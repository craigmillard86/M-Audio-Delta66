# Porting notes: M-Audio Delta 66 on macOS Sonoma (Intel)

## 1. Card identification (2026-10-25)

Host: Dell Precision T3610, macOS 14.8.9 (23J631), x86_64.

From `ioreg -l -w0` (node `pci1412,1712@4`, pcidebug `8:4:0`):

| Field               | Value  |
|---------------------|--------|
| Vendor ID           | 0x1412 (VIA/ICEnsemble) |
| Device ID           | 0x1712 (ICE1712 / Envy24) |
| Subsystem vendor ID | 0x1412 |
| Subsystem ID        | 0xd632 |
| Revision ID         | 0x02 (chip revision) |
| Class code          | 0x040100 (multimedia audio) |
| IRQ                 | io-apic-0, pin 7 |

The card enumerates, so no OpenCore/PCI bridge work is needed. Device ID is 0x1712, not 0x1724,
so this is the ICE1712 path.

Note: `system_profiler SPPCIDataType` lists only the Quadro K600. It reports devices that have a
driver attached, so the absence of the Envy24 there is expected.

### Matching against ALSA

Reference: Linux v6.12 `sound/pci/ice1712/` (copies in `reference/alsa/`).

The `ICE1712_SUBDEVICE_*` constants in `delta.h` are not PCI subsystem IDs. They are the first
four EEPROM bytes, or, if the EEPROM is blank, the byte-swapped PCI subsystem vendor/ID
(`ice1712.c:2248-2254`):

    swab16(0x1412) << 16 | swab16(0xd632) = 0x121432d6 = ICE1712_SUBDEVICE_DELTA66 (delta.h:29)

So this is a Delta 66 family card. **Classic or rev E cannot be determined from PCI config
space.** ALSA tells them apart only at runtime (`delta.c:624-626`): if the EEPROM GPIO direction
byte is `0xfb` the card is treated as `ICE1712_SUBDEVICE_DELTA66E`. Both variants share the same
PCI IDs, so an Info.plist personality cannot separate them either.

Status: board revision is open. It has to be read from the EEPROM by driver code.

## 2. Building the original driver (Phase 2)

Upstream: https://github.com/vrudikov/Envy24 at `4d66ef3`, imported unchanged into `driver/`.

### Toolchain

There's no Xcode on this machine, only the Command Line Tools (Apple clang 16, SDKs 13.3 to 15.2).
Those include everything needed to build a kext (`Kernel.framework` headers including
`IOKit/audio`, `libkmod.a`, `libkmodc++.a`, `libclang_rt.cc_kext.a`). So instead of
`Envy24_64bit_Driver.xcodeproj`, the top-level `Makefile` builds the same thing:

- Sources: the kext target's six files (`AudioDevice`, `AudioEngine`, `I2C`, `misc`, `ak_codec`,
  `Phase88`), plus `SampleAudioClip.cpp`, which the project built as the static
  `libAudioFloatLib` and linked into the kext.
- `build-support/kmod_info.c` replaces the `_info.c` Xcode generates (the `kmod_info` struct).
- SDK 14.5, `-mmacosx-version-min=14.0`, `-arch x86_64` only, `-mkernel -fapple-kext`, no
  exceptions or RTTI, unsigned (no codesign step at all).
- `make` builds `build/Envy24PCIAudioDriver.kext`.

### Changes needed to compile

Upstream source wasn't changed at all. The only fixes are build settings:

1. `-Idriver`: `misc.h` uses `#include <DriverData.h>`, which Xcode's header map used to resolve.
2. `-Wno-deprecated-declarations -Wno-#warnings`: IOAudioFamily and `IOPCIDevice` are deprecated
   but still present on Intel Sonoma. We're deliberately staying on them, and the warnings drown
   out everything else.

