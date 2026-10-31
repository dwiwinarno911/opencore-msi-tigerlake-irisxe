# OpenCore EFI — Intel 11th Gen (Tiger Lake-U) Hackintosh

Working OpenCore EFI for an **Intel 11th Gen Tiger Lake-U laptop** running **macOS Sequoia 15.8.2 (24H232)**, with Intel Xe-LP graphics acceleration via [WhateverXe](https://github.com/b00t0x/WhateverXe).

> ⚠️ **SMBIOS IS ZEROED OUT.** `MLB`, `SystemSerialNumber`, `SystemUUID` and `ROM` are all zeros.
> The EFI will boot, but iMessage / FaceTime / App Store will not work until you generate your **own** SMBIOS.
> Use [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) with model **MacBookPro16,2** and put the values in
> `PlatformInfo > Generic`. Never share your real serials.

## Hardware

| | |
|---|---|
| Laptop | MSI (ACPI OEM ID `MSI_NB`), Insyde-based firmware |
| CPU | Intel Core i7-1195G7 (Tiger Lake-U, 4C/8T) |
| iGPU | Intel Iris Xe, PCI `0x8086:0x9A49` rev `0x03` (96 EU) |
| Audio | Realtek **ALC256** (`10EC:0256`), HDA controller `8086:A0C8` |
| WiFi / BT | Intel combo module (USB BT transport) |
| macOS | 15.8.2 (24H232) |
| OpenCore | see `EFI/OC/OpenCore.efi` (config `MinimumVersion`/`Version` in `Misc`) |

## Status

**Working**
- Boot, native framebuffer, internal panel 1920×1080, brightness slider
- **Metal 2 acceleration** (`IntelAccelerator` active) — see the WEX patch below
- Audio: speakers + **headphones** (jack detection) with `layout-id = 21`
- WiFi via `itlwm` + [HeliPort](https://github.com/OpenIntelWireless/HeliPort)
- USB, battery, keyboard, S3-related ACPI, NVMe

**Known problems on this hardware / OS build**
- **PS/2 ELAN v4 trackpad stutters** with `VoodooPS2Trackpad`. It is a PS/2 device (`ApplePS2Elan`), *not* I2C, so VoodooI2C is irrelevant here. Disabling the trackpad plugin makes the pointer perfectly smooth but loses tap-to-click/scroll (this is a clickpad with no physical buttons, so that trade-off is usually not acceptable).
- **Safari video can freeze the GPU.** This is the payload/OS mix (WhateverXe Graphics 0.3.2 targets build `24H32`, this machine runs `24H232`). Same root cause as the ControlCenter crash below.
- Internal Intel Bluetooth is unreliable once `itlwm` is loaded, because `itlwm` exposes no `IO80211` interface (`bluetoothd` logs `WiFiManagerClientCopyDevices failed: NULL`, so BT/WiFi coexistence cannot be coordinated). Using a USB Bluetooth dongle (BCM20702) resolves it.

## Customizations in this EFI

### 1. WhateverXe fingerprint patch for macOS build 24H232 ⭐
WhateverXe 0.6.1 hard-codes the kernel instruction patterns it validates for Sequoia in
`tglPrepareSequoiaMMIOUC()`. On build **24H232** the ninth pattern does not match, which makes
`TGLMMIOStrongUCRouted = No` → `tglMMIOUCReady = false` → **`TGLAcceleratorAdmissionReady = No`**
and the accelerator never starts (framebuffer only, no Metal).

| | |
|---|---|
| Expected by WEX (build 24H32) | `_IOProtectCacheMode+0xCA` = `E8 71 CB 02 00` (call rel32 `0x2CB71`) |
| Actual on build 24H232 | `E8 A1 CB 02 00` (call rel32 `0x2CBA1`, target shifted `+0x30`) |

The bundled `Kexts/WhateverXe.kext` is **modified**: the expected constant is changed in the two
places it is stored (the inlined `movl $0x2cb71e8` in `XE::processKernel`, file offset `0x16703`,
and the `__const` array `protectCall`, offset `0xcbabe`). Binary hash: pristine `e92b65b0…` →
patched `db41df41…`.

Verify after boot:
```bash
ioreg -w0 -l | grep -aoE '"TGL(MMIOStrongUCRouted|AcceleratorAdmissionReady)" = [^|]*'
ioreg -rc IntelAccelerator | head -1
system_profiler SPDisplaysDataType | grep -i metal
```
Expected: `TGLMMIOStrongUCRouted = Yes`, `TGLAcceleratorAdmissionReady = Yes`, `IntelAccelerator` present, `Metal Support: Metal 2`.

To go back to the untouched kext, download `WhateverXe-0.6.1-RELEASE.zip` from the upstream release page and replace `EFI/OC/Kexts/WhateverXe.kext` (you will lose acceleration on 24H232).

Other Sequoia builds may need a different displacement — after any macOS update, re-check the two `ioreg` properties above. If `TGLAcceleratorAdmissionReady` is `No`, decode `TGLSequoiaMMIOGuard` (20 × u64; index 16 is the ninth comparison) and compare `_IOProtectCacheMode+0xCA` against the expected bytes.

### 2. `boot-args` must be in `NVRAM > Delete`
OpenCore does **not** overwrite an existing NVRAM variable. Without `boot-args` listed in
`NVRAM > Delete > 7C436110-…`, OpenCore logs
`OCVAR: Setting NVRAM …:boot-args - ignored, exists` and the required arguments never apply.
This config lists it.

Current boot-args:
```
-allow3d -disablegfxfirmware -igfxdvmt
```

### 3. DVMT fixed at 60 MB
The firmware cannot change DVMT (GMS = `0xFE` → 60 MB), so the config uses the WEX recipe for that
case: `-igfxdvmt` plus, on `PciRoot(0x0)/Pci(0x2,0x0)`:
`framebuffer-patch-enable = 01000000`, `framebuffer-stolenmem = 0000C003`.
If your BIOS allows 96 MB, remove all three.

### 4. Audio `layout-id = 21`
`PciRoot(0x0)/Pci(0x1f,0x3) > layout-id = 21`. AppleALC's ALC256 layout 14 ("Dell with subwoofer")
exposes no headphone port on this machine; 21 does. Try 11 / 13 / 66 / 88 if your jack behaves differently.

### 5. Brightness keys (experimental, unverified)
The EC queries `_QB7`/`_QB8` call `BRTN(0x87/0x86)`, but `_QB7/_QB8` are gated on
`EOSV() >= 2` (a `_OSI("Windows …")` probe) and `BRTN` itself on `DIDX/DIDY`, so the notification
never reaches the display device on macOS. This config renames the OEM definitions to `XB07`/`XB08`
(two `ACPI > Patch` entries, `TableSignature = DSDT`, `Count = 1`) and `SSDT-BRTN.aml` provides
replacements that keep the OEM EC bookkeeping and notify `\_SB.PC00.GFX0.DD1F` unconditionally.
`BrightnessKeys.kext` is loaded for the ACPI path. **Not confirmed working on this machine** — the
on-screen brightness slider works regardless (PNLF + WEX).

### 6. WiFi: `itlwm` + HeliPort (AirportItlwm also included)
`itlwm.kext` is enabled and `AirportItlwm.kext` disabled. `itlwm` exposes an Ethernet-like interface
and needs the HeliPort app; `AirportItlwm` (2.4.0, Sequoia fork) gives the native WiFi menu and correct
BT/WiFi coexistence but was slower on this 2.4 GHz-only network. Flip `Enabled` in `Kernel > Add` to switch.

### 7. Metal binary-archive workaround (script)
On 24H232 the payload's Metal driver does not line up with Apple's precompiled archives, so
`ControlCenter` crash-loops (`MTLLoader sliceIDForDevice:…` → assert inside
`RB::load_library_archive`). `scripts/metal-archive-workaround.plist` is a per-user LaunchAgent that sets
`RB_USE_BINARY_ARCHIVE=0`, `MTL_DEBUG_IGNORE_PRECOMPILED_BINARIES=1` and
`MTL_DEBUG_IGNORE_PSO_FAIL_ON_MISS=1`, then restarts ControlCenter. Install with:
```bash
cp scripts/metal-archive-workaround.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.wex.metal-archive-workaround.plist
```

### 8. Diagnostics left at defaults
`Misc > Debug > SysReport = false`, and VoodooPS2's `LogScanCodes = false` (both were enabled only while
debugging).

## ACPI

| File | Purpose |
|---|---|
| `SSDT-PLUG.aml` | `plugin-type = 1` for XCPM |
| `SSDT-PNLF.aml` | Backlight device (required by WEX for internal-panel brightness) |
| `SSDT-BRTN.aml` | **Custom** — brightness-key notification fix (see above) |
| `SSDT-XOSI.aml` | `_OSI` → `XOSI` rename target |
| `SSDT-GPI0.aml` | GPIO controller stub for VoodooI2C (unused on this PS/2 trackpad, harmless) |
| `SSDT-USBX.aml` | USB power properties |
| `SSDT-USB-Reset.aml` | USB reset / port map helper |
| `SSDT-EC`-style helpers | `SSDT-ALS0`, `SSDT-SBUS`, `SSDT-MCHC`, `SSDT-RTCAWAC` |

`ACPI > Patch`: `GPI0 _STA → XSTA`, `PNLF → XNLF`, `_OSI → XOSI`, plus the two `_QB7/_QB8 → XB07/XB08` renames.

## Kexts

| Kext | Version | Upstream |
|---|---|---|
| Lilu | 1.7.3 | https://github.com/acidanthera/Lilu |
| VirtualSMC / SMCBatteryManager / SMCLightSensor / SMCProcessor / SMCSuperIO | 1.3.9 | https://github.com/acidanthera/VirtualSMC |
| **WhateverXe** | 0.6.1 (**patched**) | https://github.com/b00t0x/WhateverXe |
| AMFIPass (**Mod** build) | 1.4.1 | https://github.com/b00t0x/WhateverXe-Graphics (`extras/`) |
| AppleALC | 1.9.9 | https://github.com/acidanthera/AppleALC |
| BrightnessKeys | 1.0.4 | https://github.com/acidanthera/BrightnessKeys |
| ECEnabler | 1.0.6 | https://github.com/1Revenger1/ECEnabler |
| NVMeFix | 1.1.4 | https://github.com/acidanthera/NVMeFix |
| RestrictEvents | 1.1.7 | https://github.com/acidanthera/RestrictEvents |
| USBToolBox + UTBDefault + XHCI-unsupported | 1.2.0 / 1.0 / 0.9.2 | https://github.com/USBToolBox/kext |
| VoodooPS2Controller (+ Keyboard/Mouse/Trackpad) | 2.3.8 | https://github.com/acidanthera/VoodooPS2 |
| VoodooI2C (+ HID / GPIO / Services) | 2.9.1 | https://github.com/VoodooI2C/VoodooI2C |
| VoodooInput | 1.1.7 | https://github.com/acidanthera/VoodooInput |
| itlwm | 2.3.0 | https://github.com/OpenIntelWireless/itlwm |
| AirportItlwm (disabled) | 2.4.0 | OpenIntelWireless fork for Sequoia |
| IntelBluetoothFirmware / IntelBTPatcher | 2.5.0 | https://github.com/OpenIntelWireless/IntelBluetoothFirmware |
| BlueToolFixup | 2.7.3 | https://github.com/acidanthera/BrcmPatchRAM |

`Drivers`: `HfsPlus.efi`, `OpenRuntime.efi`, `ResetNvramEntry.efi`.

## Not in this EFI — install separately

1. **WhateverXe Graphics pkg** — https://github.com/b00t0x/WhateverXe-Graphics/releases
   Boot this EFI first (WEX must be loaded), then install the pkg; it puts the signed graphics stack in
   `/Library` and rebuilds AuxKC. Required `csr-active-config = 03080000` (set here).
2. **HeliPort** (only if you keep `itlwm`) — https://github.com/OpenIntelWireless/HeliPort/releases
3. **GenSMBIOS** — generate your own SMBIOS before signing in to iCloud.

## Order of operations for a fresh install

1. Copy `EFI/` to the ESP (`EFI/BOOT/BOOTx64.efi` + `EFI/OC/…`).
2. Generate SMBIOS (MacBookPro16,2) and fill `PlatformInfo > Generic`.
3. Boot macOS with this EFI.
4. Install **WhateverXe Graphics** pkg, reboot.
5. Verify GPU: `ioreg -rc IntelAccelerator` and `system_profiler SPDisplaysDataType | grep -i metal`.

## Credits / license

All kexts belong to their respective authors — see the table above. WhateverXe is BSD-3-Clause and is
redistributed here **modified** (constant patch described in section 1) for build 24H232 only.
Everything else in this repo is configuration data; use at your own risk.
