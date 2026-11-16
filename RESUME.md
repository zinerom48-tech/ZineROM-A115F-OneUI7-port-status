# RESUME — A115F One UI 7 Port

Snapshot: **2026-11-16**, legacy-fscrypt implementation checkpoint.

Status board: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-port-status>
Working branch for this checkpoint: `audit/2026-11-03-keymaster-barrier` (do not overwrite `main`).
Source branch: `manus/a115f-legacy-fscrypt`, based on `manus/a115f-vendor-fingerprint@23f8998f0200969f5748e9eb012d1dc52037a054`; pushed at `48438e6` (parent `325dcd2`).

Forks on the same account:

- Source: <https://github.com/zinerom48-tech/ZineROM-a11q>
- Release: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-boot-test>
- Firmware tool: <https://github.com/zinerom48-tech/SamFWDumper1>

GitHub forks do **not** copy release assets; the 1.84 GB ZIP remains only at the upstream release URL. The upstream repo (`nova-arch008/ZineROM-a11q`) currently returns 404; the fork at `23f8998f…` still contains the source.

## 1. Is it safe to disconnect?

**Yes.** No build, flash, format, or device operation is running. The legacy-fscrypt source is committed and pushed (`manus/a115f-legacy-fscrypt@48438e6`), and both binaries were compiled and host-tested locally. The remaining limitation is external: the GitHub repository currently has zero registered Actions workflows, so the ROM build cannot be dispatched (see §4). Do not delete the audit files listed in section 6 or the source files.

## 2. Exact current blocker

The boot reaches Android second-stage init and vendor services, then fails at `/data` file-based encryption:

```text
cryptfs enablefilecrypto -> Status(-8, EX_SERVICE_SPECIFIC)
Rescue Party reason: enablefilecrypto_failedK00R
```

The code-level root cause is now established. It has two independent parts, and both must be fixed:

1. **Kernel/`vold` mismatch.** The A11 4.9.227 kernel implements only legacy fscrypt key management. It has no `FS_IOC_ADD_ENCRYPTION_KEY`, `FS_IOC_REMOVE_ENCRYPTION_KEY`, or `FS_IOC_GET_ENCRYPTION_KEY_STATUS`. Donor Samsung `vold` is ioctl-only (`0xC0506617`, `0xC0406618`, `0xC080661A`) and has no legacy fallback symbols. `install_storage_key()` therefore always fails.
2. **Missing keyrings.** Legacy fscrypt resolves keys with `request_key` in each I/O process, through PID 1's inherited session keyring and the named `fscrypt` keyring inside it. Donor Samsung `init` creates neither. A helper cannot inject the keyring because `keyctl_session_to_parent()` rejects `parent->pid <= 1`.

**Correction:** the earlier claim that “Android 15 removed legacy fscrypt” is wrong. LineageOS 22.2 (Android 15) and 23.1 (Android 16) still contain `isFsKeyringSupported`, `installKeyLegacy`, and `evictKeyLegacy`. Samsung removed the code from the donor `vold` and `init`.

**Consequence:** the current `f51460ef…` asset cannot reach the framework even with a perfectly healthy KM4. A diagnostic run on it can only confirm the failure stage and whether key retrieval failed earlier.

**Implemented fix (source only; not compiled or built):**

- **Part A — `/system/bin/init` wrapper.** AOSP 12 first-stage init executes `/system/bin/init selinux_setup` as PID 1 before policy load. The wrapper creates PID 1's session keyring and the named `fscrypt` keyring, preserves `argv[0]`, skips keyring setup for the `ueventd` symlink path, and execs the donor init at `/system/bin/init.real`. Both files are labelled `init_exec` because the post-policy `second_stage` re-exec runs the wrapper again. The boot image is untouched.
- **Part B — 64-bit `vold` `LD_PRELOAD` shim.** Calls the real `ioctl` first; only on `ENOTTY` translates ADD/REMOVE/GET_STATUS to legacy `ext4:`/`f2fs:`/`fscrypt:` logon keys with a 72-byte `fscrypt_key` payload. The shim creates the `fscrypt` keyring if it is missing (fallback).
- SELinux already permits the intended pre-policy path: `(allow domain init (key (search)))`, `(allow domain vold (key (search)))`, `(allow init init (key (write search setattr)))`, `(allow vold init (key (write search setattr)))`, `(allow vold vold (key (write search setattr)))`, plus the versioned `kernel_31_0` rules.
- Full ABI, structs, syscall numbers, packaging, and risks: `FIX_LEGACY_FSCRYPT.md`.

