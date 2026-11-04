# RESUME — A115F One UI 7 Port

Snapshot: **2026-11-03**, Keymaster-barrier audit checkpoint.

Status board: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-port-status>
Working branch for this checkpoint: `audit/2026-11-03-keymaster-barrier` (do not overwrite `main`).

Forks on the same account:

- Source: <https://github.com/zinerom48-tech/ZineROM-a11q>
- Release: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-boot-test>
- Firmware tool: <https://github.com/zinerom48-tech/SamFWDumper1>

GitHub forks do **not** copy release assets; the 1.84 GB ZIP remains only at the upstream release URL.

## 1. Is it safe to disconnect?

**Yes.** No build, flash, format, or download is running. The only pending work is committing/pushing this documentation checkpoint and continuing static audits. The partial source clone at `C:\Users\claymi5433\ZineROM-a11q-src` is stalled and should be abandoned; use raw GitHub files/API instead. Do not delete the audit files listed in section 5.

## 2. Exact current blocker

The boot reaches Android second-stage init and vendor services, then fails at `/data` file-based encryption:

```text
cryptfs enablefilecrypto -> Status(-8, EX_SERVICE_SPECIFIC)
Rescue Party reason: enablefilecrypto_failedK00R
```

The exact failing operation is vold’s AIDL `fbeEnable()`, which calls `fscrypt_initialize_systemwide_keys()`. The failing sub-step is still unknown. Candidates:

1. `retrieveOrGenerateKey()` — KeyStorage/Keystore2/Keymaster device-key retrieval. **Leading unverified branch.**
2. `install_storage_key()` — kernel fscrypt key installation/policy.
3. Writing `/data` encryption options/references.
4. `init_data_file_encryption_options()` — fstab parsing. **Least likely:** the A11 `/data` entry is structurally valid (`fileencryption=ice`, no key directory).

**Leading root-cause hypothesis:** the only raw failure log came from a build **before** the Keymaster barrier. The barrier was added on 2026-10-25 and is present in the current `f51460ef…` asset, but no barrier-containing build is known to have been device-tested.

**Important limitation:** the barrier runs the A11 Android 12 `wait_for_keymaster`, which waits for Keystore2 registration, not KM4. It logs `getSecurityLevel` failures but still exits 0. Therefore `A115F_KEYMASTER_BARRIER_COMPLETE` does **not** prove KM4 is usable.

There is **no successful boot**, no boot animation, and no Setup Wizard. No fix has been built or tested.

## 3. What is already done

- Public source and release confirmed; current artifact hash is `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95`, size `1,842,649,818` bytes.
- Current ZIP contains the `A115F_KEYMASTER_BARRIER_BEGIN` / `exec_start wait_for_keymaster` / `A115F_KEYMASTER_BARRIER_COMPLETE` / `mount_all --late` sequence. Released vendor `init.target.rc` MD5 is `e111ab7f49ebcc8fa70d3253ccba9bce`.
- Build timeline: `1103fc00…` (prior diagnostic ZIP, no barrier) → `190586c…` (barrier, 2026-10-25) → `23f8998f…` (markers, 2026-10-27) → current asset `f51460ef…` built from `23f8998f…`.
- The release is **not** boot-tested; it passed 62 static/package tests.
- The earlier barrier-deadlock theory is **retracted**. AOSP `ImportParser::EndFile()` parses imports at end-of-file, so system `on late-fs` `class_start early_hal` runs before the imported vendor `on late-fs` barrier. The barrier is correctly placed.
- Effective FBE policy is **v1**: A11 vendor `ro.product.first_api_level=29`; donor system, system_ext, and product build.prop files do not override it.
- A11 kernel is Linux/arm64 `4.9.227` with fscrypt/ICE/XTS/PFK support; the earlier v2 kernel-support risk is closed.
- Donor system is not 64-bit-only. `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`, all `ELF32` ARM. ISSUE-014 is closed.
- Donor `/system/lib64` contains `libkm_compat.so`, `libkm_compat_service.so`, `libkeymint_support.so`, and KeyMint V1/V2/V3 libraries. `libkm_compat.so` includes the HIDL Keymaster 3.x/4.x fallback; it skips HIDL only if the HIDL service manager is unavailable.
- `hwservicemanager` is present at `/system/system_ext/bin/hwservicemanager`; `/system_ext` is a symlink to `/system/system_ext`.
- Android 15 FCM 202404 and FCM 8 contain no Keymaster/KeyMint entries; VINTF is not rejecting the A11 KM4 HAL.
- A11 vendor SELinux defines `hal_keymaster_qti`/`hal_keymaster_qti_exec`; KM4 is labeled correctly; donor policy includes `keystore` in `hal_keymaster_client`.
- The vold path is confirmed: `installkey /data` → `vdc cryptfs enablefilecrypto` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`.
- `ro.crypto.type=file` is set by `fs_mgr` from the fstab; the A11 vendor init does not call `enablefilecrypto` directly.
- A11 `/data` fstab: `fileencryption=ice`, no `keydirectory=`, `latemount`, `check`, `quota`, `checkpoint=fs`.
- A11 `/keydata` and `/keyrefuge`: `ext4`, `fileencryption=ice,nofail`.
- `K00R` and QSEE `0x72000206` are **not** root causes; both occur on booting devices.
- The `/oem` zero-superblock mount is a stock A11 init action; boot continues past it.
- Full donor system image reconstructed: `system_full.dat`, exactly `3,213,774,848` bytes. The earlier partial-image resume state is obsolete.
- Full donor product stream reconstructed: `product_final.dat`, exactly `312,815,616` bytes; `/product/etc/build.prop` has no `first_api_level`, crypto, or zygote override.
- `vndservicemanager` is not in the donor system, but the A11 vendor provides `/vendor/bin/vndservicemanager` and `/vendor/etc/init/vndservicemanager.rc`, so donor `init.rc` can start it. This branch is closed.
- Samsung ODE/UCM compatibility remains open: donor vold/libfscrypt reference `/efs/sec_efs/ucm_ode_mode`, `/data/sec_backup_v2`, `/metadata/sec_backup`, and `/sec_backup_unencrypted`; A11 mounts `/keydata` and `/keyrefuge` with file encryption.

## 4. Build results

- **No build was run in this session.**
- No new ZIP was produced.
- No local build has been reproduced.
- The only artifact is the public `f51460ef…` ZIP from Actions run `#40494388352`, commit `23f8998f…`.
- That artifact is not boot-tested.
- Do not build or publish another ZIP until the root cause or a decisive device-log branch is established.

