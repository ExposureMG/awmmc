# AWMC0001 (Allwinner H616 SMHC0) sdport miniport — Phase 1 & 2

Status: phases 1 (reference summary) and 2 (architecture proposal). **Historical: the architecture was settled and
implemented afterwards — see `02-design.md`, which supersedes §4–§5 here.** Corrections made after phase 2 are listed at the end.
Every claim below cites the file/function it came from. Items marked **UNVERIFIED** could not be
confirmed from the reference sources and must not be treated as fact.

Sources were fetched on 2026-10-04 from the `master` branches of the repos listed in the task.

## 1. Findings that change the plan

| # | Finding | Evidence | Impact |
|---|---|---|---|
| F1 | The Orange Pi Zero 3 DT in Linux is `sun50i-h618-orangepi-zero3.dts` (H618 SoC), not `sun50i-h616-orangepi-zero3.dts`; the latter does not exist (404). It reuses the H616 SMHC node/compatible. | `arch/arm64/boot/dts/allwinner/sun50i-h618-orangepi-zero3.dts`, `&mmc0` node | SMHC0 behaviour = `allwinner,sun50i-h616-mmc` config either way. |
| F2 | SMHC0 base `0x04020000`, size `0x1000` matches. | `sun50i-h616.dtsi`, node `mmc0: mmc@4020000`, `reg = <0x04020000 0x1000>` | ACPI `Memory32Fixed` is correct. |
| F3 | IRQ: `GIC_SPI 35` → INTID 67 (35+32) matches ACPI `Interrupt {67}`, level, active-high. | `sun50i-h616.dtsi`, `mmc0`, `interrupts = <GIC_SPI 35 IRQ_TYPE_LEVEL_HIGH>` | ACPI interrupt is correct. |
| F4 | **The `SDC0`/`AWMC0001` node is not in the OrangePiZero3Pkg repo.** Its `Dsdt.asl` is a Juno-derived placeholder (CLU0/CPU0.. ACPI0010/ACPI0007, KMI0 `ARMH0501`, ETH0 `ARMH9118`, COM0, USB0). grep for `mmc|smhc|SDC|AWMC` in the repo finds nothing relevant. | `OrangePiZero3Pkg/AcpiTables/Dsdt.asl` | The statement "already in the DSDT" refers to a DSDT I cannot see. I treat the node in the task text as authoritative. Needs confirmation (Q1). |
| F5 | The firmware repo does **not** initialise SMHC0 at all. UEFI is chain-loaded **from U-Boot** off the SD card (`fatload mmc 0:3 0x40080000 UEFI` then `go 0x40080000`). | `OrangePiZero3Pkg/README.md`, "Booting UEFI" | The SD card is the boot medium, so U-Boot has already enabled SMHC0 bus clock, de-asserted reset, programmed `MMC0_CLK_REG`, muxed PF0-PF5 and powered the card via the AXP313A. EDK2 neither helps nor undoes this. |
| F6 | Card detect on the Zero 3: schematic wires CD to PF6 through an inverter but upstream says it "just doesn't work" → `broken-cd`. Card VDD is `reg_dldo1` of the AXP313A (`vmmc-supply`). | `sun50i-h618-orangepi-zero3.dts`, `&mmc0 { broken-cd; vmmc-supply = <&reg_dldo1>; }` | No usable GPIO CD. Treat card as always present (see §4.5). No regulator control from Windows without an I2C/PMIC driver, so 1.8V is out of scope. |
| F7 | The "dwcmshc" repo is the **Samsung Exynos 7870** driver (ACPI `EXNS000A`), itself derived from WoR's Rockchip dwcmshc (Synopsys DWC MSHC = SDHCI-compatible). README: "disabled PIO ... does not support tuning". | `AistopGit/dwcmshc/README.md`, `dwcmshc.inx` | Useful only as proof that a non-Microsoft sdport miniport is feasible and as a structure reference. Not a register source, as the task states. |
| F8 | The Microsoft sample's `SdhcGetSlotCount` returns 1 for ACPI buses. | `sdhc.c`, `SdhcGetSlotCount`, `case SdBusTypeAcpi` | Matches our single-slot ACPI node. |
| F9 | `arch/arm/include/asm/arch-sunxi/mmc.h` in u-boot is now a 1-function stub. The real register map is `drivers/mmc/sunxi_mmc.h`. | both files | I cite `drivers/mmc/sunxi_mmc.h`. |