There is **no successful boot**, no boot animation, and no Setup Wizard. No fix has been compiled into a ROM or tested. Readiness remains **R0**.

## 3. What is already done

- Public source and release confirmed; current artifact hash is `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95`, size `1,842,649,818` bytes.
- Current ZIP contains the `A115F_KEYMASTER_BARRIER_BEGIN` / `exec_start wait_for_keymaster` / `A115F_KEYMASTER_BARRIER_COMPLETE` / `mount_all --late` sequence. Released vendor `init.target.rc` MD5 is `e111ab7f49ebcc8fa70d3253ccba9bce`.
- Build timeline: `1103fc00…` (prior diagnostic ZIP, no barrier) → `190586c…` (barrier, 2026-10-25) → `23f8998f…` (markers, 2026-10-27) → current asset `f51460ef…` built from `23f8998f…`.
- The release is **not** boot-tested; it passed 62 static/package tests.
- The earlier barrier-deadlock theory is **retracted**. AOSP `ImportParser::EndFile()` parses imports at end-of-file, so system `on late-fs` `class_start early_hal` runs before the imported vendor `on late-fs` barrier. The barrier is correctly placed.
- Effective FBE policy is **v1**: A11 vendor `ro.product.first_api_level=29`; donor system, system_ext, and product build.prop files do not override it.
- A11 kernel is Linux/arm64 `4.9.227` with `CONFIG_FS_ENCRYPTION`, `CONFIG_EXT4_FS_ENCRYPTION`, `CONFIG_EXT4_FS_ICE_ENCRYPTION`, `CONFIG_F2FS_FS_ENCRYPTION`, `CONFIG_CRYPTO_XTS`, and `CONFIG_PFK`.
- Kernel byte scan of `vmlinux.bin`: 6 `FS_IOC_SET_ENCRYPTION_POLICY` (`0x800C6613`), 4 `FS_IOC_GET_ENCRYPTION_POLICY` (`0x400C6615`), **0** `FS_IOC_ADD_ENCRYPTION_KEY` (`0xC0506617`). Kernel source uses `request_key(&key_type_logon, …)`.
- Donor `vold` disassembly: `installKey` (`0x47114`) → `ioctl(0xC0506617)` at `0x472FC`; `evictKey` (`0x47B74`) → `ioctl(0xC0406618)` at `0x47C30`; `waitForBusyFiles` (`0x48054`) uses REMOVE + `0xC080661A`. No `installKeyLegacy`/`evictKeyLegacy`/`isFsKeyringSupported`/`fscryptKeyring` symbols; no `add_key`/`keyctl` imports.
- Donor `init`: no keyctl strings/symbols, no `libkeyutils.so` DT_NEEDED, no `svc` in the full `SecondStageMain` (`0x155DFC`, size `0x5574`).
- AOSP/Lineage `init.cpp` still calls `keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1)`; `FscryptInstallKeyring()` creates the named `fscrypt` keyring.
- Legacy key format confirmed from Android 14 source: `fscrypt_key` = 72 bytes (`mode=0`, `raw[64]`, `size=64`); names `ext4:<16 hex>`, `f2fs:<16 hex>`, `fscrypt:<16 hex>`; keyring searched in PID 1's session keyring.
- Fscrypt ioctl structs confirmed from `include/uapi/linux/fscrypt.h`: v1 specifier `type=1`, descriptor at offset 8; ADD raw payload at offset 80; REMOVE 0x40 bytes; GET_STATUS 0x80 bytes; status `ABSENT=1`, `PRESENT=2`, `INCOMPLETELY_REMOVED=3`.
- SELinux key rules confirmed in the A11 target `plat_sepolicy.cil` (API 31, from `system_big3.dat`) and A11 `vendor_sepolicy.cil`; no policy change is needed for the intended pre-policy path.
- Donor system is not 64-bit-only. `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`, all `ELF32` ARM.
- Donor `/system/lib64` contains the complete 64-bit KM4 HIDL stack plus KeyMint V1/V2/V3 libraries; donor `keystore2` is ELF64.
- A11 `libkeymasterdeviceutils.so` `KeymasterUtils` constructor calls `QSEECom_start_app(handle, "/vendor/firmware_mnt/image", "keymaster64", 0xa000)`, retries `"keymaster"`, then aborts via `__android_log_assert(“…”, "KeymasterUtils", "QSEECom_start_app failed")`. `libQSEEComAPI.so` opens the TA image with `fopen` and has no sleep/retry.
- A11 init order: `on init` starts `keymaster-4-0` (expected abort) → `on fs` mounts `apnhlos` at `/vendor/firmware_mnt` → `on post-fs` starts `vendor.qseecomd` and blocks in `init.qti.qseecomd.sh` until `vendor.sys.listeners.registered=true` → system `on late-fs` `class_start early_hal` starts KM4 afresh → vendor `on late-fs` barrier → `mount_all --late` → `on post-fs-data` `installkey /data`.
- The A11 `keymaster-4-0` rc has no `interface` declaration, so there is no lazy-start fallback.
- `restart_period` defaults to 5 s in Android 12/15; Android 15’s 4-crash fatal limit does not apply to `keymaster-4-0`.
- `hwservicemanager` is present at `/system/system_ext/bin/hwservicemanager`; `/system_ext` is a symlink to `/system/system_ext`.
- Android 15 FCM 202404 and FCM 8 contain no Keymaster/KeyMint entries; VINTF is not rejecting the A11 KM4 HAL.
- A11 vendor SELinux defines `hal_keymaster_qti`/`hal_keymaster_qti_exec`; KM4 is labeled correctly; donor policy includes `keystore` in `hal_keymaster_client`.
- `installkey /data` → `vdc cryptfs enablefilecrypto` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()` is confirmed.
- `ro.crypto.type=file` is set by `fs_mgr` from the fstab; the A11 vendor init does not call `enablefilecrypto` directly.
- A11 `/data` fstab: `fileencryption=ice`, no `keydirectory=`, `latemount`, `check`, `quota`, `checkpoint=fs`.
- A11 `/keydata` and `/keyrefuge`: `ext4`, `fileencryption=ice,nofail`.
- `K00R` and QSEE `0x72000206` are **not** root causes; both occur on booting devices.
- The `/oem` zero-superblock mount is a stock A11 init action; boot continues past it.
- Samsung ODE/UCM compatibility is **closed** for this failure: donor `vold` calls `isUcmOdeMode()` only while parsing `wrappedkey_v0`; A11 `/data` has no such flag, and donor `fbeEnable()` does not consume `/keydata` or `/keyrefuge`.
- Source-branch audit: only `manus/a115f-vendor-fingerprint` at `23f8998f…` contains `scripts/patch_a115f_init.py` and the barrier. `build-once` is an ancestor; `preflight` is 2 ahead/15 behind and lacks the barrier.
- `/system/bin/ueventd` is a build symlink to `init` (`symlinks: ["ueventd"]` in `init/Android.bp`), and `service ueventd … seclabel u:r:ueventd:s0`. A `/system/bin/init` wrapper must preserve `argv[0]` and skip keyring setup for ueventd; the mod verifies the symlink target before installing it.
- Full donor system image reconstructed: `system_full.dat`, exactly `3,213,774,848` bytes.
- Full donor product stream reconstructed: `product_final.dat`, exactly `312,815,616` bytes.
- Compatibility audit: no additional kernel, VINTF, boot-image, or installer defect beyond the FBE root cause. The installer does not touch `/data`; dynamic-partition geometry matches the donor images.

## 4. Build results

- **No ROM build was run.**
- No new ZIP was produced.
- The only ROM artifact is the public `f51460ef…` ZIP from Actions run `#40494388352`, commit `23f8998f…`.
- That artifact is not boot-tested and cannot succeed at FBE on this kernel.
- The fix binaries were compiled locally with Zig 0.17.1 (clang 22.1.8 + lld), host tests pass, and the ELF checks pass. They have **not** been built by CI, packaged into a ROM, or device-tested.
- GitHub currently reports zero registered workflows for `zinerom48-tech/ZineROM-a11q`; a dispatch of `build.yml` returns HTTP 404 even though the default branch contains the file and Actions is enabled. This is an external blocker requiring user action.
- Do not build or publish another ZIP until CI is running and the ROM build is inspected. A candidate ROM build must start from `manus/a115f-legacy-fscrypt`, must not run `publish-public.yml`, and must stay R0 until a device reaches the framework.

