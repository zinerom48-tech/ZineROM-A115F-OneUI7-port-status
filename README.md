# ZineROM One UI 7 — Galaxy A11 (SM-A115F) Port Status

Public status board for the **SM-A115F / a11q** One UI 7 port based on the A05s `A057FXXUFDZH2` donor.

Maintained for **zinerom48-tech** so the latest evidence and next actions are available whenever there is internet access.

## Read this first

- **This is not a ROM download and not a flashable package.**
- **Current readiness: R0.** No successful boot has been observed.
- **Do not install the test ZIP** unless you have a verified stock `A115FXXS6CWK2` restore path and have explicitly approved the risk.
- Do **not** format `/data`, `/keydata`, or `/keyrefuge` to “fix” the boot loop. That destroys data and does not prove the cause.

## Progress right now (2026-10-29)

| Measure | Estimate | Meaning |
|---|---|---|
| Successful boot | **0%** | No completed boot, no boot animation, no Setup Wizard. |
| Boot stages reached | **~45%** | Bootloader → kernel → first-stage init → second-stage init → vendor services/HALs. It fails at `/data` file-based encryption before zygote and `system_server`. |
| Offline root-cause isolation | **~65%** | Failure narrowed to vold `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`. The exact failing sub-step is still unknown. |
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
- Donor Keystore2 contains Keymaster 4.0 support, but the **64-bit** Keymaster HIDL client has not been verified yet. The earlier “no Keymaster library” result was a truncated-image artifact, not evidence of absence.

## What is still unknown

1. Which sub-step of `fscrypt_initialize_systemwide_keys()` fails:
   - fstab option parsing,
   - device-key retrieval through KeyStorage/Keystore2/Keymaster,
   - kernel fscrypt key installation,
   - or writing the `/data` encryption options/references.
2. Whether the A11’s Keymaster 4.0 **HIDL** HAL can be bound by Android 15 Keystore2 at runtime.
3. Whether the A11’s Samsung ODE `/keydata`/`/keyrefuge` state is compatible with Android 15 vold.
4. Whether the effective FBE policy is v1 (likely) or v2 (product/system_ext properties still unread).
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

- [RESUME.md](RESUME.md) — **start here after disconnecting.** Exact offsets, hashes, running-task state, and resume commands.
- [PORTING_STATUS.md](PORTING_STATUS.md) — milestones, confirmed facts, hypotheses, recovery prerequisites, and the 2026-10-29 checkpoint.
- [ISSUE_REGISTER.md](ISSUE_REGISTER.md) — every open/closed issue and the evidence behind it.
- [NEXT_DEVICE_TEST.md](NEXT_DEVICE_TEST.md) — the next device-only test and log capture procedure.

## Latest checkpoint

**2026-10-29:** all current progress is saved here. No build, flash, format, or download is running. After this commit is pushed, it is safe to disconnect. Resume from [RESUME.md](RESUME.md); do not repeat the completed audit.

## Links

- Source repository: <https://github.com/nova-arch008/ZineROM-a11q>
- Release repository: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test>
- Current asset: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip>

## Disclaimer

This is an experimental, unofficial port. It is not affiliated with Samsung or Google. Flashing modified firmware can brick the device, void warranties, and destroy data. Nothing here is a promise that the ROM will boot or that the device can be restored. Use a verified recovery path and proceed only with informed approval.