## 2. Reference summary

### 2.1 Microsoft sdport sample (`sd/miniport/sdhc/sdhc.c`, 3465 lines; `sdhc.h`, 2126 lines; `sdhc.inx`)

Skeleton to keep (callback table assigned in `DriverEntry`):

`GetSlotCount`, `GetSlotCapabilities`, `Initialize` (`SdhcSlotInitialize`), `IssueBusOperation`,
`GetCardDetectState`, `GetWriteProtectState`, `Interrupt`, `IssueRequest`, `GetResponse`,
`ToggleEvents`, `ClearEvents`, `RequestDpc`, `SaveContext`, `RestoreContext`,
`PowerControlCallback`, `Cleanup`; plus `PrivateExtensionSize`, `CrashdumpSupported`;
registered via `SdPortInitialize`.

Hardware-specific code that has to be replaced (all SDHCI-register based):

- `SdhcSlotInitialize` – capability register decode, ADMA enable via `SDHC_HC_DMA_SELECT_*` (sdhc.c ~1173-1183).
- `SdhcSlotIssueBusOperation` → `SdhcResetHost`, `SdhcSetVoltage`, `SdhcSetClock`, `SdhcSetBusWidth`, `SdhcSetSpeed`, `SdhcSetSignaling`, `SdhcExecuteTuning`.
- `SdhcBuildTransfer`/`SdhcStartTransfer` / `SdhcBuildAdmaTransfer` / `SdhcCreateAdmaDescriptorTable` / `SdhcStartAdmaTransfer` – ADMA2 descriptor build uses `Request->Command.DmaVirtualAddress/DmaPhysicalAddress` and `SDHC_ADMA2_MAX_LENGTH_PER_ENTRY` (sdhc.c ~2913-3290). This is where IDMAC chain descriptors go.
- `SdhcSlotInterrupt`, `SdhcRequestDpc`, `SdhcSlotToggleEvents`, `SdhcSlotClearEvents`, `SdhcGetResponse`, `SdhcEnableInterrupt`/`SdhcDisableInterrupt`/`SdhcGetInterruptStatus`/`SdhcGetErrorStatus`/`SdhcAcknowledgeInterrupts` – map SDHCI normal/error status onto `SDPORT_EVENT_*`/`SDPORT_ERROR_*`.
- PIO path: `SdhcBuildPioTransfer`/`SdhcStartPioTransfer` with `SdhcReadDataPort`/`SdhcWriteDataPort`, selected by `Request->Command.TransferMethod == SdTransferMethodPio` in `SdhcBuildTransfer`/`SdhcStartTransfer` (sdhc.c ~2787-3040). `SdhcSlotInitialize` also sets `Capabilities->PioTransferMaxThreshold = 64`, `Flags.UsePioForRead/Write` (sdhc.c ~317-323). We keep the same split.

The ADMA code relies on the port allocating the descriptor buffer (`Capabilities->DmaDescriptorSize`,
sdhc.c ~307) and giving a physical address in the request. That is the same contract we need for
the IDMAC descriptor list and means **the port owns the DMA adapter, cache maintenance for the
data buffers, and descriptor-memory allocation** (**UNVERIFIED** for the cacheability of
`DmaVirtualAddress` memory on `_CCA=0`; see Q3).

### 2.2 Allwinner SMHC register map (u-boot `drivers/mmc/sunxi_mmc.h`, `struct sunxi_mmc`; cross-checked with Linux `sunxi-mmc.c`)