## 5. What the fix needs, in order

1. **Unblock Actions workflow registration (next; user action).** The repository has no registered workflows: `POST .../actions/workflows/build.yml/dispatches` returns HTTP 404 and `GET /actions/workflows` returns `total_count: 0`, although `oneui7-a11q-boot` contains `.github/workflows/build.yml`, Actions is enabled, and the token has `repo` scope. Enable/register workflows in the GitHub UI or provide a `workflow`-scoped credential. Do not force-push the default branch.
2. **Dispatch `build.yml` on `manus/a115f-legacy-fscrypt`.** The first step runs all `tests/`; the mod then compiles the wrapper and shim, runs the host tests, runs the `vold.rc` patcher tests, and performs the ELF/symbol checks. Fix any failure before building.
3. **Record the result.** The ROM build must produce and inspect a ZIP; record the ZIP SHA-256 and the SHA-256 of `/system/bin/init`, `/system/bin/init.real`, and the shim. Do not publish.
4. **Review the built image statically.** Confirm `system/bin/init` is the wrapper, `init.real` is the donor init, `ueventd` still resolves to `init`, `vold.rc` has exactly one `setenv LD_PRELOAD`, and the shim is present in `/system/lib64` with the expected label.
5. **Device test only after recovery is verified.** Capture wrapper markers (`A115F_FSCRYPT_WRAPPER: ENTER`, `SESSION_OK`, `FSCRYPT_OK`, `EXEC`), logcat tag `A115F_FSCRYPT` (`SHIM_LOADED`, `ADD_OK`/`ADD_FAIL`, `REMOVE_*`, `STATUS_*`), the vold fscrypt error (if any), both KM4 starts, and AVC denials. Do not format `/data`, `/keydata`, or `/keyrefuge`.
6. **Keep the status docs current.** Update `PORTING_STATUS.md`, `ISSUE_REGISTER.md`, `NEXT_DEVICE_TEST.md`, and `FIX_LEGACY_FSCRYPT.md` with the build hashes and any CI failure output.

