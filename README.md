# msi-mystic-light-re

Reverse-engineering the USB protocol variant of MSI Mystic Light used by a specific board revision, so it can be driven natively on Linux via [OpenRGB](https://openrgb.org/) instead of requiring Windows + MSI Center.

**Status: work in progress, not yet usable.** This repo currently documents findings, not a finished fix.

> **Türkçe özet:** Bu depo, MSI Mystic Light'ın bu anakart revizyonunda kullandığı USB protokol varyantını tersine mühendislik yaparak `OpenRGB`'de düzeltilmesini hedefliyor. Cihaz (`0db0:0076`) OpenRGB tarafından kısmen tanınıyor ama yanlış protokol varyantı hedeflendiği için çalışmıyor. USB trafiği yakalama altyapısı (usbmon + tshark) çalışır durumda; ancak üretici yazılımının (MSI Center) bir VM-tespit engelini henüz aşamadığımız için gerçek bir renk/efekt değiştirme komutu yakalanamadı. Proje devam ediyor. Depo içeriği (kod, teknik dokümantasyon) İngilizce; bu paragraf sadece hızlı bir özet için Türkçe eklendi.

## Target device

**USB ID:** `0db0:0076` ("MYSTIC LIGHT"), on an MSI MAG B850 TOMAHAWK MAX WIFI (MS-7E62) motherboard. Controls the board's fan/case ARGB headers.

`OpenRGB` already has a detection hook for this exact VID:PID, but it targets the wrong protocol variant and fails with a "packet length = 0" error. Tracked upstream at [OpenRGB#4645](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/4645) (open, unresolved — the original reporter is Linux-only and can't capture real traffic to fix it themselves, which is exactly what this project aims to supply).

## Method

MSI only ships Windows software for full control of this device. The approach:

1. Run the vendor's official control app (**MSI Center**, which downloads "Mystic Light" as a sub-module) inside a Windows VM (`libvirt`/`qemu-kvm`) with the real USB device passed straight through via `usb-host` (`virsh`/`virt-xml --add-device --hostdev <vid>:<pid>`) — the guest talks to the real hardware, not an emulation.
2. Capture the actual USB traffic on the **host** side while operating the vendor app (`usbmon` + Wireshark/`tshark`).
3. Use the captured color/effect-change commands to correct OpenRGB's existing (wrong-variant) protocol implementation for this device.

## Current blocker: vendor app won't run in the VM

MSI Center installs fine in the passthrough VM but perpetually reports "no internet connection," even though DNS/ping/HTTPS all work correctly from inside the guest.

Root cause found: MSI Center's "Case" service sets an internal `bCaseConnected = false` flag for this device, which hides the entire RGB control screen — this looks like a VM/environment detection check rather than an actual connectivity problem.

Attempts to defeat this (none resolved it so far, kept here so they aren't retried blindly):

- Spoofing SMBIOS data (via `virt-xml --sysinfo`) to match the real motherboard (manufacturer, `MS-7E62` product string)
- Switching the virtual NIC model from `virtio` to `e1000e` (virtio is more readily fingerprinted as "virtual")
- Adjusting vCPU topology to a single socket / multiple cores (avoids both an obvious multi-socket "this is a VM" signal and Windows Pro's 2-socket licensing cap)
- A static string search across `API_Case.dll` and `MsiHid.dll` for an obvious environment check — nothing conclusive found this way; a real disassembly/decompilation pass is the logical next step

## USB traffic capture — confirmed working

Captured on the host via `usbmon` (`/dev/usbmon1`) + `tshark`, while the device just sits idle (MSI Center never got far enough to send a color command, due to the block above):

- The device responds with standard USB HID descriptors.
- It emits a periodic 64-byte interrupt-IN packet roughly every 1.1 seconds.

This confirms the capture pipeline itself works end-to-end; what's missing is a captured **color/effect-change command**, which requires either getting past the `bCaseConnected` block or bypassing MSI Center entirely.

## Status / next steps

- [x] Confirm USB capture pipeline works (`usbmon` + `tshark`)
- [x] Root-cause the "no internet" symptom to MSI Center's `bCaseConnected` flag
- [ ] Disassemble/decompile `API_Case.dll` / `MsiHid.dll` to find and defeat the actual detection check (string search alone wasn't enough)
- [ ] Alternative path: install `OpenRGB` directly inside the guest VM (bypassing MSI Center entirely) and capture whatever traffic *it* generates when changing colors — since OpenRGB already has partial detection logic for this VID:PID, this may be the faster route to a real color-change capture
- [ ] Once a color-change command is captured: correct OpenRGB's existing (wrong-variant) protocol implementation for this device, and/or contribute findings back to [OpenRGB#4645](https://gitlab.com/CalcProgrammer1/OpenRGB/-/issues/4645)

## What's *not* in this repo

Vendor-supplied binaries (MSI Center installer, extracted DLLs) are **not** committed here — they're copyrighted third-party software and are only used locally for research. If you want to reproduce the analysis, obtain MSI Center yourself from MSI.

## Related

- [zalman-oz-lcd-re](https://github.com/droltr/zalman-oz-lcd-re) — companion project reverse-engineering a Zalman AIO's LCD/LED controller on the same test system.

## License

GPL-3.0 (see [`LICENSE`](LICENSE)).