The remaining warnings are left as-is (Phase 2 doesn't change logic): missing `override`,
narrowing `int` to `unsigned char` in `misc.cpp`/`ak_codec.cpp`, shifts of negative values in
`misc.cpp`, and a broken header guard in `misc.h` (`AHI_Drivers_misc_h` vs. the `#define` after it).

`ld` warns that `libkmod.a` was built for 14.5 while we target 14.0. That's harmless; the host
runs 14.8.9.

### Validation (no load)

`kextutil -n -t` no longer works on Sonoma (`-n is not a supported kmutil mode`). What was run instead:

| Check | Result |
|---|---|
| `file` | `Mach-O 64-bit kext bundle x86_64`; `_kmod_info`, `__start`, `__stop` present |
| `plutil -lint Info.plist` | OK |
| `kmutil print-diagnostics -p` | **Dependencies: OK**. Also reports ownership 501:20 instead of 0:0 and a bad code signature, both expected for an unsigned build in a user folder and irrelevant to OpenCore injection |
| `kextlibs` | Reports 487 symbols missing, but that's a false result: on Sonoma library kexts have no binary on disk (they're only in the kernel collection), so `kextlibs` can't see them |
| `kmutil create` (scratch aux collection) | Not possible: needs a KDK matching build 23J631. Not installed |
| Imports vs. XNU `xnu-10063.141.1/config/*.exports` | All non-IOAudio, non-IOPCIFamily imports are exported (`kernel_task` in Mach, `IOMallocContiguous`/`IODelay`/`IOSleep` in IOKit, stack protector in Libkern). `_RESERVED*` vtable slots are handled by the kext linker |
| IOAudioFamily | Loaded on host at 540.3, compatible version 1.0, so the `OSBundleLibraries` requirement of 1.1 is satisfied. Its symbols couldn't be checked without a KDK |

`install.sh` and `go` in `driver/` copy the kext into `/System/Library/Extensions`.
**Never run them.** They're kept only because they're part of the upstream import.

## 3. Adding the Delta 66 (Phase 3)

### What the upstream driver already had (a change of plan)

The brief assumed the macOS driver only knew the Phase 88 and that the Delta 66 would be a copy
of the Phase 88 board class with a new GPIO map. That's not the case:

- `card_init()` (`misc.cpp`) reads the EEPROM subvendor and already recognized `0x121432d6`
  as `MAUDIO_DELTA66`, sharing one code path with the Delta 44.
- That path already used the classic Delta 66 codec pins from `delta.h` (data 0x10, clock 0x20,
  CS 0x40/0x80) and the same AK4524 write routine as ALSA (`akm4xxx_write_new` matches
  `snd_ice1712_akm4xxx_write` in `ak4xxx.c` line for line).
- The Phase 88 drives its codecs through an I²C port expander (CS by writing 0x48), which is a
  different method. Copying it would have been the wrong starting point.

So `Delta66.cpp`/`Delta66.h` were made by moving the Delta 66 branch out of the shared Delta
44/66 code and porting the missing parts of ALSA `delta.c` onto it. The layout (Init, lock,
unlock, pin defines in the header) follows `Phase88.cpp` so the two can be compared. The Delta 44
code is unchanged.

`ak4xxx.c` wasn't on the brief's list of reference files, but that's where the ICE1712 GPIO
bit-banging lives; it was fetched into `reference/alsa/`.

### GPIO mapping

Phase 88 uses only three GPIO pins; codec select goes over I²C to an expander (`Phase88.cpp:135`).

| Function | Phase 88 bit | Delta 66 bit | Delta 66 rev E bit | Source (`delta.h`) |
|---|---|---|---|---|
| Sample rate mode (DFS, 1 above 48 kHz) | — | 0x01 | 0x01 | :48 `ICE1712_DELTA_DFS` |
| S/PDIF input status (input) | — | 0x02 | — | :50 `ICE1712_DELTA_SPDIF_IN_STAT` |
| SPI clock (codecs + CS8427) | — | — | 0x02 | :137 `ICE1712_DELTA_66E_CCLK` |
| CS8403 status clock | — | 0x04 | — | :55 `ICE1712_DELTA_SPDIF_OUT_STAT_CLOCK` |
| SPI data in (from CS8427) | — | — | 0x04 | :138 `ICE1712_DELTA_66E_DIN` |
| CS8403 status data | — | 0x08 | — | :60 `ICE1712_DELTA_SPDIF_OUT_STAT_DATA` |
| SPI data out (codecs + CS8427) | — | — | 0x08 | :139 `ICE1712_DELTA_66E_DOUT` |
| Codec read/write direction | 0x08 (`PHASE88_RW`) | — | — | — |
| Codec serial data | 0x10 (`PHASE88_DATA`) | 0x10 | (0x08, shared DOUT) | :90 `ICE1712_DELTA_CODEC_SERIAL_DATA` |
| Codec serial clock | 0x20 (`PHASE88_CLOCK`) | 0x20 | (0x02, shared CCLK) | :92 `ICE1712_DELTA_CODEC_SERIAL_CLOCK` |
| CS8427 chip select (low = selected) | — | — | 0x10 | :140 `ICE1712_DELTA_66E_CS_CS8427` |
| AK4524 chip A select | I²C expander | 0x40 | 0x20 | :95 `ICE1712_DELTA_CODEC_CHIP_A`, :141 `..._66E_CS_CHIP_A` |
| AK4524 chip B select | I²C expander | 0x80 | 0x40 | :96 `ICE1712_DELTA_CODEC_CHIP_B`, :142 `..._66E_CS_CHIP_B` |

