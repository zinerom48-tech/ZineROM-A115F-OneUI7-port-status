# Next Device Test — A115F FBE Diagnostic

Last updated: 2026-11-16

This is the **only** device test that should be run next. It is a diagnostic run, not a proven fix. The current ZIP contains a Keymaster barrier and kmsg markers; the prior `enablefilecrypto` failure log most likely came from a build before that barrier. The barrier waits for Keystore2 registration, not for KM4, so a completed barrier is not proof that the FBE key path is healthy.

**Updated 2026-11-16 — the current asset cannot succeed at FBE even with a healthy KM4.** The A11 4.9.227 kernel implements only legacy fscrypt key management, while donor `vold` is ioctl-only and donor `init` creates no keyring. Test this asset only to capture where the failure occurs; do not treat a successful boot as a possible outcome of this run. The fix source is compiled and host-tested locally (`manus/a115f-legacy-fscrypt@5651d76`), and its 76-test suite passes in CI (run `43913667988`), including compile and ELF checks of both targets, but the CI build stops before `make_rom.sh` on the workflow package defect tracked as ISSUE-028. It has not been built into a ROM, so there is still no fixed ZIP to test.

Two expectations to keep straight while reading the log:

- `keymaster-4-0` starts once at `on init`, before `/vendor/firmware_mnt` is mounted, and is **expected to abort** with `QSEECom_start_app failed`. That first abort is not the root cause.
- The decisive start is the system `on late-fs` `class_start early_hal`, which happens after the `apnhlos` mount and before the imported vendor barrier. The log must show whether that second start registers KM4 with TEE security level.

## Before you start

Stop if any of these is false:

- [ ] The phone is an **SM-A115F** with bootloader **`A115FXXS6CWK2`**.
- [ ] You have a matching stock `A115FXXS6CWK2` firmware package for your exact CSC.
- [ ] You can enter Download Mode and restore stock firmware with Odin/Heimdall.
- [ ] You have backed up anything important.
- [ ] You accept that this can brick the phone or destroy data.
- [ ] You will **not** format `/data`, `/keydata`, or `/keyrefuge`.

If you cannot restore the phone, do not flash the test ZIP.

## 1. Verify the ZIP

Use the exact current asset:

```text
ZineROM_OneUI7_A115F-Port-A057F.zip
1,842,649,818 bytes
SHA-256 f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95
```

Windows:

```bat
certutil -hashfile ZineROM_OneUI7_A115F-Port-A057F.zip SHA256
```

Linux/macOS:

```sh
sha256sum ZineROM_OneUI7_A115F-Port-A057F.zip
```

Do not test a different ZIP. The older `29384d92…` asset in `BOOT_PROGRESS.md` is stale and may predate the barrier.

## 2. Record the device identity

From Android (if it boots) or Download Mode/Recovery:

```sh
adb shell getprop ro.product.model
adb shell getprop ro.boot.em.model
adb shell getprop ro.boot.bootloader
adb shell getprop ro.csc.sales_code
adb shell getprop ro.build.display.id
adb shell getprop ro.product.first_api_level
adb shell getprop ro.crypto.state
adb shell getprop ro.crypto.type
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.flash.locked
adb shell getprop init.svc.keymaster-4-0
adb shell getprop init.svc.vendor.qseecomd
adb shell getprop vendor.sys.listeners.registered
adb shell getprop init.svc.wait_for_keymaster
```

If ADB is unavailable, record the model/CSC/firmware from the phone’s About screen or the stock firmware package name.

## 3. Flash and boot once

1. Flash the exact ZIP using the documented recovery/OTA method.
2. Let it attempt one boot.
3. Do not repeatedly reboot, do not factory reset, and do not format anything.
4. When it reaches Recovery, stop and collect logs.

Prior attempt (pre-barrier build): `cryptfs enablefilecrypto` → `Status(-8, EX_SERVICE_SPECIFIC)` → Rescue Party reason `enablefilecrypto_failedK00R`. If this run fails, the important question is whether the barrier completed and whether Keystore2 reported KM4 before the failure.

The 2026-11-16 analysis shows that this run must fail at key installation regardless of KM4: donor `vold` uses an ioctl the A11 kernel does not implement, and donor `init` provides no keyring. The log is still useful because it shows whether key *retrieval* failed earlier, which determines whether the wrapper/shim fix is sufficient or whether KM4/Keystore2 also needs work.

