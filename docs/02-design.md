# AWMC0001 — design: sdport miniport for the Allwinner SMHC (H616/H618), SMHC0

Status: **implemented, not yet run on hardware, not yet built with the WDK.**
What *has* been checked is described in §9. Every item that no reference source
confirms is marked **UNVERIFIED** and is collected in §8.

Reference tags used throughout (all fetched from `master`, 2026-10-04):

| Tag | File |
|---|---|
| `[UB-H]` | u-boot `drivers/mmc/sunxi_mmc.h` (struct `sunxi_mmc`, `SUNXI_MMC_*`) |
| `[UB-C]` | u-boot `drivers/mmc/sunxi_mmc.c` |
| `[UB-CCU]` | u-boot `arch/arm/include/asm/arch-sunxi/clock_sun50i_h6.h` |
| `[LX]` | Linux `drivers/mmc/host/sunxi-mmc.c` |
| `[LX-DT]` | Linux `arch/arm64/boot/dts/allwinner/sun50i-h616.dtsi`, `sun50i-h618-orangepi-zero3.dts` |
| `[MS]` | Microsoft `Windows-driver-samples/sd/miniport/sdhc/sdhc.c`, `sdhc.h` |
| `[DW]` | `AistopGit/dwcmshc` `dwcmshc.cpp`, `dwcmshc.h` (DesignWare-MMC miniport for sdport) |

## 1. Platform facts and decisions

| Question | Answer | Source |
|---|---|---|
| SMHC0 base / size | `0x04020000` / `0x1000` — matches the ACPI node | `[LX-DT]` `mmc0: mmc@4020000` |
| SMHC0 interrupt | SPI 35 → GIC INTID 67, level high — matches the ACPI node | `[LX-DT]` `interrupts = <GIC_SPI 35 IRQ_TYPE_LEVEL_HIGH>` |
| Controller variant | `allwinner,sun50i-h616-mmc` (`idma_des_size_bits = 16`, `idma_des_shift = 2`, `can_calibrate`, `mask_data0`, `needs_new_timings`) | `[LX]` `sun50i_h616_cfg`; `[LX-DT]` |
| Board DT | Orange Pi Zero 3 is `sun50i-h618-orangepi-zero3.dts` (H618); `sun50i-h616-orangepi-zero3.dts` does not exist. Same SMHC. | Linux tree |
| Card detect | `broken-cd` — schematic wires PF6 through an inverter "but it just doesn't work" | `[LX-DT]` zero3 `&mmc0` |
| Card supply | `vmmc-supply = <&reg_dldo1>` (AXP313A, I²C PMIC) | `[LX-DT]` zero3 `&mmc0` |
| Bus width | 4-bit only (pins PF0–PF5 = CLK, CMD, D0–D3) | `[LX-DT]` `mmc0_pins` |
| What firmware leaves | Per the project owner: **mu-silicium UEFI leaves the card powered and clocked at 24 MHz**. (The OrangePiZero3Pkg repo named in the task contains no SMHC code and its DSDT has no SDC0; it is not what runs.) | task owner; `OrangePiZero3Pkg/AcpiTables/Dsdt.asl` |

### Option (a)/(b)/(c) — clock, pinmux, power, CD

**Chosen: (a), with no CCU access in this version.**

* Pinmux, bus clock gate/reset, module clock and card VDD are assumed configured by firmware (it booted from
  or initialised this very slot, and the owner states the card is left powered at 24 MHz).
* The driver therefore never touches the CCU. The card clock is derived from the firmware's module clock
  (default **24 MHz**, registry `ModuleClockHz`) using only the SMHC-internal divider `CLKCR[7:0]`:
  400 kHz identification and 24 MHz normal speed (≤ 25 MHz, SD default speed). **50 MHz High-Speed is
  therefore not offered** (`Supported.HighSpeed = 0`); UHS-I is not offered either (no switchable 1.8 V).
* Card detect: `GetCardDetectState` returns TRUE, `GetWriteProtectState` FALSE (microSD has no WP).
  No hot-plug; an empty slot makes sdport's enumeration fail on CMD0/CMD8 timeouts.
