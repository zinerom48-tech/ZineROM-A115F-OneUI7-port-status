# ZineROM One UI 7 — Galaxy A11 (SM-A115F) Port Status

Last updated: 2026-10-29

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
| Offline root-cause isolation | **~65%** | Failure narrowed to vold `fbeEnable()` → `fscrypt_initialize_systemwide_keys()`. The exact failing sub-step is still unknown. |
| Validated fix | **0%** | No fix has been built, flashed, or tested. |
| Physical-device validation | **0%** | No A11 is available for installation, logcat, pstore, or hardware testing. |
| Build/package audit | **~85% static** | The artifact exists and its package structure is audited. This is not evidence that it boots. |

These percentages are project bookkeeping, **not** a probability that the ROM will boot. The remaining framework/UI work is larger than the stage count suggests.

## 2. Milestone status

| Milestone | Status | Evidence / reason |
|---|---|---|
| M0 — Repository and hardware audit | **In progress** | Public source repo, release metadata, package guards, donor/target identities, dynamic-partition geometry, kernel container, and A11 vendor HALs inspected. Hardware compatibility still has open items (Samsung ODE/keydata, QSEE/TA, SELinux). |
| M1 — Compatibility assessment and build environment | **Partial** | Source is public at `nova-arch008/ZineROM-a11q`; the A115F target and generic QSSI configuration were read. A build has **not** been reproduced locally. |
| M2 — Configuration and prerequisite validation | **Partial** | Model/bootloader guards, BLOCK OTA metadata, transfer lists, partition sizes, FBE version default, A11 fstab, and the Keymaster barrier were checked. Product/system_ext property overrides and the full 64-bit HAL set remain unverified. |
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
19. `fileencryption=ice` with no key directory is consistent with the A11’s FBE v1 + inline-crypto layout. The exact policy version still depends on the effective `ro.product.first_api_level`; product and system_ext properties remain unread.
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
30. The donor `/system/lib` listing is complete, but the `/system/lib64` listing is **not**. The earlier empty `/system/lib64` result ended with `ERROR: read beyond end of image`; it is a truncated-image artifact, not evidence that the 64-bit Keymaster client is absent. The `/system/lib64` inode points to an extent leaf at physical block `393,336` (byte offset ~1.611 GB), beyond the current 1.325 GB decompressed prefix.
31. No physical Galaxy A11 is available in this session. No installation, boot, logcat, pstore, or hardware test has been performed.

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

The effective `ro.product.first_api_level` is likely `29`:

- A11 vendor: `29`.
- Donor system: value absent.
- Product and system_ext build.prop files: not yet fully read because their image data is outside the decompressed window.

If the effective value is `29`, Android 15 `libfscrypt` selects FBE policy v1, matching the A11’s Android 12-era kernel expectations. If product or system_ext explicitly sets a value greater than `29`, the port would select v2 and the A11 4.9 kernel’s v2 support would become a serious risk. This is a concrete, testable branch, not a conclusion.

### Keymaster / 32-bit / 64-bit assessment

The earlier concern that the donor system might be 64-bit-only is resolved: the donor declares 32-bit ABI support, ships 32-bit binaries, and its `/system/lib` contains every 32-bit dependency of `wait_for_keymaster`. The A11 Keymaster HAL and the helper are both `ELF32` ARM.

The remaining Keymaster risks are:

- whether the donor’s **64-bit** `/system/lib64` contains the Keymaster 4.0 HIDL client used by Keystore2 (`android.hardware.keymaster@4.0.so`, `libkeymaster4support.so`) and whether Keystore2’s dependencies resolve;
- whether Android 15 Keystore2 can discover and bind the A11’s Keymaster **4.0 HIDL** service under the donor SELinux policy;
- whether the A11 Keymaster TA/QSEE path can generate the key type/capability requested by Android 15 FBE;
- whether the A11’s Samsung ODE `/keydata`/`/keyrefuge` state is compatible with Android 15 vold.

### Hypotheses (not confirmed)

- **H1 — Keymaster/TA mismatch:** the A11 Keymaster 4.0 HAL starts, but the TA/QSEE path cannot create the key type or capability requested by Android 15’s FBE setup.
- **H2 — Helper runtime gap:** largely resolved. The 32-bit helper dependencies are present and `ELF32`; a missing 32-bit library is no longer a leading explanation. ABI incompatibility or SELinux transition is still possible.
- **H3 — Samsung ODE/keydata mismatch:** Android 15 vold’s handling of `/keydata`/`/keyrefuge` or Samsung VaultKeeper/DRK differs from the A11 Android 12 format, so the FBE key cannot be unwrapped.
- **H4 — `/data` state:** a previous failed attempt left `/data` or its metadata partially encrypted/corrupt, so `enablefilecrypto` fails before Keymaster is reached. This must not be tested by formatting without explicit approval.
- **H5 — Kernel feature gap:** the A11 4.9 kernel lacks a crypto/inline-encryption feature required by the donor’s FBE path. This is less likely if policy v1 is selected, but not ruled out.
- **H6 — Barrier not exercised in the prior log:** the prior build may not have contained the barrier, or the helper produced no kernel messages because it never started. The current `f51460ef…` package contains the barrier and is the correct artifact for the next device test.
- **H7 — Keystore2/HAL ABI gap:** Keystore2’s 64-bit Keymaster HIDL client is missing, cannot resolve its dependencies, or cannot bind the A11 KM4 service. This is now the main unverified Keymaster branch.

