# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A [ZMK](https://zmk.dev) firmware config repo (a "zmk-config") for a custom split-less numpad/macropad called **dumbpad** (hardware: [imchipwood/dumbpad](https://github.com/imchipwood/dumbpad) v1.1, the `combo`/`v1x` PCB family), built on a `nice_nano_v2` controller via a `pro_micro` pinout adapter. This repo does not contain the ZMK application source itself — it's a west-managed overlay that ZMK's build system pulls in as `config/`. There is no local compiler/toolchain setup; firmware is built by GitHub Actions using ZMK's reusable workflow.

The keymap is single-purpose: it turns the pad into a dedicated control surface for the **Claude Desktop app on Mac** (not Claude Code CLI), not a general-purpose macropad. There is deliberately only one layer — don't add layers/`&mo` back in without checking that's actually wanted, since the last redesign explicitly removed the numpad/adjust/desktop-shortcut layers this board used to have.

## Build

Firmware is built entirely in CI, not locally:

- [.github/workflows/build.yml](.github/workflows/build.yml) delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main` on every push/PR/manual dispatch.
- [build.yaml](build.yaml) defines the build matrix: `nice_nano@2//zmk` + `dumbpad` shield, and `nice_nano@2//zmk` + `settings_reset` shield. Add entries here (or an `include:` block for less common board/shield pairings) to add new build targets.
- `config/west.yml` tracks `zmkfirmware/zmk` at `revision: main` (unpinned) — every build compiles against ZMK's latest `main`, not a fixed release. This already broke once: ZMK's Hardware Model V2 migration (Zephyr 4.1, Dec 2025) renamed the board from `nice_nano_v2` to `nice_nano` (default revision `2.0.0`), so the old `board: nice_nano_v2` in `build.yaml` started failing CMake configure with "Invalid BOARD" out of nowhere, with no changes on this repo's side. If a build ever fails with a board/Kconfig/devicetree-binding error that doesn't map to anything actually changed here, suspect an upstream `main` change first — check [ZMK's blog](https://zmk.dev/blog) for a recent breaking-change post before debugging this repo's own files. Pinning `revision:` to a specific ZMK release tag would avoid this class of breakage, at the cost of not getting new features automatically.
- Compiled `.uf2` firmware is produced as a GitHub Actions artifact — flash it onto the nice_nano_v2 by copying it to the board's UF2 bootloader drive (double-tap reset).

There is no local `west build` setup configured in this repo (no `.west/`, no toolchain). If local builds are ever needed, they'd follow standard ZMK west workflow against the upstream `zmk` project pulled via [config/west.yml](config/west.yml).

## Repo layout

- `config/west.yml` — west manifest; pulls in `zmkfirmware/zmk` (`main` branch) as the upstream ZMK app, with this repo's `config/` as the `self` overlay path.
- `config/boards/shields/dumbpad/` — the actual, active shield definition:
  - `dumbpad.dtsi` — base devicetree: kscan matrix (4 rows × up to 5 cols via `RC(row,col)` matrix transform), GPIO pin assignments on the pro_micro, the encoder node (`left_encoder`, disabled by default in the base dtsi), and the 3 onboard status LEDs (`leds` + `indicators` nodes — see below).
  - `dumbpad.overlay` — nice_nano_v2-specific overlay: defines the column GPIOs and re-enables `left_encoder` (`status = "okay"`).
  - `dumbpad.keymap` — the actual key layout/behaviors (layers, macros, sensor/encoder bindings). This is the file to edit for keybinding changes.
  - `dumbpad.conf` / `Kconfig.defconfig` / `Kconfig.shield` — Kconfig options (shield name, encoder enable flags) and the `SHIELD_DUMBPAD` shield-detection config.
  - `dumbpad.zmk.yml` — shield metadata (id, name, required interconnect `pro_micro`, feature flags `keys`/`encoder`) used by ZMK's shield listing/tooling.
- `config/murphpad.conf` / `config/murphpad.keymap` — **not part of the active build** (no `murphpad` shield exists under `config/boards/shields/`, and `build.yaml` never references it). These appear to be leftover/reference files from the initial zmk-config template. Don't assume changes here affect any built firmware unless a corresponding shield is added to `build.yaml`.

## Keymap architecture (dumbpad.keymap)

The physical layout is a 4×5 grid (with row 3/the bottom row having an extra key, position 0) plus one rotary encoder. Key positions map via the `default_transform` in `dumbpad.dtsi` — when changing physical wiring/pinout, the matrix transform and GPIO lists in `dumbpad.dtsi`/`dumbpad.overlay` must stay in sync with each other and with the keymap's physical-layout comments. **Row 3, position 0 is the encoder's push-button**, not a regular key — this PCB's combo Cherry MX/EC11 socket wires the encoder click into the matrix instead of a separate GPIO, so it shows up as an ordinary entry in `bindings`, not in the `left_encoder` devicetree node.

A single `default_layer` (no other layers, no `#define`d indices, no `&mo`/`&tog`), entirely dedicated to controlling Claude Desktop on Mac. Every physical key has a binding — none are left as `&none`:
- `F19`/`F20` drive Claude's own configurable global shortcuts (Settings → Desktop app → General → "Quick access"/"Voice" set to Custom, captured from these two keys) — this is how the pad reaches Claude from any app without any Mac-side automation software.
- `Cmd+O`/`Cmd+K`/`Cmd+Shift+S`/`Escape`/`Enter` cover new chat/search-conversations/sidebar/stop-generation/send, and only take effect when Claude Desktop is already frontmost.
- `claude_proj_a`/`claude_proj_b`/`claude_proj_c`/`claude_proj_d` macros: `Cmd+K` + type a pinned conversation/project name + `Enter` — the placeholder text ("PROJECT A"/"B"/"C"/"D") must be edited to the user's real project names before flashing. Only A/B have a matching LED flag (see below) — C/D are quick-jump-only, since there are just 3 physical LEDs and 2 are already spoken for.
- `KP_NUM`/`SLCK` toggle the right/center status LEDs as free-form manual flags (see below) — chosen because neither has any real effect on typing on macOS.
- `Cmd +`/`Cmd -`/`Cmd 0` (zoom in/out/reset) — confirmed real Claude Desktop shortcuts.
- The encoder's push-button (row 3, position 0) sends `Enter`/send, intentionally duplicating the dedicated Send key elsewhere — lets you scroll the conversation (rotation) and send (click) without moving your hand.
- Encoder rotation = page up/down (scroll the open conversation).

Custom macros (`claude_proj_a` through `claude_proj_d`) live in the `macros { }` block. They chain one `&macro_tap &kp <KEY>` per character as separate comma-separated bindings — putting multiple `&kp` inside a single `&macro_tap` step presses them as a simultaneous chord, not as sequential characters, so don't collapse a word into one step.

If a layer is ever added back, remember both `bindings` and `sensor-bindings` (the encoder behavior) — a layer without `sensor-bindings` falls back to `&trans`, silently inheriting the layer below's encoder behavior.

## Status LEDs (dumbpad.dtsi)

The v1.1 PCB has 3 onboard LEDs (QMK's `LED_00`/`LED_01`/`LED_02`, right/center/left), wired to `pro_micro` pins 10/15/14 respectively — derived from the original ATmega32u4 pins (`B6`/`B1`/`B3`) via the same pin-numbering table already validated against this repo's matrix/encoder wiring. They're driven by ZMK's built-in `zmk,indicator-leds` feature (no custom C code, no extra Kconfig): a `leds` node (`compatible = "gpio-leds"`) defines the 3 physical LEDs, and an `indicators` node (`compatible = "zmk,indicator-leds"`) maps each one to a standard HID lock indicator — right→Num Lock, center→Scroll Lock, left→Caps Lock. Num Lock/Scroll Lock are toggled deliberately from the keymap as inert status flags; Caps Lock is left without a keymap binding on purpose so it only ever reflects the host's *real* Caps Lock state (adding a dedicated Caps Lock key here would actually flip case-sensitivity on the Mac while typing). If a LED lights inverted, flip that LED's `GPIO_ACTIVE_HIGH` to `GPIO_ACTIVE_LOW` in the `leds` node — nothing else needs to change.

## Conventions from git history

- Pin/GPIO changes and encoder resolution/behavior tweaks have historically been done as small, focused commits (e.g. "Change pin", "Change colums settings", "Change encoder resolution") rather than bundled — keep pin/wiring changes separate from keymap/behavior changes when committing.