| Offset | Name | u-boot field / Linux macro |
|---|---|---|
| 0x00 | GCTRL | `gctrl` / `SDXC_REG_GCTRL` |
| 0x04 | CLKCR | `clkcr` / `SDXC_REG_CLKCR` |
| 0x08 | TIMEOUT | `timeout` / `SDXC_REG_TMOUT` |
| 0x0C | WIDTH | `width` / `SDXC_REG_WIDTH` |
| 0x10 | BLKSZ | `blksz` |
| 0x14 | BYTECNT | `bytecnt` / `SDXC_REG_BCNTR` |
| 0x18 | CMD | `cmd` / `SDXC_REG_CMDR` |
| 0x1C | ARG | `arg` / `SDXC_REG_CARG` |
| 0x20-0x2C | RESP0-3 | `resp0..3` |
| 0x30 | IMASK | `imask` |
| 0x34 | MINT (masked) | `mint` / `SDXC_REG_MISTA` |
| 0x38 | RINT (raw) | `rint` / `SDXC_REG_RINTR` |
| 0x3C | STATUS | `status` / `SDXC_REG_STAS` |
| 0x40 | FTRGLEVEL | `ftrglevel` / `SDXC_REG_FTRGL` |
| 0x44 | FUNCSEL | `funcsel` / `SDXC_REG_FUNS` |
| 0x58 | A12A (auto-CMD12 arg) | `a12a` / `SDXC_REG_A12A` |
| 0x5C | NTSR (new timing) | `ntsr` / `SDXC_REG_SD_NTSR` |
| 0x78 | HWRST | `SUNXI_MMC_HWRST` |
| 0x80 | DMAC (IDMAC ctrl) | `dmac` / `SDXC_REG_DMAC` |
| 0x84 | DLBA (descriptor list base) | `dlba` / `SDXC_REG_DLBA` |
| 0x88 | IDST (IDMAC status) | `idst` / `SDXC_REG_IDST` |
| 0x8C | IDIE | `idie` / `SDXC_REG_IDIE` |
| 0x100 | THLDC | `SUNXI_MMC_THLDC` (gated by `CONFIG_SUN50I_GEN_H6`) |
| 0x144 | SAMP_DL | struct layout; Linux `SDXC_REG_SAMP_DL_REG 0x144` |
| 0x200 | FIFO (H6-class) | **UNVERIFIED** – derived from the u-boot struct padding (`res3[16]`, `samp_dl`, `res4[46]`); not stated in Linux (Linux never uses PIO). Must be confirmed on hardware / vendor manual. |

Bit definitions: GCTRL, CMD, RINT, STATUS, IDMAC, CLKCR bits come from `sunxi_mmc.h` (`SUNXI_MMC_GCTRL_*`,
`SUNXI_MMC_CMD_*`, `SUNXI_MMC_RINT_*`, `SUNXI_MMC_STATUS_*`, `SUNXI_MMC_IDMAC_*`, `SUNXI_MMC_IDIE_*`,
`SUNXI_MMC_CLK_*`) and Linux `sunxi-mmc.c` lines 82-231 (`SDXC_*`). Additional Linux-only bits I plan to use:
`SDXC_DMA_ENABLE_BIT`(5), `SDXC_INTERRUPT_ENABLE_BIT`(4), `SDXC_IDMAC_*` status bits and
descriptor bits `DES0_DIC/LD/FD/CH/ER/CES/OWN` (bits 1/2/3/4/5/30/31), `SDXC_CAL_DL_SW_EN`(7), `SDXC_2X_TIMING_MODE`(31).

H616 CCU registers (from u-boot `arch/arm/include/asm/arch-sunxi/clock_sun50i_h6.h`):
`CCU_MMC0_CLK_CFG 0x830`, `CCU_H6_MMC_GATE_RESET 0x84c`,
mod-clock fields `CCM_MMC_CTRL_M(x)=(x)-1`, `N(x)<<8`, source `OSCM24=0<<24`, `PLL6=1<<24`,
`PLL_PERIPH2X2=2<<24`, `ENABLE=1<<31`; gate/reset bits `SUNXI_MMC_COMMON_CLK_GATE = 1<<16`,
`SUNXI_MMC_COMMON_RESET = 1<<18` (`sunxi_mmc.h`). Per-instance bit positions in `0x84c`
(MMC0 = bit 0 / bit 16) are **UNVERIFIED** here (not read from the clock driver body); the CCU base
`0x03001000` is **UNVERIFIED** (not in the files I fetched).

### 2.3 H616 init and command flow (u-boot `sunxi_mmc.c`; Linux `sunxi-mmc.c`)