## 6. Audit workspace state

The audit workspace is `C:\Users\claymi5433\AppData\Local\Temp\opencode`.

The source workspace is `C:\Users\claymi5433\Documents\Default Project\ZineROM-a11q-src`. It is a depth-1, blobless partial clone of `zinerom48-tech/ZineROM-a11q`; `HEAD` is `refs/heads/manus/a115f-legacy-fscrypt` at `48438e6`, the full donor tree is in the index, and 12 new files are committed on the branch. The standalone CI workflow is preserved on local branch `local/a115f-legacy-fscrypt-ci-workflow` at `5551184` and is not pushed. The worktree does not contain the donor tree, so never run `git add -A` or `git reset --hard`.

| Source path | Meaning |
|---|---|
| `prebuilts/a115f_legacy_fscrypt/src/init_wrapper.c` | Freestanding `/system/bin/init` wrapper |
| `prebuilts/a115f_legacy_fscrypt/src/fscrypt_legacy.c` | Legacy key payload/name/install/evict/status helpers |
| `prebuilts/a115f_legacy_fscrypt/src/fscrypt_legacy_shim.c` | vold `ioctl` `LD_PRELOAD` shim |
| `prebuilts/a115f_legacy_fscrypt/build.sh` | AArch64 build and host-test entry point |
| `prebuilts/a115f_legacy_fscrypt/tests/test_legacy.c` | Host tests for the legacy helpers |
| `unica/mods/zz_a115f_legacy_fscrypt/customize.sh` | Installs wrapper/init.real/shim, patches `vold.rc`, sets metadata |
| `scripts/patch_a115f_legacy_fscrypt_vold_rc.py` | Idempotent `setenv LD_PRELOAD` patcher |
| `tests/test_legacy_fscrypt_vold_rc.py` | Patcher unit tests |
| `.github/workflows/legacy-fscrypt-ci.yml` | Compile, host-test, and ELF/symbol checks (local-only branch; not pushed) |