## 5. Audit workspace state

The audit workspace is `C:\Users\claymi5433\AppData\Local\Temp\opencode`.

Important files:

| File | Meaning |
|---|---|
| `system_full.dat` | Complete donor system image, `3,213,774,848` bytes |
| `product_final.dat` | Complete donor product stream, `312,815,616` bytes |
| `vendor.new.dat` | Complete donor vendor image |
| `vendor_init_target_zip.rc` | Released vendor `init.target.rc`; MD5 `e111ab7f49ebcc8fa70d3253ccba9bce` |
| `donor_init_full.rc` | Donor system `init.rc`; `class_start early_hal` before imported vendor actions |
| `wfk12.cpp` / `km12.cpp` | A11 Android 12 `wait_for_keymaster` / `Keymaster` source |
| `ssn.rs` | Android 15 Keystore2 shared-secret negotiation |
| `ks2main.rs` | Android 15 Keystore2 main; registers after negotiation |
| `kmcompat.cpp` | Android 15 `libkm_compat` HIDL KM4 fallback |
| `parser15.cpp` / `importparser15.cpp` | AOSP init import-order proof |
| `wait_for_keymaster_rel.bin` | Released 32-bit helper; SHA-256 `0e64ac8862492b8f7455a5320189c754dc8f3fef720dbf0b0f71849cb86811c9` |

The partial clone `C:\Users\claymi5433\ZineROM-a11q-src` has a stale `tmp_pack_dxxDeC` and `shallow.lock`; do not resume it. Use raw GitHub/API fetches.

## 6. Next offline work

1. **Samsung ODE/UCM audit:** determine whether donor vold/KeyStorage actually consumes `/keydata`/`/keyrefuge` and `/efs/sec_efs/ucm_ode_mode`, or whether A11’s ODE state is irrelevant to the first `/data` FBE setup. Do not format these partitions.
2. **Service-discovery check:** `vndservicemanager` is already confirmed in the A11 vendor; no further offline work is needed unless a device log shows vendor HIDL discovery failing.
3. **KM4 startup audit:** inspect `qseecom-service`, `keymaster-4-0`, and QSEECom ordering; determine what the device log must show to prove KM4 is registered before `fscrypt_initialize_systemwide_keys()`.
4. **Keep the status docs current** in `PORTING_STATUS.md` and `ISSUE_REGISTER.md`.

Do not change fstab, FBE policy, or add another Keymaster bridge based only on static speculation.

## 7. Next device command — capture the decisive log

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
Vold unable to get security level from keystore2
A115F_KEYMASTER_BARRIER_COMPLETE
Successfully registered Keystore 2.0 service
Skipping keymaster compat
found android.hardware.keymaster
Failed to get service for
keymaster-4-0
qseecom
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

The exact barrier/Keystore2/KM4/vold line selects the next code change. Do not edit fstab or add another Keymaster bridge before that line is known.

Branch tree:

1. No `A115F_KEYMASTER_BARRIER_BEGIN` → helper did not run; check init/SELinux.
2. BEGIN, no COMPLETE → helper hung; check Keystore2/servicemanager.
3. COMPLETE + `Vold unable to get security level` → Keystore2 up, KM4 absent/unusable.
4. COMPLETE + `Keymaster device ready` without error, then FBE failure → KM4 may be present; inspect key generation/profile/KeyStorage.
5. COMPLETE + FBE success → barrier likely fixed the race; continue to next boot stage.

## 8. Do not repeat

- Do not rebuild or publish another ZIP before the FBE sub-step is identified.
- Do not format `/data`, `/keydata`, or `/keyrefuge`.
- Do not treat `K00R` or QSEE `0x72000206` as the root cause.
- Do not treat `A115F_KEYMASTER_BARRIER_COMPLETE` as proof that KM4 is ready.
- Do not move the barrier into vendor `on post-fs-data`; system `installkey /data` runs before it.
- Do not treat a truncated ext4 listing as proof that a library is missing.
- Do not claim the port boots from static tests or package validation alone.

## 9. Resume checklist

- [ ] Read this file and `PORTING_STATUS.md`.
- [ ] Confirm the status repo is at the latest audit commit.
- [ ] Continue the Samsung ODE/UCM and KM4 startup audits in section 6.
- [ ] If the phone is available, run the one device test in section 7 and capture the full logs.
- [ ] Do not build or publish a new ZIP until the log identifies the branch.
