# Tauri v2+ Capabilities & Permissions Reference

## Contents

- Security Model: v1 vs v2
- Overview
- Capability File Structure
- Core Permissions
- Plugin Permissions
- Scopes
- Permission Sets
- Window and Webview Targeting
- Plugin Permission Authoring
- Capability Best Practices
- Common Capability Patterns
- Anti-Patterns

## Security Model: v1 vs v2

Tauri v2 replaces the v1 `allowlist` with a capabilities-first security model. In v1, you listed allowed API calls in `tauri.conf.json`'s `allowlist`. In v2, permissions must be explicitly granted via capability files in `src-tauri/capabilities/`.

**Three-layer security model:**
- **Capability**: A named collection of permissions, scoped to specific windows/webviews. Lives in `src-tauri/capabilities/*.json`.
- **Permission**: An identifier that grants access to a specific command or feature (e.g., `fs:allow-read-file`). Defined per-plugin.
- **Scope**: Optional constraint on a permission that limits what it can access (e.g., only `$APPDATA/*` paths). Part of a permission object.

## Overview

Tauri v2+ uses a capabilities-based security model. By default, **nothing is allowed** - you must explicitly grant permissions through capability files.

*Last verified: 2026-04-02. Check the official Tauri changelog when capability semantics or permission names change.*

## Capability File Structure

Location: `src-tauri/capabilities/`

```json
{
    "$schema": "../gen/schemas/desktop-schema.json",
    "identifier": "capability-name",
    "description": "What this capability allows",
    "windows": ["main", "settings"],
    "webviews": [],
    "permissions": [
        "core:default",
        "plugin-name:permission-name"
    ]
}
```

## Core Permissions

### Essential (Almost Always Needed)

```json
{
    "permissions": [
        "core:default",
        "core:window:default",
        "core:event:default"
    ]
}
```

### Window Permissions

| Permission | Description |
|------------|-------------|
| `core:window:default` | Basic window operations |
| `core:window:allow-close` | Allow closing windows |
| `core:window:allow-set-title` | Allow changing window title |
| `core:window:allow-minimize` | Allow minimizing |
| `core:window:allow-maximize` | Allow maximizing |
| `core:window:allow-set-size` | Allow resizing |
| `core:window:allow-set-position` | Allow repositioning |
| `core:window:allow-set-fullscreen` | Allow fullscreen toggle |

### Event Permissions

| Permission | Description |
|------------|-------------|
| `core:event:default` | Basic event listening |
| `core:event:allow-emit` | Allow emitting events |
| `core:event:allow-listen` | Allow listening to events |

## Plugin Permissions

### File System (`tauri-plugin-fs`)

```json
{
    "permissions": [
        "fs:default",
        "fs:allow-read-dir",
        "fs:allow-read-file",
        "fs:allow-write-file",
        "fs:allow-create-dir",
        "fs:allow-remove-file",
        "fs:allow-rename"
    ]
}
```

**With Scopes:**
```json
{
    "permissions": [
        {
            "identifier": "fs:allow-read-file",
            "allow": [
                { "path": "$APPDATA/*" },
                { "path": "$HOME/Documents/*" }
            ]
        }
    ]
}
```

### Dialog (`tauri-plugin-dialog`)

```json
{
    "permissions": [
        "dialog:default",
        "dialog:allow-open",
        "dialog:allow-save",
        "dialog:allow-message",
        "dialog:allow-ask",
        "dialog:allow-confirm"
    ]
}
```

### Shell (`tauri-plugin-shell`)

```json
{
    "permissions": [
        "shell:default",
        "shell:allow-open",
        "shell:allow-execute"
    ]
}
```

**Scoped Execute:**
```json
{
    "permissions": [
        {
            "identifier": "shell:allow-execute",
            "allow": [
                { "name": "git", "args": true },
                { "name": "npm", "args": ["install", "run"] }
            ]
        }
    ]
}
```

### HTTP (`tauri-plugin-http`)

```json
{
    "permissions": [
        "http:default"
    ]
}
```