* Power/voltage callbacks accept 3.3 V (and "off"/"set power") as no-ops; 1.8 V signaling is refused
  (`STATUS_NOT_SUPPORTED`). Consequence: **sdport cannot power-cycle the card**; CMD0 is what resets it.

### ACPI changes required

**None** for this version — the node in the task is used unchanged (`acpi/sdc0.asl`). Future needs (not used by the
driver today): two small `Memory32Fixed` windows for the CCU to go beyond 24 MHz; a GPIO and a PMIC path for
card detect and 1.8 V. See the comment block in `acpi/sdc0.asl`. (CCU base `0x03001000` is confirmed by `[LX-DT]`; the MMC gate/reset bit positions are not, U10.)

## 2. Source layout

| File | Contents |
|---|---|
| `driver/smhc_regs.h` | register offsets and bit fields, each tagged with its source |
| `driver/smhc_core.h` | pure logic shared with the host tests: divider choice, SMHC→sdport event/error mapping, R2 formatting, IDMAC chain builder |
| `driver/smhc.h` | extension, tunables, logging, prototypes |
| `driver/smhc.c` | `DriverEntry` and the sdport callback table (one function per `[MS]` callback) |
| `driver/smhc_hw.c` | reset, clock, width, command issue, PIO and IDMAC data paths, completion |
| `driver/awmmc.inx`, `awmmc.rc`, `awmmc.vcxproj`, `../awmmc.sln` | INF source (`ACPI\AWMC0001`), version resource, VS/WDK project (ARM64) |
| `tests/` | host-side unit tests and a register-level emulator (see `tests/README.md`) |

## 3. sdhc-sample concept → SMHC equivalent

| sdport/SDHCI concept (`[MS]`) | SMHC implementation | Reference |
|---|---|---|
| `SdhcResetHost` software reset (all/cmd/dat) | `GCTRL` = SOFT\|FIFO\|DMA reset (bits 0–2), poll until clear (250 ms); then rewrite everything the driver owns | `[LX]` `sunxi_mmc_reset_host`, `[UB-C]` `sunxi_mmc_reset` |
| `SdhcSetClock`, clock-control divider | gate card clock; write `CLKCR[7:0]`; `NTSR[31]` new-timing; `SAMP_DL = SW_EN` (delay 0); ungate. Each gate change is `CLKCR` write → `CMD = START\|UPCLK_ONLY\|WAIT_PRE_OVER` → poll START → clear `RINT`, with `MASK_DATA0` (CLKCR[31]) set around it | `[LX]` `sunxi_mmc_oclk_onoff`, `sunxi_mmc_clk_set_rate`, `sunxi_mmc_calibrate`; `[UB-C]` `mmc_update_clk`, `mmc_config_clock` |
| `SdhcSetBusWidth` (HOST_CONTROL bits) | `WIDTH` = 0 (1-bit) / 1 (4-bit) | `[LX]` `sunxi_mmc_set_bus_width`, `SDXC_WIDTH*` |
| `SdhcSetVoltage` / power control | not available → accepted, no effect | §1 |
| `SdhcSetHighSpeed` / UHS / tuning | not offered | §1 |
| `SdhcSendCommand` (COMMAND + TRANSFER_MODE + ARGUMENT) | `ARG`, `BLKSZ`, `BCNTR` (bytes), single `CMD` word | `[UB-C]` `sunxi_mmc_send_cmd_common`, `[LX]` `sunxi_mmc_request` |
| response regs (R2 shifted by SDHCI) | `RESP0..3`; R2: drop the lowest byte (CRC7\|end) | `[DW]` `MshcSlotGetResponse`; `[LX]` `sunxi_mmc_finalize_request` |
| interrupt status/enable/ack (`SdhcSlotInterrupt`, `ToggleEvents`, `ClearEvents`) | `MINT`/`RINT`/`IMASK`; write-1-to-clear; `IDST`/`IDIE` for the IDMAC | `[LX]` `sunxi_mmc_irq`; `[DW]` `MshcSlotInterrupt` |
| ADMA2 table + `SYSADDR` (`SdhcCreateAdmaDescriptorTable`) | IDMAC chain descriptors + `DLBA` (address >> 2) + `DMAC` | `[LX]` `sunxi_mmc_init_idma_des`, `sunxi_mmc_start_dma`; `[DW]` `MshcCreateIdmacDescriptorTable` |
| buffer data port PIO (`SdhcReadDataPort`) | `FIFO` register, `GCTRL.ACCESS_BY_AHB`, `STATUS` FIFO level/full | `[UB-C]` `mmc_trans_data_by_cpu` |
| Auto-CMD12 | `CMD.AUTO_STOP`; completion = `RINT.AUTO_COMMAND_DONE` | `[UB-C]`, `[LX]` `sunxi_mmc_request` |
| busy after R1b / writes (Present State DAT0) | `STATUS.CARD_DATA_BUSY` poll; work item at DISPATCH_LEVEL | `[UB-C]` `sunxi_mmc_send_cmd_common`; `[DW]` `MshcCompleteRequest` |
| error after data (abort) | polled CMD12 with `STOP_ABORT` | `[LX]` `sunxi_mmc_send_manual_stop` |
| card detect / WP | constants | §1 |