Codec write protocol (both revisions, `akm_delta44_priv` delta.c:531 and `akm_delta66e_priv`
delta.c:508): `caddr = 2`, `cif = 0`, `cs_mask == cs_addr` = the chip's select bit. That means CS
is driven low, 16 bits are clocked out MSB first on the rising edge, then CS goes high to latch.

### What was ported from ALSA

| ALSA (`delta.c`) | macOS (`Delta66.cpp`) |
|---|---|
| `delta_ak4524_lock` :247 / `delta66e_ak4524_lock` :274 | `Delta66_SetupCodec` sets the per-chip select once (we keep one `akm_codec` per chip); `Delta66_ak4524_lock` saves GPIO dir and mask like `snd_ice1712_save_gpio_status` |
| `snd_ice1712_restore_gpio_status` (`ice1712.h`) | `Delta66_ak4524_unlock` restores dir and mask, **not** data |
| `delta_ak4524_set_rate_val` :322 | `Delta66_set_rate_val`: only acts if DFS changes; resets codecs (reg 1 = 0), flips DFS, releases reset (reg 1 = 3) and rewrites regs 4-7 |
| `ak4524_reset` (`ak4xxx-adda.c`) | `Delta66_ak4524_reset`, using a new per-codec register cache (`akm_codec.regs`, filled in `akm4xxx_write`) |
| rev E detection :624-626 | `card_init`: Delta 66 ID + EEPROM GPIO dir `0xfb` → `MAUDIO_DELTA66E` |
| "initialize the SPI clock to high" :665 | `Delta66_Init` sets GPIO 0x02 high on both revisions |

`set_rate_val` is called from `performFormatChange` after the rate register is written, the same
order as ALSA. With only 44.1/48 kHz offered, it just makes sure DFS is clear.

Locking: ALSA takes `gpio_mutex`. Here, every codec write happens either in `card_init` or under
the IOAudioDevice command gate (control changes, format changes), and the interrupt filter never
touches GPIO, so no extra lock was added.

### EEPROM

The macOS driver reads the EEPROM like ALSA (`ice1712.c:2234-2310`), with the same layout:
config bytes 0x60-0x63 from data[0-3], GPIO mask/state/dir from data[4-6] (`ICE_EEP1_GPIO_*`).
Nothing is hard-coded for the Delta 66. Two fixes, following ALSA:

- the size byte is now bounded (<6 → 32, >32 → error). Upstream read up to 249 bytes into
  `eeprom[128]` on the stack.
- GPIO mask, state and direction are logged at load so the board revision can be confirmed from
  `log show`.

ALSA writes mask, then dir, then data; upstream writes mask, data, dir. Left as upstream: all
three are written before any codec access.

### S/PDIF

- **Classic Delta 66:** the transmitter is a CS8403 programmed by bit-banging GPIO 0x04/0x08.
  Upstream did write it (`InitDigitalOut`), and ALSA does too. For this pass it's **disabled**:
  `InitDigitalOut` is no longer called for the Delta 66, and the DMA S/PDIF slots (8-9) aren't
  exposed.