The first `QSEECom_start_app failed` in the new log is not the same failure as the prior FBE error. It is expected before `/vendor/firmware_mnt` is mounted; keep the full log so the second `keymaster-4-0` start can be evaluated.

## 4. Capture the logs

### If Android or Recovery exposes ADB

```sh
adb devices
adb shell getprop > getprop.txt
adb shell dmesg > dmesg.txt
adb logcat -b all -v threadtime -d > logcat-all.txt
adb pull /sys/fs/pstore pstore
adb pull /cache/recovery/last_log last_log.txt
adb pull /cache/recovery/last_install last_install.txt
```

If `adb logcat` is empty because Android never reaches the framework, the kernel log and pstore are the important ones.

### If ADB is unavailable

Use a custom recovery with a terminal/file manager, or copy these files to an SD card:

```text
/sys/fs/pstore/*
/proc/last_kmsg
/cache/recovery/last_log
/cache/recovery/last_install
```

Do not wipe these partitions.

## 5. Grep these exact strings

Keep the **full** log, not only the grep output. The surrounding lines identify the failing branch.

```sh
grep -iE "starting service 'keymaster-4-0'|QSEECom_start_app failed|KeymasterUtils|mount_all /vendor/etc/fstab.qcom" logcat-all.txt dmesg.txt
grep -iE "vendor.sys.listeners.registered|QSEECOM DAEMON RUNNING|IQSEECom registerAsService" logcat-all.txt dmesg.txt
grep -iE "A115F_KEYMASTER_BARRIER|Waiting for Keymaster|Keymaster device ready|Vold unable to get security level" logcat-all.txt dmesg.txt
grep -iE "Successfully registered Keystore 2.0 service|Skipping keymaster compat|found android.hardware.keymaster|Failed to get service for|No secure Keymaster implementation" logcat-all.txt dmesg.txt
grep -iE "keymaster-4-0|qseecom|QSEECom|android.hardware.keymaster@4.0" logcat-all.txt dmesg.txt
grep -iE "fscrypt_initialize_systemwide_keys|No mount point entry for /data|Unable to parse encryption options|EncryptionOptions not initialized" logcat-all.txt dmesg.txt
grep -iE "Failed to get ephemeral wrapped key|decryptWithKeystoreKey failed|decryptWithoutKeystore failed|encryptWithKeystoreKey failed|Version mismatch" logcat-all.txt dmesg.txt
grep -iE "Failed to install fscrypt key|Failed to open /data to install key|Failed to evict fscrypt key|Failed to get status for fscrypt key|Inappropriate ioctl for device" logcat-all.txt dmesg.txt
grep -iE "A115F_FSCRYPT|fscrypt.*keyring|Creating new key in|No key found in" logcat-all.txt dmesg.txt
grep -iE "vold|keystore2|keymaster|linker|fscrypt" logcat-all.txt dmesg.txt
```

On Windows, use `findstr /i /c:"..."` or open the files in a text editor.

## 6. What the result means

