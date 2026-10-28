# M-Audio Delta 66 for macOS Sonoma

A Core Audio driver for the M-Audio Delta 66 PCI card (ICE1712 / Envy24), built on the
IOAudioFamily-based [vrudikov/Envy24](https://github.com/vrudikov/Envy24) driver (originally
Audio Evolution's, written for the Terratec Phase 88). Developed on a Dell Precision T3610 (OpenCore, macOS Sonoma 14, x86_64). The ALSA driver (`reference/alsa/`) is the
source of truth for registers, GPIO bits and codec sequences.

## Status

| Feature | State |
|---|---|
| Analog out 1-4, analog in 1-4 | Working (outputs 1-4 and inputs 1/2 verified by ear; inputs 3/4 only seen at the noise floor) |
| 44.1 / 48 kHz | Working (measured) |
| 88.2 / 96 kHz | Capture rate measured correct; playback at 96 kHz confirmed working by the user |
| 0.5 dB level controls, device name | Built and loaded; sliders not yet checked in Audio MIDI Setup |
| S/PDIF out (outputs 5/6) | Built; user reports S/PDIF running (direction not specified) |
| S/PDIF in (inputs 5/6), Clock Source selector | Built; needs a live S/PDIF source to verify |
| Delta 66 rev E | Analog only (4 in / 4 out); CS8427 not set up. Untested, no hardware |
| Hardware monitoring (digital mixer) | Not implemented |

The card shows up as **M-Audio Delta 66**, 6 in / 6 out on the classic board.
Details, the GPIO table, differences from ALSA and open items are in [PORTING.md](PORTING.md).

## Requirements

- Intel Mac, macOS Sonoma (built against the 14.5 SDK, deployment target 14.0).
- **VT-d on, and `DisableIoMapper` off.** The ICE1712 can only DMA below 256 MB. After boot there
  is no free contiguous memory that low, so the driver maps its buffers through the IOMMU
  (`IODMACommand`, 28-bit spec).
- Command Line Tools (no Xcode needed).

## Build

    make                      # -> build/Envy24PCIAudioDriver.kext (x86_64, unsigned)
    swiftc -O tools/devtest.swift -o build/devtest

## Install and load

OpenCore cannot inject this kext: IOAudioFamily is only in the system kernel collection, not
the boot one. It loads from the auxiliary kernel collection instead:

    sudo cp -R build/Envy24PCIAudioDriver.kext /Library/Extensions/
    sudo chown -R root:wheel /Library/Extensions/Envy24PCIAudioDriver.kext
    sudo chmod -R go-w /Library/Extensions/Envy24PCIAudioDriver.kext
    sudo kmutil load -p /Library/Extensions/Envy24PCIAudioDriver.kext

Approve it in System Settings if asked, then reboot. To remove it, delete the kext from
`/Library/Extensions` and reboot. Do not use `driver/install.sh` or `driver/go`; they copy into
`/System/Library/Extensions`.

## Testing

`build/devtest` drives the device through Core Audio:

    devtest play [rate]            n beeps on output n, for every output
    devtest record [secs] [rate]   peak level per input (5/6 = S/PDIF), plus measured rate
    devtest rate [hz]              show or set the nominal sample rate

Driver messages: `/usr/bin/log show --last boot --predicate 'sender contains "Envy24"'`

## Layout

    driver/          the driver (upstream import plus Delta 66 changes: Delta66.cpp/.h, ak_codec, ...)
    build-support/   kmod_info.c for the Command Line Tools build
    reference/alsa/  ALSA sources from Linux v6.12 used as reference
    tools/           devtest
    PORTING.md       card ID, GPIO map, ALSA differences, test log, open items

## License

GPL-2.0-or-later; see [LICENSE](LICENSE). The Delta 66 code is ported from the ALSA ICE1712
driver (GPL-2.0-or-later), and the files in `reference/alsa/` keep their original SPDX headers.
The upstream Envy24 driver this builds on has no license file of its own, so its original
copyright and terms remain with its authors.
