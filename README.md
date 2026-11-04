# ZineROM One UI 7 — Galaxy A11 (SM-A115F) Port Status

Public status board for the **SM-A115F / a11q** One UI 7 port based on the A05s `A057FXXUFDZH2` donor.

Maintained for **zinerom48-tech** so the latest evidence and next actions are available whenever there is internet access.

## Read this first

- **This is not a ROM download and not a flashable package.**
- **Current readiness: R0.** No successful boot has been observed.
- **Do not install the test ZIP** unless you have a verified stock `A115FXXS6CWK2` restore path and have explicitly approved the risk.
- Do **not** format `/data`, `/keydata`, or `/keyrefuge` to “fix” the boot loop. That destroys data and does not prove the cause.

## Progress right now (2026-11-03)

| Measure | Estimate | Meaning |
|---|---|---|
| Successful boot | **0%** | No completed boot, no boot animation, no Setup Wizard. |
| Boot stages reached | **~45%** | Bootloader → kernel → first-stage init → second-stage init → vendor services/HALs. It fails at `/data` file-based encryption before zygote and `system_server`. |
| Offline root-cause isolation | **~80%** | Failure narrowed to vold `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`. The exact failing sub-step is still unknown; the barrier build has not been device-tested. |
| Validated fix | **0%** | No fix has been built, flashed, or tested. |
| Device validation | **0%** | No physical A11 available. |

These are bookkeeping estimates, **not** a probability that the ROM will boot. The remaining Android framework/UI work is larger than the stage count suggests.

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
- Donor Keystore2 contains Keymaster 4.0 support. The 64-bit HIDL compat stack is present (`libkm_compat.so`, `libkm_compat_service.so`, `libkeymint_support.so`), and `hwservicemanager` is present. The remaining question is runtime KM4 registration/TA compatibility, not missing libraries.
- The effective FBE policy is **v1**: A11 vendor `ro.product.first_api_level=29`; donor system, system_ext, and product build.prop files do not override it.
- The Keymaster barrier is correctly placed, but the A11 Android 12 helper waits for Keystore2 registration, not KM4; `A115F_KEYMASTER_BARRIER_COMPLETE` is not proof that KM4 is ready.

## What is still unknown

1. Which sub-step of `fscrypt_initialize_systemwide_keys()` fails:
   - fstab option parsing,
   - device-key retrieval through KeyStorage/Keystore2/Keymaster,
   - kernel fscrypt key installation,
   - or writing the `/data` encryption options/references.
2. Whether the A11’s Keymaster 4.0 **HIDL** HAL registers and can be bound by Android 15 Keystore2 at runtime.
3. Whether the A11’s Samsung ODE `/keydata`/`/keyrefuge` state is compatible with Android 15 vold.
4. Whether the prior failure log came from a pre-barrier build (leading hypothesis) and whether the current barrier build behaves differently.
5. Whether `/data` was left in a partially encrypted/corrupt state by earlier attempts.

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
- [PORTING_STATUS.md](PORTING_STATUS.md) — milestones, confirmed facts, hypotheses, recovery prerequisites, and the 2026-11-03 checkpoint.
- [ISSUE_REGISTER.md](ISSUE_REGISTER.md) — every open/closed issue and the evidence behind it.
- [NEXT_DEVICE_TEST.md](NEXT_DEVICE_TEST.md) — the next device-only test and log capture procedure.

## Latest checkpoint

**2026-11-03:** the init import-order/deadlock theory was retracted; the barrier is correctly placed and is present in the current `f51460ef…` asset. The prior failure log most likely predates the barrier. The barrier waits for Keystore2, not KM4, so the next device log must capture Keystore2/KM4 discovery as well as the barrier markers. No build, flash, or format is running. Resume from [RESUME.md](RESUME.md).

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
