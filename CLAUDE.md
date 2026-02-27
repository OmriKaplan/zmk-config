# CLAUDE.md — ZMK Keyboard Configuration for NARFBLOQ

This file provides guidance for AI assistants working in this repository.

## Repository Overview

This is a **ZMK firmware configuration repository** for the **NARFBLOQ** keyboard — a custom 10-key wireless macro pad with a rotary encoder, powered by a Seeed XIAO BLE microcontroller.

The repository does **not contain ZMK firmware source code**. It contains only the user-specific configuration (keymaps, hardware overlay, build matrix) that is fed into ZMK's build system via GitHub Actions.

---

## Directory Structure

```
zmk-config/
├── build.yaml                       # Build matrix: board/shield pairs to compile
├── zephyr/
│   └── module.yml                   # Registers repo as a Zephyr module (enables custom boards/shields)
├── .github/
│   └── workflows/
│       └── build.yml                # GitHub Actions CI/CD (delegates to ZMK's official workflow)
├── boards/
│   └── shields/
│       └── narfbloq/                # Custom shield definition for the NARFBLOQ keyboard
│           ├── Kconfig.defconfig    # Default Kconfig values (sets keyboard name)
│           ├── Kconfig.shield       # Defines SHIELD_NARFBLOQ Kconfig symbol
│           ├── narfbloq.overlay     # Device tree hardware description (pins, matrix, encoder)
│           └── narfbloq.zmk.yml    # ZMK shield metadata (id, type, requirements)
└── config/
    ├── narfbloq.conf                # Kconfig runtime options (BT power, encoder, logging)
    ├── narfbloq.keymap              # Keymap layers and key bindings (device tree syntax)
    └── west.yml                     # West manifest: pins ZMK dependency to a specific commit
```

---

## Hardware Specification

| Component        | Value                              |
|------------------|------------------------------------|
| MCU              | Seeed XIAO BLE (nRF52840)         |
| Shield           | NARFBLOQ (custom macropad)         |
| Key matrix       | 4 columns × 3 rows (10 active keys)|
| Encoder          | Alps EC11 rotary encoder           |
| Connectivity     | Bluetooth LE + USB                 |
| BT TX power      | +8 dBm (maximum)                  |

**GPIO Pinout** (XIAO D-pins):
- Rows (outputs, active high, pull-down): D2, D3, D4
- Columns (inputs, active high): D7, D8, D9, D10
- Encoder A: D5 (pull-up), Encoder B: D6 (pull-up)
- Diode direction: `col2row`

---

## Key Files Explained

### `config/narfbloq.keymap`

The main file most people edit. Uses **ZMK device tree syntax**.

- **Layers** are defined as numbered nodes under the `keymap` node.
- `DEFAULT` (layer 0): Media controls, window navigation, layer tap to LOWER.
- `LOWER` (layer 1): Bluetooth device management and system reset/bootloader.

**Common binding prefixes:**
| Binding       | Meaning                                    |
|---------------|--------------------------------------------|
| `&kp KEY`     | Simple key press                           |
| `&lt LAYER KEY` | Layer tap (hold = activate layer, tap = key) |
| `&bt BT_OP`   | Bluetooth operation                        |
| `&none`       | No-op (empty binding)                      |
| `&sys_reset`  | Reset to normal firmware                   |
| `&bootloader` | Enter DFU/bootloader mode                  |
| `&inc_dec_kp A B` | Encoder: A on CW rotation, B on CCW  |

**Modifier syntax:** `LC(X)` = Left Ctrl+X, `LS(X)` = Left Shift+X.

**Required includes at top of keymap:**
```c
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/outputs.h>
```

### `config/narfbloq.conf`

Kconfig flags that customize the build. Current settings:
```
CONFIG_ZMK_USB_LOGGING=n          # Disable USB logging (saves resources)
CONFIG_EC11=y                      # Enable Alps EC11 encoder driver
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y  # Encoder uses global thread
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y   # Max Bluetooth TX power (+8 dBm)
```

### `config/west.yml`

