# AGENTS.md

Fork goal: make tpmix safely edit a **Topping E4X4** (`152a:8754`) mixer config from Linux. Upstream fork targeted E1X2/E2X2 OTG and never tested the E4X4. The user's real device is currently in a good state (restored via the Windows TOPPING app) — do not break it.

## Layout
- Single-file app: `tpmix.cpp` (~3.9k lines, Qt 6 Widgets + hidapi). No other sources, no tests, no linter/formatter config.
- `tpmix.cpp.bak` is a stale older copy (pre-Qt-port leftovers); never edit it, don't treat it as reference.
- `tpmix` binary is **committed**; `make` overwrites it. Don't commit a rebuilt binary unless asked.
- `topping.e4x4.hid.md`: the only real E4X4 USB capture. It is the source of truth for E4X4 register IDs; code constants not found there are guesses from OTG models.
- `packaging/` + `.github/workflows/appimage.yml`: AppImage build, CI runs only on `v*` tags / manual dispatch.

## Build / verify
- `make` (plain `g++` + `pkg-config Qt6Widgets` + `-lhidapi-hidraw`, C++20, needs g++ ≥ 13 for `std::format`).
- Fast compile check without touching the binary: `g++ tpmix.cpp $(pkg-config --cflags Qt6Widgets) -fPIC -Wall --std=c++20 -fsyntax-only`
- No automated tests. Without a device (or no hidraw permission) the app runs in **demo mode** with fake meters — usable for UI checks.
- `./tpmix --tray` starts hidden in tray.

## Protocol essentials
- 16-byte HID packets: `22 33 20 01 01 <ch> <sub> <int32 BE value> <tail>`; settings key = `(ch<<8)|sub`.
- E4X4: bytes 11–12 are `00 00`. E1X2/E2X2 models instead put a CRC16 there (`enqueue()` branches on `pid`). Keep that branch intact.
- Gains are linear fixed-point: `0x02000000` = 0 dB, negative = phase inverted, 0 = mute.
- Save-to-device for E4X4 is `0x11 0x05` (per capture). The original GitLab code's `0x11 0x06` does not persist.
- Device only pushes switch states and meters; mixer gains/volumes can't be read back. GUI state comes from `~/.config/toppingmixer/toppingmixer.settings` (`%04x %08x` lines), not the device.

## Device-safety gotchas (cause of a past bricked-sounding config)
- On startup the app calls `pushGuiStateToDevice()` (`TPMixer` constructor, after `loadSettings()`), which writes the entire GUI/default state to the hardware, and `OnResetDefaults()` does too. Treat any write path as hazardous.
- E4X4 shows 3 output strips, but `setOutputVol(2)` hits `0x3503/0x3603` = phone monitor-mix (0..100) with a gain value. Likely cause of the past distortion; E4X4 output volumes are only `0x31..0x34 sub 03`. `0x35/0x36 sub 01` = output 1+2 / 3+4 source select.
- Unverified on E4X4, do not write them to an 8754 device without a capture: `0x3901` (standby), `0x3a01` (OTG), `0x1104` (LED), `0x3705`.
- `initializeSettingsWithDefaults()` routes all 12 sources at 0 dB into every mix bus — loud summed mix if pushed.
- Phone mix is double-converted: `setPhoneMix()` maps -100..100 → 0..100, but stored/pushed values are already raw, so each push drifts toward 100.
- 48V restore is intentionally disabled; keep it that way.
- Input UI is laid out for 3 columns (IN1, IN2, IN3+4 linked); 48V/MON/INST only exist for the first two columns — E4X4 has 4 independent inputs.

## Device access
hidraw needs a udev rule (`KERNEL=="hidraw*", ATTRS{idVendor}=="152a", TAG+="uaccess"`) or the app silently falls into demo mode.
