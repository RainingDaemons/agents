# Logging and Debugging

## Description

How logging, tracing, and debugging code can dominate a program's runtime, and how to keep diagnostics cheap when they are disabled. Two costs hide here: the logging call itself (formatting, locking, I/O), and the **data collection** that feeds it (building strings, walking structures, computing values). If the second runs unconditionally, the logging level makes no difference.

## When to use

- A profile shows time in `fmt`, `format!`, `to_string`, `write`, or `stdout`.
- Throughput drops when verbose logging is enabled, but the logs are not the bottleneck.
- Debug assertions or validation run in release builds.
- `println!` appears per line or per item.
- A service logs on the hot path or per request.
- Error construction or diagnostics are expensive on the success path.

## Checks

```bash
perf record -g ./target/release/app
rg -n 'println!|print!|eprintln!|eprint!' src
rg -n 'format!\(|to_string\(|to_owned\(' src | head -50
rg -n 'assert!\(|debug_assert!' src | head -50
rg -n 'log::|tracing::|log_enabled!|enabled!' src
rg -n 'stdout\(\)|stderr\(\)|BufWriter|BufReader' src
```

| Symptom | Likely cause | Fix |
|---|---|---|
| Time in `fmt`/`write` | Logging on the hot path | Guard, defer, or remove |
| Allocations in logging | Building strings before the level check | Compute lazily inside the macro |
| Release build runs validation | `assert!` instead of `debug_assert!` | `debug_assert!` for non-safety checks |
| Slow per-line output | `println!` locks and flushes per call | Lock stdout once + `BufWriter` |
| Debug code collects data | Metrics/backtraces computed unconditionally | Gate collection on the level |

## Code examples

### Do not build the data before checking the level

```rust
// SLOW: the expensive `Debug` formatting runs even when debug logging is disabled
let snapshot = format!("{:?}", expensive_state());
log::debug!("state: {snapshot}");
```

```rust
// FAST: arguments are only formatted when the level is enabled
log::debug!("state: {:?}", expensive_state());
```

`log` and `tracing` macros check the level before formatting their arguments, so passing a value is cheap; pre-formatting it is not. The same applies to computing the value itself:

```rust
// SLOW: `expensive_state()` always runs
let state = expensive_state();
if enabled!(Level::DEBUG) {
    tracing::debug!(?state);
}
```

```rust
// FAST: compute only when the level is enabled
if tracing::enabled!(tracing::Level::DEBUG) {
    let state = expensive_state();
    tracing::debug!(?state);
}
```

### Guard optional data collection

```rust
// SLOW: metrics are collected even when nobody exports them
metrics.record_latency(start.elapsed());
metrics.record_cache_stats(cache.stats());
```

```rust
// FAST: gate collection on a cheap flag
if metrics.enabled() {
    metrics.record_latency(start.elapsed());
    metrics.record_cache_stats(cache.stats());
}
```

This is common in servers: the diagnostics path walks maps and computes percentiles on every request even when the exporter is off.

### `assert!` vs `debug_assert!`

```rust
// SLOW in release: the check runs forever
assert!(is_sorted(&data));
```

```rust
// FAST: only in dev builds
debug_assert!(is_sorted(&data));
```

Use `assert!` only for invariants required for safety or correctness. Use `debug_assert!` for expensive sanity checks that are not part of the contract.

### Lock stdout once for repeated output

```rust
// SLOW: `println!` locks stdout on every call
for line in lines {
    println!("{line}");
}
```

```rust
// FAST: lock once, buffer writes
use std::io::Write;

let stdout = std::io::stdout();
let mut out = std::io::BufWriter::new(stdout.lock());
for line in lines {
    writeln!(out, "{line}")?;
}
out.flush()?;
```

For machine-readable output, build into a reused `String` and write once.

### Defer formatting

```rust
// SLOW: allocates a String even when the value is never printed
let msg = format!("user {id} failed: {err}");

// FAST: format_args! borrows and defers
let msg = format_args!("user {id} failed: {err}");
```

`format_args!` cannot outlive the statement, so use it where the value is consumed immediately. When the message may not be used, a closure or the logging macro is better.

### Structured logging instead of formatted strings

```rust
// SLOW: allocates a message per event
tracing::info!("processed {} items in {:?}", count, elapsed);

// FAST: fields are recorded lazily by the subscriber
tracing::info!(count, ?elapsed, "processed items");
```

Structured fields let the subscriber decide what to record and avoid formatting work that is filtered out.

## Anti-patterns

1. Building a `String`/`format!` before a logging level check.
2. Collecting metrics or diagnostics unconditionally.
3. `println!`/`eprintln!` per line or per item.
4. Hot `assert!` for non-safety invariants instead of `debug_assert!`.
5. Logging full request bodies, tokens, or secrets (also a security problem; see the `security-review` skill).
6. `dbg!` left in hot paths.
7. Unbounded log buffers or channels that grow under load.
8. Logging inside a lock or on the async executor's hot path.
9. Formatting error messages eagerly on the success path (`ok_or(format!(...))`).
10. Enabling verbose logging in production without sampling or filtering.

## Checklist

- Is logging on a measured hot path?
- Is all data collection gated by a cheap level check?
- Are log arguments passed lazily to the macro, not pre-formatted?
- Are non-safety assertions `debug_assert!`?
- Is repeated output locked and buffered once?
- Is error construction lazy (`ok_or_else`, `map_err` with a closure)?
- Are logs bounded, sampled, and free of secrets/PII?
- Is the tracing/log filter configured for production?
- Did you re-measure with logging disabled and enabled?

## Related

- `profiling.md` for finding logging and formatting hot spots.
- `std.md` for lazy `Option`/`Result` combinators.
- `async.md` for logging that blocks the executor.
- `heap.md` for the allocation cost of `format!` and `to_string`.
