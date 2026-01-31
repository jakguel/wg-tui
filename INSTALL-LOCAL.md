# Installing Local Development Version

This guide explains how to install your local development version of `wg-tui` globally on your system.

## Prerequisites

- Rust and Cargo installed
- Git repository cloned to `/home/jiyan/Projects/wg-tui` (or your local path)

## Installation Steps

### 1. Uninstall existing version (if any)

If you previously installed `wg-tui` from crates.io:

```bash
cargo uninstall wg-tui
```

### 2. Install from local source

Navigate to your local repository and install:

```bash
cd /home/jiyan/Projects/wg-tui
cargo install --path .
```

This will:
- Build the release binary from your local source code
- Install it to `~/.cargo/bin/wg-tui`
- Make your changes available globally as the `wg-tui` command

### 3. Verify installation

```bash
which wg-tui
# Should output: /home/jiyan/.cargo/bin/wg-tui

wg-tui --version
# Should show: wg-tui 0.2.0
```

## Usage

Run from anywhere on your system:

```bash
sudo wg-tui
```

## Updating After Code Changes

Whenever you make changes to the source code and want to update the global installation:

```bash
cd /home/jiyan/Projects/wg-tui
cargo install --path . --force
```

The `--force` flag overwrites the existing installation without prompting.

## Uninstalling

To remove the globally installed version:

```bash
cargo uninstall wg-tui
```

## Notes

- The binary is installed to `~/.cargo/bin/`, which should be in your `$PATH`
- You need `sudo` to run `wg-tui` because it requires root privileges to manage WireGuard tunnels
- Changes to the source code require reinstalling with `cargo install --path . --force`
