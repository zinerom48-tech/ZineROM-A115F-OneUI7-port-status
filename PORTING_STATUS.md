# ZineROM One UI 7 — Galaxy A11 (SM-A115F) Port Status

Last updated: 2026-11-04

Status board: <https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-port-status>

## 1. Scope and readiness

| Item | Value |
|---|---|
| Target device | Samsung Galaxy A11 `SM-A115F` (codename `a11q`), Binary S6 |
| Required bootloader | `A115FXXS6CWK2` (exact match enforced by package) |
| Donor firmware | `A057FXXUFDZH2` — `samsung/a05snsdxx/qssi:15/AP3A.240905.015.A2/A057FXXUFDZH2:user/release-keys` |
| Donor Android / One UI | Android 15 / One UI 7, security patch `2026-08-05` |
| Public artifact | `ZineROM_OneUI7_A115F-Port-A057F.zip` |
| Artifact size | `1,842,649,818` bytes |
| Artifact SHA-256 | `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95` |
| Package type | AOSP block OTA (`ota-type=BLOCK`), writes logical dynamic partitions plus `boot` and `dtbo` |

**Readiness level: R0 — artifact identified and partially audited; no successful boot observed. Do not install.**

R0 means only that the release exists, its identity/hash are confirmed, and part of the binary payload has been inspected. No build has been reproduced, no physical A11 has been tested in this project, and no boot milestone has been observed.

### Progress at a glance

| Measure | Estimate | Meaning |
|---|---|---|
| Successful boot outcome | **0%** | No completed boot, no boot animation, no Setup Wizard, no home screen. |
| Boot stages reached | **~45%** | Bootloader → kernel → first-stage init → second-stage init → vendor services/HALs. It fails at `/data` FBE before zygote and `system_server`. |
| Offline root-cause isolation | **~80%** | Failure narrowed to vold `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`. KM4 start ordering and the ODE/UCM branch have been audited; the exact failing sub-step still requires a device log. |
| Validated fix | **0%** | No fix has been built, flashed, or tested. |
| Physical-device validation | **0%** | No A11 is available for installation, logcat, pstore, or hardware testing. |
| Build/package audit | **~85% static** | The artifact exists and its package structure is audited. This is not evidence that it boots. |

These percentages are project bookkeeping, **not** a probability that the ROM will boot. The remaining framework/UI work is larger than the stage count suggests.

## 2. Milestone status

| Milestone | Status | Evidence / reason |
|---|---|---|
| M0 — Repository and hardware audit | **Complete for offline evidence** | Public source repo, release metadata, package guards, donor/target identities, dynamic-partition geometry, kernel container, A11 vendor HALs, VINTF/SELinux/boot-image/installer compatibility, and source-branch reconciliation are audited. Remaining hardware items (QSEE/TA runtime, SELinux runtime, KM4 late-fs registration) require a device log. Samsung ODE/keydata is closed for this failure. |
| M1 — Compatibility assessment and build environment | **Partial (source/build only)** | Source is public at `nova-arch008/ZineROM-a11q`; the A115F target and generic QSSI configuration were read; kernel, VINTF, SELinux, boot-image, and installer compatibility are audited. A build has **not** been reproduced locally. |
| M2 — Configuration and prerequisite validation | **Partial (config verified; runtime pending)** | Model/bootloader guards, BLOCK OTA metadata, transfer lists, partition sizes, FBE version default, A11 fstab, and the Keymaster barrier were checked. Product/system_ext property overrides and the 64-bit HAL set are audited; the remaining items are device-side runtime behavior. |
| M3 — Images compiled and inspected | **Partial (inspection only)** | Prebuilt `system`, `vendor`, `product`, `odm`, `boot`, and the patched vendor init were inspected. No image was compiled from source. |
| M4 — Installer and package validated | **Partial (static only)** | ZIP structure, CRC/member list, updater script, partition geometry, and asset digest. No installation performed. |
| M5 — Installation and early boot on target | **Not tested — device required** | No physical A11 available. |
| M6 — Android userspace / boot animation | **Not tested — device required** | Prior log excerpt reaches Android init and the FBE attempt; no boot animation observed. |
| M7 — Setup Wizard / home screen | **Not tested — device required** | No device. |
| M8 — Core hardware functionality | **Not tested — device required** | No device. |
| M9 — Stability / regression | **Not tested — device required** | No device. |

## 3. Confirmed facts

1. `nova-arch008/ZineROM-a11q` is public. The default boot branch is `oneui7-a11q-boot`; the inspected working branch is `manus/a115f-vendor-fingerprint` at `23f8998f0200969f5748e9eb012d1dc52037a054`.
2. The current public release asset is `ZineROM_OneUI7_A115F-Port-A057F.zip`, size `1,842,649,818` bytes, SHA-256 `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95` (GitHub asset metadata and release notes agree).
3. The public `target/a115f/BOOT_PROGRESS.md` still lists an older asset hash, `29384d92ba37f6603cf0dfe08eec027e4049e3781414ffc9d197479b3388e2e0`, and size `1,842,620,087`. That text is stale relative to the current release; the current asset is the `f51460ef…` build.
4. The current release body says the build came from GitHub Actions run `#40494388352` on commit `23f8998f0200969f5748e9eb012d1dc52037a054`, with 62 static/package tests passing. It explicitly says the ROM was **not** boot-tested on a phone.
5. The package is a BLOCK OTA. It writes `system`, `vendor`, `product`, `odm`, `boot`, and `dtbo`, and rewrites logical dynamic-partition metadata inside the existing `super` device. It does not flash `vbmeta` or DLKM images.
6. The updater requires `ro.boot.em.model == SM-A115F` and the exact bootloader `A115FXXS6CWK2`.
7. The shipped A11 vendor image contains the diagnostic barrier in `/vendor/etc/init/hw/init.target.rc`:

   ```text
   on late-fs
       write /dev/kmsg A115F_KEYMASTER_BARRIER_BEGIN
       exec_start wait_for_keymaster
       write /dev/kmsg A115F_KEYMASTER_BARRIER_COMPLETE
       mount_all /vendor/etc/fstab.qcom --late
   ```

   This ordering is present in the current `f51460ef…` asset. It proves the barrier is packaged; it does not prove the helper ran or completed on a device.