**With URL Scopes:**
```json
{
    "permissions": [
        {
            "identifier": "http:default",
            "allow": [
                { "url": "https://api.example.com/*" },
                { "url": "https://*.myapp.com/*" }
            ]
        }
    ]
}
```

### Store (`tauri-plugin-store`)

```json
{
    "permissions": [
        "store:default",
        "store:allow-get",
        "store:allow-set",
        "store:allow-delete",
        "store:allow-keys",
        "store:allow-clear"
    ]
}
```

### Clipboard (`tauri-plugin-clipboard-manager`)

```json
{
    "permissions": [
        "clipboard-manager:default",
        "clipboard-manager:allow-read",
        "clipboard-manager:allow-write"
    ]
}
```

### Notification (`tauri-plugin-notification`)

```json
{
    "permissions": [
        "notification:default",
        "notification:allow-send",
        "notification:allow-request-permission"
    ]
}
```

### Global Shortcut (`tauri-plugin-global-shortcut`)

```json
{
    "permissions": [
        "global-shortcut:default",
        "global-shortcut:allow-register",
        "global-shortcut:allow-unregister"
    ]
}
```

## Permission Sets

Permission sets allow grouping multiple permissions into a single reusable identifier. You can use preset permission sets provided by plugins (like `fs:default`) or define your own in `src-tauri/permissions/`.

```json
{
  "permissions": [
    "fs:default",          // Permission set: includes common fs operations
    "fs:allow-read-file",  // Individual permission: specific operation
    {
      "identifier": "fs:allow-read-file",  // Permission with scope
      "allow": [{ "path": "$APPDATA/*" }]
    }
  ]
}
```

## Platform-Specific Capabilities

```json
{
    "identifier": "desktop-only",
    "platforms": ["linux", "macos", "windows"],
    "permissions": ["global-shortcut:default"]
}
```

```json
{
    "identifier": "mobile-only",
    "platforms": ["iOS", "android"],
    "permissions": ["biometric:default", "haptics:default"]
}
```

## Windows and Webviews Targeting

Capabilities are applied to specific windows and webviews by their labels. A window or webview can be part of multiple capabilities, in which case their permissions are merged.

```json
{
  "identifier": "main-window-cap",
  "windows": ["main"],        // Target by window label
  "webviews": [],             // Or target specific webviews
  "permissions": ["core:default", "fs:default"]
}
```

## Remote URL Access

Allow Tauri commands from remote URLs:

```json
{
    "identifier": "remote-access",
    "remote": {
        "urls": ["https://*.myapp.com"]
    },
    "permissions": ["http:default"]
}
```

## Custom Permission Files

Create custom permissions in `src-tauri/permissions/`:

**`custom.toml`:**
```toml
[[permission]]
identifier = "allow-home-documents"
description = "Allow access to home documents"
commands.allow = ["read_file", "write_file"]

[[scope.allow]]
path = "$HOME/Documents/**"
```

Reference in capability:
```json
{
    "permissions": ["custom:allow-home-documents"]
}
```

## Plugin Permission Authoring

Everything above is the **consumer** side: what an app grants. The producer side lives inside the plugin crate and is what makes `<plugin-name>:<permission>` resolvable at all. A plugin has no `capabilities/` folder; it ships permissions, and each consuming app decides which ones to grant.

### Permission Files

Each `.toml` or `.json` file under `permissions/` declares permissions, permission sets, or the default set. Identifiers follow the same rules as capability identifiers: lowercase ASCII, digits and hyphens, with a single `:` only when prefixed.

```toml title="permissions/start-server.toml"
[[permission]]
identifier = "allow-start-server"
description = "Enables the start_server command."
commands.allow = ["start_server"]

[[permission]]
identifier = "deny-start-server"
description = "Denies the start_server command."
commands.deny = ["start_server"]
```

### Autogenerated Command Permissions

List every command in `build.rs` and `tauri-plugin` generates `allow-<command>` / `deny-<command>` under `permissions/autogenerated/commands/`, plus a `permissions/autogenerated/reference.md` table for docs:

```rust title="build.rs"
const COMMANDS: &[&str] = &["list_devices", "play_device", "stop_device"];

fn main() {
    tauri_plugin::Builder::new(COMMANDS)
        .android_path("android") // optional, omit for desktop-only plugins
        .ios_path("ios")         // optional
        .build();
}
```

```toml title="Cargo.toml"
[build-dependencies]
tauri-plugin = { version = "2", features = ["build"] }
```

Names must match the `#[tauri::command]` functions in snake_case. A command registered in `generate_handler!` but missing from `COMMANDS` still compiles, and then the frontend gets a permission-denied error at runtime.

### Default Permission Set

`permissions/default.toml` is what `"<plugin-name>:default"` expands to, so it is the permission most consumers will add:

```toml title="permissions/default.toml"
[default]
description = "Default permissions for the plugin"
permissions = ["allow-list-devices", "allow-play-device"]
```

Enable only the commands without which the plugin is useless. Destructive or privileged commands should stay opt-in.

### Permission Sets

Group commands that form one API surface so consumers grant them in one line:

```toml title="permissions/playback.toml"
[[set]]
identifier = "allow-playback"
description = "Allows starting and stopping playback"
permissions = ["allow-play-device", "allow-stop-device"]
```

### Scopes

Scopes are Rust structs deriving `schemars::JsonSchema`, so consumers get autocomplete and validation inside their `allow` / `deny` blocks:

```rust title="src/scope.rs"
#[derive(Debug, schemars::JsonSchema)]
pub struct Entry {
    pub binary: String,
}
```

```rust title="build.rs"
#[path = "src/scope.rs"]
mod scope;

fn main() {
    tauri_plugin::Builder::new(COMMANDS)
        .global_scope_schema(schemars::schema_for!(scope::Entry))
        .build();
}
```

`schemars` belongs in both `[dependencies]` and `[build-dependencies]` because `scope.rs` is shared by the crate and the build script. Commands read scopes through `tauri::ipc::CommandScope<'_, Entry>` or `tauri::ipc::GlobalScope<'_, Entry>`, both exposing `allows()` and `denies()`.

### Plugin Permission Checklist

- [ ] Every command appears in `COMMANDS` in `build.rs`
- [ ] `permissions/default.toml` lists the commands the plugin cannot work without
- [ ] Generated permission files are committed under `permissions/autogenerated/`
- [ ] Scope structs derive `JsonSchema` and the schema is registered in `build.rs`
- [ ] Renaming or removing a command is treated as a breaking change, because `allow-<old-name>` disappears

## Capability Best Practices

1. **Principle of Least Privilege**: Only grant what's needed
2. **Use Scopes**: Limit file/URL access to specific paths
3. **Separate Capabilities**: Create focused capability files for different features
4. **Platform-Specific**: Use platform filtering for platform-specific features
5. **Document**: Add descriptions to explain why permissions are needed

**See also:** [Plugin Development](plugins-development.md) for the crate layout and lifecycle hooks | [Mobile Plugin Development](mobile-plugins-development.md) for native permission requests | [Publishing](publishing.md) for shipping the crate and its npm package

## Anti-Pattern: Missing Capability

Plugin installed but **NOT** in capabilities = silent permission denied at runtime. Always add plugin permissions to a capability file that targets the window using the plugin.

## Common Capability Patterns

### Minimal App

```json
{
    "identifier": "minimal",
    "windows": ["main"],
    "permissions": ["core:default"]
}
```

### File Manager

```json
{
    "identifier": "file-manager",
    "windows": ["main"],
    "permissions": [
        "core:default",
        "fs:default",
        "dialog:allow-open",
        "dialog:allow-save"
    ]
}
```

### Web-Connected App

```json
{
    "identifier": "web-app",
    "windows": ["main"],
    "permissions": [
        "core:default",
        "http:default",
        "shell:allow-open"
    ]
}
```

### Full Desktop App

```json
{
    "identifier": "full-desktop",
    "windows": ["main"],
    "permissions": [
        "core:default",
        "core:window:default",
        "core:event:default",
        "fs:default",
        "dialog:default",
        "shell:default",
        "clipboard-manager:default",
        "notification:default",
        "global-shortcut:default",
        "store:default"
    ]
}
```
