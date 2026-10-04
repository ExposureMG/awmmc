# AWMC0001 bring-up test plan

Order matters: each stage has an exit condition; do not start the next one before it holds.
Everything below the "expected" lines is an *expectation derived from the references*, not an observed result — the driver has not run on hardware yet.

> **Do not bring this driver up on the disk Windows boots from.** If SMHC0 holds the boot SD card, a faulty driver means an unbootable system and a
> corrupt filesystem. Boot Windows from USB (or another medium) and use a **scratch microSD** in SMHC0. Only after stage 6 consider making `awmmc` the boot storage driver
> (`BootFlags = 8` is already in the INF; the device also has to be in the offline image's critical-device database, which this repo does not cover).

## 0. Pre-flight (UEFI side, mu-silicium)

Goal: confirm the assumptions the driver is built on, from the UEFI shell, before Windows is involved.

| Check | How | Expected |
|---|---|---|
| Memory map has SMHC0 | UEFI shell `memmap` / `dmem 0x04020000 0x10` | `0x04020000` is device memory, readable |
| Module clock is 24 MHz (U11) | `mm 0x03001830 -w 4` (CCU `MMC0_CLK_REG`; CCU base `0x03001000` per the H616 DT, register offset `0x830` per u-boot) | bit 31 set (enabled); bits 25:24 = 0 (OSC24M); N (9:8) = 0, M (3:0) = 0 |
| Bus clock/reset released | `mm 0x0300184C -w 4` (`MMC_BGR_REG`) | MMC0 gate (bit 0) and reset (bit 16) set — bit positions UNVERIFIED |
| SMHC alive | `mm 0x04020000 -w 4` … dump 0x00–0x8C | `GCTRL` readable (not `0xFFFFFFFF`); `CLKCR` (0x04) may show bit 16 (clock on) |
| Card VDD up | multimeter on the socket VDD pin | 3.3 V (DLDO1) |
| Pinmux (PF0–PF5 = mmc0) | `dmem` of the PIO PF bank (`0x0300B0B4` + …, UNVERIFIED) or just continue; if the controller sees no card the enumeration in stage 2 fails | — |
| FIFO window (U2) | `dmem 0x04020200 0x10` | readable without a bus fault |

Record the values; they become the baseline for `ModuleClockHz` (registry) and for stage 2 debugging. **Exit:** SMHC0 registers readable, module clock confirmed 24 MHz (else set `ModuleClockHz` to the real value in the INF/registry).

## 1. Build, sign, install, attach the debugger

See `docs/04-build-and-install.md`. Required: test signing on and a kernel debugger. KDNET needs a supported NIC (the Zero 3's Allwinner EMAC is probably not one); a serial debugger over the UART described by
the firmware's DBG2 table is the realistic option — whichever works on your mu-silicium build. Before the first boot with the driver:

```
bcdedit /store <EFI>\EFI\Microsoft\Boot\BCD /set {default} testsigning on
bcdedit /store <EFI>\EFI\Microsoft\Boot\BCD /debug on
bcdedit /store <EFI>\EFI\Microsoft\Boot\BCD /dbgsettings serial debugport:1 baudrate:115200
        (or: /dbgsettings net hostip:<ip> port:50000 key:<key>   where KDNET is usable)
```

In WinDbg once attached (and after every boot):

```
ed nt!Kd_IHVDRIVER_Mask 0xF      ; DbgPrintEx: 1=error 2=warn 4=info 8=trace (the driver's "SMHC:" lines)
lm m awmmc                       ; driver loaded?
```

Useful breakpoints (symbols from the build's `.pdb`):

```
bp awmmc!SmhcSlotInitialize      ; stage 1
bp awmmc!SmhcSetClock            ; stage 2
bp awmmc!SmhcSendCommand "r $t0=poi(@x1+0); gc"   ; or just watch the SMHC: CMD<n> lines
ba w4 <virtual SMHC base>+0x18   ; every CMD register write (find the base in the "slot init" log line / !devnode)
```

**Exit:** `lm m awmmc` shows the module and the `SMHC: DriverEntry` line appeared.

## 2. Device manager and slot bring-up (no card traffic yet)

Expected log (`SMHC:` prefix), in order:

```
DriverEntry
slot init: phys 0x4020000 len 0x1000 module clock 24000000 Hz max 50000 kHz mode PIO
bus op: reset 0
bus op: clock 400 kHz
clock: requested 400 kHz, module 24000000 Hz, CLKCR div field 59, ~400 kHz
```

* Device Manager: *SD host adapters* → "Allwinner SMHC SD/MMC Host Controller", status OK (code 0).
* If the device shows Code 10/31/39: check the INF is bound to `ACPI\AWMC0001` (`pnputil /enum-devices /instanceid ACPI\AWMC0001\0`), that test signing is on,
  and that the log has no `register window ... too small` / `initial reset failed`.
* Read back the SMHC registers in WinDbg (`dd <base> L24`): `CLKCR` = `0x0001003B` (clock on, field 59) after the 400 kHz request, `NTSR` bit 31 set, `SAMP_DL` = `0x80`, `GCTRL` bit 4 set, `THLDC` = `0x02000005`.

**Exit:** slot init clean, clock programmed as above. **Measure CLK now** (U1): 400 kHz (linear reading) or ≈ 203 kHz (DesignWare reading). Either is fine for the next stage; note which.

## 3. Enumeration (CMD0/8/55+41/2/3/9/7/…)

Insert the scratch card, reboot or *Scan for hardware changes*. Expected log (indices; `cmd` shows the command word):

```
CMD0   ... cmd 0x80008000   (START | SEND_INIT_SEQ)
CMD8   arg 0x000001AA       (R7, handled as R1: RESP_EXPIRE|CHK_CRC)
CMD55  -> CMD41 (R3, no CRC check)  repeated until the card reports ready (busy bit)
CMD2 (R2) -> CMD3 (R6) -> CMD9 (R2) -> CMD7 (R1b, busy wait)
ACMD51 (SCR, 8 byte PIO read)  -> ACMD6 (width 4) -> CMD16/…  -> ACMD13/CMD6 (64 byte reads)
```

What each failure means:

| Symptom | Likely cause | Check |
|---|---|---|
| `CMD0` ok, `CMD8` → `CMD_TIMEOUT` every time | no card / pins not muxed / clock not reaching the card / wrong divider | scope CLK and CMD; U11 (module clock), pinmux |
| CMD2 → `CMD_CRC_ERROR` | R2 CRC checking on a response that does not carry a CRC7 in this position | try without `CHK_RESPONSE_CRC` for R2 (one-line change in `SmhcSendCommand`) |
| enumeration stops after ACMD51 / CMD6 with `DATA_TIMEOUT` | small PIO read broken: FIFO offset (U2), `FTRGL` layout (U3), `THLDC` (U13) | read `STATUS`, dump FIFO window; set `TransferMode=0`; compare with u-boot behaviour |
| sdport reports the card as inaccessible after CMD7 | busy wait wrong (U14) | log `STATUS` bit 9 after CMD7 |
| card appears but 1-bit only | ACMD6 refused / `WIDTH` not applied | `dd <base>+0x0C L1` should be `1` after the width op |

**Exit:** card enumerates (Device Manager: *SD Storage / SD memory card*, Disk Management shows a disk of the right size).

## 4. Read tests (PIO first)

Prepare the scratch card on a PC: a FAT32/NTFS partition with a few files and a 256 MiB file `big.bin`; keep `sha256sum` of every file.

1. Single block: `Get-Disk`, `Get-Partition`, `Get-Volume` (reads the MBR/GPT and the boot sector — small PIO reads and 512-byte reads).
2. Directory listing of the volume; `Get-Content <small file>` — multi-block PIO read with auto-CMD12 (log shows `CMD18 ... cmd 0x...` with bit 12 set).
3. `Get-FileHash -Algorithm SHA256 X:\big.bin` — compare with the PC value. Run three times.
4. Switch to DMA (`reg add` below, then disable/enable the device) and repeat 1–3:

```
reg add "HKLM\SYSTEM\CurrentControlSet\Enum\ACPI\AWMC0001\0\Device Parameters" /v TransferMode /t REG_DWORD /d 1 /f
pnputil /restart-device ACPI\AWMC0001\0       (or reboot)
```

   Expected: log shows `PIO` → `DMA` on the `CMD17/18` lines; hashes identical; throughput clearly higher than PIO.

Failure triage:

| Symptom | Likely cause |
|---|---|
| PIO hash mismatch, DMA ok | FIFO/threshold handling (U2–U4, U13) |
| DMA hash mismatch (random corruption), PIO ok | cache maintenance (U7) or descriptor/barrier problem; try small transfers first; dump descriptors in WinDbg (`dd <desc> L10`): a 1-descriptor chain has `config = 0x8000003C` (OWN\|FD\|LD\|ER\|CH), a longer one starts `0x8000001A` (OWN\|FD\|CH\|DIC) and ends `0x80000034` (OWN\|LD\|ER\|CH); `buf_addr`/`next` are physical addresses >> 2 |
| DMA hangs, log shows `req-events 0x...` never satisfied | missing `IDST.RI` (IDIE) or `AUTO_COMMAND_DONE`; dump `IDST` (+0x88) and `RINT` (+0x38) |
| intermittent `DATA_CRC_ERROR` at 24 MHz | sample delay / timing (`SAMP_DL`, `NTSR`), U5 |
| `FIFO_RUN_ERROR` | watermark/burst settings (U3); try `FTRGL` = `0x20070008` for PIO as well |

**Exit:** reads verified (PIO and DMA), no errors in the log over a 256 MiB read repeated 3×.

## 5. Write tests (scratch card only)

1. Create a small file (`Set-Content`), read it back, compare hash. Log: `CMD24`, then the busy wait after the data phase.
2. Copy a 64 MiB file to the card, read it back, compare SHA-256 (PIO, then DMA).
3. `robocopy` 256 MiB of many small files; `chkdsk X:` read-only afterwards; then remove the card and verify on the PC (`sha256sum`, `fsck`).
4. Repeat with the card removed/re-inserted between runs (no hot-plug support: reboot or `pnputil /restart-device`).

Failure triage: corruption only on write → busy handling (U14: next command issued while DAT0 low), or `THLDC.WRITE_EN`; writes time out → TXDR handling (U4); `CMD25` multi-block errors → auto-stop (`AUTO_COMMAND_DONE` ordering).

**Exit:** write + read-back identical, chkdsk clean.

## 6. Soak and fault handling

* 1 GiB read/verify loop, 1 GiB write/verify loop, 1 h each, DMA mode.
* Pull the card during a transfer: expect failed requests, `SMHC: ... failed` lines, `NeedStop` → polled CMD12, then recovery after reinsertion + restart-device (no hot-plug detection).
* Sleep/hibernate: **not supported** (no crash-dump, no D-state work); confirm the system does not enter it with the driver loaded.

## 7. After it works

Order of the next improvements: (1) CCU access for 50 MHz High-Speed (ACPI windows in `acpi/sdc0.asl`), (2) INF `TransferMode=1` as default, (3) card detect (GPIO/`broken-cd` override), (4) 1.8 V/UHS-I if a PMIC path exists, (5) ETW/TraceLogging instead of `DbgPrintEx`.