- Reset (`sunxi_mmc_reset`): `GCTRL = SOFT|FIFO|DMA reset`; delay 1 ms; on H6/H616: HWRST assert, 10 µs, deassert, 300 µs; `THLDC = READ_THLD(512)|WRITE_EN|READ_EN` ("Needed on H616").
- Host init (Linux `sunxi_mmc_init_host`): `FTRGL=0x20070008`, `TMOUT=0xffffffff`, `RINTR=0xffffffff`, `DBGC=0xdeb` (undocumented), `FUNS=SDXC_CEATA_ON`, `DLBA = sg_dma >> idma_des_shift`, `GCTRL |= INTERRUPT_ENABLE; &= ~ACCESS_DONE_DIRECT`.
- Clock (`mmc_config_clock`): disable `CLKCR.ENABLE`; `mmc_update_clk` (CMD with `START|UPCLK_ONLY|WAIT_PRE_OVER`, poll START clear, clear RINT); program CCU mod clock; clear internal divider; for H616 write `SAMP_DL = CAL_DL_SW_EN` (Linux `sunxi_mmc_calibrate`: "best rates obtained by simply setting the delay to 0, as Allwinner does in its BSP"); re-enable `CLKCR.ENABLE`, `update_clk` again.
- `H616 cfg` (Linux `sun50i_h616_cfg`): `idma_des_size_bits=16`, `idma_des_shift=2`, `can_calibrate`, `mask_data0`, `needs_new_timings`.
- Width register: 0=1-bit, 1=4-bit, 2=8-bit (`sunxi_mmc_set_ios_common`, Linux `SDXC_WIDTH*`).
- Command build (`sunxi_mmc_send_cmd_common`): `START`, `SEND_INIT_SEQ` for CMD0, `RESP_EXPIRE`, `LONG_RESPONSE` (R2), `CHK_RESPONSE_CRC`, `DATA_EXPIRE|WAIT_PRE_OVER`, `WRITE`, `AUTO_STOP` if blocks>1; `BLKSZ`, `BYTECNT` written before `CMD`; R2 is read `resp3,resp2,resp1,resp0` into `response[0..3]`; R1b busy = poll `STATUS.CARD_DATA_BUSY`.
- Error masks: `SUNXI_MMC_RINT_INTERRUPT_ERROR_BIT` (0xbfc2); done masks `AUTO_COMMAND_DONE|DATA_OVER|COMMAND_DONE|VOLTAGE_CHANGE_DONE`.
- IDMAC descriptor (Linux `struct sunxi_idma_des`): `{config, buf_size, buf_addr_ptr1, buf_addr_ptr2}`; chain mode with `CH|OWN|DIC` per descriptor (`sunxi_mmc_init_idma_des`); `buf_size=0` means max; addresses and `next` are `>> idma_des_shift`. `FD`/`LD` handling and the trailing bits are in Linux lines ~365-400 (to be re-read in phase 3).
- Linux also **masks DATA0** (`mask_data0`, lines ~673/694) around clock changes on H616 – must be replicated.

### 2.4 What I could **not** find (must be treated as unknown)

- Cache maintenance for descriptors/data: Linux relies on the DMA API; there is no Allwinner-specific rule. Windows equivalent is the port-provided DMA adapter (Q3).
- Any delay/phase calibration beyond "write `CAL_DL_SW_EN`=delay 0". No numeric tuning table exists for H616 in either source. U-Boot's `oclk_dly/sclk_dly` table only applies to controllers *without* calibration (`sunxi_mmc_can_calibrate()`), and on H6 `CCM_MMC_CTRL_*_DLY` are compiled out.
- Whether the AXP313A DLDO1 voltage can be switched to 1.8V by the driver (needs PMIC I²C). Not attempted.
- IDMAC descriptor chain `FD/LD/DIC` exact end-of-list semantics for H616 beyond Linux's code (re-verify when implementing).
- UHS-I behaviour on this slot (no 1.8V supply control) → future work.

## 3. Mapping: SDHCI-sample concept → SMHC equivalent

| sdport/SDHCI concept (sample) | Allwinner SMHC |
|---|---|
| Software reset (all/cmd/data) | `GCTRL.SOFT_RESET/FIFO_RESET/DMA_RESET` (bits 0/1/2) |
| Clock control, divider, SD clock enable | `CLKCR.ENABLE(16)`, `CLKCR.POWERSAVE(17)`, div[7:0] + CCU `MMC0_CLK_REG` (0x830) mod-clock; every change followed by `UPCLK_ONLY` command |
| Power control reg | none in SMHC; DLDO1 on AXP313A (external) → no-op at 3.3V |
| Host control 1 (width, high-speed) | `WIDTH` reg (0/1/2); high-speed handled by clock + delay (no HS bit) |
| Transfer Mode / Command | single `CMD` reg (bits 5:0 index, flags above, `START`=31) |
| Block size / count / argument | `BLKSZ` + `BYTECNT` (bytes, not blocks) / `ARG` |
| Response regs (R2 shifted) | `RESP0-3` raw; R2 stored as `resp3..resp0` |
| Normal/error status + enables | `RINT`(raw), `MINT`(masked), `IMASK`; write-1-to-clear on `RINT` |
| ADMA2 table + `SYSADDR` | IDMAC chain descriptors + `DLBA` (>>2), `DMAC.IDMA_ON|FIX_BURST`, `GCTRL.DMA_ENABLE`, status in `IDST` |
| Buffer data port (PIO) | FIFO (offset UNVERIFIED), `GCTRL.ACCESS_BY_AHB`, `STATUS.FIFO_EMPTY/FULL/LEVEL` |
| Card inserted/removed int | `RINT.CARD_INSERT(30)/CARD_REMOVE(31)` – unreliable on Zero3 (broken-cd) |
| Present State (CD, DAT0 busy) | `STATUS.CARD_PRESENT(8)`, `STATUS.CARD_DATA_BUSY(9)` |
| Auto-CMD12 | `CMD.SEND_AUTO_STOP(12)`; completion `RINT.AUTO_COMMAND_DONE(14)` |
| Tuning (UHS) | none implemented; `SAMP_DL` software delay only → UHS future work |