8. The A11 Keymaster 4.0 HAL `/vendor/bin/hw/android.hardware.keymaster@4.0-service-qti` is `ELF32` ARM with interpreter `/system/bin/linker`. Its service is started from `on init` as `keymaster-4-0`, class `early_hal`, user `system`, group `system drmrpc`.
9. The donor system is **not** 64-bit-only: `/system/build.prop` declares `ro.system.product.cpu.abilist=arm64-v8a,armeabi-v7a,armeabi` and `ro.system.product.cpu.abilist32=armeabi-v7a,armeabi`; the image contains `app_process32`, `mediaserver32`, `boringssl_self_test32`, `/system/bin/linker`, and `/system/bin/linker64`.
10. `/system/bin/linker` and `/system/bin/linker64` are symlinks to `/apex/com.android.runtime/bin/linker` and `/apex/com.android.runtime/bin/linker64`, respectively. The runtime linker lives in the runtime APEX, not in `/system/lib`.
11. The copied `/system/bin/wait_for_keymaster` is `ELF32` ARM, 11,312 bytes, built from `system/vold/wait_for_keymaster.cpp`. Its strings include `Waiting for Keymaster device` and `Keymaster device ready`; its service file is `/system/etc/init/wait_for_keymaster.rc` (`user root`, `group root system`, `priority -20`, `ioprio rt 0`, no `oneshot`, no `disabled`).
12. The helper’s dynamic dependencies include `libc.so`, `libbase.so`, `libbinder.so`, `libbinder_ndk.so`, `libhardware.so`, `libhardware_legacy.so`, `libhidlbase.so`, `libutils.so`, `libc++.so`, `libm.so`, `libdl.so`, `android.system.keystore2-V1-ndk_platform.so`, `android.security.maintenance-ndk_platform.so`, and `libkeymint_support.so`.
13. The A115F patch copies only the Keymaster/Keystore2-specific 32-bit libraries into `/system/lib`; it relies on the donor system for `libhardware_legacy.so`, `libbinder*.so`, `libbase.so`, `libc.so`, and the other platform libraries.
14. **The donor `/system/lib` contains every required 32-bit helper dependency.** All inspected dependencies are `ELF32` ARM. `libc.so`, `libm.so`, and `libdl.so` are symlinks into the runtime APEX. This resolves the 32-bit helper-library question (ISSUE-014).
15. The A11 vendor `/build.prop` contains `ro.product.first_api_level=29`, `ro.zygote=zygote32`, and `ro.vndk.version=31`.
16. The donor `/system/build.prop` contains no `ro.product.first_api_level` line. AOSP Android 15 `libfscrypt` defaults to policy v2 when `ro.product.first_api_level > 29`, and to v1 when it is `<= 29`; an explicit `:v1` fstab option can force v1.
17. AOSP init loads partition property files in the order system → system_ext → system_dlkm → vendor/default → vendor → vendor_dlkm → odm_dlkm → odm → product. The load map is later-wins, then applied once; `ro.*` properties are write-once at application time. The A11 vendor’s `29` therefore overrides an earlier system value. Product is loaded last and could override if it explicitly contains the key.
18. `/data` in the A11 vendor fstab is `f2fs` with `fileencryption=ice` and **no** explicit `keydirectory=` or `:v1`/`:v2` option. The same fstab mounts Samsung ODE partitions `/keydata` and `/keyrefuge` as `ext4` with `fileencryption=ice,nofail`.
19. `fileencryption=ice` with no key directory is consistent with the A11’s FBE v1 + inline-crypto layout. The exact policy version depends on the effective `ro.product.first_api_level`; the donor system, system_ext, and product property files have all been read and do not override the vendor value `29`.
20. The A11 vendor image contains Samsung’s FBE hardware-crypto stack (`libcryptfshwcommon.so`, `libcryptfshwhidl.so`, `vendor.qti.hardware.cryptfshw@1.0.so`) and security services (`vaultkeeper`, `drk`, `libsecure_storage.so`).
21. The A11 vendor init contains an explicit stock command, not an fstab entry:
    `mount ext4 /dev/block/bootdevice/by-name/oem /oem ro nosuid nodev barrier=1`.
    The prior log’s `/oem` mount failure at ~12.142 s is therefore a stock A11 init action; it occurs before QSEE and does not abort init.