- **Rev E:** has a CS8427 on the shared SPI lines with select 0x10. The upstream CS8427 code is
  written for the Audiophile pin map. It **isn't used** in this pass. `Delta66_Init` drives 0x10
  high so the CS8427 ignores codec traffic on DOUT/CCLK (the state ALSA's
  `ap_cs8427_codec_deassert` leaves it in).

### Channel mapping

The ICE1712 multi-track DMA always moves full frames: playback 10 × 32-bit slots (PSDOUT0-3 L/R,
then S/PDIF L/R), capture 12 slots (PSDIN0-3 L/R, S/PDIF L/R, digital mixer L/R). Upstream gave
Core Audio the DMA buffers directly, so every card showed up as 10 out / 12 in; `NumChannels` was
set but never read.

Now Core Audio gets shadow sample buffers sized for the exposed channel count, and
`clipOutputSamples`, `convertInputSamples` and `eraseOutputSamples` read and write the DMA
buffers with the 10/12-slot stride. Core Audio channel *n* is DMA slot *n*:

| Core Audio | DMA slot | Codec |
|---|---|---|
| Out 1/2 | PSDOUT0 L/R (0, 1) | AK4524 A DAC |
| Out 3/4 | PSDOUT1 L/R (2, 3) | AK4524 B DAC |
| In 1/2 | PSDIN0 L/R (0, 1) | AK4524 A ADC |
| In 3/4 | PSDIN1 L/R (2, 3) | AK4524 B ADC |

This matches ALSA, where Delta 66 PCM channels 0-3 are DACs/ADCs 1-4. Other cards keep 10/12.

### Where ALSA and the macOS driver disagreed

| Item | Upstream macOS | ALSA | Now |
|---|---|---|---|
| MT 0x30 PSDOUT route | `0x0101`: outputs 1/2 from the digital monitor mixer | reset value 0: each PSDOUT plays its DMA slots | ALSA (Delta 66 only) |
| MT 0x34 capture route | `0x10` loopback | 0 | ALSA (Delta 66 only) |
| DFS / rate change | never touched | `delta_ak4524_set_rate_val` | ALSA |
| Rev E | treated as classic (wrong pins) | detected via GPIO dir 0xfb | ALSA |
| EEPROM size check | none | 6..32 | ALSA |
| CS8403 init on classic | yes | yes | neither, for now (brief: digital off) |
| AK4524 regs 4-7 at init | ADC 0x80 (0 dB), DAC 0x7E | 0 (muted), then restored from the ALSA mixer | **upstream**: macOS has no mixer-restore step, so ALSA's values would leave the card silent |

### Other fixes found along the way

- Clip: `float * 2147483647.0` rounds to 2^31 and overflows `SInt32` at exactly +1.0, giving a
  full-scale negative sample. It now scales to 24 bits and shifts left by 8.
- Stream format now says what the hardware does: 24 bits in 32-bit little-endian slots. The
  advertised 16-bit format was dropped, since the clip routine always wrote 32-bit samples.
- `performFormatChange` reset the hardware to 44.1 kHz on any change that had no new rate.
- Delta 66 offers 44.1 and 48 kHz only for now.

### Info.plist

The single `IOPCIMatch 0x17121412` is replaced by two personalities, both `IOPCIPrimaryMatch
0x17121412`:

- `Envy24 Delta 66`: `IOPCISecondaryMatch 0xd6321412` (subsystem 1412:d632 from section 1).
  This matches both the classic and the rev E; `card_init` tells them apart.
- `Envy24 Phase 88`: `IOPCISecondaryMatch 0x1151153b 0x1115153b 0x1125153b 0x117c153b`. These
  are upstream's Terratec EEPROM IDs converted with ALSA's swab16 rule. They can't match this card.

Other ICE1712 boards (Audiophile, 1010LT, Delta 44, ...) no longer match at all.

### Build check

`make clean && make` has no errors. `kmutil print-diagnostics`: **Dependencies: OK** (plus the
expected unsigned/ownership notes). The new imports (`IOMalloc`, `IOFree`, `bzero`) are in the
XNU exports (`IOMalloc` as an alias of `IOMalloc_external`).

## Open items

See section 5 for S/PDIF out, 88.2/96 kHz, the volume scale and the device name.

- S/PDIF input (CS8414 on the classic board): slots 8/9 of capture are not exposed. Recording a
  digital source cleanly also needs the card clocked from the input (MT rate register bit 0x10).
