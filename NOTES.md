# Project Notes — Crosses 36 + dual trackballs

Working notes from the July 2026 session that added Fusion 360 support.
CLAUDE.md holds the rules for AI-assisted changes; this file is the human manual.

## Fusion 360 "space mouse" mode (left trackball)

Toggle: hold the **Lower** thumb key (middle left thumb) and press the
**bottom-right key on the right half** (the `/` key on base). The OLED shows
"Fusion". Same combo exits. Typing stays normal while Fusion mode is on.

| Action | How | What it sends to Fusion |
|---|---|---|
| Zoom  | roll left ball, nothing held | scroll wheel |
| Orbit | hold **J** + roll left ball  | Shift + middle-button drag |
| Pan   | hold **K** + roll left ball  | middle-button drag |

### How it works

- The left half sends **raw XY** over the split (`config/crosses_left.overlay`,
  `&trackball_peripheral_split` — only the BLE rate limiter runs there).
- The central (right) half maps it per-layer in `config/crosses.keymap` on
  `&trackball_peripheral_listener`: scroll by default, XY passthrough when the
  Orbit (5) or Pan (6) layer is active.
- Holding J/K on the Fusion layer runs the `fusion_orbit` / `fusion_pan`
  macros: they activate the Orbit/Pan layer *and* hold Shift+MMB / MMB, and
  release everything when the key is released.
- Never move the scroll mapping back onto the peripheral split node — the
  per-layer switching only works on the central listener.

### Tuning

- **Scroll/zoom speed**: `zip_scroll_scaler 1 8` in crosses.keymap (currently
  1/8; smaller second number = faster).
- **Orbit/pan speed**: `zip_xy_scaler 1 1` in the `orbit_mode` / `pan_mode`
  child nodes (e.g. `2 3` = ~0.66x, `2 1` = 2x).
- **Invert a direction**: add `&zip_x_input_code_transform` /
  `&zip_xy_transform` processors — ask Claude, or see
  https://zmk.dev/docs/keymaps/input-processors/transformer.

## Build & flash

No local toolchain — GitHub Actions builds on every push.

```bash
git push
gh run list --limit 1              # note the run id
gh run watch <run-id> --exit-status
gh run download <run-id>           # puts .uf2 files in ./firmware/ (gitignored)
```

Flash: double-tap reset on a half → `NICENANO` drive appears → copy the
matching `crosses_36_left.uf2` / `crosses_36_right.uf2` → it reboots.
Flash **both halves** whenever keymap or split behavior changes.

## Why west.yml is fully pinned (important)

In July 2026 the build broke without any config change: the upstream
`gggw-zmk-keebs` repo tracks its modules on floating branches, and two of them
had moved (left trackball driver switched to `pmw3610-alt`/`swap-xy` in May;
`zmk-report-rate-limit` adopted a newer ZMK API in March). `config/west.yml`
now pins **everything** — zmk, gggw-zmk-keebs, pmw3610 driver,
report-rate-limit, input-processor-xyz — to the exact commits of the last
known-good state (2026-02-20). When upgrading, bump all pins together and
verify the build before flashing.

## Freeze issue (watch this)

Symptom: both halves occasionally freeze (OLED stuck on last state, no keys
register); reboot of both halves fixes it. Suspected cause:
`CONFIG_ZMK_BLE_MOUSE_REPORT_QUEUE_SIZE` was `1` — a 1-deep BLE mouse report
queue that can stall the central's event thread during trackball bursts. Now
set to `20` (ZMK default) in `config/crosses.conf`. **If a freeze happens
again after this fix**, note what was happening right before, and build the
usb-logging variant to capture logs.

## OLED status screen (right half)

Left to right: output icon (Bluetooth = sending over BLE; changes for USB) →
active BT profile number + connection state (checkmark = connected; profiles
0–4 are selected with the `BT_SEL` keys on the Lower layer) → battery level →
active layer name.

## Layers

0 Base · 1 Lower (middle left thumb) · 2 Mouse (hold Space) · 3 Raise (right
thumb) · 4 Fusion (toggled, Lower + `/`) · 5 Orbit / 6 Pan (only while J/K
held on Fusion — never activate these directly).

## New machine setup

```bash
git clone https://github.com/Tardusas/my_kb.git && cd my_kb
brew install gh && gh auth login        # browser login
claude mcp add -t http deepwiki https://mcp.deepwiki.com/mcp   # for Claude Code
```