| Audit file | Meaning |
|---|---|
| `system_full.dat` | Complete donor system image, `3,213,774,848` bytes |
| `product_final.dat` | Complete donor product stream, `312,815,616` bytes |
| `vendor.new.dat` | Complete donor vendor image |
| `vmlinux.bin` | Packaged A11 kernel; legacy-only fscrypt scan |
| `donor_vold` / `donor_init` | Donor binaries with recovered symbols/debugdata |
| `vold_all.txt` | `vold` symbol table + PLT map (`ioctl=0x140408`, `syscall=0x1429B8`) |
| `vold_syms.txt` | Recovered `.gnu_debugdata` symbols |
| `plat_sepolicy.cil` | A11 target platform policy (API 31, from `system_big3.dat`); key-permission proof |
| `vendor_sepolicy.cil` | A11 vendor policy (`VE=SEPF_SM-A115F_12_0001`); versioned key rules |
| `KeyUtil22.cpp` / `KeyUtilL23.cpp` | Lineage 22.2/23.1 legacy fallback reference |
| `keyutil14.cpp` | Exact legacy key names/payload/mode |
| `vold15src/KeyUtil.cpp` | Android 15 ioctl-only reference |
| `fscrypt_uapi.h` | Exact struct layouts and constants |
| `fstab_a11.txt` | A11 `/data`, `/keydata`, `/keyrefuge`, `/vendor/firmware_mnt` entries |
| `init_android15.bp` | `symlinks: ["ueventd"]` proof |
| `main22.cpp` / `first_stage22.cpp` / `ueventd22.cpp` | init dispatch and first-stage exec reference |
| `fc22` | AOSP `file_contexts`; `/system/bin/init` label |
| `a11q_build/build.yml` | CI workflow |
| `a11q_build/config.sh` | A115F target config |
| `a11q_build/qssi.sh` | QSSI donor config |
| `a11q_build/patch_a115f_init.py` | Barrier patcher |
| `a11q_build/boot_fit.sh` | Boot-image repack mod |
| `a11q_build/selinux.sh` | SELinux entry removal list |
| `fork_tree.json` | Fork tree at `23f8998f…` |
| `vendor_init_target_zip.rc` | Released vendor `init.target.rc`; MD5 `e111ab7f49ebcc8fa70d3253ccba9bce` |
| `donor_init_full.rc` | Donor system `init.rc` |
| `wait_for_keymaster_rel.bin` | Released 32-bit helper; SHA-256 `0e64ac88…` |
| `kmdev_ctor_va.txt` / `kmdev_ctor_tail.txt` | `KeymasterUtils` constructor disassembly |
| `km4.rc` | A11 `keymaster-4-0` service definition |
| `qseescript.sh` | `init.qti.qseecomd.sh` listener loop |
| `donor_lib64.txt` | Complete donor 64-bit library listing |
| `libqsee_strings.txt` | TA image-open behavior and no retry sleep |

The old stalled partial clone at `C:\Users\claymi5433\ZineROM-a11q-src` is not the active workspace; do not use it. The active workspace is `Documents\Default Project\ZineROM-a11q-src`.

## 7. Next offline work

1. **Unblock Actions workflow registration (next; user action):** the repository reports zero registered workflows despite a valid `build.yml` on the default branch; dispatch returns HTTP 404. Enable/register workflows in the GitHub UI or provide a `workflow`-scoped credential. Do not rewrite the default branch.
2. **Dispatch the ROM build:** run `build.yml` on `manus/a115f-legacy-fscrypt`; inspect the unit-test step, then the mod's compile/host-test/`readelf` checks. Fix any failure before producing a ZIP.
3. **Review the built system image statically** before any device test; record all hashes and logs.
4. **Review the built system image statically** before any device test.
5. **Keep the status docs current** in `PORTING_STATUS.md`, `ISSUE_REGISTER.md`, `NEXT_DEVICE_TEST.md`, and `FIX_LEGACY_FSCRYPT.md`.
6. **Optional provenance check:** reproduce `23f8998f…` in a clean environment and compare image hashes with `f51460ef…`. This validates provenance only, not bootability.

