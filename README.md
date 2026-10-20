# daxs ZMK configuration

ZMK firmware for the daxs split keyboard: 39 keys, a rotary encoder on the left
half, and a PAW3222 trackball on the right half, which is the central.

The firmware is built on the ZMK main line (Zephyr 4.1) from the `cormoran/zmk`
fork, so most settings are changed on the running keyboard with
[daxial](https://wadackel.github.io/daxial/) instead of rebuilding: key
bindings, encoder rotation, trackball direction and speed, mouse gestures,
combos and macros. Values in this repository are the defaults; what is saved
from the app wins over them.

What the app cannot change lives here: the number and order of layers, behavior
definitions, hold-tap timing, and the structure of the input chains.

`AGENTS.md` is the detailed guide (hardware, key positions, input chains,
pitfalls).

## Build

Needs [Nix](https://nixos.org/download.html) with flakes enabled. Commands run
through the development shell, either inside `nix develop` or one at a time:

```bash
nix develop --command just setup                  # once: fetch ZMK, Zephyr and the modules
nix develop --command just build                  # both halves and the settings reset image
nix develop --command just build-target daxs_R    # one target: daxs_R, daxs_L or settings_reset
nix develop --command just clean                  # remove build output
nix develop --command just pristine               # also remove the west workspace
```

| Output | For |
|---|---|
| `build/daxs_R.uf2` | Right half: central, trackball, ZMK Studio |
| `build/daxs_L.uf2` | Left half: peripheral, encoder |
| `build/settings_reset.uf2` | Erases stored settings, including BLE bonds and everything saved from the app |

To flash, put a half into its bootloader and copy the matching file onto the
drive that appears. After `settings_reset.uf2`, flash the half's own firmware
again and re-pair it with each host; flash both halves if they no longer find
each other.

Do not run `nix flake update`: the toolchain depends on the pinned `nixpkgs`
and on Zephyr SDK 0.17.0.

## Layout

| Path | Contents |
|---|---|
| `config/` | Keymap (`daxs.keymap`), gesture defaults (`gestures.dtsi`), west manifest |
| `boards/shields/daxs/` | Shield definition, per-half overlays and configuration |
| `module/` | Empty Zephyr module files that keep this repository from picking up Zephyr's own |

## Modules

Pinned by commit in `config/west.yml`, at the same revisions as
[zmk-config-dax6](https://github.com/wadackel/zmk-config-dax6).

| Module | Role |
|---|---|
| `cormoran/zmk` (`main+custom-studio-protocol`) | ZMK with custom Studio RPC |
| `wadackel/zmk-dax-modules` | Mouse gestures, the per-profile OS layer, trackball CPI, direction and speed, combos and macros editable at runtime |
| `zmk-driver-paw3222` | Trackball sensor driver |
| `zmk-behavior-runtime-sensor-rotate` | Encoder bindings editable at runtime |
| `zmk-rgbled-widget` | Status LED |
