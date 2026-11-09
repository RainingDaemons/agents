# Async and Tokio

## Description

Why async Rust services stall, and how to keep the executor responsive. An async task is **cooperative**: it runs until it `.await`s, so any blocking call or long CPU burst stops every other task scheduled on that worker thread. Most async performance problems are not slow functions; they are work that should not run on the executor at all, or too many tasks competing for it.

## When to use

- A service has high tail latency, low CPU utilization, or throughput that collapses under load.
- Requests intermittently take seconds while the CPU looks idle.
- A task holds a lock across `.await`, or calls `std::fs`, `std::net`, `std::thread::sleep`, or a CPU-heavy loop.
- The service spawns one task per item or uses unbounded channels.
- A client, pool, or regex is constructed per request.
- `tokio-console` shows long poll times or many idle workers.

## Checks

| Symptom | Likely cause | First check |
|---|---|---|
| Workers idle while requests wait | Blocking call inside async code | Search for `std::fs`, `std::net`, `thread::sleep`, heavy CPU |
| High tail latency, low throughput | Long task between `.await`s | `tokio-console`, `tracing` spans, poll times |
| Deadlock or `!Send` future | `std::sync::MutexGuard` held across `.await` | Scope the guard; drop before awaiting |
| Memory grows under load | Unbounded channels or task spawning | Bounded channels, `Semaphore`, `buffer_unordered` |
| Connection setup dominates | New `Client`/pool per request | Shared, cloned client |
| Futures are huge / stack overflow | Large or recursive async state machines | `Box::pin`, reduce captured state |
| Executor starved by one task | Long CPU loop | `spawn_blocking`, `yield_now` |

Probes:

```bash
rg -n 'std::thread::sleep|std::fs::|std::net::|TcpStream|File::' src
rg -n '\.lock\(\)|MutexGuard|RwLock' src
rg -n 'tokio::spawn|spawn\(' src
rg -n 'unbounded\(|channel\(' src
rg -n 'Client::new|reqwest::Client::new|Regex::new' src
rg -n '\.await' src | rg -i 'lock|guard'
```

## Code examples

### Blocking I/O in an async handler

```rust
// SLOW: blocks the executor thread while the OS reads the file
async fn load(path: &str) -> std::io::Result<String> {
    std::fs::read_to_string(path)
}
```

```rust
// FAST: async I/O yields to the executor
async fn load(path: &str) -> std::io::Result<String> {
    tokio::fs::read_to_string(path).await
}
```

For CPU-heavy or genuinely blocking libraries, move the work off the executor:

```rust
// FAST: CPU work runs on the blocking pool
async fn parse(data: Vec<u8>) -> Report {
    tokio::task::spawn_blocking(move || parse_cpu(&data))
        .await
        .expect("blocking task panicked")
}
```

`spawn_blocking` is for work that blocks for a while; it is not a fix for a long-running loop that could be chunked. Do not spawn a blocking task per tiny item.

### Holding a lock across `.await`

```rust
// SLOW: guard lives across the await; risks deadlock and makes the future !Send
async fn update(state: &std::sync::Mutex<State>) {
    let mut guard = state.lock().unwrap();
    some_io().await;
    guard.count += 1;
}
```

```rust
// FAST: do the I/O first, then take the lock only for the mutation
async fn update(state: &std::sync::Mutex<State>) {
    let value = some_io().await;
    {
        let mut guard = state.lock().unwrap();
        guard.apply(value);
    }
}
```

If the lock must be held across an `.await` by design, use `tokio::sync::Mutex` and keep the critical section short. Prefer restructuring over holding any lock across I/O.

### Long CPU loop on an async worker

```rust
// SLOW: monopolizes one worker until the loop finishes
async fn work(items: Vec<Item>) {
    for item in items {
        heavy_cpu(item);
    }
}
```

```rust
// FAST: move the burst to the blocking pool
async fn work(items: Vec<Item>) {
    tokio::task::spawn_blocking(move || {
        for item in items {
            heavy_cpu(item);
        }
    })
    .await
    .unwrap();
}
```

If the work must stay on the executor, insert `tokio::task::yield_now().await` between chunks. Yield more often than you think: a task that runs for tens of milliseconds without yielding is already a latency problem.

### One task per item

```rust
// SLOW: unbounded concurrency, memory and scheduling overhead
for item in items {
    tokio::spawn(handle(item));
}
```

```rust
// FAST: bounded concurrency
use futures::stream::{self, StreamExt};

stream::iter(items)
    .map(|item| async move { handle(item).await })
    .buffer_unordered(32)
    .for_each(|_| async {})
    .await;
```

`buffer_unordered(N)` keeps at most `N` futures in flight. A `tokio::sync::Semaphore` gives the same control when the tasks must be spawned.

### Unbounded channels

```rust
// SLOW: hides overload and grows memory without bound
let (tx, rx) = tokio::sync::mpsc::unbounded_channel();
```

```rust
// FAST: backpressure propagates to the producer
let (tx, rx) = tokio::sync::mpsc::channel(1_000);
```

Bounded channels make the producer wait when the consumer falls behind. Choose the bound from the acceptable latency and memory, not arbitrarily.

### Client per request

```rust
// SLOW: new connection pool, TLS setup, and DNS per call
async fn fetch(url: &str) -> reqwest::Result<String> {
    reqwest::Client::new().get(url).send().await?.text().await
}
```

```rust
// FAST: build once, clone the cheap handle
static CLIENT: std::sync::LazyLock<reqwest::Client> =
    std::sync::LazyLock::new(reqwest::Client::new);

async fn fetch(url: &str) -> reqwest::Result<String> {
    CLIENT.get(url).send().await?.text().await
}
```

The same applies to database pools (`sqlx::Pool`, `deadpool`, `bb8`) and compiled regexes.

### Large futures

Deeply nested `async` functions and large captured state produce large futures that are moved on every poll. Box the future (`Box::pin`) when it is large or recursive, and reduce what the future captures.

## Anti-patterns

1. `std::thread::sleep` instead of `tokio::time::sleep`.
2. Synchronous file, socket, or database calls in async code.
3. Holding a `std::sync` guard across `.await`.
4. One task per element with no concurrency limit.
5. Unbounded channels used as a queue.
6. Creating a client, pool, or regex per request.
7. Long CPU bursts without `spawn_blocking` or `yield_now`.
8. `block_on` inside a runtime, or nesting runtimes.
9. Oversubscribing workers, or mixing blocking work into the compute pool.
10. Profiling with `println!` on the hot path (see `logging-debugging.md`).

## Checklist

- Is every blocking call either async or on `spawn_blocking`?
- Is any lock held across an `.await`?
- Is concurrency bounded (`buffer_unordered`, `Semaphore`, bounded channel)?
- Are clients, pools, and regexes created once and reused?
- Is the runtime flavor (`current_thread` vs `multi_thread`) and worker count deliberate?
- Do long CPU loops yield or move off the executor?
- Are large futures boxed?
- Does `tokio-console` show sane poll times under load?

## Related

- `parallelism.md` for CPU-bound work and thread pools.
- `profiling.md` for finding the hot task or call.
- `logging-debugging.md` for cheap diagnostics and `tracing`.
- `buildconfig.md` for release builds: async code in a dev build is misleadingly slow.
