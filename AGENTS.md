# AGENTS.md

Fork goal: make tpmix safely edit a **Topping E4X4** (`152a:8754`) mixer config from Linux. Upstream fork targeted E1X2/E2X2 OTG and never tested the E4X4. The user's real device is currently in a good state (restored via the Windows TOPPING app) — do not break it.

## Layout
- Single-file app: `tpmix.cpp` (~3.9k lines, Qt 6 Widgets + hidapi). No other sources, no tests, no linter/formatter config.
- `tpmix.cpp.bak` is a stale older copy (pre-Qt-port leftovers); never edit it, don't treat it as reference.
- `tpmix` binary is **committed**; `make` overwrites it. Don't commit a rebuilt binary unless asked.
- `topping.e4x4.hid.md`: the only real E4X4 USB capture. It is the source of truth for E4X4 register IDs; code constants not found there are guesses from OTG models.
- `packaging/` + `.github/workflows/appimage.yml`: AppImage build, CI runs only on `v*` tags / manual dispatch.
- packaging/pipewire/90-tpmix-e4x4-channels.conf: PipeWire module-loopback drop-in (E4X4 capture AUX4-9 → 3 loopback sources, playback AUX0-7 → 4 playback sinks); installed by the PKGBUILD to /usr/share/pipewire/pipewire.conf.d/.
- `packaging/arch/PKGBUILD` (`tpmix-git`) clones GitHub `main`, not the local tree: packaging changes only build with `makepkg` after they are pushed. Never `makepkg -i`/`sudo` from an agent.

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
- Sample rate / buffer size (shown in the Windows app) are USB-audio driver settings, not HID: on Linux PipeWire owns them (`clock.rate`, `clock.allowed-rates`, `clock.quantum`). User decision: do not add them to tpmix.
- Device only pushes switch states and meters; mixer gains/volumes can't be read back. GUI state comes from `~/.config/toppingmixer/toppingmixer.settings` (`%04x %08x` lines), not the device.

## Device-safety rules (a past build corrupted the E4X4 config)
- Nothing may write to the device implicitly. Startup and "Reset Defaults" send nothing; a full write only happens via Download dialog → "Push full workspace" (confirmed). Keep it that way.
- E4X4 write guard: `ToppingHID::isAllowedE4x4Write()` in `enqueue()` is the allow-list derived from `topping.e4x4.hid.md`; refused packets print `[blocked] ...` and are not stored. To support a new E4X4 register, add a capture to the .md first, then extend the guard. `loadSettings()` drops saved values the guard would refuse.
- Blocked on E4X4 until captured: `0x39xx` standby, `0x3axx` OTG, `0x1101/1103/1104`, `0x3705`. Their Settings-dialog widgets are disabled.
- `0x3503/0x3603` are phone monitor-mix (raw 0..100, 50 = center), not volumes. `setPhoneMix()` takes -100..100; stored settings are raw.
- Never send or restore 48V except from the 48V button handler.
- Gate E4X4 behaviour on `pid == 0x8754` (`panelInputs->fourMono` for inputs); other models' code paths must stay unchanged.
- E4X4 layout: 4 mono inputs (INST only IN1/IN2), 2 output strips (1+2: `3501`, `3103/3203`, MON `3701`, LINE `3703`; 3+4: `3601`, `3303/3403`, `3702`, `3704`).

## Device access / testing
Test with `./tpmix --dry-run` first (packets printed as `[dry-run] >> <hex>`, comparable with the capture).

hidraw needs a udev rule (`KERNEL=="hidraw*", ATTRS{idVendor}=="152a", TAG+="uaccess"`) or the app silently falls into demo mode.