Pins the ZMK firmware dependency to a specific commit hash. **Do not change the revision** unless intentionally upgrading ZMK — breaking changes between ZMK versions are common.

### `build.yaml`

Defines the GitHub Actions build matrix:
```yaml
- board: xiao_ble
  shield: narfbloq
  artifact-name: narfbloq_with_xiao
```

To build additional variants, add more entries to the `include` list.

### `boards/shields/narfbloq/narfbloq.overlay`

Device tree overlay that maps the physical hardware to ZMK abstractions. Edit this only when changing hardware pin assignments or adding/removing physical components.

---

## Development Workflow

### Building Firmware

Firmware is built automatically via **GitHub Actions** on every push to `master`/`main` and on pull requests. To trigger a manual build:

1. Go to **Actions** → **Build** → **Run workflow**.
2. Download firmware artifacts from the completed workflow run.
3. Flash `.uf2` files to the keyboard by entering bootloader mode.

### Entering Bootloader Mode

- Hold the BOOT button on the XIAO BLE while connecting USB, **or**
- Trigger `&bootloader` binding from the LOWER layer (position: row 1, col 2 on LOWER).

### Editing Keymaps

1. Edit `config/narfbloq.keymap`.
2. Commit and push to `master`/`main` (or open a PR).
3. Download the built `.uf2` artifact from GitHub Actions.
4. Flash the firmware.

For keymap syntax reference, see the [ZMK documentation](https://zmk.dev/docs).

### Adding Behaviors (Combos, Macros, Hold-Tap, etc.)

Add new behaviors as nodes inside the `/` root in the keymap file, then reference them with `&behavior_name`. For example:

```dts
/ {
    combos {
        compatible = "zmk,combos";
        combo_esc {
            timeout-ms = <50>;
            key-positions = <0 1>;
            bindings = <&kp ESC>;
        };
    };
};
```

---

## Conventions & Constraints

- **File naming**: Config files must be named `<shield_id>.keymap` and `<shield_id>.conf` to be picked up by the build system automatically.
- **Layer numbering**: Layers are zero-indexed. Always define `#define LAYER_NAME N` macros for readability.
- **Encoder bindings**: Must be listed under `sensor-bindings` in the layer node, one entry per encoder.
- **No custom C code**: All configuration is declarative (Kconfig + device tree). Avoid adding C source files unless implementing a custom ZMK behavior module.
- **ZMK version pinning**: The `west.yml` commit hash should only change when deliberately upgrading ZMK. Sync `narfbloq.overlay` and `narfbloq.conf` after any ZMK upgrade that changes APIs.
- **Shield vs Board**: The XIAO BLE is the **board**; NARFBLOQ is the **shield**. Board-level changes (MCU, clocks) belong in `boards/`, keymap changes belong in `config/`.

---

## Common Tasks

### Change a key binding
Edit `config/narfbloq.keymap`, find the layer and key position, update the binding.

### Add a new layer
1. Add `#define NEWLAYER N` at the top of the keymap.
2. Add a new `newlayer_layer: layer_N { ... }` node.
3. Use `&mo NEWLAYER` or `&lt NEWLAYER KEY` to activate it.

### Change Bluetooth behavior
Use `&bt` bindings on any layer:
- `&bt BT_SEL 0` through `&bt BT_SEL 4` — select profile 0–4
- `&bt BT_PRV` / `&bt BT_NXT` — cycle profiles
- `&bt BT_CLR` — clear current profile's pairing

### Upgrade ZMK version
1. Find the desired commit hash in the [ZMK repository](https://github.com/zmkfirmware/zmk).
2. Update the `revision:` field in `config/west.yml`.
3. Check ZMK's changelog for breaking changes affecting `.overlay` or `.conf` files.
4. Test build via GitHub Actions.

---

## CI/CD

The `.github/workflows/build.yml` delegates entirely to ZMK's official reusable workflow (`zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`). It reads `build.yaml` for the build matrix. No custom build scripts exist in this repository.

Artifacts are uploaded automatically and available for download from the Actions tab for a limited retention period.