22. `security.fbe.fail_cause=K00R` is not unique to this port. The same value appears in getprop dumps from booted Samsung devices (for example, `sm-a135f`, `sm-g973c`, and `sm-s938b` in public OTA archives), so `K00R` alone is not evidence of the root cause.
23. The QSEE `scm_call failed: func id 0x72000206, ret: -1` sequence appears on a booting LineageOS 23.1 daisy device that mounts `/data` and runs vold/keystore2. The error is therefore not sufficient to explain the A11 FBE failure.
24. The donor platform SELinux policy defines `wait_for_keymaster` and `wait_for_keymaster_exec` as domain, file, exec, and coredomain types.
25. The A11 vendor policy is version `31.0` and has no `wait_for_keymaster` type. The donor system ships versioned mappings `29.0.cil` through `34.0.cil`, including `31.0.cil`, so the vendor policy can be paired with the donor platform policy at the policy-version level. Runtime behavior is still unverified.
26. The donor platform policy includes the `keystore` domain in both `hal_keymaster_client` and `hal_keymint_client`; the A11 versioned platform policy maps `hal_keymaster_client` to `vold_31_0`. Keystore2 runs in the `keystore` domain. This reduces the SELinux client-attribute gap, but the exact runtime transition and HAL discovery remain unproven.
27. The donor `vold` path is: `installkey /data` → `vdc` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`. The literal `enablefilecrypto` string is in init, not vold; `vdc` translates the command to the AIDL `fbeEnable()` call. The earlier “donor vold is missing the command” contradiction is therefore resolved.
28. `ro.crypto.type=file` is set by `fs_mgr` from the fstab (`fs_mgr.cpp:1092`), so `installkey` runs and reaches `vdc`. The A11 vendor init scripts do not call `enablefilecrypto` or `vdc` directly; the call comes from the donor system init.
29. Donor Keystore2 contains Keymaster 4.0 support (`KEY_MASTER_V4_0`, `IKeymasterDevice`, `km_compat`) and Samsung KeyMint extension code. KM4 is not architecturally excluded.
30. The donor `/system/lib64` listing is now complete. It contains the 64-bit HIDL Keymaster client stack: `android.hardware.keymaster@3.0.so` (177,400 bytes), `android.hardware.keymaster@4.0.so` (203,584), `android.hardware.keymaster@4.1.so` (142,520), `android.hardware.keymaster-V4-ndk.so` (23,296), `libkeymaster4support.so` (112,584), `libkeymaster4_1support.so` (147,088), `libkeymaster_messages.so` (118,240), `libkeymaster_portable.so` (561,912), `libpuresoftkeymasterdevice.so` (103,504), `libhidlbase.so` (660,592), and `libhidltransport.so` (10,584). The donor `keystore2` binary is ELF64. The earlier empty `/system/lib64` result was a truncated-image artifact, not evidence of a missing client.
31. The A11 Keymaster HAL’s `libkeymasterdeviceutils.so` `KeymasterUtils` constructor calls `QSEECom_start_app(handle, "/vendor/firmware_mnt/image", "keymaster64", 0xa000)`; if that fails it retries with app name `"keymaster"`; if both fail it calls `__android_log_assert` with tag `KeymasterUtils` and message `QSEECom_start_app failed`, aborting the process. `libQSEEComAPI.so` opens the TA image by path with `fopen` and has no sleep/retry import.
32. The A11 Keymaster service is started twice in the boot sequence. The first `on init` start occurs before `/vendor/firmware_mnt` exists; `/vendor/etc/fstab.qcom` mounts `apnhlos` there during `on fs` `mount_all --early`. That first start is therefore expected to abort on the missing `image/keymaster64` TA. The effective start is the system `on late-fs` `class_start early_hal`, which runs before the imported vendor `on late-fs` barrier. AOSP `Service::Start()` clears `SVC_RESTARTING`, so it can start a fresh KM4 process after the firmware mount. `restart_period` defaults to 5 seconds in Android 12 and 15; Android 15’s 4-crash fatal limit applies only to `SVC_CRITICAL` or updatable services, not `keymaster-4-0`.
33. `vendor.qseecomd` and its listener gate are part of the same path: A11 `init.target.rc` starts `vendor.qseecomd` at `on post-fs` and then runs `/vendor/bin/init.qti.qseecomd.sh` with blocking `exec`; the script loops until `vendor.sys.listeners.registered=true`. The `qseecom-service` HAL is started at `on init` and logs `IQSEECom registerAsService is successful`. The prior boot reaching FBE implies this listener gate completed at least once.
34. The Samsung ODE/UCM branch is closed for first `/data` encryption. Disassembly of donor `vold` shows exactly one direct call to `isUcmOdeMode()`, inside `ParseOptionsForApiLevel` while parsing the `wrappedkey_v0` flag. `isUcmOdeMode()` reads `/efs/sec_efs/ucm_ode_mode`, parses it as base-10, and returns true only for the value `2`; missing, unreadable, or other values return false. The A11 `/data` fstab entry uses plain `fileencryption=ice` and has no `wrappedkey_v0`, so ODE is not engaged for first `/data` encryption. The `/keydata` and `/keyrefuge` fstab entries exist but donor vold has no code path that consumes them in this operation.
35. Source-branch audit: only `manus/a115f-vendor-fingerprint` at `23f8998f0200969f5748e9eb012d1dc52037a054` contains `scripts/patch_a115f_init.py` and the barrier. The GitHub compare API shows `23f8998f…` is 13 commits ahead of `ec9239a121e57b43d985d7c5756adf3cb754aba3` (build-once), so build-once is a strict ancestor containing nothing that `23f8998f…` lacks. `a61d5a41fb0871c5a6ed908e8d98c77da5965661` (preflight) is 2 commits ahead and 15 behind: it adds stock-blob extraction (`0266bc3c`) and the public-artifact-upload visibility guard (`a61d5a41`), but lacks the barrier and the 13 audit commits. A rebuild must start from `23f8998f…` and may cherry-pick `0266bc3c`; never the reverse.
36. No physical Galaxy A11 is available in this session. No installation, boot, logcat, pstore, or hardware test has been performed.

## 4. Current failure analysis

### Confirmed failure and its limits

The prior Android attempt reaches second-stage init, starts vendor processes, attempts file-based encryption for `/data`, and reboots to Recovery. The exact failure is reported as `cryptfs enablefilecrypto` returning `Status(-8, EX_SERVICE_SPECIFIC)`; the Rescue Party reason is `enablefilecrypto_failedK00R`.

The available log is an excerpt, not the raw capture. It does not include the vold service-specific error code, the full `logcat`, or a SHA-256 tying it to a particular ZIP. The prior attempt may predate the `wait_for_keymaster` barrier now present in `f51460ef…`. Therefore:

- The failure is confirmed at the stage level: FBE setup for `/data` fails and init reboots to Recovery.
- The root cause is **not** established.
- `K00R` is not the root cause; it is a Samsung property value that also appears on booted devices.
- The QSEE `0x72000206` messages are suspicious but not sufficient; identical messages occur on a booting device.

### Earliest visible events versus boot-critical failure

The prior log sequence is approximately:

- ~12.142 s: `/oem` ext4 mount fails with an all-zero superblock. This is a stock A11 init command and boot continues.
- ~12.66–12.69 s: `qseecomd` reports `scm_call` failures with function `0x72000206`.
- ~13.27 s: `security.fbe.fail_cause` is set/read as `K00R`.
- ~13.81 s: `cryptfs enablefilecrypto` fails.
- ~14.23 s: init reboots to Recovery.

The `/oem` and QSEE events are earlier, but neither is proven to be the boot-critical cause. The first event that actually stops Android is the `enablefilecrypto` failure.

### Exact failing operation

`vdc cryptfs enablefilecrypto` maps to vold’s AIDL `fbeEnable()`. In Android 15, `fbeEnable()` calls `fscrypt_initialize_systemwide_keys()`. That function can fail at four distinct points:

1. `init_data_file_encryption_options()` — parses the `/data` fstab entry. The A11 entry (`fileencryption=ice`, no key directory) is structurally valid, so this is the least likely failure.
2. `retrieveOrGenerateKey(device_key_path, device_key_temp, …)` — reads or creates the device key through KeyStorage, Keystore2, and the Keymaster HAL.
3. `install_storage_key(DATA_MNT_POINT, …)` — installs the key into the kernel and sets the fscrypt policy.
4. Writing the `/data` encryption options/references after the key is installed.

The current evidence does **not** distinguish these four. A raw log containing the vold/keymaster/keystore2 lines is required before changing fstab, FBE policy, or adding another Keymaster bridge.

### FBE version assessment

**Confirmed: the effective `ro.product.first_api_level` is `29` and the port uses FBE policy v1.**

- A11 vendor build.prop: `ro.product.first_api_level=29`.
- Donor `/system/build.prop`: key absent.
- Donor `/system/system_ext/etc/build.prop`: key absent.
- Donor `/product/etc/build.prop`: key absent.
- The donor `/system/lib64/libfscrypt.so` reads `ro.product.first_api_level` and selects the legacy v1 policy unless the value is greater than `29`. No assembled property overrides the A11 vendor value.

The earlier v2/kernel-support branch is closed. The A11 kernel is Linux/arm64 `4.9.227` with `CONFIG_FS_ENCRYPTION=y`, `CONFIG_EXT4_FS_ENCRYPTION=y`, `CONFIG_EXT4_FS_ICE_ENCRYPTION=y`, `CONFIG_F2FS_FS_ENCRYPTION=y`, `CONFIG_CRYPTO_XTS=y`, `CONFIG_CRYPTO_AES=y`, and `CONFIG_PFK=y`. The v1 policy matches the A11 Android 12-era expectations.

### Keymaster / 32-bit / 64-bit assessment

The 32-bit and 64-bit library questions are narrowed substantially:

- The donor declares 32-bit ABI support, ships 32-bit binaries, and `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`. The A11 Keymaster HAL and the helper are both `ELF32` ARM.
- The 64-bit Keymaster HIDL client stack is present: `android.hardware.keymaster@3.0.so`, `@4.0.so` (203,584 bytes), `@4.1.so`, `libkeymaster4support.so`, `libkeymaster4_1support.so`, `libkeymaster_messages.so`, `libkeymaster_portable.so`, `libhidlbase.so`, and the KeyMint V1/V2/V3 libraries. `libkm_compat.so` (227,632 bytes) contains HIDL `IKeymasterDevice::getService` symbols and the Keymaster 3.x/4.x fallback; the donor `keystore2` is ELF64.
- `hwservicemanager` is present at `/system/system_ext/bin/hwservicemanager`; `/system_ext` is a symlink to `/system/system_ext`. `libkm_compat` skips HIDL fallback only when the HIDL service manager itself is unavailable, so the A11 KM4 service is reachable in principle.

The remaining Keymaster risks are runtime, not library presence:

- whether KM4 is registered and usable when Keystore2/fscrypt asks for it (see the barrier limitation in the 2026-11-03 checkpoint);
- whether the late-fs KM4 start succeeds after `/vendor/firmware_mnt` is mounted, and whether its TA can generate the key type/capability requested by Android 15 FBE;
- whether the first `on init` KM4 abort leaves any state that prevents the late-fs restart from registering;
- whether the A11’s QSEECom listener gate and `qseecom-service` registration complete before KM4 calls `QSEECom_start_app`.

### KM4 start ordering and failure path

The A11 Keymaster service definition has no `oneshot`, no `disabled`, and no explicit `restart_period`; it is class `early_hal`. The boot sequence is:

1. `on init` starts `keymaster-4-0`. `/vendor/firmware_mnt` is not mounted yet, so `libkeymasterdeviceutils.so` cannot open `/vendor/firmware_mnt/image/keymaster64`. Its constructor retries with app name `keymaster`, then aborts through `__android_log_assert("...", "KeymasterUtils", "QSEECom_start_app failed")`.
2. `on fs` runs `mount_all /vendor/etc/fstab.qcom --early`, which mounts the `apnhlos` partition at `/vendor/firmware_mnt`.
3. `on post-fs` starts `vendor.qseecomd` and blocks in `init.qti.qseecomd.sh` until `vendor.sys.listeners.registered=true`.
4. system `on late-fs` runs `class_start early_hal`. AOSP `Service::Start()` clears `SVC_RESTARTING`, so the crashed `keymaster-4-0` is started again, now with the firmware image available.
5. vendor `on late-fs` runs the diagnostic barrier and `wait_for_keymaster`, then `mount_all --late`.

The first abort is therefore expected and is not by itself a boot-critical failure. The decisive evidence is whether the **second** start succeeds, registers `android.hardware.keymaster@4.0`, and is visible to Keystore2 before `fscrypt_initialize_systemwide_keys()`. `restart_period` defaults to 5 seconds; Android 15’s 4-crash fatal limit does not apply to this `early_hal` service.

### Hypotheses (not confirmed)

- **H1 — Keymaster/TA mismatch:** the A11 Keymaster 4.0 HAL starts after late-fs, but the TA/QSEE path cannot create the key type or capability requested by Android 15’s FBE setup.
- **H2 — Helper runtime gap:** largely resolved. The 32-bit helper dependencies are present and `ELF32`; a missing 32-bit library is no longer a leading explanation. ABI incompatibility or SELinux transition is still possible.
- **H3 — Samsung ODE/keydata mismatch:** **disproven for first `/data` encryption.** Donor vold consults `isUcmOdeMode()` only while parsing `wrappedkey_v0`; the A11 `/data` fstab has no such flag. `/keydata` and `/keyrefuge` are not consumed by donor vold in this operation. Keep the partition-state caution, but do not treat ODE as a cause of this failure.
- **H4 — `/data` state:** a previous failed attempt left `/data` or its metadata partially encrypted/corrupt, so `enablefilecrypto` fails before Keymaster is reached. This must not be tested by formatting without explicit approval.
- **H5 — Kernel feature gap:** the A11 4.9 kernel lacks a crypto/inline-encryption feature required by the donor’s FBE path. This is less likely if policy v1 is selected, but not ruled out.
- **H6 — Prior log predates the barrier:** confirmed as the leading explanation. The prior diagnostic ZIP `1103fc00…` predates commit `190586c` (barrier) and `23f8998f…` (markers); the current `f51460ef…` asset was built from `23f8998f…` and contains the barrier. No barrier-containing build is known to have been device-tested.
- **H7 — Keystore2/KM4 runtime binding gap:** the 64-bit HIDL client is present, but KM4 may not be registered or usable when Keystore2/fscrypt needs it. This is the main remaining Keymaster branch; the first-start failure path and late-fs restart must be checked in the same log.
- **H8 — Late-fs restart is sufficient:** the first `on init` abort is benign because `class_start early_hal` starts KM4 again after `/vendor/firmware_mnt` is mounted. This is consistent with the init ordering but has not been observed on a device.

## 5. Untested assumptions

- That the barrier’s `wait_for_keymaster` exit actually implies KM4 is usable. It does not: the helper waits for Keystore2 registration and logs `getSecurityLevel` failures without returning nonzero.
- That KM4 registers before Keystore2’s shared-secret negotiation and before vold’s FBE key generation.
- That the helper can execute under the donor SELinux policy and reach Keystore2.
- That the first `on init` KM4 abort is harmless and the system `on late-fs` `class_start early_hal` restart succeeds after `/vendor/firmware_mnt` is mounted. The ordering is confirmed; the device behavior is not.
- That the QSEECom listener gate completed for the current asset and that `qseecom-service` registered before the late-fs KM4 start.
- That the prior log belongs to the current `f51460ef…` asset (it does not; no SHA linkage exists).
- That the A11 kernel supports the FBE policy version actually selected and the inline-encryption path used by `/data`.
- That `/data` is intact and was not altered by earlier experiments. `/keydata` and `/keyrefuge` are no longer considered a likely cause of this failure, but their state is still unknown and must not be modified.
- That the package’s `boot.img` kernel is the stock A11 kernel and is compatible with the donor system.
- That the dynamic-partition geometry is valid for the user’s exact A115F variant/CSC.
- That a stock `A115FXXS6CWK2` package for the user’s exact CSC can be obtained and flashed successfully.
- That the device’s bootloader is unlocked and AVB/verification is disabled enough to boot the modified `boot`/`dtbo`.
- That Keystore2’s HIDL KM4 fallback and the A11 KM4 HAL are runtime-compatible.
- That the public asset really came from the only branch containing the barrier. The release metadata and the packaged `init.target.rc` agree, but the build has not been reproduced locally.

## 6. Device-only tests (cannot be completed offline)

1. With the exact `f51460ef…` ZIP and a verified recovery path, capture the kernel log and check for, in order:
   - `init: starting service 'keymaster-4-0'` — the first, pre-firmware start
   - `QSEECom_start_app failed` / `KeymasterUtils` — the expected first-start abort, if it occurs
   - `mount_all /vendor/etc/fstab.qcom --early` or the `apnhlos` mount at `/vendor/firmware_mnt`
   - `QSEECOM DAEMON RUNNING`
   - `vendor.sys.listeners.registered`
   - `IQSEECom registerAsService is successful`
   - `init: starting service 'keymaster-4-0'` — the late-fs start
   - `A115F_KEYMASTER_BARRIER_BEGIN`
   - `Waiting for Keymaster device`
   - `Keymaster device ready`
   - `A115F_KEYMASTER_BARRIER_COMPLETE`
2. Do not stop at the first `QSEECom_start_app failed`; that line is expected before `/vendor/firmware_mnt` is mounted. The decisive question is whether the **late-fs** start succeeds and registers KM4.
3. Capture `adb logcat -b all -v threadtime -d` and inspect `vold`, `keystore2`, `keymaster`, `QSEECom`, and linker errors.
4. Run `getprop` for `ro.product.first_api_level`, `ro.crypto.state`, `ro.crypto.type`, `ro.crypto.volume.options`, and `init.svc.wait_for_keymaster`.
5. Run `dmesg | grep -iE "linker|fscrypt|keymaster|qsee|dm-"` and preserve the full output.
6. If ADB never appears, collect `/sys/fs/pstore/*`, `/proc/last_kmsg`, `/cache/recovery/last_log`, and `/cache/recovery/last_install` from Recovery.
7. Compare the `/oem` mount failure on the port against a stock A11 boot log; only treat it as a defect if stock A11 does not show the same failure.
8. Inspect `/data` read-only from Recovery (mount/fsck read-only, no format) to determine whether it is intact. Treat `/keydata` and `/keyrefuge` as unrelated to this specific failure unless the log shows otherwise.
9. Confirm the device still reaches Download Mode and can be restored with the matching stock firmware after the test.

## 7. Recovery prerequisites (must be satisfied before installation)

1. Exact model/CSC and current firmware confirmed from the device (not inferred).
2. Matching stock `A115FXXS6CWK2` firmware downloaded from a trusted source and hash-recorded.
3. Verified ability to enter Download Mode and flash stock firmware; Odin/Heimdall available.
4. Bootloader unlock status and AVB implications documented.
5. User explicitly approves installation after reading the risks.
6. A backup of important data exists; the test must not require formatting `/data` to diagnose the failure.

Without items 1–6, installation must not be recommended, regardless of ZIP availability.

## 8. Blockers and next actions

### Highest-priority blocker

The only raw `enablefilecrypto` failure log predates the Keymaster barrier. The current `f51460ef…` package contains the barrier and markers but has not been device-tested. The barrier is correctly placed, but it waits for Keystore2 registration, not for KM4; therefore barrier completion alone cannot prove that the FBE key path is healthy. In addition, the first `on init` KM4 start is expected to abort before `/vendor/firmware_mnt` is mounted, so the log must show whether the late-fs restart actually succeeds.

### Next concrete actions

1. Device-test the exact `f51460ef…` asset with a verified stock restore path and explicit approval; capture the barrier, both `keymaster-4-0` starts, Keystore2, KM4, QSEECom, and FBE lines.
2. Use the branch tree in `NEXT_DEVICE_TEST.md` to decide whether the failure is before the barrier, inside Keystore2/KM4 discovery, or inside KeyStorage/kernel FBE.
3. Do not move the barrier into vendor `on post-fs-data`: system `installkey /data` would run before it. Do not add another Keymaster bridge before the log identifies the branch.
4. Samsung ODE/UCM audit: **complete.** Donor vold consults `isUcmOdeMode()` only for `wrappedkey_v0`; A11 `/data` has no such flag, so ODE is not engaged. `vndservicemanager` is also confirmed present in the A11 vendor. Both branches are closed.
5. Compatibility audit: **complete.** Kernel (4.9.227) has FS_ENCRYPTION/ICE/XTS/PFK; VINTF has no missing required framework HAL and no boot-blocking extra HAL; the boot image is stock A11; the installer does not touch `/data`; dynamic-partition geometry matches the donor images. No new blocker found.
6. Keep readiness at R0; do not build or publish another ZIP until the root cause or a decisive device-log branch is established.
6. Do not rebuild from `manus/a115f-build-once` or `manus/a115f-bootability-preflight`: both lack `scripts/patch_a115f_init.py`, so their ZIPs would not contain the barrier. If a rebuild is ever approved, start from `23f8998f…` or merge that patcher deliberately.

### What requires the user / physical device

- The original failure logs and the exact tested ZIP SHA.
- The exact A11 CSC and current firmware string.
- The physical device for installation and log collection, after a recovery path is verified.
- Explicit approval before any installation or destructive `/data` action.

## 9. Artifacts and evidence

- Public source repo: <https://github.com/nova-arch008/ZineROM-a11q> (branch `manus/a115f-vendor-fingerprint`, commit `23f8998f…`)
- Public release repo: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test>
- Release asset URL: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip>
- Key evidence used for this report:
  - donor `system_build.prop` — donor system properties
  - A11 `vendor_build.prop` — A11 vendor properties
  - `wait_for_keymaster.rc` — helper service definition
  - `wait_for_keymaster` binary and strings — helper dependencies
  - patched `init.target.rc` — Keymaster barrier
  - A11 `libkeymasterdeviceutils.so` — `QSEECom_start_app("/vendor/firmware_mnt/image", "keymaster64")` and abort path
  - A11 `libQSEEComAPI.so` — TA image opened by path, no retry sleep
  - A11 `init.qti.qseecomd.sh` — listener-registration loop
  - A11 `vendor_fstab` — `/data`, `/keydata`, `/keyrefuge`, `/vendor/firmware_mnt`
  - donor `plat_sepolicy.cil` and A11 `vendor_sepolicy.cil` — policy types and attributes
  - donor `keystore2` and `vold` strings/disassembly — KM4 support and FBE call path
  - donor `/system/lib64` listing — complete 64-bit Keymaster HIDL client stack

Local audit files are not published because they contain machine-specific paths.

## 10. Session checkpoint — 2026-10-29 (historical pre-disconnect)

> **Historical note (2026-11-03):** this section records the state on 2026-10-29. The `/system/lib64` and product build.prop audits mentioned below are now complete (`system_full.dat` and `product_final.dat` are fully reconstructed), and the 2026-11-03 checkpoint supersedes the resume commands. Keep this section only as a record of what was known at that time.

### Exact current blocker

The boot stops at vold’s AIDL `fbeEnable()`, which fails inside `fscrypt_initialize_systemwide_keys()`. The exact failing sub-step is still unknown. The four candidates are:

1. `init_data_file_encryption_options()` — fstab parsing. This is the least likely: the A11 `/data` entry is structurally valid (`fileencryption=ice`, no key directory).
2. `retrieveOrGenerateKey()` — device-key retrieval through KeyStorage/Keystore2/Keymaster. This is the leading unverified branch because Android 15 Keystore2 must bind the A11’s Keymaster 4.0 HIDL HAL.
3. `install_storage_key()` — kernel fscrypt key installation/policy.
4. Writing `/data` encryption options/references after key installation.

No successful boot has been observed. No fix has been built or tested.

### Changes made in this session

- Created and published the public status repository `zinerom48-tech/ZineROM-A115F-OneUI7-port-status`; initial commit `e58626e`.
- Updated `PORTING_STATUS.md` and `ISSUE_REGISTER.md` with the new policy, library, fstab, vold-path, and Keymaster findings.
- Added `README.md`, `NEXT_DEVICE_TEST.md`, `RESUME.md`, and this checkpoint.
- Closed ISSUE-014: the donor `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`; all inspected dependencies are `ELF32` ARM.
- Added ISSUE-019 (unverified 64-bit Keymaster HIDL client/Keystore2 ABI) and ISSUE-020 (unknown FBE sub-step).
- Confirmed the Android 15 call chain: `installkey /data` → `vdc cryptfs enablefilecrypto` → vold `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`.
- Confirmed `ro.crypto.type=file` is set by `fs_mgr` from the fstab, and the A11 vendor init does not call `enablefilecrypto` directly.
- Confirmed the A11 `/data` fstab uses `fileencryption=ice` with no `keydirectory=`; `/keydata` and `/keyrefuge` use `fileencryption=ice,nofail`.
- Confirmed donor Keystore2 contains Keymaster 4.0 support (`KEY_MASTER_V4_0`, `IKeymasterDevice`, `km_compat`) and that the donor platform policy includes `keystore` in `hal_keymaster_client` and `hal_keymint_client`.
- Corrected the earlier `/system/lib64` result: the empty listing was caused by `ERROR: read beyond end of image`, not by a missing 64-bit Keymaster library.
- Extended the donor system image to `system_big5.dat` (`1,324,875,776` bytes). The `/system/lib64` inode (2690) has its extent leaf at physical block `393,336` (byte offset `1,611,104,256`), so the listing still needs more decompressed data.
- Saved a partial next range at `system_next5.br` (`12,731,492` bytes) after the download reset. No complete `/system/lib64` listing exists yet.
- No ROM build, no new ZIP, no flash, no format, and no destructive `/data` operation was performed.

### Build results

- **No build was run in this session.**
- The existing public artifact remains `ZineROM_OneUI7_A115F-Port-A057F.zip`, size `1,842,649,818` bytes, SHA-256 `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95`.
- That artifact came from GitHub Actions run `#40494388352` on commit `23f8998f0200969f5748e9eb012d1dc52037a054`, passed 62 static/package tests, and was **not** boot-tested on a phone.
- No local build has been reproduced, so the asset is still treated as a binary artifact.
- No new ZIP was generated.

### Evidence behind the `/data` encryption failure

1. **Prior log:** `cryptfs enablefilecrypto` returns `Status(-8, EX_SERVICE_SPECIFIC)`; Rescue Party reason is `enablefilecrypto_failedK00R`; init reboots to Recovery at ~14.23 s.
2. **Exact call chain:** donor init calls `installkey /data`; `vdc` maps `cryptfs enablefilecrypto` to the AIDL `fbeEnable()`; Android 15 `fbeEnable()` calls `fscrypt_initialize_systemwide_keys()`.
3. **A11 fstab:** `/data` is `f2fs` with `fileencryption=ice`, no `keydirectory=`, `latemount`, `check`, `quota`, `checkpoint=fs`. `/keydata` and `/keyrefuge` are `ext4` with `fileencryption=ice,nofail`.
4. **`ro.crypto.type`:** set to `file` by `fs_mgr` when the fstab reports file encryption (`fs_mgr.cpp:1092`), so `installkey` reaches `vdc`.
5. **32-bit helper:** `wait_for_keymaster` is `ELF32` ARM; the donor `/system/lib` contains all its 32-bit dependencies, all `ELF32` ARM; `libc.so`, `libm.so`, and `libdl.so` are runtime-APEX symlinks.
6. **Keystore2/Keymaster:** donor Keystore2 contains KM4 support, so the HAL is not architecturally excluded. The **64-bit** Keymaster HIDL client (`android.hardware.keymaster@4.0.so`, `libkeymaster4support.so`) is still unverified because `/system/lib64` has not been listed.
7. **Samsung ODE:** `/keydata` and `/keyrefuge` are mounted with file encryption; the A11 vendor contains `libcryptfshw*`, VaultKeeper, DRK, and secure-storage HALs. Android 15 vold compatibility is unproven.
8. **Non-causes:** `K00R` appears on booted Samsung devices; QSEE `0x72000206` appears on a booting LineageOS daisy device; the `/oem` zero-superblock mount is a stock A11 init action and boot continues past it.

### Required next commands

**Device test (only with a verified stock restore path and explicit approval):**

```sh
adb devices
adb shell getprop > getprop.txt
adb shell dmesg > dmesg.txt
adb logcat -b all -v threadtime -d > logcat-all.txt
adb pull /sys/fs/pstore pstore
adb pull /cache/recovery/last_log last_log.txt
adb pull /cache/recovery/last_install last_install.txt
```

Then grep the full logs for:

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

**Offline resume — complete the `/system/lib64` audit:**

```bat
curl.exe -L --fail --retry 5 -r 830236850-997505357 -o system_next5b.br https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip
copy /b system_big5.br+system_next5.br+system_next5b.br system_big6.br
brotli\brotli.exe -d -c system_big6.br > system_big6.dat 2> brotli_big6_err.txt
AuditTool.exe ext4ls system_big6.dat /system/lib64 1 > system_lib64_big6.txt
cscript //nologo searchfile.vbs system_lib64_big6.txt keymaster
cscript //nologo searchfile.vbs system_lib64_big6.txt keymint
```

If the partial range is corrupt, re-download the full range `817505358-997505357` to `system_next5.br` and repeat. The exact offsets and hashes are in `RESUME.md`.

**Offline resume — product properties (not yet complete):**

`product_full.dat` is only `83,886,080` bytes; the product partition is `312,815,616` bytes, so `/product/etc/build.prop` is still outside the decompressed prefix. Extend the product image before running:

```bat
AuditTool.exe ext4cat product_full.dat /etc/build.prop product_build.prop.txt
```

### Tasks still running / disconnect safety

- No build, flash, format, or download is running. The only Git activity is this checkpoint commit/push. The last range download was reset; its partial file is saved.
- It is safe to disconnect **after this checkpoint is committed and pushed**. The public repo is the saved state.
- Do not delete `system_big5.dat`, `system_big5.br`, or `system_next5.br`; they are the resume point.

### Resume state

- Status repo: `https://github.com/zinerom48-tech/ZineROM-A115F-OneUI7-port-status`, branch `main`.
- Quick-resume file: `RESUME.md` in that repo.
- Current local image prefix: `system_big5.dat`, `1,324,875,776` bytes.
- Current compressed prefix: `system_big5.br`, `656,441,987` bytes, SHA-256 `ff872e36e4c8902f88ac855e3a682e2fae36387501ba16d842d6e7f05b4cda5e`.
- Partial next range: `system_next5.br`, `12,731,492` bytes, SHA-256 `2e9e05e9450e2bedb838e9ac609b5618b3b19d7afae3926a6423122195ebd223`; starts at ZIP absolute offset `817,505,358` (compressed-relative `656,441,987`); resume at absolute `830,236,850`.
- `/system/lib64` inode `2690`; extent leaf physical block `393,336`; byte offset `1,611,104,256`.

## 11. Session checkpoint — 2026-11-03 (barrier/build-timeline correction)

### Correction: the barrier-deadlock theory is retracted

The earlier concern that the vendor `on late-fs` barrier might run before the system `on late-fs` `class_start early_hal` is disproven. AOSP `ImportParser::EndFile()` parses imports at the end of the parent file, so system `init.rc` actions are queued before imported vendor actions. The actual order is:

1. system `on late-fs`: `class_start early_hal`
2. vendor `on late-fs`: `A115F_KEYMASTER_BARRIER_BEGIN`
3. vendor `on late-fs`: `exec_start wait_for_keymaster`
4. vendor `on late-fs`: `A115F_KEYMASTER_BARRIER_COMPLETE`
5. vendor `on late-fs`: `mount_all /vendor/etc/fstab.qcom --late`

This matches Android metadata-encryption guidance. Moving the barrier to vendor `on post-fs-data` would be wrong because system `installkey /data` executes before that trigger.

### Build timeline

| Build / commit | Date | Barrier? | Evidence |
|---|---|---|---|
| `1103fc00b36b7aa427a7bec4dd43db8aad92f2254f6022253d09704f23667c3b` — prior diagnostic ZIP | before 2026-10-25 | No | `BOOT_PROGRESS.md` states the previous diagnostic ZIP omitted the barrier |
| `190586c39cd86057887d7ae84a4631761ace7c54` | 2026-10-25 | Yes | Commit message: `fix(a115f): gate late data mount on keymaster readiness` |
| `23f8998f0200969f5748e9eb012d1dc52037a054` | 2026-10-27 | Yes + markers | Commit message: `diag(a115f): log keymaster readiness barrier` |
| `f51460ef…` — current public asset | built from `23f8998f…` | Yes + markers | Released vendor `init.target.rc` MD5 `e111ab7f49ebcc8fa70d3253ccba9bce` contains `exec_start wait_for_keymaster` before `mount_all --late` |

No barrier-containing build is known to have been device-tested. The only raw `enablefilecrypto` log predates the barrier.

### What the barrier proves — and does not prove

- The packaged `wait_for_keymaster` is the A11 Android 12 helper (`ELF32`, SHA-256 `0e64ac8862492b8f7455a5320189c754dc8f3fef720dbf0b0f71849cb86811c9`), not a donor Android 15 binary. Its source is `system/vold/wait_for_keymaster.cpp`.
- The helper calls `AServiceManager_waitForService("android.system.keystore2.IKeystoreService/default")`, then constructs `android::vold::Keymaster`.
- `Keymaster::Keymaster()` calls `getSecurityLevel(TRUSTED_ENVIRONMENT)`. On failure it logs `Vold unable to get security level from keystore2.` and returns; the helper still logs `Keymaster device ready` and exits 0.
- Therefore `A115F_KEYMASTER_BARRIER_COMPLETE` proves that Keystore2 registered. It does **not** prove that KM4 registered, that KM4 is usable, or that `getSecurityLevel(TE)` succeeded.
- Android 15 Keystore2 calls `perform_shared_secret_negotiation()` before registering. Negotiation lists running HIDL KM4 4.1/4.0 instances and retries the listed participants; if KM4 is not registered when the list is built, the participant list can be empty and Keystore2 still registers.
- `libkm_compat` `getDevice()` caches the TEE device only if it is found. If it is not found, a later call re-enumerates, so a late KM4 registration can still be picked up. However, the diagnostic barrier does not wait for that.

### VINTF, system_ext, and service discovery

- Android 15 FCM 202404 and FCM 8 contain no Keymaster/KeyMint entries; FCM 7 (Android 12) lists `android.hardware.keymaster` twice. VINTF is not rejecting the A11 KM4 HAL.
- `/system_ext` is a symlink to `/system/system_ext`; `hwservicemanager` is present at `/system/system_ext/bin/hwservicemanager` and is started by donor `init.rc`.
- `vndservicemanager` is not in the donor system, but the A11 vendor provides `/vendor/bin/vndservicemanager` and `/vendor/etc/init/vndservicemanager.rc` (`service vndservicemanager /vendor/bin/vndservicemanager /dev/vndbinder`, class `core`). The donor `init.rc` `start vndservicemanager` therefore resolves. This is not a service-discovery gap.
- A11 vendor SELinux defines `hal_keymaster_qti` and `hal_keymaster_qti_exec`; the KM4 binary is labeled `hal_keymaster_qti_exec`; `/data/vendor/keymaster` is `vendor_keymaster_data_file`. Donor policy includes `keystore` in `hal_keymaster_client` and `hal_keymaster_default`/`hal_keymaster_qti` in `hal_keymaster_server`. Runtime denials remain possible but are not proven.

### Leading root-cause hypothesis

The `enablefilecrypto_failed` log most likely came from a build before the barrier (`1103fc00…` or earlier). The barrier was added on 2026-10-25 and packaged in the current asset, but it has not been device-tested. This is a hypothesis, not a confirmed fix. Even with the barrier, KM4 readiness is not guaranteed because the helper waits on Keystore2, not KM4.

### Device-log branch tree

1. **No `A115F_KEYMASTER_BARRIER_BEGIN`** — the helper service did not run. Check init service state and SELinux, not FBE.
2. **BEGIN but no COMPLETE** — `wait_for_keymaster` hung. Check Keystore2 registration and servicemanager.
3. **COMPLETE plus `Vold unable to get security level from keystore2`** — Keystore2 is up but KM4 is not usable. Check KM4/qseecom startup, HIDL registration, and SELinux.
4. **COMPLETE plus `Keymaster device ready` with no security-level error, then FBE failure** — KM4 may be present but key generation/profile handling failed. Inspect vold KeyStorage and Keystore2 error codes.
5. **COMPLETE plus FBE success** — the barrier likely fixed the original race; continue to the next boot stage.

## 12. Compatibility audit — kernel, VINTF, boot image, installer (2026-11-04)

**Kernel (`ikconfig.txt`, Linux 4.9.227, ARM64):** `CONFIG_FS_ENCRYPTION=y`, `CONFIG_EXT4_FS_ENCRYPTION=y`, `CONFIG_EXT4_FS_ICE_ENCRYPTION=y`, `CONFIG_F2FS_FS_ENCRYPTION=y`, `CONFIG_PFK=y`, `CONFIG_CRYPTO_XTS=y`. The kernel supports FBE v1 with inline encryption (ICE) and PFK, matching the A11 `/data` f2fs `fileencryption=ice` layout. No kernel feature gap is evident for the selected policy.

**Boot image (`boot.img`):** the ramdisk `build.prop` identifies stock A11 firmware `samsung/a11qnsxx/a11q:12/SP1A.210812.016/A115FXXS6CWK2:user/release-keys`. The ramdisk `fstab.qcom` matches the vendor fstab for `/data` (`f2fs`, `fileencryption=ice`, `latemount`, `checkpoint=fs`), `/keydata` and `/keyrefuge` (`ext4`, `fileencryption=ice,nofail`), and `/vendor/firmware_mnt` (`vfat`, `apnhlos`). The package therefore retains the stock A11 boot image and its first-stage mount table.

**VINTF — framework-matrix direction (`vendor_manifest.xml` vs `donor_compat_*.xml`):** the vendor manifest declares 74 HALs. 51 are vendor-specific (`vendor.*`, `com.qualcomm.*`, `com.samsung.*`, etc.) and are correctly absent from the donor framework compatibility matrices, which only list framework-relevant HALs. Of the `android.hardware.*` entries, three are absent from every donor matrix: `android.hardware.configstore`, `android.hardware.keymaster`, and `android.hardware.radio.deprecated`. All three are legacy/optional for an Android 15 framework: `configstore` was superseded by system properties, `keymaster` HIDL by KeyMint AIDL, and `radio.deprecated` by the radio 1.x/2.x interfaces. VINTF treats an undeclared device HAL as an extra provided by the device, not a failure, so none of these is a boot blocker by itself. The `keymaster` absence is consistent with the port’s reliance on `libkm_compat`’s HIDL fallback at runtime.

**VINTF — device-matrix direction (`vendor_compat_matrix.xml` vs `system_manifest.xml`):** the device matrix requires six framework HALs: `android.frameworks.sensorservice`, `android.hidl.allocator`, `android.hidl.manager`, `android.hidl.memory`, `android.hidl.token`, and `android.system.wifi.keystore`. Of these, `allocator`, `manager`, and `token` are core HIDL services provided by `hwservicemanager`/`servicemanager` and are not enumerated in the donor framework manifest, which is normal. `sensorservice`, `memory`, and `wifi.keystore` are present. No required framework HAL is missing.

**Installer (`updater-script.txt`):** the package checks `ro.boot.em.model == SM-A115F` and `ro.boot.bootloader == A115FXXS6CWK2`, then patches `system`, `vendor`, `product`, and `odm` via block-image update, writes `dtbo` and `boot`, and updates dynamic-partition metadata. It does not modify, wipe, or format `/data`, `/keydata`, or `/keyrefuge`. The dynamic-partition geometry (`system` 3,213,774,848; `vendor` 527,249,408; `product` 312,815,616; `odm` 1,642,496; group 4,320,133,120) matches the donor image sizes and the A11 super partition (4,324,327,424).

**Vendor properties (`vendor_build.prop.txt`):** `ro.product.first_api_level=29` and `ro.vendor.radio.first_api_level=29`, confirming FBE policy v1. The donor system `ro.system.product.cpu.abilist` is `arm64-v8a,armeabi-v7a,armeabi`, so both ABIs are available and `ro.zygote` resolves to `zygote64_32`.

**Conclusion of this audit:** no additional kernel, VINTF, boot-image, or installer defect was found beyond the already-tracked Keymaster/FBE blocker. The remaining decisive work is the device-side log capture for the exact `f51460ef…` asset.

## 13. Session checkpoint — 2026-11-04 (KM4 start ordering, ODE closure, source branches)

### KM4 constructor and the first-start failure

Disassembly of the A11 `/vendor/lib/libkeymasterdeviceutils.so` shows that the `KeymasterUtils` constructor calls:

```text
QSEECom_start_app(handle, "/vendor/firmware_mnt/image", "keymaster64", 0xa000)
```

If that returns nonzero, it retries with app name `"keymaster"`. If both calls fail, it reaches `__android_log_assert` with tag `KeymasterUtils` and message `QSEECom_start_app failed`, which aborts the KM4 process. `libQSEEComAPI.so` opens the TA image with `fopen` / `%s/%s.mdt` and has no `sleep`/`usleep` retry loop.

`/vendor/firmware_mnt` is the `apnhlos` partition, mounted from `/vendor/etc/fstab.qcom` during `on fs` `mount_all --early`. The A11 service definition starts `keymaster-4-0` at `on init`, **before** that mount. The first start is therefore expected to abort on the missing `image/keymaster64` file. The effective start is the system `on late-fs` `class_start early_hal`, which runs before the imported vendor `on late-fs` barrier. AOSP `Service::Start()` clears `SVC_RESTARTING`, so the late-fs class start can launch a fresh process after the firmware mount.

`restart_period` defaults to 5 seconds in Android 12 and 15. Android 15’s 4-crash fatal limit applies only to `SVC_CRITICAL` or updatable services; `keymaster-4-0` is neither, so repeated aborts do not permanently disable it.

The A11 `keymaster-4-0` rc has no `interface` declaration. AOSP init uses that option only for lazy service start; the service is still started explicitly by `on init` and by `class_start early_hal`. The practical consequence is that there is no lazy `interface_start` fallback if the process exits before the late-fs class start, so the log must show the explicit second start.

### QSEECom listener gate

A11 `init.target.rc` starts `vendor.qseecomd` at `on post-fs` and then runs `/vendor/bin/init.qti.qseecomd.sh` with blocking `exec`. The script loops until `vendor.sys.listeners.registered=true`. The `qseecom-service` HAL starts at `on init`, class `hal`, and logs `IQSEECom registerAsService is successful`. The prior boot reaching FBE implies this gate completed at least once, but the current asset’s log must show it for the same boot as the barrier.

### ODE/UCM branch closed

Donor `vold` has exactly one direct call to `isUcmOdeMode()`, at `0x102C68`, inside `ParseOptionsForApiLevel` while parsing the `wrappedkey_v0` flag. `isUcmOdeMode()` reads `/efs/sec_efs/ucm_ode_mode`, parses it as base-10, and returns true only for `2`; any missing/unreadable/other value returns false. The A11 `/data` fstab uses plain `fileencryption=ice` with no `wrappedkey_v0`, so ODE is not engaged for first `/data` encryption. `/keydata` and `/keyrefuge` are mounted by the A11 fstab but donor vold has no code path that consumes them in `fbeEnable()`. This branch is closed for the current failure; do not format either partition.

### Source-branch audit

| Branch | Head | Relationship to `23f8998f` | Has barrier patcher? | Relevant change |
|---|---|---|---|---|
| `manus/a115f-vendor-fingerprint` | `23f8998f` (2026-10-27) | base (13 commits ahead of `build-once`) | **Yes** — `scripts/patch_a115f_init.py` | Barrier + markers; released asset was built from this commit |
| `manus/a115f-build-once` | `ec9239a1` (2026-10-24) | ancestor of `23f8998f` (13 commits behind) | **No** — file absent | Older branch; `23f8998f` is a strict superset, so it contains everything `build-once` has plus the barrier and audits |
| `manus/a115f-bootability-preflight` | `a61d5a41` (2026-11-01) | diverged: 2 commits ahead, 15 behind `23f8998f` | **No** — file absent | Adds stock-blob extraction (`0266bc3c`) and the public-artifact-upload visibility guard (`a61d5a41`); lacks the barrier and the 13 audit commits |

The branch names are misleading: `build-once` is an ancestor, not a newer candidate, and `bootability-preflight` is a side branch with only two commits. A ZIP built from either would silently lack the Keymaster barrier. If a rebuild is ever approved, start from `23f8998f…` and, if desired, cherry-pick `0266bc3c` (stock blob extraction) — never the reverse.

### What the next device log must decide

1. Did the first `on init` KM4 start abort with `QSEECom_start_app failed`?
2. Did `/vendor/firmware_mnt` mount, and did `vendor.qseecomd` reach `vendor.sys.listeners.registered`?
3. Did the late-fs `keymaster-4-0` start register `android.hardware.keymaster@4.0` with TEE security level?
4. Did `wait_for_keymaster` complete, and did `getSecurityLevel(TE)` succeed?
5. If FBE still fails, which `fscrypt_initialize_systemwide_keys()` sub-step or Keystore2 error is first?

No build, flash, format, or new ZIP was produced in this session. Readiness remains **R0**.

## 14. Change log

- 2026-10-28: Initial status created from repository, release metadata, and static package metadata. No build or device test performed.
- 2026-10-28: Corrected the source-access claim (repo is public), added the current release asset hash and the stale `BOOT_PROGRESS.md` hash discrepancy, confirmed the donor’s 32-bit support and the helper’s ELF32 dependencies, recorded the FBE v1/v2 decision path, demoted `K00R` and QSEE `0x72000206`, and identified the stock `/oem` mount and Samsung ODE partitions as separate open items. No build or device test performed.
- 2026-10-29: Closed the 32-bit helper-dependency question after listing the donor `/system/lib`; confirmed the `vdc` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()` call chain; confirmed `ro.crypto.type=file` comes from the fstab; recorded the A11 `fileencryption=ice` layout; confirmed donor Keystore2 KM4 support and the platform policy `keystore` membership; corrected the `/system/lib64` result as a truncated-image artifact; added the four FBE sub-step hypotheses and the 64-bit Keymaster/HAL verification branch. Still R0; no build or device test performed.
- 2026-10-29: Added the pre-disconnect checkpoint with the exact blocker, all session changes, build results, `/data` FBE evidence, resume offsets/hashes, and next commands; published the status repo. Still R0; no build or device test performed.
- 2026-11-03: Retracted the init import-order/deadlock theory after reading AOSP `ImportParser::EndFile()`; confirmed the barrier is correctly placed after system `class_start early_hal` and before vendor `mount_all --late`; recorded the barrier build timeline (`1103fc00…` pre-barrier, `190586c…` barrier, `23f8998f…` markers, current `f51460ef…`); confirmed FBE v1, the 64-bit KM4 compat libraries, merged system_ext, VINTF non-blocking, and the A11 vendor `vndservicemanager`; documented that `wait_for_keymaster` gates on Keystore2 registration, not KM4, and added the decisive device-log branch tree. Still R0; no build or device test performed.
- 2026-11-04: Decoded the A11 `KeymasterUtils` constructor (`QSEECom_start_app(…, "/vendor/firmware_mnt/image", "keymaster64", …)` with a `"keymaster"` retry and a `QSEECom_start_app failed` abort); established that the first `on init` KM4 start necessarily precedes the `apnhlos` mount at `on fs` and is expected to abort; confirmed AOSP `Service::Start()` clears `SVC_RESTARTING`, so the system `on late-fs` `class_start early_hal` can start a fresh KM4; confirmed the `restart_period` default of 5 s and that Android 15’s 4-crash limit does not apply; documented the `vendor.qseecomd` listener gate and `qseecom-service` registration; confirmed the KM4 rc has no `interface` declaration and therefore no lazy-start fallback; closed the Samsung ODE/UCM branch from donor `vold` disassembly (`isUcmOdeMode()` is consulted only for `wrappedkey_v0`, which A11 `/data` does not use); completed the donor `/system/lib64` listing and confirmed the full 64-bit KM4 HIDL stack; reconciled the source branches via the GitHub compare API (`23f8998f…` is 13 commits ahead of `build-once`, which is a strict ancestor; `preflight` is 2 commits ahead and 15 behind, adding stock-blob extraction and the upload-visibility guard); completed the kernel/VINTF/boot-image/installer compatibility audit (kernel 4.9.227 has FS_ENCRYPTION/ICE/XTS/PFK; no missing required framework HAL; stock A11 boot image; installer does not touch `/data`; geometry matches donor images). No build, flash, format, or new ZIP was produced. Still R0; no device test performed.
