# ZineROM One UI 7 — Galaxy A11 (SM-A115F) Port Status

Public status board for the **SM-A115F / a11q** One UI 7 port based on the A05s `A057FXXUFDZH2` donor.

Maintained for **zinerom48-tech** so the latest evidence and next actions are available whenever there is internet access.

## Read this first

- **This is not a ROM download and not a flashable package.**
- **Current readiness: R0.** No successful boot has been observed.
- **Do not install the test ZIP** unless you have a verified stock `A115FXXS6CWK2` restore path and have explicitly approved the risk.
- Do **not** format `/data`, `/keydata`, or `/keyrefuge` to “fix” the boot loop. That destroys data and does not prove the cause.

## Progress right now (2026-11-16)

| Measure | Estimate | Meaning |
|---|---|---|
| Successful boot | **0%** | No completed boot, no boot animation, no Setup Wizard. |
| Boot stages reached | **~45%** | Bootloader → kernel → first-stage init → second-stage init → vendor services/HALs. It fails at `/data` file-based encryption before zygote and `system_server`. |
| Offline root-cause isolation | **~95%** | Code-level root cause established: the A11 4.9 kernel implements only legacy fscrypt, while the Samsung donor `vold` and `init` have the legacy key path stripped. The fix compiles, passes ELF checks, and is exercised through the real mod install and `apply_modules.sh` driver paths in CI; the remaining question is runtime behavior. |
| Validated fix | **0%** | The source fix is compiled, host-tested, and CI-tested (including the in-ROM mod install and real build driver), but it has not been ROM-built or device-tested. |
| Device validation | **0%** | No physical A11 available. |

These are bookkeeping estimates, **not** a probability that the ROM will boot. The remaining Android framework/UI work is larger than the stage count suggests.

**2026-11-16 update:** the fix branch `manus/a115f-legacy-fscrypt@3aa20d4` adds a PID-1 `/system/bin/init` keyring wrapper, a 64-bit `vold` `LD_PRELOAD` shim, and an in-ROM mod-install suite. CI run `43925890423` passed all 81 tests, including the `vold.rc` patcher tests, the build tests that compile both targets with clang 18 and verify their ELF/symbol contract, four tests that drive the real mod install (donor init move, wrapper/shim install, ueventd symlink, exactly-once metadata, one `LD_PRELOAD`, idempotence, fail-closed), and a fifth test that invokes the real `scripts/internal/apply_modules.sh` driver and proves `SKIPUNZIP=1` suppresses the automatic system-tree copy. The mod now uses coreutils `od` instead of `xxd`. The same run’s **build** stopped before `make_rom.sh` because `.github/workflows/build.yml` installs a nonexistent versioned kernel package (`linux-modules-extra-7.0.0-1017-azure` in that run); the token lacks `workflow` scope to fix that line. No ROM build or device test exists yet.

## What is confirmed

- The current public artifact is `ZineROM_OneUI7_A115F-Port-A057F.zip`, **1,842,649,818 bytes**, SHA-256:

  ```text
  f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95
  ```

- The release was built from `nova-arch008/ZineROM-a11q` commit `23f8998f0200969f5748e9eb012d1dc52037a054` and passed 62 static/package tests. It has **not** been boot-tested on a phone.
- The port reaches Android second-stage init and starts vendor services, then fails at `cryptfs enablefilecrypto` with `Status(-8, EX_SERVICE_SPECIFIC)` and reboots to Recovery.
- `K00R` and QSEE `0x72000206` are **not** the root cause. Both appear on devices that boot normally.
- The failing operation is vold’s `fbeEnable()`, which calls `fscrypt_initialize_systemwide_keys()`. The failure is inside that function, not a missing `vdc` command.
- The A11 `/data` fstab uses `fileencryption=ice` with no key directory, consistent with the A11’s FBE v1 + inline-crypto layout.
- The donor `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`; all are `ELF32` ARM. The old “32-bit library missing” concern is closed.
- Donor Keystore2 contains Keymaster 4.0 support. The complete 64-bit HIDL client stack is present (`android.hardware.keymaster@3.0/4.0/4.1`, `libkeymaster4support.so`, `libkeymaster4_1support.so`, `libhidlbase.so`, `libkm_compat.so`, `libkm_compat_service.so`, `libkeymint_support.so`), and `hwservicemanager` is present. The remaining question is runtime KM4 registration/TA compatibility, not missing libraries.
- The effective FBE policy is **v1**: A11 vendor `ro.product.first_api_level=29`; donor system, system_ext, and product build.prop files do not override it.
- The Keymaster barrier is correctly placed, but the A11 Android 12 helper waits for Keystore2 registration, not KM4; `A115F_KEYMASTER_BARRIER_COMPLETE` is not proof that KM4 is ready.
- The A11 `keymaster-4-0` starts at `on init`, before `/vendor/firmware_mnt` is mounted, and is expected to abort with `QSEECom_start_app failed`. The decisive start is the system `on late-fs` `class_start early_hal`, after the `apnhlos` mount and before the imported vendor barrier.
- Samsung ODE/UCM is **not** engaged for this failure: donor `vold` consults `isUcmOdeMode()` only for `wrappedkey_v0`, and A11 `/data` uses plain `fileencryption=ice`. `/keydata` and `/keyrefuge` remain untouched.
- Only the `manus/a115f-vendor-fingerprint` branch at `23f8998f…` contains the barrier patcher. `manus/a115f-build-once` and `manus/a115f-bootability-preflight` do not, so a rebuild from either would silently lack the barrier.

