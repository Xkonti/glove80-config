# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **ZMK firmware configuration repository** for the **MoErgo Glove80** split ergonomic keyboard. The project uses Nix flakes for reproducible builds and includes automated workflows for firmware compilation, flashing, and keymap visualization.

## Common Commands

### Building and Flashing
```bash
# Build firmware (outputs to ./result/glove80.uf2)
nix build

# Flash firmware to connected keyboard (interactive, auto-detects mounted keyboard)
nix run

# Generate keymap SVG diagrams
nix run .#draw

# Validate flake and run checks
nix flake check

# Update dependencies
nix flake update

# Format Nix code
nix fmt
```

### Flashing Process
The firmware file contains both keyboard halves:
1. Build firmware: `nix build`
2. Connect **right half** in bootloader mode (hold C6R6 + C3R2 while powering on)
3. Flash: `nix run` (or manually copy `result/glove80.uf2` to mounted volume)
4. Connect **left half** in bootloader mode
5. Flash: `nix run`

## Architecture

### Nix Flake Structure

The project uses **flake-parts** for modular organization:

- **`flake.nix`**: Main entry point, imports sub-modules
- **`flake-pkgs.nix`**: Custom overlay that exposes flake attributes (`flake.inputs`, `flake.packages`, etc.) to packages via `callPackage`
- **`packages/flake-module.nix`**: Auto-discovers all `.nix` files in `packages/` and registers them as outputs

This architecture enables packages to access flake internals. For example, `firmware.nix` can reference `flake.inputs.glove80-zmk` because of the overlay.

**Adding new packages**: Simply drop a `.nix` file in `packages/` - it will be auto-discovered and callable as `nix run .#<name>`.

### ZMK Firmware Build Process

**Dual-Half Architecture** (`packages/firmware.nix`):
1. Builds separate firmware images for left (`glove80_lh`) and right (`glove80_rh`) halves
2. Uses `firmware.combine_uf2` to merge both into a single `.uf2` file
3. Single file is flashed to both keyboard halves sequentially

**Configuration Files**:
- **`config/glove80.keymap`**: Main keymap in DeviceTree syntax (338 lines)
  - Defines layers (Colemak-DH base, Qwerty, gaming, symbols, navigation, magic)
  - Uses advanced ZMK features: behaviors, tap-dance, mod-morphs
  - Mod-morphs: keys that change when shift is held (e.g., `?` → `!`)
  - Tap-dance: different actions on single vs. double tap
- **`config/glove80.conf`**: ZMK build configuration (currently empty)
- **`config/keymap.json`**: JSON representation for web-based Keymap Editor

### Smart Flashing (`packages/flash.nix`)

The flash package is a shell application with:
- **Platform-agnostic detection**: Works on Linux, macOS, Windows (WSL/Cygwin)
- **Automatic keyboard detection**: Finds mounted Glove80 in bootloader mode
- **Validation**: Ensures exactly one keyboard is connected before flashing
- **Terminal UI**: Uses `gum` for interactive prompts

### Automated Keymap Visualization (`packages/draw.nix`)

Uses **keymap-drawer** (Python tool) to:
1. Parse `.keymap` files (DeviceTree) into YAML representation
2. Generate SVG diagrams (both full keyboard and per-layer views)
3. Output to `img/` directory

**Workflow**: Changes to `config/glove80.keymap` trigger GitHub Actions that automatically regenerate SVGs and commit them to the repository.

## CI/CD Workflows

All workflows defined in `.github/workflows/`:

- **`build.yml`**: Triggered on config changes, builds firmware and uploads as artifact
- **`draw-keymaps.yml`**: Triggered on config changes, generates keymap SVGs and auto-commits
- **`check.yaml`**: Runs `nix flake check` on all pushes
- **`update.yml`**: Weekly (Saturdays) automated `flake.lock` updates via pull request

**Binary Cache**: Uses Cachix (`matt-sturgeon`) to speed up CI builds.

## Integration Points

### Keymap Editor
This repository integrates with **nickcoutsos/keymap-editor** (web-based visual editor):
- Editor reads/writes `config/keymap.json`
- Changes sync to `config/glove80.keymap` via the editor's backend
- Preferred over MoErgo's official Layout Editor

### ZMK Fork
Uses **moergo-sc/zmk** (MoErgo's ZMK fork) rather than upstream ZMK:
- Includes Glove80-specific features and board definitions
- Referenced in `flake.nix` as `glove80-zmk` input

## Key Concepts

### DeviceTree Syntax
`.keymap` files use DeviceTree syntax, not C. Key patterns:
```dts
&kp ESC          // Key press behavior
&lt LAYER TAB    // Layer-tap: hold for layer, tap for key
&mt LSHIFT A     // Mod-tap: hold for modifier, tap for key
```

### Package Auto-Discovery
The `packages/` directory uses Nix auto-discovery:
1. `packages/flake-module.nix` scans directory for `.nix` files
2. Each file is called with `callPackage`, receiving overlay attributes
3. Packages can access `flake.inputs`, `flake.packages`, etc.
4. No manual registration needed - just add file

### Local Development = CI
Nix ensures local builds are **identical** to CI builds:
- Same toolchain versions (pinned in `flake.lock`)
- Same build process
- Same dependencies
- If it builds locally, it will build in CI

## Development Workflow

### Making Keymap Changes
1. Edit `config/glove80.keymap` (or use Keymap Editor for `keymap.json`)
2. Test build: `nix build`
3. Flash to keyboard: `nix run`
4. Commit changes
5. CI automatically generates updated SVG diagrams

### Adding New Layers or Behaviors
Study the existing keymap structure:
- Behaviors defined in `behaviors {}` node
- Layers defined in `keymap {}` node
- Reference existing patterns for tap-dance, mod-morph usage

### Debugging Build Issues
- Check `nix flake check` for validation errors
- ZMK build errors typically indicate DeviceTree syntax issues in `.keymap`
- Compare against ZMK documentation for behavior syntax
