# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A [ZMK](https://zmk.dev) firmware config repo (a "zmk-config") for a custom split-less numpad/macropad called **dumbpad**, built on a `nice_nano_v2` controller via a `pro_micro` pinout adapter. This repo does not contain the ZMK application source itself — it's a west-managed overlay that ZMK's build system pulls in as `config/`. There is no local compiler/toolchain setup; firmware is built by GitHub Actions using ZMK's reusable workflow.

## Build

Firmware is built entirely in CI, not locally:

- [.github/workflows/build.yml](.github/workflows/build.yml) delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main` on every push/PR/manual dispatch.
- [build.yaml](build.yaml) defines the build matrix: `nice_nano_v2` + `dumbpad` shield, and `nice_nano_v2` + `settings_reset` shield. Add entries here (or an `include:` block for less common board/shield pairings) to add new build targets.
- Compiled `.uf2` firmware is produced as a GitHub Actions artifact — flash it onto the nice_nano_v2 by copying it to the board's UF2 bootloader drive (double-tap reset).

There is no local `west build` setup configured in this repo (no `.west/`, no toolchain). If local builds are ever needed, they'd follow standard ZMK west workflow against the upstream `zmk` project pulled via [config/west.yml](config/west.yml).

## Repo layout

- `config/west.yml` — west manifest; pulls in `zmkfirmware/zmk` (`main` branch) as the upstream ZMK app, with this repo's `config/` as the `self` overlay path.
- `config/boards/shields/dumbpad/` — the actual, active shield definition:
  - `dumbpad.dtsi` — base devicetree: kscan matrix (4 rows × up to 5 cols via `RC(row,col)` matrix transform), GPIO pin assignments on the pro_micro, and the encoder node (`left_encoder`, disabled by default in the base dtsi).
  - `dumbpad.overlay` — nice_nano_v2-specific overlay: defines the column GPIOs and re-enables `left_encoder` (`status = "okay"`).
  - `dumbpad.keymap` — the actual key layout/behaviors (layers, macros, sensor/encoder bindings). This is the file to edit for keybinding changes.
  - `dumbpad.conf` / `Kconfig.defconfig` / `Kconfig.shield` — Kconfig options (shield name, encoder enable flags) and the `SHIELD_DUMBPAD` shield-detection config.
  - `dumbpad.zmk.yml` — shield metadata (id, name, required interconnect `pro_micro`, feature flags `keys`/`encoder`) used by ZMK's shield listing/tooling.
- `config/murphpad.conf` / `config/murphpad.keymap` — **not part of the active build** (no `murphpad` shield exists under `config/boards/shields/`, and `build.yaml` never references it). These appear to be leftover/reference files from the initial zmk-config template. Don't assume changes here affect any built firmware unless a corresponding shield is added to `build.yaml`.

## Keymap architecture (dumbpad.keymap)

The physical layout is a 4×5 grid (with row 3/the bottom row having an extra key, position 0) plus one rotary encoder. Key positions map via the `default_transform` in `dumbpad.dtsi` — when changing physical wiring/pinout, the matrix transform and GPIO lists in `dumbpad.dtsi`/`dumbpad.overlay` must stay in sync with each other and with the keymap's physical-layout comments.

Three layers, referenced by `#define` index (`DEFAULT 0`, `NUMPAD 1`, `ADJUST 2`):
- `default_layer` — everyday shortcuts (screenshot macro, arrow keys, virtual-desktop-switch macros, mute, layer-momentary keys to NUMPAD/ADJUST). Encoder = volume.
- `numpad_layer` — numeric keypad entry. Encoder = volume.
- `adjust_layer` — Bluetooth profile selection (`BT_SEL 0/1/2`, `BT_CLR`), navigation (Home/End/PgUp/PgDn). Encoder = page up/down.

Custom macros (`scrnsht`, `desk_lft_kp`, `desk_rgt_kp`, `desk_up_kp`) are defined in the `macros { }` block at the top of the keymap using `macro_press`/`macro_tap`/`macro_release` triples — follow this pattern for any new multi-key-combo macro rather than relying on ZMK's simpler tap-only macro shorthand, since these need modifier hold-then-release semantics.

When adding a layer, remember to add both `bindings` and `sensor-bindings` (the encoder behavior for that layer) — a layer without `sensor-bindings` falls back to `&trans`, silently inheriting the layer below's encoder behavior, which is easy to get wrong when testing.

## Conventions from git history

- Pin/GPIO changes and encoder resolution/behavior tweaks have historically been done as small, focused commits (e.g. "Change pin", "Change colums settings", "Change encoder resolution") rather than bundled — keep pin/wiring changes separate from keymap/behavior changes when committing.