- Rev E support (CS8427) is written but untested; the card here is a classic Delta 66.
- Sample rate is written even while DMA is running. ALSA skips the change in that case
  (`snd_ice1712_set_pro_rate`).
- OpenCore-only install would need a PCI-only kext + Core Audio HAL plug-in. Not planned.

## 4. Load and test (Phase 4)

### OpenCore injection doesn't work for this kext on Sonoma

OpenCore `Kernel > Add` injects into the boot kernel collection, and an injected kext can only
link against kexts in that collection. `kmutil inspect` on this host shows IOPCIFamily in the
boot collection but **IOAudioFamily only in the system collection**. OpenCore would fail to link
the kext and skip it. Nothing was written to the EFI partition (`SONOMAEFI`, disk1s1) and
`config.plist` is untouched.

Options looked at and rejected:
- injecting a copy of IOAudioFamily (from a KDK) via OpenCore: AppleHDA, used by AppleALC for the
  onboard ALC280, is pre-linked against the system collection's copy, so a second copy risks
  breaking onboard audio or panicking.
- splitting into a PCI-only kext (OpenCore-injectable) plus a Core Audio HAL plug-in: viable,
  but it's a rewrite of the audio side. Deferred until the hardware code has been proven on the card.

### Install via /Library/Extensions (test only)

**Rule waiver:** the user approved breaking the brief's "never install to `/Library/Extensions`"
and "don't touch kext caches" rules for this test (2026-10-25). Third-party kexts on Sonoma load
from the auxiliary kernel collection in `/Library/KernelCollections`. No SIP, NVRAM or OpenCore
change is involved; SIP already allows unsigned kexts (`csr-active-config` 03080000).

Kext installed: `build/Envy24PCIAudioDriver.kext` from commit `74a3ee5`, executable SHA-256
`e73dde23ac80237e1da88d608a16e684d2e33cbff5b79a922f70fd1fc6fbf5fa`.

Steps (run by the user in Terminal):

    sudo cp -R ~/src/envy24-delta66/build/Envy24PCIAudioDriver.kext /Library/Extensions/
    sudo chown -R root:wheel /Library/Extensions/Envy24PCIAudioDriver.kext
    sudo chmod -R go-w /Library/Extensions/Envy24PCIAudioDriver.kext
    sudo kmutil load -p /Library/Extensions/Envy24PCIAudioDriver.kext
    # approve in System Settings > Privacy & Security, let it rebuild, then reboot

### Revert

    sudo rm -rf /Library/Extensions/Envy24PCIAudioDriver.kext
    sudo kmutil clear-staging
    sudo kmutil create -n aux --boot-path /System/Library/KernelCollections/BootKernelExtensions.kc --system-path /System/Library/KernelCollections/SystemKernelExtensions.kc
    # reboot

**If it panics at boot:** boot in safe mode (add `-x` to boot-args from the OpenCore picker,
or hold Shift). Safe mode skips the auxiliary kernel collection, so the kext won't load. Then
collect `/Library/Logs/DiagnosticReports/*.panic` and run the revert steps. Don't reboot again
until the panic has been looked at.

### Test 1 (2026-10-25 22:04): loads, engine fails on DMA allocation

Installed as above, approved, rebooted. No panic.

- `kextstat`: `com.audio-evolution.driver.Envy24 (0.22.0)` loaded; attached through the
  `Envy24 Delta 66` personality (`IOPCISecondaryMatch 0xd6321412`).
- Kernel log (`/usr/bin/log show --predicate 'sender == "Envy24PCIAudioDriver"'`; note that plain
  `log` in zsh is a shell builtin):

      M-Audio Delta 66 detected!
      Envy24: EEPROM subvendor 121432d6, GPIO mask 02, state c0, dir fd
      Envy24: Delta 66 initialized, GPIO dir fd mask 02 data e2
      Failed to allocate memory within the first 256MB address space!
      Envy24: couldn't allocate DMA output buffer below 256 MB
      Something failed in the engine!