| Observation | Meaning | Next action |
|---|---|---|
| First `keymaster-4-0` start, then `QSEECom_start_app failed` before the firmware mount | Expected pre-`/vendor/firmware_mnt` abort; not the root cause by itself. | Keep reading; find the second start after `on late-fs`. |
| No second `keymaster-4-0` start after `mount_all --early` | `class_start early_hal` did not reach KM4, or init parsing/SELinux blocked it. The rc has no `interface` declaration, so there is no lazy-start fallback. | Check init class/action parsing and SELinux, not FBE. |
| Second start aborts again with `QSEECom_start_app failed` | `/vendor/firmware_mnt` or the TA image path is still unavailable; KM4 cannot load its TA. | Check the `apnhlos` mount and `/vendor/firmware_mnt/image/keymaster64`. |
| No `vendor.sys.listeners.registered` | `vendor.qseecomd`/`init.qti.qseecomd.sh` gate did not complete; QSEECom listeners are not ready. | Check qseecomd and `qseecom-service`, not fstab. |
| No `A115F_KEYMASTER_BARRIER_BEGIN` | The barrier action or helper service did not run. | Check init service state/SELinux, not FBE. |
| BEGIN, but no `A115F_KEYMASTER_BARRIER_COMPLETE` | `wait_for_keymaster` hung, usually before Keystore2 registered. | Check Keystore2/servicemanager; do not touch fstab. |
| COMPLETE + `Vold unable to get security level from keystore2` | Keystore2 is up, but KM4 is absent or unusable. `BARRIER_COMPLETE` does **not** prove KM4 readiness. | Inspect `keymaster-4-0`, `qseecom`, HIDL registration, and SELinux. |
| COMPLETE + `Keymaster device ready` with no security-level error, then FBE failure | KM4 may be present; the failure is in key generation/profile or KeyStorage. | Use the vold/keystore2 error code to select the sub-step. |
| COMPLETE + `found android.hardware.keymaster@4.0…` (seclevel TEE) + FBE success | The barrier likely fixed the original race. | Continue to the next boot stage; still not a complete boot claim. |
| COMPLETE + `found android.hardware.keymaster@4.0…` + FBE failure | KM4 is discovered but cannot satisfy the Android 15 request. | Investigate the KM4 TA/capability and Keystore2 error, not fstab. |
| `decryptWithKeystoreKey failed` / `Failed to get ephemeral wrapped key` | KeyStorage/Keystore2/Keymaster key retrieval failed. | Investigate the Keystore2 ↔ KM4 path. |
| `Unable to parse encryption options` / `No mount point entry for /data` | fstab/option parsing failed. | Re-check the `/data` fstab entry before touching Keymaster. |
| `install_storage_key`/kernel fscrypt error | Kernel key installation or policy failed. | Check A11 kernel fscrypt/ICE support. |
| `Failed to install fscrypt key to /data: Inappropriate ioctl for device` | **Confirmed code-level root cause.** Donor `vold` issued `FS_IOC_ADD_ENCRYPTION_KEY`, and the A11 4.9 kernel returned `ENOTTY`. The current asset cannot install a key; the wrapper/shim fix is required. | No further diagnosis needed for this sub-step; build the fix from ISSUE-024/025. |
| `Failed to open /data to install key` | `vold` could not open the mountpoint; the key install never reached the ioctl. | Check `/data` mount state and the fstab, but do not format. |
| `Creating new key in` / `No key found in` with no `Failed to install fscrypt key` line | Key retrieval (KeyStorage/Keystore2/KM4) failed before installation. | The KM4/Keystore2 branch is the earlier blocker; fixing legacy fscrypt alone will not boot. |
| `A115F_FSCRYPT_WRAPPER` kmsg lines or `A115F_FSCRYPT` logcat messages | Only present in a fixed build, not in `f51460ef…`. Their absence here is expected and is not evidence. | Use them only when testing the wrapper/shim build. |
| `Version mismatch` | A key blob or policy version is incompatible. | Check FBE version and Keymaster key format. Samsung ODE is already excluded for this `/data` entry. |

Do not treat `Keymaster device ready` alone as success: the A11 helper logs it even when `getSecurityLevel(TE)` failed, and then exits 0.

## 7. Report back

Include:

- Exact ZIP SHA-256.
- Phone model, CSC, firmware string, bootloader unlock/AVB state.
- Whether the phone reached Recovery or stayed in a boot loop.
- `logcat-all.txt`, `dmesg.txt`, `getprop.txt`, and pstore/recovery logs.
- The barrier marker order and timestamps: first `keymaster-4-0` start → expected `QSEECom_start_app failed` → `mount_all --early` → qseecomd/`vendor.sys.listeners.registered` → second `keymaster-4-0` start → BEGIN → `Waiting for Keymaster device` → `Keymaster device ready` → COMPLETE.
- Any `Vold unable to get security level from keystore2` line and the Keystore2 KM4 discovery/security-level lines.
- The first line that shows the failure, with 20 lines before and after it.

Do not send only “it didn’t boot.” The exact barrier/Keystore2/KM4/vold line is what decides the next code change.

## 8. What not to do