Do not change fstab, FBE policy, or add another Keymaster bridge based only on static speculation.

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
init: starting service 'keymaster-4-0'
QSEECom_start_app failed
KeymasterUtils
mount_all /vendor/etc/fstab.qcom --early
vendor.sys.listeners.registered
IQSEECom registerAsService is successful
A115F_KEYMASTER_BARRIER_BEGIN
Waiting for Keymaster device
Keymaster device ready
Vold unable to get security level from keystore2
A115F_KEYMASTER_BARRIER_COMPLETE
Successfully registered Keystore 2.0 service
Skipping keymaster compat
found android.hardware.keymaster
Failed to get service for
qseecom
fscrypt_initialize_systemwide_keys
No mount point entry for /data
Unable to parse encryption options
EncryptionOptions not initialized
Failed to get ephemeral wrapped key
Creating new key in
No key found in
decryptWithKeystoreKey failed
decryptWithoutKeystore failed
encryptWithKeystoreKey failed
Version mismatch
Failed to install fscrypt key to /data: Inappropriate ioctl for device
Failed to open /data to install key
Failed to evict fscrypt key
Failed to get status for fscrypt key
A115F_FSCRYPT_WRAPPER: ENTER
A115F_FSCRYPT_WRAPPER: SESSION_OK <serial>
A115F_FSCRYPT_WRAPPER: SESSION_FAIL <errno>
A115F_FSCRYPT_WRAPPER: FSCRYPT_OK <serial>
A115F_FSCRYPT_WRAPPER: FSCRYPT_FAIL <errno>
A115F_FSCRYPT_WRAPPER: UEVENTD_PASSTHROUGH
A115F_FSCRYPT_WRAPPER: EXEC /system/bin/init.real
logcat tag A115F_FSCRYPT: SHIM_LOADED
logcat tag A115F_FSCRYPT: ADD_OK
logcat tag A115F_FSCRYPT: ADD_FAIL <errno>
logcat tag A115F_FSCRYPT: REMOVE_OK
logcat tag A115F_FSCRYPT: REMOVE_FAIL <errno>
logcat tag A115F_FSCRYPT: STATUS_PRESENT
logcat tag A115F_FSCRYPT: STATUS_ABSENT
```

The exact vold/KeyStorage line selects the next code change. Do not edit fstab or add another Keymaster bridge before that line is known. In particular, the **first** `QSEECom_start_app failed` before `/vendor/firmware_mnt` mounts is expected; keep reading until the second `keymaster-4-0` start.

Branch tree for the current asset:

1. No second `keymaster-4-0` start → late-fs class start did not run; check init parsing/SELinux.
2. Second start aborts again with `QSEECom_start_app failed` → firmware mount or TA image path problem.
3. No `vendor.sys.listeners.registered` → qseecomd/listener gate failed; KM4 cannot reach QSEECom.
4. No `A115F_KEYMASTER_BARRIER_BEGIN` → helper did not run; check init/SELinux.
5. BEGIN, no COMPLETE → helper hung; check Keystore2/servicemanager.
6. COMPLETE + `Vold unable to get security level` → Keystore2 up, KM4 absent/unusable.
7. COMPLETE + `Failed to install fscrypt key to /data: Inappropriate ioctl for device` → **expected**; confirms the legacy-fscrypt root cause.
8. COMPLETE + `Creating new key in` / `No key found in` / `decryptWithKeystoreKey failed` and no install error → key retrieval failed before key installation; KM4/Keystore2 is the earlier blocker.
9. COMPLETE + FBE success → not a possible outcome for this asset on this kernel; treat as evidence the kernel/asset identity assumption is wrong and re-verify the SHA.

## 9. Do not repeat

- Do not claim the current asset can boot with a healthy KM4; the kernel/`vold`/`init` gap is unconditional.
- Do not claim “Android 15 removed legacy fscrypt”; LineageOS 22.2/23.1 still contain it.
- Do not install a `/system/bin/init` wrapper without preserving `argv[0]`, verifying the `ueventd` symlink, and labelling both `init` and `init.real` as `init_exec`.
- Do not try to inject a session keyring into PID 1 from a helper; the kernel rejects it.
- Do not publish another ZIP before the compile/host-test CI is green and the ROM build is statically inspected.
- Do not format `/data`, `/keydata`, or `/keyrefuge`.
- Do not treat `K00R` or QSEE `0x72000206` as the root cause.
- Do not treat `A115F_KEYMASTER_BARRIER_COMPLETE` as proof that KM4 is ready.
- Do not treat the first `on init` `QSEECom_start_app failed` as the root cause; that start precedes `/vendor/firmware_mnt`.
- Do not rebuild from `manus/a115f-build-once` or `manus/a115f-bootability-preflight`; neither contains the barrier patcher.
- Do not move the barrier into vendor `on post-fs-data`; system `installkey /data` runs before it.
- Do not treat a truncated ext4 listing as proof that a library is missing.
- Do not claim the port boots from static tests or package validation alone.

## 10. Resume checklist

- [ ] Read this file, `PORTING_STATUS.md`, `ISSUE_REGISTER.md`, and `FIX_LEGACY_FSCRYPT.md`.
- [ ] Confirm the status repo is at the latest audit commit.
- [x] Source committed and pushed as `manus/a115f-legacy-fscrypt@48438e6`.
- [ ] Unblock GitHub Actions workflow registration (ISSUE-027); dispatch `build.yml` and record the outcome.
- [ ] Build only after CI is green, then inspect the built image before any device test.
- [ ] If the phone is available, run the one device test in section 8 after verifying the recovery path and getting explicit approval; capture both `keymaster-4-0` starts, the vold fscrypt line, and the wrapper/shim markers.
- [ ] Use the branch tree to pick exactly one next branch.
- [ ] Do not publish a new ZIP until a device reaches the framework.
