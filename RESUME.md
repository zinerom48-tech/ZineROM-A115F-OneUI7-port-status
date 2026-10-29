# RESUME — A115F One UI 7 Port

Snapshot: **2026-10-29**, pre-disconnect checkpoint.

Status board: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-port-status>  
Branch: `main`

This file is the exact resume point. It records what is done, what is blocked, which files are partial, and the commands needed to continue without repeating the audit.

## 1. Is it safe to disconnect?

**Yes — after this checkpoint is committed and pushed.** No build, flash, format, or download is running; the only Git activity is this checkpoint commit/push. The last range download was reset; its partial file is saved.

Do **not** delete these resume files:

- `system_big5.dat`
- `system_big5.br`
- `system_next5.br`

## 2. Exact current blocker

The boot reaches Android second-stage init and vendor services, then fails at `/data` file-based encryption:

```text
cryptfs enablefilecrypto -> Status(-8, EX_SERVICE_SPECIFIC)
Rescue Party reason: enablefilecrypto_failedK00R
```

The exact failing operation is vold’s AIDL `fbeEnable()`, which calls `fscrypt_initialize_systemwide_keys()`. The failing sub-step is still unknown. Candidates, in order of current suspicion:

1. `retrieveOrGenerateKey()` — KeyStorage/Keystore2/Keymaster device-key retrieval. **Leading unverified branch:** Android 15 Keystore2 must bind the A11’s Keymaster 4.0 HIDL HAL.
2. `install_storage_key()` — kernel fscrypt key installation/policy.
3. Writing `/data` encryption options/references.
4. `init_data_file_encryption_options()` — fstab parsing. **Least likely:** the A11 `/data` entry is structurally valid (`fileencryption=ice`, no key directory).

There is **no successful boot**, no boot animation, and no Setup Wizard. No fix has been built or tested.

## 3. What is already done

- Public source and release confirmed; current artifact hash is `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95`.
- The current ZIP has the `A115F_KEYMASTER_BARRIER_BEGIN` / `wait_for_keymaster` / `A115F_KEYMASTER_BARRIER_COMPLETE` diagnostic sequence.
- The release is **not** boot-tested; it passed 62 static/package tests.
- Donor system is not 64-bit-only. `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`, all `ELF32` ARM. ISSUE-014 is closed.
- The vold path is confirmed: `installkey /data` → `vdc cryptfs enablefilecrypto` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`.
- `ro.crypto.type=file` is set by `fs_mgr` from the fstab; the A11 vendor init does not call `enablefilecrypto` directly.
- A11 `/data` fstab: `fileencryption=ice`, no `keydirectory=`, `latemount`, `check`, `quota`, `checkpoint=fs`.
- A11 `/keydata` and `/keyrefuge`: `ext4`, `fileencryption=ice,nofail`.
- Donor Keystore2 has Keymaster 4.0 support (`KEY_MASTER_V4_0`, `IKeymasterDevice`, `km_compat`) and Samsung KeyMint code.
- Donor platform policy includes `keystore` in `hal_keymaster_client` and `hal_keymint_client`.
- `K00R` and QSEE `0x72000206` are **not** root causes; both occur on booting devices.
- The `/oem` zero-superblock mount is a stock A11 init action; boot continues past it.
- The earlier empty `/system/lib64` listing was a **truncated-image artifact**, not proof that the 64-bit Keymaster client is missing.

## 4. Build results

- **No build was run in this session.**
- No new ZIP was produced.
- No local build has been reproduced.
- The only artifact is the public `f51460ef…` ZIP from Actions run `#40494388352`, commit `23f8998f…`.
- That artifact is not boot-tested.

## 5. Partial-image resume state

The full donor system partition is `3,213,774,848` bytes. The current decompressed prefix is only `1,324,875,776` bytes.

| File | Size | Meaning |
|---|---:|---|
| `system_big5.br` | `656,441,987` | Compressed system prefix, compressed-relative bytes `0..656,441,986` |
| `system_big5.dat` | `1,324,875,776` | Decompressed system prefix from `system_big5.br` |
| `system_next5.br` | `12,731,492` | Partial next range; starts at ZIP absolute `817,505,358` (compressed-relative `656,441,987`) |

Hashes:

```text
system_big5.br   ff872e36e4c8902f88ac855e3a682e2fae36387501ba16d842d6e7f05b4cda5e
system_next5.br  2e9e05e9450e2bedb838e9ac609b5618b3b19d7afae3926a6423122195ebd223
```

Important alignment facts:

- ZIP `system.new.dat.br` data starts at absolute `161,063,371`.
- There is a 2-byte gap at ZIP absolute `679,498,827..679,498,829` (compressed-relative `518,435,456..518,435,457`).
- `system_next4.br` starts at compressed-relative `582,435,459`; its first `37,523,456` bytes were already present in `system_big3.br`. Only its non-overlapping tail was appended to make `system_big5.br`.
- `system_next5.br` begins at compressed-relative `656,441,987`; the saved partial covers `12,731,492` bytes, so resume at absolute `830,236,850`.
- Brotli will report `corrupt input` at the truncated end while still writing partial output. That is expected for a prefix.

The `/system/lib64` directory inode is `2690`. Its extent leaf is at physical block `393,336`, byte offset `1,611,104,256`. The current prefix ends at `1,324,875,776`, so more decompressed data is required.

## 6. Next offline command — finish `/system/lib64`

Asset URL:

```text
https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip
```

Resume the partial range:

```bat
curl.exe -L --fail --retry 5 -r 830236850-997505357 -o system_next5b.br https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip
```

If `system_next5.br` is corrupt or the wrong size, re-download the full range instead:

```bat
curl.exe -L --fail --retry 5 -r 817505358-997505357 -o system_next5.br https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip
```

Then combine and list:

```bat
copy /b system_big5.br+system_next5.br+system_next5b.br system_big6.br
brotli\brotli.exe -d -c system_big6.br > system_big6.dat 2> brotli_big6_err.txt
AuditTool.exe ext4ls system_big6.dat /system/lib64 1 > system_lib64_big6.txt
cscript //nologo searchfile.vbs system_lib64_big6.txt keymaster
cscript //nologo searchfile.vbs system_lib64_big6.txt keymint
```

If the listing still ends at `read beyond end of image`, repeat with the next ZIP range and append. Do not conclude that a library is absent from a truncated listing.

If `/system/lib64` is available, verify:

```text
android.hardware.keymaster@4.0.so
libkeymaster4support.so
android.hardware.security.keymint-V1-ndk_platform.so
android.hardware.security.keymint-V2-ndk.so
libkeymint_support.so
```

Also inspect Keystore2’s `DT_NEEDED` entries:

```bat
ElfInfo.exe donor_keystore2
```

## 7. Next offline command — product properties

`product_full.dat` is only `83,886,080` bytes; the product partition is `312,815,616` bytes. `/product/etc/build.prop` is still outside the decompressed prefix. After extending the product image:

```bat
AuditTool.exe ext4cat product_full.dat /etc/build.prop product_build.prop.txt
```

The goal is to confirm whether product or system_ext overrides `ro.product.first_api_level` above `29`, which would select FBE policy v2 and make the A11 4.9 kernel’s v2 support a boot-critical risk.

Do not block the device test on this audit.

## 8. Next device command — capture the decisive log

Only run this with a verified stock `A115FXXS6CWK2` restore path and explicit approval. Do **not** format `/data`, `/keydata`, or `/keyrefuge`.

```sh
adb devices
adb shell getprop > getprop.txt
adb shell dmesg > dmesg.txt
adb logcat -b all -v threadtime -d > logcat-all.txt
adb pull /sys/fs/pstore pstore
adb pull /cache/recovery/last_log last_log.txt
adb pull /cache/recovery/last_install last_install.txt
```

Grep for:

```text
A115F_KEYMASTER_BARRIER_BEGIN
Waiting for Keymaster device
Keymaster device ready
A115F_KEYMASTER_BARRIER_COMPLETE
fscrypt_initialize_systemwide_keys
No mount point entry for /data
Unable to parse encryption options
EncryptionOptions not initialized
Failed to get ephemeral wrapped key
decryptWithKeystoreKey failed
decryptWithoutKeystore failed
encryptWithKeystoreKey failed
Version mismatch
```

The exact vold/keystore2/keymaster line selects the next code change. Do not edit fstab or add another Keymaster bridge before that line is known.

## 9. Do not repeat or do

- Do not rebuild or publish another ZIP before the FBE sub-step is identified.
- Do not format `/data`, `/keydata`, or `/keyrefuge`.
- Do not treat `K00R` or QSEE `0x72000206` as the root cause.
- Do not treat a truncated ext4 listing as proof that a library is missing.
- Do not claim the port boots from static tests or package validation alone.

## 10. Resume checklist

- [ ] Read this file and `PORTING_STATUS.md`.
- [ ] Confirm the status repo is at the latest commit.
- [ ] If continuing offline, verify `system_big5.br` and `system_next5.br` sizes/hashes above.
- [ ] Resume the ZIP range at absolute `830,236,850` or re-download `817505358-997505357`.
- [ ] Extend the image, list `/system/lib64`, and check the Keymaster/KeyMint libraries.
- [ ] If the phone is available, run the one device test in section 8 and capture the full logs.