## 4. Proposed architecture

### 4.1 Skeleton
Keep the Microsoft sample's file layout: `smhc.sys` (`DriverEntry` → `SdPortInitialize` with the same callback table), a
`SMHC_EXTENSION` replacing `SDHC_EXTENSION`, and `PrivateExtensionSize`. `GetSlotCount` returns 1 (ACPI path). `CrashdumpSupported = FALSE` initially (the sample sets TRUE; enabling needs a polled path with no interrupts/allocations – later).

### 4.2 Clocks/pinmux/power decision (a / b / c)
**Recommendation: (a) with a guarded fallback to (b).**

- (a) Assume the firmware chain (U-Boot, F5) left SMHC0 clocked, un-reset, muxed and powered. This is true by construction because the platform boots from this very SD card.
- Windows must still change the card clock (400 kHz → 25/50 MHz). That needs `MMC0_CLK_REG`. Therefore the driver needs a CCU window. Two options:
  - (b) Map only the two words we need (`0x03001830` MMC0 clock, `0x0300184C` gate/reset) — this requires a second `Memory32Fixed` in the ACPI node (change C1). **UNVERIFIED**: CCU base `0x03001000`.
  - Alternative that avoids any CCU access: keep the module clock U-Boot programmed and use only the SMHC internal divider (`CLKCR[7:0]`). **Rejected as the plan**: the module clock rate U-Boot left is unknown to Windows (UNVERIFIED), and an 8-bit divider only reaches 400 kHz if that rate is low enough.
- (c) Full ACPI extension is *required only for* CD GPIO and PMIC control, which we are not using (F6).
- Interop alternative: if the ACPI authors prefer `_DSD` properties (`"clock-frequency"` of the module clock) we can read the actual module rate instead of assuming it.

**ACPI changes required (proposed, minimal):**
- C1: add `Memory32Fixed (ReadWrite, 0x03001830, 0x4)` and `Memory32Fixed (ReadWrite, 0x0300184C, 0x4)` (or one 0x1000 window at `0x03001000` — larger blast radius, not preferred), **CCU base UNVERIFIED**.
- C2: `Name (_DSD, ...)` with `"broken-cd" = 1` is optional; the driver defaults to always-present.
- C3 (optional): `_DSD` `"max-frequency" = 50000000`.
- No GPIO/regulator resources.

### 4.3 Bring-up scope (strictly minimal first)
1. Init: soft reset, THLDC, FTRGL, TMOUT, interrupt enables, `DLBA`.
2. 400 kHz identification clock (`CLKCR` + CCU, `UPCLK_ONLY`), 1-bit.
3. CMD0 / CMD8 / ACMD41 / CMD2 / CMD3 / CMD7, then CMD6/ACMD6 → 4-bit, 25 MHz.
4. Single-block read via **PIO/FIFO** first (simplest, no cache issues).
5. Single-block read via IDMAC, then multi-block (auto-CMD12) via IDMAC chain.
6. Writes, then 50 MHz high speed (`SAMP_DL` delay 0 as in Linux).
UHS-I, 1.8V, tuning, DDR, eMMC modes: **not implemented** (future work).

### 4.4 Interrupt / DPC model
ISR (`SdhcSlotInterrupt` equivalent): read `MINT`/`IDST`, latch into the extension, mask, return TRUE if ours; DPC (`RequestDpc`) maps `RINT` → `SDPORT_EVENT_*`/`SDPORT_ERROR_*` (resp error/CRC/timeout, data CRC/timeout, FIFO run error, start/end bit error). `ToggleEvents`/`ClearEvents` write `IMASK`/`RINT`.