## 4. Request flows

### 4.1 Command without data
`IssueRequest` → `SmhcSendCommand` builds the `CMD` word (response flags from `ResponseType`; `SEND_INIT_SEQ` for CMD0 only;
`STOP_ABORT` for CMD12), sets `RequiredEvents = CARD_RESPONSE`, clears `RINT`, ORs the needed bits into `IMASK`, writes `ARG` then `CMD`,
returns `STATUS_PENDING`. ISR maps `COMMAND_DONE` → `SDPORT_EVENT_CARD_RESPONSE`; `SmhcRequestDpc` clears the required bit and completes. R1b: if
`STATUS.CARD_DATA_BUSY` is set the completion waits (work item at DISPATCH_LEVEL, spin otherwise).

### 4.2 PIO (bring-up default, registry `TransferMode = 0`)
`ScatterGatherDma` is not advertised so sdport picks PIO. Command phase: FIFO reset, `GCTRL.ACCESS_BY_AHB=1`, `DMA_ENABLE=0`, thresholds
(`FTRGL` RX_TL = min(words,8)−1 / TX_TL = min(words,8), re-programmed per phase so the last request is never
below the watermark), `BLKSZ`, `BCNTR`. Required events: `CARD_RESPONSE | BUFFER_FULL` (read) / `BUFFER_EMPTY` (write).
`StartTransfer` moves everything the FIFO can give/take (reads: `STATUS` level; writes: until `FIFO_FULL`),
acks the data-request bits, re-arms them, and answers `STATUS_MORE_PROCESSING_REQUIRED` until the last word, then waits for `CARD_RW_END`
(already-seen events are remembered in `CurrentEvents` so a `DATA_OVER` that arrived while draining is not lost).
The ISR masks `RX/TX_DATA_REQUEST` when they fire (they cannot be cleared before the FIFO is serviced) — same approach as `[DW]`.

### 4.3 Scatter/gather DMA (`TransferMode = 1`)
Command phase builds the chain in sdport's descriptor buffer (`DmaVirtualAddress`/`DmaPhysicalAddress`), one 16-byte descriptor per ≤ 64 KiB piece of each
SG element, then `GCTRL.DMA_ENABLE`, DMA reset, `DMAC` soft reset, `DLBA`, `IDIE.RX` (reads), `DMAC = FIX_BURST|IDMA_ON`, then the command.
Required events: `CARD_RESPONSE | CARD_RW_END` (+ `DMA_COMPLETE` for reads: Linux does not finish a read before the IDMAC receive
interrupt — `wait_dma` in `sunxi_mmc_irq`). Teardown (IDST clear, `DMAC=0`, DMA reset, FIFO reset) in `SmhcCompleteRequest`.
`StartTransfer` just completes (as `SdhcStartAdmaTransfer`).

