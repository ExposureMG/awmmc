# Build, sign and install

**Not exercised yet:** the driver has only been compiled against a hand-written host mock (see `tests/README.md`). The steps below are the standard WDK flow;
expect the first real build to need small fixes (see `docs/02-design.md` §10 for the list of identifiers that depend on the real `sdport.h`).

## Prerequisites (build machine: Windows 10/11, x64 or ARM64)

* Visual Studio 2022 with *Desktop development with C++* and the **MSVC ARM64 build tools** (`MSVC v143 – VS 2022 C++ ARM64 build tools`).
* Windows SDK + **Windows Driver Kit** matching it (the `sdport.lib` / `sdport.h` import library and header ship in the WDK `km` tree), with the WDK Visual Studio extension.
* Target OS: Windows 10/11 ARM64 (the INF is stamped for build 16299+, like the dwcmshc INF).

## Build

```
msbuild awmmc.sln /p:Configuration=Debug /p:Platform=ARM64
msbuild awmmc.sln /p:Configuration=Release /p:Platform=ARM64
```

Output (project default `OutDir`): `ARM64\<Config>\awmmc.sys`, `awmmc.pdb`, and the package directory with `awmmc.inf` (stamped from `driver\awmmc.inx`) and the catalog.
Set `ProviderString` in `awmmc.inx` before shipping anything. Tunables (`TransferMode`, `ModuleClockHz`, `MaxFrequencyKhz`) are in the `[AWMC_Params]` section.

Command-line equivalent of what the WDK targets do (adjust paths/versions):

```
stampinf -f awmmc.inf -d * -a arm64 -v * -k 1.15 -i awmmc.inx
inf2cat /driver:. /os:10_NI_ARM64 /uselocaltime
signtool sign /fd sha256 /s My /n "WDKTestCert <you>" awmmc.sys awmmc.cat
```

## Test signing

Windows on ARM64 requires signed kernel drivers; for development use test signing.

1. Create/trust a test certificate on the target (the VS "WDKTestCert" certificate works; or
   `New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=awmmc test" -CertStoreLocation Cert:\CurrentUser\My`,
   export the public part, and import it into *LocalMachine\Root* and *LocalMachine\TrustedPublisher* on the target).
2. Enable test signing on the target (**mu-silicium must not enforce Secure Boot**; this is a firmware matter, not covered here):

   ```
   bcdedit /set testsigning on                         (running system)
   bcdedit /store <EFI>\EFI\Microsoft\Boot\BCD /set {default} testsigning on     (offline image)
   ```
3. Reboot; the desktop shows a "Test Mode" watermark.

## Install

Running system:

```
pnputil /add-driver awmmc.inf /install
```

Offline image (preferred while the SD card is the system disk — but see the warning in `docs/03-bringup-test-plan.md`):

```
dism /image:<mounted-windows-volume>\ /add-driver /driver:<path-to>\awmmc.inf
```

The device (`ACPI\AWMC0001\0`) must exist in ACPI (`acpi/sdc0.asl`); otherwise nothing binds.

## Verify / change settings / uninstall

```
pnputil /enum-devices /instanceid ACPI\AWMC0001\0 /connected
reg query "HKLM\SYSTEM\CurrentControlSet\Enum\ACPI\AWMC0001\0\Device Parameters"
reg add   "HKLM\SYSTEM\CurrentControlSet\Enum\ACPI\AWMC0001\0\Device Parameters" /v TransferMode /t REG_DWORD /d 1 /f
pnputil /restart-device ACPI\AWMC0001\0
pnputil /delete-driver oem<N>.inf /uninstall /force
```

## Host-side tests (any Linux/macOS box with gcc/clang and make)

```
make -C tests check
```

These do **not** involve the WDK.
