# PLAN: safe E4X4 (152a:8754) support

Goal: edit the E4X4 mixer config from Linux without ever writing a value the official app wouldn't. Register truth = `topping.e4x4.hid.md`.

## Root cause found (likely source of the past distortion)
- E4X4 gets 3 output strips (`PanelOutputs`, `numOutputs()`). `setOutputVol(2)` computes `0x31 + 4/5` = **`0x3503`/`0x3603`**, which the capture shows are the **phone monitor-mix** registers (range 0..100, 0x32 = center). They receive a gain like `0x02000000`.
- `pushGuiStateToDevice()` then also sends `setPhoneMix(settings[0x3503])` → `(0x02000000+100)/2` = garbage.
- The user's old `~/.config/toppingmixer/toppingmixer.settings.bak` contains `3503 02000000` / `3603 02000000` → confirmed this happened.
- Capture: `0x35 01` / `0x36 01` = source select for Output 1+2 / 3+4. E4X4 outputs volumes are only `0x31..0x34 sub 03`.

## Phase 0 — Safety net (before touching the device again)
1. Back up `~/.config/toppingmixer/` and start from an empty settings file (old files carry the bad `3503/3603` values and OTG-only keys `1101/1103/1104/3705`).
2. Add a **dry-run mode** (`TPMIX_DRY_RUN=1` env or `--dry-run`): open the device for reading meters/switches but log every outgoing packet (`>> 22 33 20 01 01 ...`) instead of `hid_write`. All later phases are verified with this first.
3. Until Phases 1–2 land, test only in demo mode (device unplugged) or dry-run.

## Phase 1 — No unsolicited writes
1. Remove `pushGuiStateToDevice()` from the `TPMixer` constructor. Startup must send nothing.
2. `OnResetDefaults()`: reset GUI + settings file only; push only after an explicit confirm dialog.
3. Add an explicit menu action "Push workspace to device" (confirm dialog) for the cases a full push is wanted.
4. Writes happen only when the user moves a control, and only for that control.

## Phase 2 — Write guard in `ToppingHID::enqueue()` (pid 0x8754 only)
Reject + log any packet whose key/value isn't in the captured set:

| Keys | Meaning | Allowed values |
|---|---|---|
| `2101..2401` / `2102..2402` / `2103..2403` | MON / 48V / INST | 0, 1 (48V still never auto-restored) |
| `2105..2405` | input gain | linear gain, abs ≤ +36 dB scale |
| `6101..680c` | mix bus L/R × 12 sources | linear gain |
| `3103..3403` | output 1–4 volume | linear gain ≤ 0 dB scale (`0x02000000`) |
| `3501`, `3601` | output 1+2 / 3+4 source | 1..14 |
| `3502`, `3602` | phone gain boost | 0, 1 |
| `3503`, `3603` | phone monitor mix | 0..100 |
| `3701..3704` | output/phone enables (captured) | 0, 1 |
| `5103..5603`, `5701..5901` | loopback vol / source | gain / 1..14 |
| `1105` | save to device | 1 |

Explicitly blocked on E4X4: `0x39xx`, `0x3axx`, `0x1101`, `0x1103`, `0x1104`, `0x3705` (OTG-model registers, never captured on E4X4). Hide those options in `SettingsDialog` when `pid == 0x8754`.
- Also sanitize on `loadSettings()`: drop/clamp out-of-range values so a bad file can't reach the device.

## Phase 3 — Fix E4X4 layout/logic bugs
1. **Outputs**: 2 strips on E4X4 (Output 1+2 → `0x31/0x32`, Output 3+4 → `0x33/0x34`); remove the 3rd strip / fix `numOutputs()`, `setOutputVol`, `refreshOutputUi`, `pushGuiStateToDevice` loops.
2. **Phone mix**: store the UI value (-100..100) separately or store raw 0..100 and stop re-converting in `pushGuiStateToDevice()`; default raw = 50 (0x32).
3. **Mixer defaults** (`initializeSettingsWithDefaults`): not "every source at 0 dB into every bus". Use a quiet/identity default (e.g. playback 1+2 → Mix A only, everything else 0) — or better, no defaults written to the device at all (GUI-only until touched).
4. **Inputs**: E4X4 has 4 independent inputs. Build 4 columns with MON/48V/INST each (`2401/2402` are captured). Today `sendInput()` forces MON off for IN3/IN4 because `cbMon[2]` is null. Optional stereo-link of IN3+4 stays GUI-only.
5. **Settings namespace**: GUI-only keys (`2_06..2_09` mute/solo/phase/loop-gain, `90xx..93xx`, `9999`) live in the same map as device registers; make sure they can never be enqueued (Phase 2 guard covers it, but keep them clearly GUI-only).

## Phase 4 — Verification
1. Compile check: `g++ tpmix.cpp $(pkg-config --cflags Qt6Widgets) -fPIC -Wall --std=c++20 -fsyntax-only`, then `make`.
2. Demo mode (unplugged): every panel renders, no crash, settings file round-trips.
3. Dry-run with device: move each control once, compare logged packets to `topping.e4x4.hid.md` (input gain, MON, INST, mix bus, output sel/vol, phone mix/boost, loopback). Startup must log zero packets.
4. Live, one control at a time, starting with low-risk ones (output volume down, phone mix), listening between steps. Do not toggle 48V with mics that can't take it.
5. Only then test "Download to device" (`0x1105`) and confirm via power-cycle + Windows app.

## Phase 5 — Optional / needs Windows capture
- Capture the official app for: auto-standby, OTG/mobile mode, LED brightness, factory reset, and any "read all settings" request. Add to `topping.e4x4.hid.md`, then unblock the matching registers.
- If a read-back request exists, replace "settings file is the truth" with reading device state on startup.
- Docs: update README supported-devices list and AGENTS.md as phases land. Don't commit the rebuilt `tpmix` binary unless asked.
