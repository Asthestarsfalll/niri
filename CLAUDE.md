# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is niri

Niri is a scrollable-tiling Wayland compositor written in Rust. Windows are arranged in columns on an infinite horizontal strip; workspaces are dynamic and vertical. Built on the Smithay library.

## Build & Development Commands

```bash
# Build
cargo build

# Run tests (excludes visual-tests which need a display)
cargo test --all --exclude niri-visual-tests

# Run a single test module
cargo test --all --exclude niri-visual-tests -- test_name

# Run with release profile (used for randomized/slow tests)
cargo test --all --exclude niri-visual-tests --release

# Clippy (CI uses --all --all-targets)
cargo clippy --all --all-targets

# Format (requires nightly rustfmt)
cargo +nightly fmt --all

# Check format without modifying
cargo +nightly fmt --all -- --check

# Build visual tests (needs libadwaita)
cargo build --package niri-visual-tests

# Check with specific feature combinations
cargo check --no-default-features
cargo check --no-default-features --features dbus
cargo check --no-default-features --features systemd
cargo check --no-default-features --features dinit
cargo check --no-default-features --features xdp-gnome-screencast
```

## Workspace Crates

- **niri** (root): The compositor itself
- **niri-config**: Config parsing (KDL format via knuffel). Types are split into `Foo` (final) and `FooPart` (parsed per-file); defaults are set before parsing then merged.
- **niri-ipc**: IPC types and socket protocol. JSON-based request/reply over Unix socket. Follows niri version, not semver-stable.
- **niri-visual-tests**: Visual test harness (build-only in CI, requires libadwaita)

## Architecture

### Core Flow
- `src/main.rs` — CLI entry point, event loop setup, IPC client handling
- `src/niri.rs` — `State` struct: the central compositor state, owns the event loop, handles D-Bus, outputs, and the main loop
- `src/backend/` — Backend implementations: `tty.rs` (DRM/KMS), `winit.rs` (windowed), `headless.rs` (testing)

### Layout Engine (`src/layout/`)
Pure layout logic, decoupled from Wayland. Key types:
- `Layout` — top-level, manages monitors
- `Monitor` — per-output, contains workspaces
- `Workspace` — contains tiles (columns), one empty workspace always exists
- `Tile` — wraps a single window with decorations, focus ring, shadow
- `Scrolling` — the scrolling tiling layout for columns
- `floating.rs` — floating window layout

Layout tests (`src/layout/tests.rs`, `src/layout/tests/`) use proptest for randomized property testing.

### Wayland Handling
- `src/handlers/` — Smithay protocol handlers: `compositor.rs`, `xdg_shell.rs`, `layer_shell.rs`, `background_effect.rs`
- `src/protocols/` — Custom/extended Wayland protocols
- `src/layer/` — Layer shell surface management

### Input
- `src/input/` — Input processing: keyboard bindings, pointer grabs (move, resize, pick color/window), touchpad gestures, scroll tracking, spatial navigation
- `src/input/mod.rs` — Main input dispatch

### Rendering
- `src/render_helpers/` — GPU rendering utilities (shaders, blur, custom render elements)
- `src/ui/` — On-screen UI: screenshot UI, overview, notifications

### Other
- `src/animation/` — Animation framework
- `src/dbus/` — D-Bus interfaces (freedesktop, GNOME, accessibility, power)
- `src/screencasting/` — PipeWire screencasting via xdg-desktop-portal-gnome
- `src/ipc/` — Server-side IPC handling
- `src/window/` — Window abstraction layer
- `src/a11y.rs` — Accessibility tree (accesskit)
- `src/frame_clock.rs` — Frame timing

### Tests
- `src/tests/` — Client-server integration tests using a `Fixture` that runs niri in-process with a Wayland client
- `src/layout/tests/` — Randomized property tests for the layout engine
- Snapshot tests use the `insta` crate

## Feature Flags

Default features: `dbus`, `systemd`, `xdp-gnome-screencast`

- `dbus` — D-Bus support, accessibility, power button handling
- `systemd` — systemd integration (requires `dbus`)
- `xdp-gnome-screencast` — Screencasting via PipeWire (requires `dbus`)
- `dinit` — dinit init system integration
- `profile-with-tracy` — Tracy profiler instrumentation
- `profile-with-tracy-ondemand` — On-demand Tracy profiling
- `profile-with-tracy-allocations` — Tracy allocation profiling

## Code Conventions

- Format with `cargo +nightly fmt` using settings in `rustfmt.toml`: imports grouped as std/external/crate, comments wrapped at 100 chars
- Clippy allows: `new_without_default`, `collapsible_match`
- MSRV: Rust 1.85
- Smithay is pinned to a specific git revision (not crates.io)
- Each commit should build and pass tests independently
- Rebase onto main rather than merging for PR updates