- Do not flash a different ZIP or a “fixed” build yet.
- Do not expect this asset to boot. It cannot install a legacy fscrypt key on the A11 4.9 kernel; a failed FBE attempt here does not by itself prove the KM4/Keystore2 path is broken.
- Do not interpret the absence of `A115F_FSCRYPT_WRAPPER` kmsg lines or `A115F_FSCRYPT` logcat messages as a fault; those markers exist only in the planned fixed build.
- Do not disable FBE, edit the fstab, or add another Keymaster bridge by guesswork.
- Do not move the barrier into vendor `on post-fs-data`; system `installkey /data` runs before that trigger.
- Do not treat `A115F_KEYMASTER_BARRIER_COMPLETE` as proof that KM4 is ready.
- Do not treat the first `QSEECom_start_app failed` as the root cause; it is the expected pre-firmware start.
- Do not rebuild from `manus/a115f-build-once` or `manus/a115f-bootability-preflight`; neither contains the barrier patcher.
- Do not format `/data`, `/keydata`, or `/keyrefuge`.
- Do not claim the port is fixed from this diagnostic run. It is only meant to identify the failing sub-step.

## 9. Fixed-build test — only after the build exists and recovery is verified

Do **not** use this section until a ZIP built from `manus/a115f-legacy-fscrypt@5651d76` (or a later inspected commit) exists, its system image has been statically inspected, and the stock restore path is verified. No such ZIP exists yet: CI run `43913667988` passed all 76 tests (including compile and ELF checks of both targets) but stopped before `make_rom.sh` at `Set up build environment` because of ISSUE-028. The fixed build does not modify the boot image; recovery means reflashing the previous port ZIP or the stock system image.

Before flashing, record the ZIP SHA-256, the build log, and the SHA-256 of `/system/bin/init`, `/system/bin/init.real`, and `/system/lib64/liba115f_fscrypt_shim.so`.

Expected fixed-build evidence, in order:

```text
A115F_FSCRYPT_WRAPPER: ENTER
A115F_FSCRYPT_WRAPPER: SESSION_OK <serial>
A115F_FSCRYPT_WRAPPER: FSCRYPT_OK <serial>
A115F_FSCRYPT_WRAPPER: EXEC /system/bin/init.real
A115F_FSCRYPT_WRAPPER: UEVENTD_PASSTHROUGH        (when ueventd starts)
logcat tag A115F_FSCRYPT: SHIM_LOADED
logcat tag A115F_FSCRYPT: ADD_OK
logcat tag A115F_FSCRYPT: ADD_OK
logcat tag A115F_FSCRYPT: ADD_OK
```

Interpretation:

| Observation | Meaning | Next action |
|---|---|---|
| No `WRAPPER_ENTER` | The wrapper is not `/system/bin/init`, or first-stage init did not exec it. | Re-inspect the built system image; restore the previous ZIP. |
| `SESSION_FAIL` / `FSCRYPT_FAIL` | A raw keyring syscall failed. | Check `CONFIG_KEYS`, seccomp, SELinux, and the kernel log; the shim fallback may still create the keyring. |
| `FSCRYPT_OK` but no `SHIM_LOADED` | `LD_PRELOAD` was ignored or `vold.rc` was not patched. | Check `vold.rc`, linker namespace, and `AT_SECURE`. |
| `SHIM_LOADED` but no `ADD_OK`/`ADD_FAIL` | vold did not reach `installKey()`, or the key install failed before the ioctl. | Read the preceding Keystore2/vold error; this is not necessarily the shim. |
| `ADD_OK` ×3 but `/data` still fails | The keys were installed but `request_key()` cannot resolve them. | Check session keyring inheritance, keyring labels, AVC denials, and `CONFIG_KEYS`. |
| `ADD_FAIL EACCES` | SELinux key permission or label mismatch. | Capture the AVC; do not add a blind rule. |
| `ADD_FAIL ENOKEY` | The `fscrypt` keyring is missing and the fallback failed. | Check the wrapper markers and the shim constructor. |
| No `UEVENTD_PASSTHROUGH` and ueventd fails | The symlink or `argv[0]` handling broke. | Restore the previous system image; do not unlink the symlink. |
| `/data` mounts, zygote/system_server starts | The FBE blocker is cleared on this device. | Continue to the next boot stage; this is not a stability claim. |

Do not format `/data` to force a pass. If the device boot-loops, restore the previous ZIP or stock system image and keep the full log.
