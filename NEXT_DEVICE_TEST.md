# Next Device Test — A115F FBE Diagnostic

Last updated: 2026-10-29

This is the **only** device test that should be run next. It is a diagnostic run, not a fix. The current ZIP contains a Keymaster barrier; it does **not** fix or bypass the `enablefilecrypto` failure.

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
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.flash.locked
```

If ADB is unavailable, record the model/CSC/firmware from the phone’s About screen or the stock firmware package name.

## 3. Flash and boot once

1. Flash the exact ZIP using the documented recovery/OTA method.
2. Let it attempt one boot.
3. Do not repeatedly reboot, do not factory reset, and do not format anything.
4. When it reaches Recovery, stop and collect logs.

Expected failure (from the prior attempt): `cryptfs enablefilecrypto` → `Status(-8, EX_SERVICE_SPECIFIC)` → Rescue Party reason `enablefilecrypto_failedK00R`.

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
grep -iE "A115F_KEYMASTER_BARRIER|Waiting for Keymaster|Keymaster device ready" logcat-all.txt dmesg.txt
grep -iE "fscrypt_initialize_systemwide_keys|No mount point entry for /data|Unable to parse encryption options|EncryptionOptions not initialized" logcat-all.txt dmesg.txt
grep -iE "Failed to get ephemeral wrapped key|decryptWithKeystoreKey failed|decryptWithoutKeystore failed|encryptWithKeystoreKey failed|Version mismatch" logcat-all.txt dmesg.txt
grep -iE "vold|keystore2|keymaster|QSEECom|linker|fscrypt" logcat-all.txt dmesg.txt
```

On Windows, use `findstr /i /c:"..."` or open the files in a text editor.

## 6. What the result means

| Observation | Meaning | Next action |
|---|---|---|
| Barrier BEGIN, but no `Waiting for Keymaster device` | The helper did not start. | Check the helper service/SELinux, not FBE. |
| `Waiting for Keymaster device`, no `Keymaster device ready` | The helper started but never saw the A11 KM4 HAL. | Investigate HAL discovery/SELinux, not fstab. |
| Barrier COMPLETE, then `fscrypt_initialize_systemwide_keys` failure | The barrier is not the blocker; the failure is inside FBE key setup. | Use the vold/keystore2 lines to select the sub-step. |
| `decryptWithKeystoreKey failed` / `Failed to get ephemeral wrapped key` | KeyStorage/Keystore2/Keymaster key retrieval failed. | Investigate Keystore2 ↔ KM4 HAL, not the kernel. |
| `Unable to parse encryption options` / `No mount point entry for /data` | fstab/option parsing failed. | Re-check the `/data` fstab entry before touching Keymaster. |
| `install_storage_key`/kernel fscrypt error | Kernel key installation or policy failed. | Check A11 kernel fscrypt/ICE support. |
| `Version mismatch` | A key blob or policy version is incompatible. | Check FBE version and Samsung ODE key format. |

## 7. Report back

Include:

- Exact ZIP SHA-256.
- Phone model, CSC, firmware string, bootloader unlock/AVB state.
- Whether the phone reached Recovery or stayed in a boot loop.
- `logcat-all.txt`, `dmesg.txt`, `getprop.txt`, and pstore/recovery logs.
- The first line that shows the failure, with 20 lines before and after it.

Do not send only “it didn’t boot.” The exact vold/keystore2/keymaster line is what decides the next code change.

## 8. What not to do

- Do not flash a different ZIP or a “fixed” build yet.
- Do not disable FBE, edit the fstab, or add another Keymaster bridge by guesswork.
- Do not format `/data`, `/keydata`, or `/keyrefuge`.
- Do not claim the port is fixed from this diagnostic run. It is only meant to identify the failing sub-step.