### 4.4 Event mapping (`SmhcConvertInterrupts`)
`COMMAND_DONE`→`CARD_RESPONSE`; `DATA_OVER`→`CARD_RW_END` **unless** the command uses AUTO_STOP, in which case only
`AUTO_COMMAND_DONE`→`CARD_RW_END`; `RX/TX_DATA_REQUEST`→`BUFFER_FULL/EMPTY`; `IDST.RI/TI`→`DMA_COMPLETE`.
Errors: `RESP_TIMEOUT`→`CMD_TIMEOUT`, `RESP_CRC`→`CMD_CRC`, `RESP_ERROR`→`CMD_END_BIT|CMD_INDEX`, `DATA_TIMEOUT`, `DATA_CRC`, `END_BIT`→`DATA_END_BIT`,
`FIFO_RUN|HARDWARE_LOCKED|START_BIT`→`SDPORT_GENERIC_IO_ERROR`, `IDST` error bits→`ADMA_ERROR`.
`VOLTAGE_CHANGE_DONE` (bit 10) is deliberately **not** an error although `[UB-H]`/`[LX]` list it in their error mask.
Status mapping to NTSTATUS is the `[MS]` table (timeout→`STATUS_IO_TIMEOUT`, CRC→`STATUS_CRC_ERROR`, …).
After `CMD_TIMEOUT` the DPC waits for the late `COMMAND_DONE` before completing (`[LX]` `sunxi_mmc_irq`).

### 4.5 Errors and reset
Completion with failure keeps the outstanding pointer and sets `NeedStop`. sdport then calls `ResetHost(Cmd)` (returns early if a data transfer failed, as `[DW]`)
and `ResetHost(Dat)` (sends the polled CMD12 once, soft-resets, **rewrites CLKCR, WIDTH, NTSR, SAMP_DL, FTRGL, TMOUT, THLDC and IMASK** — the references do not say whether a soft reset keeps them).
`ResetHost(All)` additionally gates the clock, selects 1-bit and masks all interrupts (sdport re-enables via `ToggleEvents`).

## 5. Clock divider (the one place the references are silent)
`CLKCR[7:0]` is only used by Linux with field 0 (÷1) and field 1 (÷2); u-boot never uses it. For 400 kHz from 24 MHz a larger field is
needed. Two readings are plausible — linear `module/(field+1)` (extrapolating Linux) or DesignWare `module/(2·field)`. The driver uses the **linear field**
`ceil(module/target) − 1`: field **59** gives 400 kHz if the linear reading is right, and 203 kHz if the DesignWare reading is right. Either is within the ≤ 400 kHz
identification limit; being wrong only costs speed, never exceeds the requested clock. At or above the module clock the field is 0 (bypass; both readings agree).
**UNVERIFIED** — measure CLK on hardware (§8, U1).

## 6. DMA and cache (`_CCA = 0`)
* Data buffers: sdport builds the SG list through its DMA adapter, which on a non-coherent system is responsible for cache maintenance. The
  miniport does **no** data-cache operations. **UNVERIFIED** (U7). PIO mode has no cache dependence at all, which is why it is the bring-up default.
* Descriptors live in sdport's per-request descriptor buffer (common buffer, uncached/coherent on non-coherent adapters — **UNVERIFIED**).
  A `DSB SY` (`SmhcDmaBarrier`) separates the last descriptor write from the first doorbell write (`[LX]` ends `sunxi_mmc_init_idma_des` with `wmb()` for the same reason).
* Addresses are limited to 32 bits (`Address64Bit = 0`) and 4-byte alignment (`AlignmentRequirement = 3`) because of the `>> 2` descriptor encoding
  (`[LX]` `sunxi_mmc_map_dma` rejects `offset & 3` / `length & 3`). The 34-bit reach implied by the shift is not relied upon, so memory above 4 GiB on the 4 GB board is bounced by the port.
* Descriptor limits: `[LX]` H616 `idma_des_size_bits = 16` ⇒ 64 KiB per descriptor, 0 encodes 65536; Linux uses one page of descriptors (256). The size of sdport's descriptor buffer is **UNVERIFIED** (U8).

## 7. Capabilities advertised
`SpecVersion 3` (as `[DW]`), 1 outstanding request, block size 512, up to 8192 blocks (`[LX]` `max_blk_count`), `BaseClockFrequencyKhz = ModuleClockHz/1000`, `AutoCmd12`,
`Voltage33V`, driver type B, current limits (as `[DW]`); DMA mode adds `ScatterGatherDma`, `DmaDescriptorSize = 16`; PIO mode sets `UsePioForRead/Write`.
Not offered: 8-bit bus, High-Speed, SDR50/DDR50/SDR104/HS200/HS400, 1.8 V signaling, tuning, AutoCmd23, crash-dump.

