# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal ZMK user config for a **GGGW Crosses 36-key split keyboard** (nice_nano / nRF52840) with **dual PMW3610 trackballs**:

- **Right trackball** (central, right half): pointer movement. Sensor + listener defined in the shield (`gggw-zmk-keebs` repo, `crosses_right.overlay`).
- **Left trackball** (peripheral, left half): relays **raw XY** over the split (`&trackball_peripheral_split` in `config/crosses_left.overlay`). Mode mapping happens **on the central side** in `config/crosses.keymap` via `&trackball_peripheral_listener`, so it can change per layer:
  - default: XY → scroll (`&zip_xy_to_scroll_mapper` + `&zip_scroll_scaler 1 4`) — this is also **zoom** in Fusion 360
  - `ORBIT` layer: XY passthrough while the `fusion_orbit` macro holds Shift+MMB
  - `PAN` layer: XY passthrough while the `fusion_pan` macro holds MMB

**Do not re-add scroll processors to `&trackball_peripheral_split`** — layer-dependent switching only works on the central listener (the peripheral does not track layer state for this purpose).

### Layers

| # | Name   | Activation                          |
|---|--------|-------------------------------------|
| 0 | Base   | default                             |
| 1 | Lower  | `&mo 1` (left thumb)                |
| 2 | Mouse  | `&lt 2 SPACE`                       |
| 3 | Raise  | `&mo 3` (right thumb)               |
| 4 | Fusion | `&tog FUSION` on Lower, bottom-right key. Also carries Mouse-layer clicks on the left home row (S/D/F = R/M/L click) |
| 5 | Orbit  | only via `fusion_orbit` macro (hold H on Fusion layer) |
| 6 | Pan    | only via `fusion_pan` macro (hold J on Fusion layer)   |

### Key Files

- `config/crosses.keymap` — keymap, macros, trackball listener overrides
- `config/crosses_left.overlay` — left-half hardware (kscan cols, PMW3610, OLED, input-split)
- `config/crosses.conf` — Kconfig (pointing, BLE, PMW3610 tuning)
- `config/west.yml` — pins **everything** to exact commits (zmk, gggw-zmk-keebs, pmw3610 driver, report-rate-limit, input-processor-xyz). Upstream `gggw-zmk-keebs` tracks floating branches for its modules and has broken builds before (driver switch on 2026-05, rate-limit API change on 2026-03). Never un-pin to a branch; when upgrading, bump all pins together and verify the build.
- `build.yaml` — GitHub Actions build matrix (crosses_36_left / crosses_36_right)
- `config/crosses.dtsi` — reference copy of the shield dtsi (the one actually built comes from `gggw-zmk-keebs`)

## ZMK Development Guidelines

**CRITICAL PREREQUISITES - MANDATORY BEFORE ANY IMPLEMENTATION CHANGES:**

1. **Consult the ZMK Repository Expert**: Query the Deepwiki MCP server (repoName: `zmkfirmware/zmk`) to understand the current implementation and identify potential issues.

2. **Consult the Zephyr Repository Expert**: Since ZMK is Zephyr-based, also query the Deepwiki MCP server (repoName: `zephyrproject-rtos/zephyr`) for Zephyr-specific implementation details when needed.

3. **Consult the Shield Source**: The Crosses shield, layouts, and trackball plumbing live in `Good-Great-Grand-Wonderful/gggw-zmk-keebs` (branch `zephyr-4.1`). Check it before overriding any devicetree node — many nodes (listeners, transforms, split inputs) are defined there.

4. **Share Context**: Provide the MCP server with current implementation details, identified problems or requirements, and your proposed implementation approach.

5. **Validate Implementation Strategy**: Confirm your implementation plan with the ZMK expert before proceeding.

6. **Update Documentation**: After making any changes, always update README.md with relevant information about new features, usage instructions, or implementation status.

**Why This Process is Essential:**
- **Pre-trained Knowledge Limitations**: LLM training data may not reflect the latest ZMK architecture, APIs, or best practices
- **Build Failure Prevention**: ZMK has specific conventions and dependencies that may not be obvious from general knowledge
- **Implementation Accuracy**: The ZMK codebase has evolved significantly, and outdated approaches will likely fail

**NO ASSUMPTIONS**: Always verify current ZMK practices before implementation. Note that `west.yml` pins ZMK to a specific commit — verify features exist at that revision, not just on `main`.

**EXPERT FIRST**: Consult the repository experts before coding, not after encountering build failures.

## Build Verification

There is **no local build environment**. Firmware is built by GitHub Actions (`.github/workflows/build.yml` → `zmkfirmware/zmk` user-config workflow) on every push.

**MANDATORY BUILD CHECK** after any change:

```bash
git push
gh run list --limit 1          # get the run id for the new push
gh run watch <run-id> --exit-status
```

If the build fails, inspect logs with `gh run view <run-id> --log-failed`, fix, and repeat until green.

Builds take several minutes. Wait for completion before declaring success. Firmware artifacts (`crosses_36_left`, `crosses_36_right` .uf2 files) can be fetched with `gh run download <run-id>`.

**Devicetree sanity checks before pushing:**
- Every keymap layer must have exactly **36** bindings
- Layer indices in `layers = <...>` and `#define`s must match keymap order
- New input processors may require their Kconfig options in `config/crosses.conf`

## Flashing

Double-tap reset on a half to enter the bootloader, then copy the matching `.uf2` onto the `NICENANO` mass-storage device. Flash both halves when shared config (keymap layers, split behavior) changes.
