# awmmc — Windows SD/MMC miniport for the Allwinner H616/H618 SMHC (Orange Pi Zero 3, SMHC0)

An sdport miniport (`awmmc.sys`, ACPI `AWMC0001`) adapting Microsoft's SDHC miniport sample to the Allwinner SMHC register set.
Targets mu-silicium UEFI, which leaves the SD card powered and clocked at 24 MHz.

**Status: written and unit/flow-tested against an emulator on a host; never built with the WDK and never run on hardware.**

| | |
|---|---|
| `driver/` | source, INF source, VS/WDK project |
| `acpi/sdc0.asl` | the ACPI node (unchanged from the task) with notes on what a future version would need |
| `docs/01-reference-summary-and-architecture.md` | phase 1/2: reference summary and the original architecture proposal (partly superseded) |
| `docs/02-design.md` | **design, register/flow mapping with source citations, unverified items, limitations** |
| `docs/03-bringup-test-plan.md` | UEFI-side checks, WinDbg trace points, enumeration → read → write |
| `docs/04-build-and-install.md` | build, test signing, install |
| `tests/` | host-side mock + register-level emulator + tests (`make -C tests check`) |

Scope of this version: 400 kHz identification, 24 MHz normal speed, 1/4-bit bus, PIO (default) or IDMAC scatter/gather DMA, 3.3 V only, no hot-plug, no UHS, no crash-dump.