## 8. Unverified assumptions and open questions

| # | Item | Why unverified | How to resolve |
|---|---|---|---|
| U1 | `CLKCR[7:0]` divider semantics for fields ≥ 2 | Linux uses only 0/1; u-boot none | scope CLK / time a transfer; see §5 |
| U2 | FIFO at `0x200` | derived from u-boot struct padding; Linux has no PIO; no manual | dump 0x1F0–0x20C in UEFI, do a CMD17 PIO read; compare with a known sector |
| U3 | `FTRGL` field layout (MSIZE[30:28], RX_TL[27:16], TX_TL[11:0]) | value `0x20070008` and its comment are in Linux; field positions follow DW | PIO read of 8/64/512 bytes (test plan stage 3) |
| U4 | RX/TX data-request behaviour (level vs edge; tail below watermark) | not described in references | driver is written to work with either (drains the FIFO per event, re-programs the watermark); verify no hang |
| U5 | whether `GCTRL` soft reset clears `CLKCR`/`WIDTH`/`NTSR`/`IMASK` | not stated | driver rewrites all of them regardless; check by reading back after an error injection |
| U6 | `ACCESS_BY_AHB` / `DMA_ENABLE` interplay when mixing PIO and DMA | Linux is DMA-only, u-boot PIO-only | driver sets/clears them per transfer; verify both modes back to back |
| U7 | sdport/HAL cache maintenance of data buffers on `_CCA=0` | sdport.h and its DMA setup not available | run DMA mode with `TransferMode=1`, compare SHA-256 of large files; if corrupt, test with uncached buffers |
| U8 | size and cacheability of sdport's descriptor buffer | not documented in the sample | check `NumberOfElements` vs buffer in a debugger; builder fails safely if its computed bound is exceeded |
| U9 | whether sdport honours `Supported.Voltage33V` etc. | `[MS]` assigns them without effect, `[DW]` says "SDPORT doesn't seem to care" | none needed unless enumeration fails |
| U10 | gate/reset bit positions for MMC0 in `0x84C` | CCU base `0x03001000` is confirmed (`[LX-DT]` `ccu: clock@3001000`) and the `0x830`/`0x84C` offsets are in u-boot's header, the per-instance bits are not | needed only for 50 MHz (future work) |
| U11 | whether the firmware really leaves the module clock at 24 MHz (OSC24M, no N/M division) | owner's statement, nothing to check in this repo | read `0x0300 1830` in UEFI, or measure; set `ModuleClockHz` accordingly |
| U12 | `DBGC = 0xdeb` and `FUNCSEL = CEATA_ON` (written by Linux `sunxi_mmc_init_host`) are *not* written | "undocumented" in Linux; u-boot does not write them | add them only if reads/errors point to it |
| U13 | `THLDC` card-threshold (`READ_THLD(512)\|WRITE_EN\|READ_EN`) | written by u-boot only ("Needed on H616"), Linux does not | if PIO reads of < 512 bytes misbehave, try without |
| U14 | `STATUS.CARD_DATA_BUSY` is the right busy indicator for R1b and writes | u-boot polls it for `MMC_RSP_BUSY` only | write test, then CMD13 immediately after |
| U15 | exact sdport/ntddk API surface | `sdport.h` was not available; compile-checked against a hand-written mock only | first WDK build; see the list in §10 |

Open design questions for the owner: none blocking; see the test plan for the order in which these get answered.

## 8b. Independent review (phase 4)
A separate reviewer read all driver sources against the references. Findings and disposition:

| Finding | Disposition |
|---|---|
| Polled CMD12 in `SdResetTypeDat` raced with the ISR (sdport has re-enabled interrupts; the ISR would ack the polled bit, stalling the reset ~1 s) | **fixed**: `IMASK` is zeroed during the poll, restored from the shadow afterwards; test asserts it |
| `NeedStop` set for every failed data request (incl. response timeouts) | **fixed**: set by the DPC only when the card can be in its data state |
| DMA `StartTransfer` left `OutstandingRequest` pointing at a completed request | **fixed** + test |
| `RestoreContext` did not re-enable the card clock | **fixed** (re-applies the clock) |
| Error arriving between PIO phases ignored → request waits forever | **fixed**: `StartTransfer` checks recorded errors (`CurrentErrors`) |
| Possible lost `DATA_OVER` wake-up in the last PIO phase if ISR/DPC run on another CPU | **mitigated**: re-check after arming + `PhaseClaim` so only one party completes a phase. Not testable in the single-threaded harness |
| ISR returned TRUE for events that map to nothing | **fixed**: returns `Events != 0 \|\| Errors != 0` as the references |
| Busy worker could complete an aborted request | **fixed** |
| Last PIO phase did not re-assert `DATA_OVER`/`AUTO_COMMAND_DONE` in `IMASK` | **fixed** |
| Registers/ISR touched before/without successful init; all-ones read | **fixed**: `Initialized` guards ISR/Toggle/Clear; all-ones `GCTRL` read fails init. (An external abort on an ungated block cannot be caught.) |
| PIO-only mode left `PioTransferMaxThreshold = 0` | **changed** to "everything"; meaning in sdport UNVERIFIED |
| Comment errors (Linux's error mask excludes bit 10; NTSR attribution; CCU base is in the DT) | **fixed** |
| `IDIE` only enables RX, so IDMAC error bits are never reported; errors surface as `DATA_TIMEOUT` (as in Linux) | open (bit layout unverified) |
| `THLDC` 512-byte read threshold applies to DMA too; only u-boot (PIO) sets it | open (U13) |
| Descriptor-buffer capacity unknown; Linux caps at 256 descriptors | open (U8) |
| Race findings cannot be reproduced by the single-threaded harness | open: needs an interleaving test or hardware |

## 9. What has been checked, and how
* `make -C tests check` (also run by `.github/workflows/host-tests.yml`):
  * unit tests of the divider rule (both readings of §5 must stay ≤ the target), event mapping, R2 formatting and the IDMAC chain builder;
  * flow tests (225 checks) that drive the real `smhc.c`/`smhc_hw.c` through the sdport request choreography against a register-level SMHC model:
    init + clock programming, CMD0/8/41/2/3/7 command words and R2 formatting, PIO single/multi-block reads and writes (level- *and* edge-triggered data requests,
    FIFO smaller than the transfer, tails below the watermark), scatter/gather DMA (discontiguous elements, a 192 KiB element split into 64 KiB descriptors, delayed IDMAC copy), response timeout with late `COMMAND_DONE`,
    data CRC error and data timeout followed by the sdport reset sequence, state restoration after reset, undersized register window.
  * deliberately broken variants of the driver were confirmed to be caught by the flow tests (FIFO cleanup between PIO phases, wrong R2 shift, ignoring the late `COMMAND_DONE`, auto-stop ordering, not waiting for the IDMAC receive interrupt, tearing the IDMAC down early, unshifted `DLBA`, no register restore after reset, no stop command after a failed transfer, `IDIE` not enabled for reads).
* **This proves consistency with the model in `tests/emu.c`, which encodes this document's reading of the references. It says nothing about silicon behaviour or about conformance to the real `sdport.h`.**

## 10. sdport/ntddk identifiers used (to check at the first WDK build)
Taken from `[MS]`/`[DW]`: every `SDPORT_*`, `Sd*` enum, capability, request and command field used by `smhc.c`/`smhc_hw.c` appears in `sdhc.c`/`sdhc.h` or `dwcmshc.cpp`/`dwcmshc.h`,
with these exceptions that rely on the standard WDK: `PSCATTER_GATHER_LIST` (the sample uses `ScatterGatherList->Elements[]`), `IoOpenDeviceRegistryKey` /
`ZwQueryValueKey` (as `[DW]` `MshcGetPlatformType`), `DbgPrintEx` / `DPFLTR_*`, `__dsb` / `_ARM64_BARRIER_SY`.
`SdSetPower`, `SdResetHw` and `UseAutoCmd12` come from `[DW]` only.

## 11. Known limitations
24 MHz maximum (no High-Speed, no UHS); no hot-plug; no crash-dump/hibernate support; no power cycling of the card; no SDIO interrupt, no 8-bit, no eMMC modes;
no D-state handling (`SaveContext` caches everything, `RestoreContext` rewrites it); sdport request timeout behaviour is not exploited; descriptor addresses < 4 GiB.
