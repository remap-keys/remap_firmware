# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Identity

This is **Remap Firmware**, a fork of [QMK Firmware](https://github.com/qmk/qmk_firmware) maintained by remap-keys. The goal is to layer Remap-specific extensions on top of QMK while continuing to absorb upstream QMK releases.

### Branch Strategy

- `master` — mirrors `upstream/master` (qmk/qmk_firmware). **Do not commit Remap changes here.**
- `remap-develop` — long-lived branch where Remap extensions live. Currently based on QMK tag `0.32.12`.
- Release tags follow the pattern `remap_<qmk_version>` (e.g. `remap_0.32.12`).

### Remotes

- `origin` → `git@github.com:remap-keys/remap_firmware.git` (read/write)
- `upstream` → `git@github.com:qmk/qmk_firmware.git` (**read-only** — no push permission; do not attempt to push to upstream)

### Updating Remap-develop to a new upstream version

1. `git fetch upstream --tags`
2. `git checkout remap-develop`
3. `git merge <new_qmk_tag>` and resolve conflicts
4. Verify representative keyboard builds and run `qmk pytest`
5. `git tag remap_<new_qmk_version>` and push

## Build & Test Commands

Most tasks flow through the `qmk` Python CLI (installed via `pip install -r requirements.txt`); the top-level `Makefile` is a thin dispatcher.

### Firmware builds

- `make <keyboard>:<keymap>` — build a single firmware (e.g. `make planck/rev6:default`)
- `make <keyboard>:<keymap>:flash` — build and flash via the configured bootloader
- `make <keyboard>:all` — build every keymap of a keyboard
- `make all:default` — build the default keymap of every keyboard (slow; broad regression check)
- `make clean` / `make distclean`
- `make git-submodules` — initialize/update vendored upstreams in `lib/` (ChibiOS, LUFA, vusb, pico-sdk, googletest, lvgl, printf). Run after any commit that bumps submodule pointers.
- `make list-keyboards` / `make list-tests`

### Linting & formatting

- `qmk lint -kb <keyboard>` — schema-validate `info.json` / `keyboard.json` and sanity-check layouts
- `qmk format-c --core-only -a` — clang-format core C/H (`.clang-format`)
- `qmk format-python -a` — yapf (`setup.cfg`)
- `qmk format-json -i path/to/info.json` — format and validate keyboard JSON
- Dockerized one-shots (no local toolchain needed): `make format-core`, `make pytest`, `make format-and-pytest`

### Tests

- `qmk pytest` — Python test suite (nose2; sources in `lib/python/qmk/tests` per `nose2.cfg`)
- `make test:<feature>` — run a single C unit-test target via googletest (sources in `tests/<feature>/`). Examples: `make test:basic`, `make test:tap_dance`, `make test:combo`, `make test:caps_word`, `make test:auto_shift`, `make test:pointing`.
- `make test:all` — run every C test target

## Architecture

QMK is a layered C codebase targeting AVR (LUFA) and ARM (ChibiOS) microcontrollers. Layers from low to high:

1. **`tmk_core/protocol/`** — USB/HID protocol layer inherited from `tmk_keyboard`. Rarely modified.
2. **`platforms/`** — MCU/RTOS abstraction (`avr/`, `chibios/`, `test/`). Exposes `gpio.h`, `timer.h`, `wait.h`, `eeprom.h`, `suspend.h`, etc. Always go through these — never use chip registers or `_delay_ms()` directly.
3. **`drivers/`** — Peripheral drivers (sensors, OLED, RGB controllers). Cross-keyboard reusable.
4. **`quantum/`** — QMK's feature layer: keycodes, layers, tap-hold, combos, caps word, audio, RGB matrix, encoders, dynamic keymap (the VIA-compatible runtime keymap store), etc. This is where most "QMK features" live and where most Remap extensions are likely to land.
5. **`keyboards/`** — ~1100 per-keyboard configurations. Each has `keyboard.json` (or legacy `info.json`) and optional `<kb>.c/.h`, `config.h`, `rules.mk`, plus `keymaps/<name>/`.
6. **`layouts/`** — Community-defined layout macros (e.g. `LAYOUT_ortho_4x4`) that keymaps in compatible keyboards target.
7. **`users/`** — Userspace shared code. New userspace contributions are no longer accepted upstream but are supported by the build.

### Data-driven configuration

- `keyboard.json` is the source of truth for a buildable target and is validated against `data/schemas/keyboard.jsonschema`. `info.json` is the same syntax but not directly buildable.
- Prefer JSON-driven config over `config.h`/`rules.mk` defines whenever the schema supports it (matrix pins, RGB, encoders, layouts, USB IDs).

### Build-system plumbing

- The top-level `Makefile` parses `<keyboard>:<keymap>:<target>` strings and dispatches to `builddefs/build_keyboard.mk` (firmware) or `builddefs/build_test.mk` (C tests). Build artifacts go to `.build/`.
- The `qmk` CLI source lives in `lib/python/qmk/` and is the canonical entry point for non-build tasks (lint, format, JSON ops, flashing helpers, doctor, etc.).

## Code Conventions

- **C headers**: `#pragma once` (not `#ifndef` guards). GPL2+ license header (SPDX preferred). No direct GPIO/I2C/SPI register access — use QMK abstractions. Use `wait_ms()` and `timer_read()`/`timer_read32()` instead of `_delay_ms()` and AVR delay headers.
- **Filenames/directories**: lowercase only (exceptions: vendored upstream code like LUFA/ChibiOS).
- **Python**: yapf-formatted, flake8-clean. Long lines are tolerated (E501 ignored); `max_complexity=16`.
- **JSON**: must validate against the schema; format with `qmk format-json -i`.
- **Keymap files** (`keyboards/*/keymaps/*`): prefer `#include QMK_KEYBOARD_H`, layer enums over `#define`s, custom keycodes start at `QK_USER`. Default keymaps must be pristine — no custom keycodes, tap dance, or VIA enabled.
- The full QMK upstream PR checklist lives in `.github/copilot-instructions.md` — Copilot uses it for QMK PR reviews. The checklist is also a good baseline when adding/touching keyboards on the Remap side.

## Environment Notes

- **WSL + VSCode `index.lock` contention**: VSCode's git integration runs `git status --porcelain` very frequently and can race with `git checkout`/`commit` on the index. If a git command fails with `Unable to create '.git/index.lock': File exists`, retry in a tight loop:
  ```bash
  for i in $(seq 1 30); do rm -f .git/index.lock 2>/dev/null; if git <command>; then break; fi; sleep 0.1; done
  ```
  Exporting `GIT_OPTIONAL_LOCKS=0` in the shell also reduces the collision rate.
- The shell may alias `rm` to `rm -i`; use `rm -f` (or call `\rm`) when scripted removal must be non-interactive.
