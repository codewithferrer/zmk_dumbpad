# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A [ZMK](https://zmk.dev) firmware config repo (a "zmk-config") for a custom split-less numpad/macropad called **dumbpad** (hardware: [imchipwood/dumbpad](https://github.com/imchipwood/dumbpad) v1.1, the `combo`/`v1x` PCB family), built on a `nice_nano_v2` controller via a `pro_micro` pinout adapter. This repo does not contain the ZMK application source itself — it's a west-managed overlay that ZMK's build system pulls in as `config/`. There is no local compiler/toolchain setup; firmware is built by GitHub Actions using ZMK's reusable workflow.

The keymap is single-purpose: it turns the pad into a dedicated **Minecraft Bedrock command pad for Nintendo Switch**, connected over wired USB (via the Switch dock's USB-A port, or a USB-C-to-A adapter — the Switch's own USB-C port isn't a host port in handheld mode, and Switch's Bluetooth keyboard support is unreliable). Every key sends `/` (opens Minecraft's chat box pre-filled with the command prefix) + a preset command + `Enter`, except one dedicated key that sends plain `T` to open chat for a normal message. There is deliberately only one layer — don't add layers/`&mo` back in without checking that's actually wanted.

An earlier redesign turned this same pad into a **Claude Desktop / Claude Code control surface for Mac** (Quick Entry/Voice global shortcuts, pinned-conversation macros, Claude Code's numbered permission-prompt digits). That keymap is parked, not deleted — it's fully preserved at git commit `bdd6a78` if you ever want to go back to it or fork off of it.

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

A single `default_layer` (no other layers, no `#define`d indices, no `&mo`/`&tog`). Every physical key has a binding — none are left as `&none`:
- 16 of the 17 keys are `mc_*` macros (e.g. `mc_gm_creative`, `mc_kill_hostile`, `mc_tp_up`) — each sends `/`, then the rest of a Minecraft command's text, then `Enter`. The `/` keypress is what opens Bedrock's chat box (pre-filled with `/`), so the macro doesn't need to send anything to open chat first.
- The encoder's push-button (row 3, position 0 — see the physical-layout note above) is the one non-macro key: plain `&kp T`, which opens chat for a normal (non-command) message rather than a command.
- Encoder rotation = page up/down (scroll the chat history).

Custom macros live in the `macros { }` block, named `mc_<command>`. Like the parked Claude keymap, they chain one `&macro_tap &kp <KEY>` per character as separate comma-separated bindings — putting multiple `&kp` inside a single `&macro_tap` step presses them as a simultaneous chord, not as sequential characters, so don't collapse a word into one step. A couple of commands need a shifted character mid-string — `LS(I)` for the capital I in `keepInventory`, `UNDER`/`TILDE` for `_`/`~` — using ZMK's pre-defined shifted-key aliases (`AT`, `TILDE`, `UNDER`, `FSLH` etc. in `dt-bindings/zmk/keys.h`) rather than manually wrapping every symbol in `LS(...)`.

To change which commands are on which keys, or add new ones: edit the relevant `mc_*` macro's typed characters, or add a new macro node and reference it in `bindings`. Keep the `/` + command text + `RET` pattern for anything meant to run as a command; drop the leading `/` macro step (and adjust confirmation) for anything meant as a plain chat message.

If a layer is ever added back, remember both `bindings` and `sensor-bindings` (the encoder behavior) — a layer without `sensor-bindings` falls back to `&trans`, silently inheriting the layer below's encoder behavior.

## Status LEDs (dumbpad.dtsi) — mid-repair, pins don't match the original design right now

The v1.1 PCB has 3 onboard LEDs (QMK's `LED_00`/`LED_01`/`LED_02`, right/center/left). In the *original* hardware design these are wired to `pro_micro` pins 10/15/14 (`B6`/`B1`/`B3`) — but **this specific physical board has hardware faults**, and the actual pin assignments in `dumbpad.dtsi` right now reflect a series of live workarounds, not the clean original design:

- **Row 1 of the matrix** (originally `pro_micro 20`/F5) doesn't work on this board — electrically checks out fine with a multimeter, but the ZMK matrix scan never registers it, for reasons never fully root-caused. It's bypassed with a soldered jumper wire to `pro_micro 10` (B6), which does work — confirmed by USB debug logging (`CONFIG_ZMK_DISPLAY`-style `zmk-usb-logging` snippet in `build.yaml`) showing zero position events on the original pin vs. correct events on the bypassed one.
- That bypass took over B6, which the right LED originally used — so the right LED (`led_right`) has been migrated across several pins while chasing a working one: `pro_micro 1/2/3` (D1/D2/D3) all failed (no socket continuity, or continuity but no light), landing on **`&gpio1 1` (P1.01)** — one of 3 *extra* through-hole pins nice!nano exposes beyond the standard pro_micro/Arduino footprint (silkscreened `101`/`102`/`107` on the board; see [nicekeyboards.com's pinout diagram](https://nicekeyboards.com/docs/nice-nano/pinout-schematic/)). These extra pins aren't reachable via `&pro_micro N` at all — reference them directly via `&gpio1 N` (or `&gpio0 N` for the 2 SMD pads on the back).
- **Known outstanding issue**: the right LED's resistor (R2) still has its *original* PCB trace to B6 intact in parallel with the new wire to P1.01 — since B6 is now the row-1 bypass target, driving the LED via P1.01 currently leaks into row 1 and breaks it. The old R2→B6 trace needs to be physically cut/desoldered so only the P1.01 path remains.

They're driven by ZMK's built-in `zmk,indicator-leds` feature (no custom C code, no extra Kconfig): a `leds` node (`compatible = "gpio-leds"`) defines the 3 physical LEDs, and an `indicators` node (`compatible = "zmk,indicator-leds"`) maps each one to a standard HID lock indicator — right→Num Lock, center→Scroll Lock, left→Caps Lock. All 3 currently have `inactive-brightness`/`disconnected-brightness` forced to `100` as a **test override** to keep them lit regardless of host indicator state, for wiring verification — revert that (delete those two properties per node) once the hardware is settled and you want them reflecting real indicator state again. Since the current Minecraft keymap has no keys touching Num Lock/Scroll Lock/Caps Lock at all, none of the 3 LEDs have a real, meaningful signal to show right now anyway — the indicator mapping is vestigial until repurposed or reconnected to something.

If a LED lights inverted once everything is wired cleanly, flip that LED's `GPIO_ACTIVE_HIGH` to `GPIO_ACTIVE_LOW` in the `leds` node — nothing else needs to change.

## Conventions from git history

- Pin/GPIO changes and encoder resolution/behavior tweaks have historically been done as small, focused commits (e.g. "Change pin", "Change colums settings", "Change encoder resolution") rather than bundled — keep pin/wiring changes separate from keymap/behavior changes when committing.