## What is still unknown

1. Whether the source fix works at runtime: PID 1 keyring creation before policy load, `LD_PRELOAD` interposition in Samsung `vold`, keyring inheritance, and SELinux labels.
2. Whether key retrieval through KeyStorage/Keystore2/Keymaster fails **before** key installation on the device. The kernel-key-installation gap itself is confirmed at code level.
3. Whether the A11’s Keymaster 4.0 **HIDL** HAL registers after the late-fs restart and can be bound by Android 15 Keystore2 at runtime.
4. Whether `vendor.qseecomd` reaches `vendor.sys.listeners.registered` in the same boot as the barrier.
5. Whether `/data` was left in a partially encrypted/corrupt state by earlier attempts.
6. Whether a full ROM build completes after ISSUE-028 (the workflow package line) is fixed.

## What I need from you (device side)

The next step is **one clean diagnostic run**, not another build.

1. Confirm the exact ZIP SHA-256 is `f51460ef…` above.
2. Confirm you have a matching stock `A115FXXS6CWK2` package and can restore the phone.
3. Capture the full failed-boot log:
   - `adb logcat -b all -v threadtime -d`
   - `adb shell dmesg`
   - `adb shell getprop`
   - `/sys/fs/pstore/*`
   - `/cache/recovery/last_log` and `last_install` if Android does not stay up
4. Report the exact A11 CSC/firmware string and whether the bootloader is unlocked.
5. Do **not** wipe `/data`, `/keydata`, or `/keyrefuge`.

See [NEXT_DEVICE_TEST.md](NEXT_DEVICE_TEST.md) for the exact commands and the log lines that matter.

## Documents

- [RESUME.md](RESUME.md) — **start here after disconnecting.** Current blocker, audit state, and resume commands.
- [PORTING_STATUS.md](PORTING_STATUS.md) — milestones, confirmed facts, hypotheses, recovery prerequisites, and the 2026-11-04 checkpoint.
- [ISSUE_REGISTER.md](ISSUE_REGISTER.md) — every open/closed issue and the evidence behind it.
- [NEXT_DEVICE_TEST.md](NEXT_DEVICE_TEST.md) — the next device-only test and log capture procedure.

## Latest checkpoint

**2026-11-04:** the A11 Keymaster abort path and start order were decoded. The first `on init` KM4 start is expected to fail before `/vendor/firmware_mnt` is mounted; the late-fs `class_start early_hal` restart is the decisive one. The Samsung ODE/UCM branch was closed from donor vold disassembly. Only `23f8998f…` contains the barrier patcher, so `build-once` and `preflight` must not be used for a rebuild. No build, flash, or format is running. Resume from [RESUME.md](RESUME.md).

## Forks on zinerom48-tech

All three relevant public repositories are forked to this account:

- Source: <https://github.com/zinerom48-tech/ZineROM-a11q> (default branch `oneui7-a11q-boot`; parent `nova-arch008/ZineROM-a11q`)
- Release: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-boot-test> (default branch `main`; parent `nova-arch008/ZineROM-A115F-OneUI7-boot-test`)
- Firmware tool: <https://github.com/zinerom48-tech/SamFWDumper1> (default branch `0000`; parent `nova-arch008/SamFWDumper1`, itself a fork of `Xiatsuma/SamFWDumper`)

**Important:** a GitHub fork does not copy release assets. The 1.84 GB ZIP is **not** in the forks; it exists only on the upstream release at the asset URL below. The fork can still be used to rebuild from source.

## Links

- Source repository: <https://github.com/nova-arch008/ZineROM-a11q>
- Release repository: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test>
- Current asset: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip>

## Disclaimer

This is an experimental, unofficial port. It is not affiliated with Samsung or Google. Flashing modified firmware can brick the device, void warranties, and destroy data. Nothing here is a promise that the ROM will boot or that the device can be restored. Use a verified recovery path and proceed only with informed approval.
