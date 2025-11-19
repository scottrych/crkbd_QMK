# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview
This is a custom QMK firmware for the Corne (crkbd) 6-column split keyboard (42 keys). The firmware includes custom features like automatic mouse layer support, Unicode integration, and optional PS/2 trackpoint support.

## Essential Commands

### Building and Flashing
```fish
# Main build and flash command - runs file injection then flashes firmware
./flash.sh
```

This script:
1. Executes `qmk_file_inject.sh` to copy user files to `qmk_firmware/keyboards/crkbd/keymaps/Elil_50/`
2. Applies PS/2 patches if not already applied
3. Runs `qmk flash` from the keymap directory

### File Injection Only
```fish
# Copy user files and apply patches without flashing
./qmk_file_inject.sh
```

### Commit Changes
```fish
# Commit changes to both qmk_firmware submodule and main repo
./commit_all.sh
```

### Manual QMK Commands
```fish
# Flash from the keymap directory
cd qmk_firmware/keyboards/crkbd/keymaps/Elil_50
qmk flash

# Build without flashing
qmk compile -kb crkbd -km Elil_50
```

## Architecture

### Key Components
The custom firmware lives in `/Elil_50/` and is injected into the QMK framework at build time:

- **keymap.c** (1124 lines) - Main firmware logic including:
  - Layer definitions (alphabetic, numeric, Greek unicode, mouse, scroll)
  - Automatic layer ordering system based on enabled features
  - Extensive combo definitions (~100+ combos for two-key shortcuts)
  - Key override definitions
  - Trackpoint initialization and configuration
  - Unicode character mappings (Greek letters)
  
- **config.h** - Hardware configuration:
  - Master/slave configuration (`MASTER_LEFT`)
  - PS/2 pins for trackpoint (B5=clock, B4=data)
  - Mouse speed settings (`MK_W_OFFSET_0`, `MK_W_OFFSET_1`)
  - Auto mouse layer timeout (500ms default)
  - Unicode input modes per OS

- **rules.mk** - Build flags and feature toggles:
  - `MY_TRACKPOINT_ENABLE` - Enable/disable trackpoint support
  - `MY_UNICODE_ENABLE` - Enable/disable Unicode symbols
  - Enables: combos, key overrides, mousekeys, pointing device

### Layer System
Layers are automatically numbered based on enabled features:
- Layer 0: Alphabetic (QWERTY base)
- Layer 1: Numeric + symbols
- Layer 2: Additional/gaming layers (customizable)
- Layer 3: Greek unicode (if `MY_UNICODE_ENABLE=yes`)
- Mouse layer: Auto-activated on trackpoint movement (if `MY_TRACKPOINT_ENABLE=yes`)
- Scroll layer: Activated by double-clicking Ctrl

Layer switching keys:
- `△` (LEFT_TOGGLE) - Access layer 1 temporarily
- `▢` (RIGHT_TOGGLE) - Return to layer 0
- `△ + ▢` together - Access layer 2

### Combo and Override System
The firmware uses an innovative combo system to work around QMK layer-switching delays:
- Standard combos: Press two keys simultaneously (e.g., Shift+A = capital A)
- Layer-aware combos: Combine layer toggle with another key to access that key from the target layer without waiting
- ~100+ combo definitions starting around line 400 in keymap.c

### PS/2 Integration
Custom patches in `/PS2_patches/` integrate PS/2 trackpoint with QMK's pointing device framework:
- `ps2_pointing_device.diff` - Integrates PS/2 with pointing device API (from old PR #22532)
- `ps2_vendor.diff` - Adds pull-up resistor support for vendor PS/2 driver

Patches are auto-applied by `qmk_file_inject.sh` using `git apply` with reverse-check to avoid re-applying.

## Important Implementation Details

### Modifying the Keymap
- Layout definitions are in `keymap.c` starting with layer arrays
- To add new layers: Create layer array, link from layer 2 using `TG(n)`
- Combos and overrides are defined in two sections: declarations (around line 400-600), then registrations (around line 680+)
- Double-click timing: 175ms max between clicks

### Feature Flags
Toggle features in `Elil_50/rules.mk`:
- Set `MY_TRACKPOINT_ENABLE = no` to disable trackpoint (removes mouse layers, auto-mouse, PS/2 code)
- Set `MY_UNICODE_ENABLE = no` to disable Unicode (removes Greek layer, unicode symbols)

Changes propagate via C preprocessor directives throughout keymap.c.

### Custom Keys
- `HOME_LCTL` - Hold=Ctrl, Click=Home, Double-click=Scroll layer
- `END_SHIFT` - Hold=Shift, Click=End, Double-click=Caps Lock
- `ESC_ALT` - Hold=Alt, Click=Escape
- `ACCEL` - Toggles scroll speed (fast/slow)

### Trackpoint Configuration
Initialization sets Sprintek SK8707 registers in `pointing_device_init_user()`:
- Speed: 0xFF (max)
- Sensitivity: 0xB4
- Multipliers: 2x for X and Y axis

### Unicode OS Detection
Automatic OS detection sets appropriate Unicode input mode:
- macOS/iOS: `UNICODE_MODE_MACOS`
- Linux: `UNICODE_MODE_LINUX`
- Windows: `UNICODE_MODE_WINCOMPOSE` (requires WinCompose installed)

## File Structure
```
crkbd_QMK/
├── Elil_50/              # User keymap (source of truth)
│   ├── keymap.c          # Main firmware implementation
│   ├── config.h          # Hardware config
│   └── rules.mk          # Build flags
├── PS2_patches/          # Patches for QMK PS/2 support
├── qmk_firmware/         # QMK submodule (target for injection)
├── flash.sh              # Main build script
├── qmk_file_inject.sh    # File injection script
└── commit_all.sh         # Commit helper
```

User files in `Elil_50/` are copied to `qmk_firmware/keyboards/crkbd/keymaps/Elil_50/` at build time.

## Hardware Context
- Target: Helidox Corne V3 (6 columns, 42 keys)
- Microcontroller: Elite-Pi (RP2040) - configured via `CONVERT_TO=rp2040_ce`
- Optional: Sprintek SK8707-01-002 trackpoint (3.3V)
- Split keyboard: Master/slave via TRRS cable
- OS language input must be set to English (US) for correct key mapping

## Development Workflow
1. Edit files in `/Elil_50/` (never edit files directly in qmk_firmware/)
2. Run `./flash.sh` to inject and flash
3. Test on keyboard
4. Use `./commit_all.sh` to commit changes to both repos (if needed)

When editing keymap.c, follow the existing comment structure which clearly delineates sections:
- Automatic layer ordering
- New keys and custom keycodes
- Unicode OS detection
- Trackpoint configuration
- Unicode character definitions
- Layer definitions
- Combo declarations
- Combo registrations
- Override definitions