### 4.5 Card detect / write protect
- CD: do not trust `RINT.CARD_INSERT` or PF6 (Zero 3 `broken-cd`, F6). `GetCardDetectState` returns TRUE (always present, like Linux `broken-cd`); sdport's own init probe will fail cleanly if no card is inserted (CMD0/CMD8 timeouts). Hot-plug is **not** supported initially.
- WP: no WP line on microSD → return FALSE.

### 4.6 Power/voltage
3.3 V only. `SetVoltage` is a validated no-op (`SdBusVoltage33` accepted; others `STATUS_NOT_SUPPORTED`). Signaling-voltage switch (1.8 V) → `STATUS_NOT_SUPPORTED`; capability bits for UHS cleared.

### 4.7 DMA and cache (the biggest risk)
`_CCA = 0` ⇒ non-coherent. Plan: rely on sdport's DMA adapter (`Command.DmaVirtualAddress/DmaPhysicalAddress`, `DmaDescriptorSize`) exactly as the sample's ADMA path does; use `KeFlushIoBuffers`/adapter flush semantics through the port; descriptor list memory must be uncached or flushed before `DMAC.IDMA_ON`. **UNVERIFIED** that sdport provides non-cached descriptor memory on ARM64 non-coherent systems — Q3.

## 5. Open questions (blocking, with my proposed default)

1. **ACPI source of truth.** The DSDT in OrangePiZero3Pkg has no `SDC0` (F4). Is your DSDT a local/private build? If so, can you paste/confirm `SDC0` and tell me whether you can add the CCU `Memory32Fixed` window(s)? *Default: assume yes, add `0x03001830` + `0x0300184C` (4 bytes each).*
2. **CCU base and per-instance gate/reset bits.** I could not read the H616 clock driver body. Do you have the H616 user manual (CCU `MMC0_CLK_REG`, `MMC_BGR_REG`) or can I fetch Linux `drivers/clk/sunxi-ng/ccu-sun50i-h616.c`? *Default: I will fetch the Linux CCU driver to confirm in phase 3 before writing it.*
3. **DMA/cache policy.** Do you want me to depend on the sdport-provided DMA buffers (fastest to implement, cache semantics UNVERIFIED), or to allocate my own common buffer via `IoGetDmaAdapter`/`AllocateCommonBuffer`? *Default: sdport buffers + explicit descriptor flush; PIO first so the first read test has zero cache dependence.*
4. **Build environment.** This session is a Linux container with no WDK/MSVC, so I **cannot compile or sign** the driver here. I can deliver source + VS/WDK project + INF and statically review it. Is that acceptable, or can you give me a Windows build host/CI? *Default: source-only, reviewed by reading, with a GitHub Actions workflow (windows runner + WDK) to build it.*
5. **Card hot-plug.** Always-present CD (F6) means no insert/remove events. OK for bring-up? *Default: yes.*
6. **Scope of speed.** Stop at 25 MHz first, then 50 MHz high-speed? *Default: yes.*

If you answer nothing, I proceed to phase 3 with the defaults above.


## Corrections and decisions after phase 2

* **F7 was wrong.** `AistopGit/dwcmshc` is *not* SDHCI-compatible: it is a classic **DesignWare MMC** (`CTRL/PWREN/CLKDIV/CLKENA/CTYPE/RINTSTS/IDSTS`) miniport for sdport.
  That makes it the closest structural reference for the Allwinner SMHC (same lineage: `RINT` bit layout, IDMAC, FIFO thresholds), and the driver follows its request/event handling
  where the Microsoft sample is SDHCI-specific.
* **Firmware**: the platform runs **mu-silicium**, not OrangePiZero3Pkg; the owner states it leaves the card powered at 24 MHz. Option (a) is therefore used *without* any CCU window:
  the card clock is derived from the 24 MHz module clock with the SMHC-internal divider only (see `02-design.md` §1, §5). The ACPI node needs no changes.
* **Decision outcomes**: DMA uses sdport's SG list and descriptor buffer (PIO is the default so the first bring-up has no cache dependence); source-only delivery plus a host-side emulator
  instead of a WDK CI job (a CI job for the WDK build could not be validated here); no hot-plug; stop at 24 MHz (no 50 MHz without the CCU).