## 5. Untested assumptions

- That the effective `ro.product.first_api_level` is `29` on the assembled system (product/system_ext not fully read).
- That the helper can execute under the donor SELinux policy and reach Keystore2.
- That the prior log belongs to the current `f51460ef…` asset (it does not; no SHA linkage exists).
- That the A11 kernel supports the FBE policy version actually selected and the inline-encryption path used by `/data`.
- That `/data`, `/keydata`, and `/keyrefuge` are intact and were not altered by earlier experiments.
- That the package’s `boot.img` kernel is the stock A11 kernel and is compatible with the donor system.
- That the dynamic-partition geometry is valid for the user’s exact A115F variant/CSC.
- That a stock `A115FXXS6CWK2` package for the user’s exact CSC can be obtained and flashed successfully.
- That the device’s bootloader is unlocked and AVB/verification is disabled enough to boot the modified `boot`/`dtbo`.
- That Keystore2’s 64-bit Keymaster HIDL client and the A11 KM4 HAL are runtime-compatible.

## 6. Device-only tests (cannot be completed offline)

1. With the exact `f51460ef…` ZIP and a verified recovery path, capture the kernel log and check for, in order:
   - `A115F_KEYMASTER_BARRIER_BEGIN`
   - `Waiting for Keymaster device`
   - `Keymaster device ready`
   - `A115F_KEYMASTER_BARRIER_COMPLETE`
2. Capture `adb logcat -b all -v threadtime -d` and inspect `vold`, `keystore2`, `keymaster`, `QSEECom`, and linker errors.
3. Run `getprop` for `ro.product.first_api_level`, `ro.crypto.state`, `ro.crypto.type`, `ro.crypto.volume.options`, and `init.svc.wait_for_keymaster`.
4. Run `dmesg | grep -iE "linker|fscrypt|keymaster|qsee|dm-"` and preserve the full output.
5. If ADB never appears, collect `/sys/fs/pstore/*`, `/proc/last_kmsg`, `/cache/recovery/last_log`, and `/cache/recovery/last_install` from Recovery.
6. Compare the `/oem` mount failure on the port against a stock A11 boot log; only treat it as a defect if stock A11 does not show the same failure.
7. Inspect `/data`, `/keydata`, and `/keyrefuge` read-only from Recovery (mount/fsck read-only, no format) to determine whether they are intact.
8. Confirm the device still reaches Download Mode and can be restored with the matching stock firmware after the test.

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

The boot-critical `enablefilecrypto` failure has no established root cause, and the current evidence cannot distinguish the four sub-steps of `fscrypt_initialize_systemwide_keys()`. The `f51460ef…` package contains the barrier but has not been device-tested.

### Next concrete actions

1. Obtain the raw failed-boot log and the exact tested ZIP SHA from the user/tester; correlate `enablefilecrypto`, vold, keystore2, Keymaster, and QSEE timestamps.
2. Require the next device test to use `f51460ef…` and capture the barrier plus vold/keystore2/keymaster/QSEE logs.
3. Use the log to identify the failing sub-step before changing fstab, FBE policy, or adding another Keymaster bridge.
4. Finish the 64-bit `/system/lib64` audit: verify `android.hardware.keymaster@4.0.so`, `libkeymaster4support.so`, and Keystore2’s `DT_NEEDED` entries. The current decompressed prefix ends before the directory data; this is a pending offline check, not a confirmed absence.
5. Finish the product build.prop audit only if cheap; do not block on it.
6. Keep readiness at R0; do not build or publish another ZIP until the root cause or a decisive device-log branch is established.

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
  - A11 `vendor_fstab` — `/data`, `/keydata`, `/keyrefuge`
  - donor `plat_sepolicy.cil` and A11 `vendor_sepolicy.cil` — policy types and attributes
  - donor `keystore2` and `vold` strings — KM4 support and FBE call path

Local audit files are not published because they contain machine-specific paths.

## 10. Change log

- 2026-10-28: Initial status created from repository, release metadata, and static package metadata. No build or device test performed.
- 2026-10-28: Corrected the source-access claim (repo is public), added the current release asset hash and the stale `BOOT_PROGRESS.md` hash discrepancy, confirmed the donor’s 32-bit support and the helper’s ELF32 dependencies, recorded the FBE v1/v2 decision path, demoted `K00R` and QSEE `0x72000206`, and identified the stock `/oem` mount and Samsung ODE partitions as separate open items. No build or device test performed.
- 2026-10-29: Closed the 32-bit helper-dependency question after listing the donor `/system/lib`; confirmed the `vdc` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()` call chain; confirmed `ro.crypto.type=file` comes from the fstab; recorded the A11 `fileencryption=ice` layout; confirmed donor Keystore2 KM4 support and the platform policy `keystore` membership; corrected the `/system/lib64` result as a truncated-image artifact; added the four FBE sub-step hypotheses and the 64-bit Keymaster/HAL verification branch. Still R0; no build or device test performed.