**Board revision settled: classic Delta 66** (GPIO dir `0xfd`, not rev E's `0xfb`). The S/PDIF
path, when it's added, is the CS8403. After init, GPIO data `0xe2` = both codec selects idle
high (0x80|0x40), codec clock high (0x20), 0x02 high (an input), DFS clear. That's the state
ALSA leaves.

The engine didn't start: 160 KB of physically contiguous memory below 256 MB wasn't available.
This is exactly the case where upstream would have fallen back to an unrestricted buffer and let
the card DMA to a truncated address. No audio device appeared in Core Audio.

Fix: try 4096, then 2048, then 1024 frames per buffer (playback 160/80/40 KB, capture
192/96/48 KB), and log the size and physical addresses that were used.

### Test 2 (2026-10-25 22:10): smaller buffers don't help

Same result at 4096, 2048 and 1024 frames, all three failing in the same millisecond:

    Failed to allocate memory within the first 256MB address space!   (x3)
    Envy24: couldn't allocate DMA buffers below 256 MB, even at 1024 frames

Why, from xnu-10063 source:
- With no IOMMU, `IOBufferMemoryDescriptor::inTaskWithPhysicalMask(..., 0x0FFFFFFC)` passes the
  limit unchanged to `kmem_alloc_contig(max_pnum = 0xFFFF)` (`IOBufferMemoryDescriptor.cpp`,
  `IOLib.cpp IOKernelAllocateWithPhysicalRestrict`).
- `vm_page_find_contiguous` (`vm_resident.c`) scans `vm_pages[]`, which on x86 is in increasing
  physical order. It stops at the first page above `max_pnum` and wraps once to index 0. So a
  failure means there's no run of even 10 free or stealable pages anywhere below 256 MB.
- Likely cause: the boot loader places the kernel and the kernel collections (system 360 MB,
  boot 63 MB, aux 9 MB) in low physical memory, where they stay wired. That fills the region
  the ICE1712 can reach. Not proven: the booter's `memory-map` entries are gone by the time
  macOS is running.

Smaller buffers won't fix this. The DMA memory has to be set aside before macOS fills low memory,
or reached through an IOMMU.

### Test 3 setup: enable VT-d

User decision (2026-10-25): try VT-d so the IOMMU can map the DMA buffers below 256 MB.
VT-d was already enabled in the BIOS. The only thing disabling it was OpenCore's
`Kernel > Quirks > DisableIoMapper = true` (no DMAR drop in `ACPI > Delete`;
`DisableIoMapperMapping` is already `false`).

Change on SONOMAEFI (disk1s1), `EFI/OC/config.plist` line 610: `DisableIoMapper` `true` →
`false`. One-line diff, `plutil -lint` OK.
Backup: `EFI/OC/config.plist.bak-20261025-2220-pre-vtd`.

**Revert:** copy the backup over `config.plist`. If macOS won't boot, do it from the other EFI
partition or another OS, or boot with the previous config.

Driver: commit `91a1d56` gets the card address from an `IODMACommand` (28-bit spec). With VT-d
off, it behaves as before.

Watch after reboot: Wi-Fi (OCLP AirPortBrcmNIC), Ethernet (IntelMausi), USB, sleep.
These are the devices most likely to be affected by VT-d on this machine.

### Test 3 (2026-10-25 22:23): VT-d on, device appears

- `AppleVTD` loaded (`IOMapper` active). No VT-d faults in the log. Ethernet (en0) and Wi-Fi
  (en1) have addresses and pass traffic; USB enumerates.
- Driver:

      Envy24: DMA buffer 28000 bytes, device address 280000
      Envy24: DMA buffer 30000 bytes, device address 2c0000
      Envy24: DMA buffers 4096 frames, playback device address 280000, capture 2c0000

  Full-size buffers on the first try, at IOMMU addresses inside the 28-bit window.
- Engine and level controls created. `system_profiler SPAudioDataType`: **Envy24, 4 in / 4 out,
  44100 Hz**. macOS made it the default input automatically.
- `say -a Envy24 ...`: engine goes to state 1 (running) with one client; the device moves to its
  active power state.

### Test 4 (2026-10-26): channels and rates

Test tool: `tools/devtest.swift` (`build/devtest`; plays or records the Envy24 device through an
AudioQueue, and sets its nominal rate the same way Audio MIDI Setup does).

| Test | Result |
|---|---|
| Speech via `say -a Envy24` | heard clean on outputs 1/2 (user) |
| `devtest play`: n beeps on output n | **outputs 1-4 correct**, each on its own output (user) |
| Inputs 1/2 with a source | working (user) |
| Inputs 3/4, idle | live noise floor about -83 dBFS (codec B ADC running), not digital zero |
| Inputs 3/4 with a source | **not yet tested** |
| 44.1 kHz capture rate | 44116 frames/s measured (+0.04%) |
| 48 kHz | device rate change works; driver writes rate code 0 (48 kHz); 47731 frames/s measured, within one 4096-frame buffer over 9 s |

DFS stays clear at both rates, as expected for rates up to 48 kHz.


## 5. Second pass: name, levels, 88.2/96 kHz, S/PDIF out

Each is a separate commit.

**Device name** (`92ea915`). Set after `card_init()` has identified the board: "M-Audio Delta 66",
short name "Delta 66", manufacturer "M-Audio". Core Audio's device UID comes from the engine
class and PCI location, not the name, so existing settings should carry over.

**Level controls** (`79aa831`). The AK4524 level registers are not linear in dB. Ported ALSA's
`vol_cvt_datt` table and `put_ak_reg` (`ak4xxx-adda.c`) into `ak4524_level_register()`:

| Control | Range | dB | Register |
|---|---|---|---|
| Output 1-4 (regs 6/7) | 0..127 | -63.5..0, 0.5 dB steps | `vol_cvt_datt[v]` |
| Input 1-4 (regs 4/5) | 0..164 | -63.5..+18.5, 0.5 dB steps | `v < 128`: `vol_cvt_datt[v]`; else `v + 1` (IPGA) |

0 is mute. All levels start at 127 (0 dB), and `Delta66_Init` writes the codecs to match.
`Parm::conversion` defaults to `PARM_RAW`, so other boards are unchanged.

**88.2 / 96 kHz** (`12afe67`). Rate codes 11 and 7 (ALSA `snd_ice1712_set_pro_rate`). DFS
handling with codec reset was already in `Delta66_set_rate_val`.

**S/PDIF output, classic Delta 66** (`602a3f8`).
- Core Audio outputs 5/6 → DMA slots 8/9 (new `playSlot[]` map in the engine). The device now
  has 6 outputs, 4 inputs.
- CS8403 status byte: ALSA default `0x31` (consumer, no emphasis, PCM), sent at init.
- On every rate change, ALSA `delta_setup_spdif`: consumer rate bits `0x02` = 48 kHz,
  `0x04` = 32 kHz, `0x00` = 44.1 kHz *and everything else*. So at 88.2/96 kHz the stream is
  sent with the 44.1 kHz code; receivers that trust the code over the actual clock may refuse it.
  This is ALSA's behaviour too.
- MT 0x32 (S/PDIF out route) written to 0, ALSA's default (PCM).
- `spdif_write()` (upstream) is identical to `snd_ice1712_delta_cs8403_spdif_write()`.
- Rev E keeps 4 outputs until its CS8427 is set up.

`devtest play` now covers every output of the device (n beeps on output n, up to 6).

## 6. S/PDIF input and clock source (classic Delta 66)

- Capture DMA slots 8/9 carry the S/PDIF receiver (CS8414, needs no setup), exposed as
  inputs 5/6, so the classic board is now 6 in / 6 out. Rev E stays 4 in (CS8427 not set up).
- "Clock Source" selector (Internal / S/PDIF), ALSA `snd_ice1712_pro_internal_clock_put()`:
  S/PDIF sets `MT_SPDIF_MASTER` (0x10 in MT_SAMPLERATE) and calls `set_rate_val(0)` (no DFS
  change); Internal rewrites the rate register without the bit. `performFormatChange` keeps the
  bit while S/PDIF is master. Rate-bit/DFS follow the incoming signal's rate, so the
  sample rate set in Core Audio must match the source.
- Status: built. User reports S/PDIF support running and 96 kHz playback working (2026-10-28,
  user report, no measurements; which S/PDIF direction and whether the Clock Source selector
  was exercised were not stated). Still to confirm: inputs 5/6 with a live source, the Clock
  Source switch, and 88.2 kHz playback.
