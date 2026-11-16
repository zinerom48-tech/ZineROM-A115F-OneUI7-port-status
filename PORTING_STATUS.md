# ZineROM One UI 7 — Galaxy A11 (SM-A115F) Port Status

Last updated: 2026-11-16

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
| Offline root-cause isolation | **~95%** | Code-level root cause established: the A11 4.9 kernel implements only legacy fscrypt, while the Samsung donor `vold` and `init` have the legacy key path stripped. The exact first failing call on the prior build still requires a device log. |
| Validated fix | **0%** | A fix is designed but not built, flashed, or tested. |
| Physical-device validation | **0%** | No A11 is available for installation, logcat, pstore, or hardware testing. |
| Build/package audit | **~85% static** | The artifact exists and its package structure is audited. This is not evidence that it boots. |

These percentages are project bookkeeping, **not** a probability that the ROM will boot. The remaining framework/UI work is larger than the stage count suggests.

## 2. Milestone status

| Milestone | Status | Evidence / reason |
|---|---|---|
| M0 — Repository and hardware audit | **Complete for offline evidence** | Public source repo, release metadata, package guards, donor/target identities, dynamic-partition geometry, kernel container, A11 vendor HALs, VINTF/SELinux/boot-image/installer compatibility, and source-branch reconciliation are audited. Remaining hardware items (QSEE/TA runtime, SELinux runtime, KM4 late-fs registration) require a device log. Samsung ODE/keydata is closed for this failure. |
| M1 — Compatibility assessment and build environment | **Partial (source/build only)** | Source is public at `nova-arch008/ZineROM-a11q`; the A115F target and generic QSSI configuration were read; kernel, VINTF, SELinux, boot-image, and installer compatibility are audited. The fix branch’s 76-test suite passes in CI, including compile and ELF checks of both targets, but the build job stops before `make_rom.sh` on a nonexistent, runner-dependent workflow package (`linux-modules-extra-7.0.0-1014-aws`, then `…-1017-azure`). No ROM build has been reproduced locally or in CI. |
| M2 — Configuration and prerequisite validation | **Partial (config verified; runtime pending)** | Model/bootloader guards, BLOCK OTA metadata, transfer lists, partition sizes, FBE version default, A11 fstab, and the Keymaster barrier were checked. Product/system_ext property overrides and the 64-bit HAL set are audited; the remaining items are device-side runtime behavior. |
| M3 — Images compiled and inspected | **Partial (inspection only)** | Prebuilt `system`, `vendor`, `product`, `odm`, `boot`, and the patched vendor init were inspected. No image was compiled from source; the CI build is blocked at environment setup. |
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
37. **The A11 4.9.227 kernel implements only legacy fscrypt key management.** Its source uses `request_key(&key_type_logon, …)` and `fs/crypto/policy.c` exposes only set/get policy. A byte scan of the packaged `vmlinux.bin` found 6 `FS_IOC_SET_ENCRYPTION_POLICY` (`0x800C6613`) sites, 4 `FS_IOC_GET_ENCRYPTION_POLICY` (`0x400C6615`) sites, and **0** `FS_IOC_ADD_ENCRYPTION_KEY` (`0xC0506617`) sites. There is no `FS_IOC_REMOVE_ENCRYPTION_KEY` or `FS_IOC_GET_ENCRYPTION_KEY_STATUS` implementation either.
38. **The donor Samsung `vold` is ioctl-only.** `installKey` (`0x47114`, size `0x51C`) opens `/data` and calls `ioctl(fd, 0xC0506617, &key_spec)` at `0x472FC`; the failure log site is `Failed to install fscrypt key to ` at file offset `0x1842B`. `evictKey` (`0x47B74`, size `0x4E0`) calls `ioctl(fd, 0xC0406618, &arg)` at `0x47C30`. `waitForBusyFiles` (`0x48054`, size `0x624`) uses REMOVE and `FS_IOC_GET_ENCRYPTION_KEY_STATUS` (`0xC080661A`). Donor `vold` imports no `add_key` or `keyctl`; its only `syscall` callers are `async_safe_write_log` at `0x3FE20`/`0x3FF74`/`0x3FF84`. It links `libkeyutils.so` but imports neither wrapper symbol.
39. **The donor `vold` has no legacy fallback symbols.** The recovered `.gnu_debugdata` symbol table contains `installKey`, `evictKey`, and `waitForBusyFiles`, but no `installKeyLegacy`, `evictKeyLegacy`, `isFsKeyringSupported`, or `fscryptKeyring`. By contrast, LineageOS 22.2 (Android 15) and 23.1 (Android 16) `KeyUtil.cpp` still contain the complete fallback: `isFsKeyringSupportedImpl()` probes ADD with a null argument and treats `ENOTTY` as legacy; `installKeyLegacy()` searches PID 1's session keyring for the `fscrypt` keyring and adds `ext4:`, `f2fs:`, and `fscrypt:` logon keys with a 72-byte `fscrypt_key` payload; `evictKeyLegacy()` unlinks all three. **The earlier claim that “Android 15 removed legacy fscrypt support” is wrong; Samsung's donor binaries removed it.**
40. **The donor Samsung `init` does not create either keyring.** It has no `keyring`/`add_key`/`keyctl`/`request_key` strings, no `libkeyutils.so` DT_NEEDED, no key-related dynsyms, and no keyctl symbols in its `.gnu_debugdata`. The full `SecondStageMain` function (`0x155DFC`, size `0x5574`) contains no `svc` instruction, so the AOSP `keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1)` call in `init.cpp` is not present in this binary. AOSP/Lineage `init.cpp` still has that call and `FscryptInstallKeyring()` still creates the named keyring.
41. **The legacy key must live in PID 1's session keyring.** Kernel 4.9 resolves legacy fscrypt keys with `request_key` in the context of every process that touches `/data`; all processes inherit PID 1's session keyring. A vold-private session keyring would be invisible to zygote, apps, and system_server. `keyctl_session_to_parent()` rejects `parent->pid <= 1`, so a helper service cannot inject the keyring into init; PID 1 itself must create it.
42. **A PID-1 wrapper at `/system/bin/init` is viable.** AOSP 12 `FirstStageMain` executes `/system/bin/init` with `selinux_setup` before it loads SELinux policy (`first_stage22.cpp:559-565`), so a wrapper installed there runs in PID 1 before the policy-loading code. Donor `SetupSelinux` loads policy and re-execs `/system/bin/init` with `second_stage`, so the wrapper runs a second time; keyring creation is idempotent. The donor binary contains `/system/bin/init` and `second_stage` and has no `/proc/self/exe` string. The wrapper must preserve `argv[0]` because `/system/bin/ueventd` is a symlink to `init`; both `/system/bin/init` and `/system/bin/init.real` need `init_exec` for the post-policy re-exec. This design does not modify the boot image.
43. **The donor platform policy and the A11 vendor policy already permit the fix.** The donor `system_full.dat` is the One UI 7 / Android 15 system (`ro.build.id=AP3A.240905.015.A2`, `ro.build.version.sdk=35`), not an A11 image. Its `system/etc/selinux/plat_sepolicy.cil` (2,237,691 bytes, SHA-256 `febc700cfaed6a081bc50e90dca514c8ac68b606e4e842901b826379153b7c4c`, 33,891 lines) contains `(allow domain init (key (search)))`, `(allow domain vold (key (search)))`, `(allow domain kernel (key (search)))`, `(allow init init (key (write search setattr)))`, `(allow vold init (key (write search setattr)))`, and `(allow vold vold (key (write search setattr)))`. Its `system/etc/selinux/mapping/31.0.cil` (141,860 bytes, SHA-256 `ff34465aa410b0db57e1382aa87397079fb2d6c91c29da8f5817ef9530c03d47`) maps `vold_31_0` to `vold`, `init_31_0` to `init`, and `kernel_31_0` to `kernel`, so the A11 vendor rules bind. The A11 vendor policy (`VE=SEPF_SM-A115F_12_0001`, 1,803,461 bytes, SHA-256 `08e2f3a2c6e4b9cff575adb3d69841402f48effd400f4d791b3f5b525e30982b`) adds `(allow vold_31_0 kernel_31_0 (key (write search)))`, `(allow domain kernel_31_0 (key (search)))`, `(allow init_31_0 kernel_31_0 (key (search)))`, `(allow vold_31_0 init_31_0 (key (write)))`, and `(allow vold_31_0 self (key (view read write search link setattr create)))`. Kernel 4.9 `key_create_or_update()` requires only `write` on the destination keyring. The donor `system_ext_sepolicy.cil` (488,747 bytes, SHA-256 `a0a7cf4e21932384f5f26e1c0d77c00d496879d7374ba2af5f58dac498365f35`) contains no key rules, and `unica/patches/selinux` only deletes named One UI 7 types from `system_ext` mapping/genfscon, so it cannot remove an allow rule. No policy change is required for keyring creation, key installation, search, or unlink. **Correction:** earlier notes attributed this `plat_sepolicy.cil` to the A11 target (`system_big3.dat`); `system_big3.dat` presents the same donor ext4 geometry and SELinux inode numbers as `system_full.dat`, whose `build.prop` identifies Android 15, and the earlier local copy is byte-identical to the donor file.
44. **Expected device evidence if key installation is reached:** the ADD ioctl returns `ENOTTY` on f2fs 4.9, and donor vold logs `Failed to install fscrypt key to /data: Inappropriate ioctl for device`. If the prior build failed earlier, the log instead shows KeyStorage/Keystore2 errors. The prior excerpt's `Status(-8, EX_SERVICE_SPECIFIC)` does not distinguish these.
45. **The fix is implemented, compiled locally and in CI, host-tested, and pushed.** Branch `manus/a115f-legacy-fscrypt` is at `3aa20d4` (`74c8198` added the in-ROM mod-install test, `5651d76` added the CI compile/ELF test, `48438e6` is the pointer-size ABI fix, all based on `23f8998f…`) and adds 14 files: a freestanding `/system/bin/init` wrapper, a 64-bit vold `LD_PRELOAD` shim, shared legacy-key helpers, a `vold.rc` `setenv` patcher, host tests, a build script, a mod, a CI compile/ELF test, and an in-ROM mod-install test that also drives the real `apply_modules.sh` install path. The separate `.github/workflows/legacy-fscrypt-ci.yml` is held on local-only branch `local/a115f-legacy-fscrypt-ci-workflow` at `5551184` because the push credential lacks GitHub's `workflow` scope. Local compilation with Zig 0.17.1 (clang 22.1.8 + lld) produced a 2624-byte static wrapper and a 6280-byte shim with no warnings; the host unit tests pass; ELF checks confirm wrapper `ET_EXEC`, no `PT_INTERP`, no `.dynamic`, 0 `DT_NEEDED`, and shim `ET_DYN`, 0 `DT_NEEDED`, exact imports `__android_log_print __errno`, exported `ioctl`, no memory exports. Local SHA-256: wrapper `e323b0018ec21215d70b250257c591e092301c81c26ec625c12aae16b6b34916`, shim `e001ac7be935312959292dc63fa80e760e15104a87e90c03407b311ca02a6353`. These are Zig-built artifacts, not the ROM artifacts; the ROM is not built and no device test exists.
46. **The wrapper creates both keyrings before policy load.** It calls `keyctl(GET_KEYRING_ID, SESSION, create=1)`, searches for the named `fscrypt` keyring, creates it with `add_key("keyring","fscrypt",NULL,0,SESSION)` if absent, logs every result, preserves `argv[0]`, skips keyring setup for `ueventd`, and always execs `/system/bin/init.real`. It never aborts boot on keyring failure.
47. **The shim translates only on `ENOTTY`.** It calls the real `ioctl` via raw `SYS_ioctl` first. ADD installs `ext4:`, `f2fs:`, and `fscrypt:` logon keys with the 72-byte `fscrypt_key` payload; REMOVE unlinks all three and fails with `ENOKEY` if any is missing; GET_STATUS reports PRESENT or ABSENT. The shim creates the `fscrypt` keyring if it is missing (fallback) and logs to logcat under tag `A115F_FSCRYPT`; the wrapper logs to `/dev/kmsg` under prefix `A115F_FSCRYPT_WRAPPER: `.
48. **The boot image is not modified by this fix.** The wrapper and shim are files in the system image; only `$WORK_DIR/system/system` and its `fs_config`/`file_context` entries change. `magiskboot` is not required. Recovery now means reflashing the previous port ZIP or the stock system image, not the stock boot image alone.
49. **GitHub Actions registration is fixed.** Re-asserting the repository Actions permissions (`PUT /repos/zinerom48-tech/ZineROM-a11q/actions/permissions` with `enabled: true`, `allowed_actions: all`, `sha_pinning_required: false`) returned HTTP 204, after which `GET /repos/.../actions/workflows` reported `total_count: 2`: `build.yml` (id `420352259`, state `active`) and `publish-public.yml` (id `420352261`, state `active`). `build.yml` was dispatched on `manus/a115f-legacy-fscrypt` at commit `48438e6ecb9ab877c76d3bfa2a4d119134604a51` as run `43909580926` (run number 1).
50. **The CI test step passes.** In run `43909580926`, step 3 `python3 -m unittest discover -s tests -v` completed successfully with `Ran 69 tests in 0.214s` and `OK`. The 69 include all 7 `test_legacy_fscrypt_vold_rc` patcher tests. This is CI evidence for the patcher and the existing package/audit tests; it is not a ROM build or a device test.
51. **The CI build is blocked by a pre-existing workflow package bug, before the mod runs.** Step 8 `Set up build environment` failed with `E: Unable to locate package linux-modules-extra-7.0.0-1014-aws` and `E: Couldn't find any package by glob 'linux-modules-extra-7.0.0-1014-aws'`, exit code 100. The workflow line is `libzstd-dev linux-modules-extra-$(uname -r) lld …`; the runner kernel is `7.0.0-1014-aws`. Ubuntu's package search returns no results for that package in any suite or architecture. All 15 branches in the repository carry the same `linux-modules-extra-$(uname -r)` line (the audited legacy-fscrypt branch blob is `02972298d5f5`, 6032 bytes), so no existing ref can run the build without a workflow edit.
52. **The credential still cannot edit workflow files.** `GET /user` reports `X-OAuth-Scopes: gist, read:org, repo`; there is no `workflow` scope. `publish-public.yml` is registered but is `workflow_dispatch`-only and must not be run: it downloads old run `38817227822` and uploads a `ZineROM_OneUI7_a52xq-V3.zip` to GoFile. The branch `build.yml` is `workflow_dispatch`-only, uses `permissions: contents: read`, has an `audit_only` input, and has no GoFile or release steps.
53. **The minimal CI fix is a one-line workflow change.** Remove `linux-modules-extra-$(uname -r)` from the apt install list, then re-dispatch and observe whether `sudo modprobe erofs f2fs` succeeds on the runner. The workflow edit is not applied because the credential lacks `workflow` scope. The mod's own compile/host-test/`readelf` checks inside `zz_a115f_legacy_fscrypt/customize.sh` are still not reached, but the equivalent checks now run in the test step (fact 54).
54. **CI now compiles and verifies both targets in the test step.** Run `43913667988` at `5651d76` ran the full suite: `Ran 76 tests in 3.028s`, `OK`. All 7 new `test_legacy_fscrypt_build` tests passed without skips, meaning Ubuntu clang 18 built both freestanding targets, the host unit tests compiled and ran, and `readelf` confirmed the dynamic-symbol contract (wrapper no `PT_INTERP`/`.dynamic`/`NEEDED`, shim no `NEEDED`, exports `ioctl`, no `memcpy`/`memmove`/`memset`, undefined imports exactly `__android_log_print` and `__errno`). CI-built SHA-256: wrapper `530e2b3c1b9ddb5f66ca70ebde6ca74656428098dac2a50332d7928c25b00974`, 5296 bytes; shim `1faab403b0b8c9c46c3e6f23d2b44270323e3bb3b860857295d502afc242397d`, 6880 bytes. These differ in size/hash from the Zig-built binaries because the toolchains and linkers differ; the verified ELF contract is identical. They are test-step artifacts, not ROM contents.
55. **The workflow package failure is runner-pool dependent.** Run `43913667988` landed on kernel `7.0.0-1017-azure`, not `7.0.0-1014-aws`, and failed with `E: Unable to locate package linux-modules-extra-7.0.0-1017-azure` (exit 100). Because the workflow interpolates `$(uname -r)`, the missing package name changes with the assigned runner pool (aws/azure), and neither version is published for the hosted image. This strengthens the case that removing `linux-modules-extra-$(uname -r)` is the correct fix rather than pinning a different kernel package.
56. **The in-ROM mod install path is now covered by a CI test, including the real build driver.** Commit `74c8198` adds `tests/test_legacy_fscrypt_mod_install.py` and removes the mod's `xxd` dependency: `customize.sh` now reads ELF header bytes with coreutils `od` (`od -An -v -tx1 -j OFFSET -N LENGTH | tr -d '[:space:]'`), because the Ubuntu 24.04 runner image exposes coreutils but not `xxd` before the (currently blocked) build step. Commit `3aa20d4` adds a fifth test that invokes the real `scripts/internal/apply_modules.sh` driver with a copied mod and a sentinel system tree, proving that the driver runs the mod and that `SKIPUNZIP=1` suppresses its automatic system-tree copy. The suite drives the real `customize.sh`, `common_utils.sh` `SET_METADATA`, and `vold.rc` patcher against a synthetic donor init/`vold.rc` layout; it covers the target compile, moving the donor init to `init.real` unchanged, installing the wrapper and shim, preserving the `ueventd` symlink, exactly-once `fs_config`/`file_context` entries, exactly one `LD_PRELOAD` line, a second idempotent run, and both fail-closed layouts (non-symlink `ueventd`, foreign `init.real`). CI run `43921516470` at `74c8198` passed all **80** tests in 6.808 s (the first four mod-install tests); CI run `43925890423` at `3aa20d4` passed all **81** tests in 17.249 s, including the `apply_modules.sh` driver test, with no skips. The build step still failed at `Set up build environment` on `linux-modules-extra-7.0.0-1017-azure` (ISSUE-028), so the surrounding `make_rom.sh` flow remains unexercised. No release, flash, or `/data` operation was performed.
57. **The policy-input audit is closed statically.** No `7z` or `debugfs` is available on this host, so a small read-only ext4 reader was written to walk `system_full.dat` directly (superblock → group descriptors → inode table → extent tree → directory entries) and extract the donor policy files. It confirmed the image is Android 15 and produced `plat_sepolicy.cil`, `mapping/31.0.cil`, and `system_ext_sepolicy.cil` with the hashes in fact 43; the earlier local `plat_sepolicy.cil` copy is byte-identical (`febc700c…`). Every policy input that contributes to the key permissions is now checked, and the `unica/patches/selinux` mod cannot remove an allow rule. The final assembled policy is still not built: the remaining risks are policy-load ordering (keyrings labelled `kernel` before policy load versus `init` after) and the runtime label of the created keyrings. Verify those through the device AVC log or a built `precompiled_sepolicy` inspection.
58. **The shim's ADD-key argument expectations match the shipped A11 path.** AOSP 15 `installKey()` places the ioctl argument in a zeroing `KeyBuffer` (`KeyUtil.cpp:137`), sets `raw_size` from the key (`:166`), and sets `__flags` bit 0 only when `EncryptionOptions::use_hw_wrapped_key` is true (`:164`). `ParseOptionsForApiLevel()` initializes `use_hw_wrapped_key = false` and sets it only for the `wrappedkey_v0` fstab flag (`libfscrypt_fscrypt.cpp:215,227-228`); `ice` maps to `FSCRYPT_MODE_PRIVATE` (`:58`), and `first_api_level <= 29` selects version 1 (`:212`). Donor `vold` behaves identically: the 80-byte ADD header is zeroed by `memset` at `0x471A4` over `key.size()+0x50`, and bit 0 is ORed into `arg+0x4c` (offset 76) only under the `[x22+0x10]` `use_hw_wrapped_key` load at `0x47288`–`0x47298`. The A11 `/data` entry is `f2fs ... fileencryption=ice,quota,reservedsize=128M,checkpoint=fs` with no `keydirectory=`, `:v1`, `:v2`, or `wrappedkey_v0`; the effective `ro.product.first_api_level` is `29`. The shipped path therefore selects v1 and keeps `__flags == 0`, so the shim's `__flags == 0` requirement and its rejection of `HW_WRAPPED` are correct for the actual argument, not a conservative guess. Runtime behavior is still unverified.
59. **The retained A11 ramdisk-init string dump matches the AOSP 12 first-stage pivot.** The earlier boot-image audit left an init string dump (`a11_init_strings.txt`, 368,325 bytes, SHA-256 `be7e2e60…`; byte-identical to `ramdisk_init_strings.txt`). It contains `selinux_setup`, `/system/bin/init`, and the Samsung markers `!@Boot_EBS_F: FirstStageMain Init` and `!@Boot_EBS_F: SELinux Setup`, and it contains no `second_stage` code string (only `/second_stage_resources`). That combination matches AOSP 12's first-stage executable, whose final pivot is `execv("/system/bin/init", {…, "selinux_setup"})`, and it came from the same earlier boot-image extraction that produced the stock A11 ramdisk `build.prop` in §12. This is string-level evidence, not a disassembly of the call site; the boot image file is no longer retained, so the wrapper's actual execution on the device remains unverified. It narrows R1 but does not close it.

## 4. Current failure analysis

### Confirmed failure and its limits

The prior Android attempt reaches second-stage init, starts vendor processes, attempts file-based encryption for `/data`, and reboots to Recovery. The exact failure is reported as `cryptfs enablefilecrypto` returning `Status(-8, EX_SERVICE_SPECIFIC)`; the Rescue Party reason is `enablefilecrypto_failedK00R`.

The available log is an excerpt, not the raw capture. It does not include the vold service-specific error code, the full `logcat`, or a SHA-256 tying it to a particular ZIP. The prior attempt may predate the `wait_for_keymaster` barrier now present in `f51460ef…`. Therefore:

- The failure is confirmed at the stage level: FBE setup for `/data` fails and init reboots to Recovery.
- The root cause is **established at code level**: the donor userspace has no legacy fscrypt path, and the A11 4.9 kernel has no modern fscrypt key ioctls. The exact first failing call in the prior boot is still unknown because the log is only an excerpt.
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

`vdc cryptfs enablefilecrypto` maps to vold’s AIDL `fbeEnable()`. In Android 15, `fbeEnable()` calls `fscrypt_initialize_systemwide_keys()`. Four sub-steps can fail:

1. `init_data_file_encryption_options()` — parses the `/data` fstab entry. The A11 entry (`fileencryption=ice`, no key directory) is structurally valid.
2. `retrieveOrGenerateKey(device_key_path, device_key_temp, …)` — reads or creates the device key through KeyStorage, Keystore2, and the Keymaster HAL.
3. `install_storage_key(DATA_MNT_POINT, …)` — installs the key into the kernel and sets the fscrypt policy.
4. Writing the `/data` encryption options/references after the key is installed.

**Sub-step 3 is a confirmed, unconditional failure in the shipped combination.** The A11 kernel 4.9.227 implements only legacy fscrypt, while donor `vold` uses only `FS_IOC_ADD_ENCRYPTION_KEY`. The ioctl cannot succeed, so `installKey()` must return false and `fbeEnable()` must fail. Steps 1, 2, and 4 are not proven on the device; step 2 may fail earlier if KM4/Keystore2 is unhealthy.

The failure has two independent parts:

- **Key installation gap:** donor `vold` never falls back to legacy `add_key`/`keyctl`; the A11 kernel has no ADD/REMOVE/GET_STATUS ioctls. Donor vold therefore logs `Failed to install fscrypt key to /data: Inappropriate ioctl for device` if it reaches that call.
- **Keyring gap:** donor `init` never creates the session keyring or the named `fscrypt` keyring, and a helper cannot inject one into PID 1. Even a restored vold fallback would fail with `Unable to find device keyring`, and no other process could resolve the key.

Fixing only one part cannot boot `/data`. The leading fix (section 14) restores both: a PID-1 wrapper creates the keyrings, and a vold `LD_PRELOAD` shim translates the three unsupported ioctls.

The available log is an excerpt, not the raw capture. It does not include the vold service-specific error code, the full `logcat`, or a SHA-256 tying it to a particular ZIP. The prior attempt may predate the `wait_for_keymaster` barrier now present in `f51460ef…`. The first *observed* failure is still the `enablefilecrypto` failure, but the first *deterministic* failure in the current ships-as-built combination is the legacy/v2 mismatch above.

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
- **H5 — Kernel feature gap:** **closed as originally stated, replaced by the confirmed userspace/kernel mismatch.** The kernel lacks the modern fscrypt key ioctls, but it implements legacy v1 fscrypt with ICE/XTS/PFK. The missing piece is userspace support, not a kernel crypto feature.
- **H6 — Prior log predates the barrier:** confirmed as the leading explanation for why the prior log has no barrier markers. The prior diagnostic ZIP `1103fc00…` predates commit `190586c` (barrier) and `23f8998f…` (markers); the current `f51460ef…` asset was built from `23f8998f…` and contains the barrier. No barrier-containing build is known to have been device-tested.
- **H7 — Keystore2/KM4 runtime binding gap:** the 64-bit HIDL client is present, but KM4 may not be registered or usable when Keystore2/fscrypt needs it. This remains a possible *earlier* failure in `retrieveOrGenerateKey()`; it is independent of the confirmed legacy-fscrypt gap.
- **H8 — Late-fs restart is sufficient:** the first `on init` abort is benign because `class_start early_hal` starts KM4 again after `/vendor/firmware_mnt` is mounted. This is consistent with the init ordering but has not been observed on a device.
- **H9 — Key retrieval fails before key installation:** if KM4 or Keystore2 is unhealthy, `retrieveOrGenerateKey()` fails before `installKey()` is reached. The device log must still distinguish this from the confirmed ioctl failure; fixing legacy fscrypt does not prove KM4 is usable.
- **H10 — Wrapper/shim runtime gaps:** `LD_PRELOAD` may be ignored by vold, `/system/bin/init.real` may not receive the `init_exec` label, or the wrapper's raw syscalls may be denied. These are build/runtime risks of the proposed fix, not causes of the original failure.

## 5. Untested assumptions

- That the barrier’s `wait_for_keymaster` exit actually implies KM4 is usable. It does not: the helper waits for Keystore2 registration and logs `getSecurityLevel` failures without returning nonzero.
- That KM4 registers before Keystore2’s shared-secret negotiation and before vold’s FBE key generation.
- That the helper can execute under the donor SELinux policy and reach Keystore2.
- That the first `on init` KM4 abort is harmless and the system `on late-fs` `class_start early_hal` restart succeeds after `/vendor/firmware_mnt` is mounted. The ordering is confirmed; the device behavior is not.
- That the QSEECom listener gate completed for the current asset and that `qseecom-service` registered before the late-fs KM4 start.
- That the prior log belongs to the current `f51460ef…` asset (it does not; no SHA linkage exists).
- That the A11 kernel’s legacy fscrypt path (v1 policy, session keyring lookup, ICE) behaves as its source and disassembly indicate. The kernel audit is static only.
- That the A11 stock first-stage init executes `/system/bin/init selinux_setup` before loading policy, and that the wrapper's pre-policy keyring creation behaves as intended. The AOSP 12 flow is confirmed and the retained A11 ramdisk-init string dump contains the matching `selinux_setup`/`/system/bin/init` pivot strings (fact 59), but the call site has not been disassembled and the boot image is no longer retained locally; runtime behavior is unverified.
- That the wrapper can be re-executed as `second_stage` with `init_exec`, and that the ueventd symlink passthrough keeps ueventd working.
- That vold honours `LD_PRELOAD` on this Samsung build, and that the shim's ADD/REMOVE/GET_STATUS emulation matches the donor `vold` ABI and the 4.9 keyring semantics.
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
   For the confirmed legacy-fscrypt branch, grep for:
   - `Failed to install fscrypt key to /data: Inappropriate ioctl for device` — the ADD ioctl reached the kernel and was rejected; this directly confirms the code-level root cause.
   - `Failed to open /data to install key` — the key install never reached the ioctl.
   - `Failed to evict fscrypt key` / `Failed to get status for fscrypt key` — REMOVE/GET_STATUS paths.
   - `Creating new key in`, `No key found in`, `encryptWithKeystoreKey failed`, `decryptWithKeystoreKey failed`, `Version mismatch` — key retrieval failed before key installation, so KM4/Keystore2 is the earlier blocker.
   - `Unable to find device keyring` is an Android 12/14 legacy string. It is absent from donor `vold`, so its absence is **not** evidence. A fixed build should emit `A115F_FSCRYPT_WRAPPER_*` markers instead.
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

The code-level root cause is established: the A11 4.9 kernel implements only legacy fscrypt, while donor `vold` and `init` have the legacy key path stripped. Even with a perfectly healthy KM4, the current `f51460ef…` asset cannot install a v1 key into the kernel or create the keyring that every process needs. The source-level fix is compiled locally and in CI, host-tested, and pushed as `manus/a115f-legacy-fscrypt@3aa20d4`; CI run `43925890423` passed all 81 tests, including the compile, host, `readelf`, in-ROM mod-install, and real `apply_modules.sh` driver suites (facts 54 and 56). The CI **build** is still blocked before it reaches `make_rom.sh` by a workflow defect: `.github/workflows/build.yml` installs `linux-modules-extra-$(uname -r)`, which resolved to `linux-modules-extra-7.0.0-1014-aws` in run 1 and `linux-modules-extra-7.0.0-1017-azure` in runs 2, 3, and 4; neither package exists. The credential lacks `workflow` scope, so the one-line fix cannot be applied yet. Compile and test results remove the build-correctness unknown for the two binaries and the mod's install logic; the remaining unknowns are runtime behavior: whether the wrapper runs in PID 1 before policy load, whether the `init_exec` labels and ueventd passthrough work, and whether vold honours `LD_PRELOAD`. Readiness remains R0.

### Next concrete actions

1. **Fix the CI workflow package line.** Remove `linux-modules-extra-$(uname -r)` from the `apt install` list in `.github/workflows/build.yml` (all 15 branches contain it). This requires a `workflow`-scoped credential or a user edit; the current token cannot change workflow files. Do not work around it by shadowing `uname`/`sudo`, adding a dummy package, or disabling the step.
2. Re-dispatch `build.yml` on `manus/a115f-legacy-fscrypt` after the workflow fix; the 81-test suite and the target compile/`readelf`/mod-install/`apply_modules.sh` checks already pass, so watch the `Set up build environment` step and then the full `make_rom.sh` flow.
3. Record the ZIP SHA-256, build log, and the SHA-256 of the wrapper, `init.real`, and shim in `PORTING_STATUS.md` and `ISSUE_REGISTER.md`.
4. Device-test only after the stock restore path is verified and the user approves; capture the wrapper markers, shim markers, KM4/Keystore2 lines, both `keymaster-4-0` starts, and AVC denials.
5. Keep readiness at R0; do not publish a new ZIP or recommend installation until at least one device boot reaches the framework.
6. Do not rebuild from `manus/a115f-build-once` or `manus/a115f-bootability-preflight`: both lack `scripts/patch_a115f_init.py`. If a rebuild is approved, start from `23f8998f…` or merge that patcher deliberately.
7. Do not move the barrier into vendor `on post-fs-data`: system `installkey /data` would run before it.
8. Do not run `publish-public.yml`: it is dispatch-only, downloads old run `38817227822`, and uploads a different (`a52xq`) ZIP to GoFile.

### What requires the user / physical device

- The original failure logs and the exact tested ZIP SHA.
- The exact A11 CSC and current firmware string.
- The physical device for installation and log collection, after a recovery path is verified.
- Explicit approval before any installation or destructive `/data` action.

## 9. Artifacts and evidence

- Public source repo: <https://github.com/nova-arch008/ZineROM-a11q> (branch `manus/a115f-vendor-fingerprint`, commit `23f8998f…`)
- Public release repo: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test>
- Release asset URL: <https://github.com/nova-arch008/ZineROM-A115F-OneUI7-boot-test/releases/download/a115f-boot-test/ZineROM_OneUI7_A115F-Port-A057F.zip>
- CI runs on the fix branch: run 1 <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43909580926> (commit `48438e6`; 69/69 tests pass), run 2 <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43913667988> (commit `5651d76`; 76/76 tests pass, including CI clang 18 compile and ELF/symbol checks), run 3 <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43921516470> (commit `74c8198`; 80/80 tests pass, including the four in-ROM mod-install tests), and run 4 <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43925890423> (commit `3aa20d4`; 81/81 tests pass, including the real `apply_modules.sh` driver test). All builds are blocked at `Set up build environment` by the workflow’s nonexistent `linux-modules-extra-$(uname -r)` package.
- Registered workflows: `build.yml` id `420352259` (active), `publish-public.yml` id `420352261` (active, dispatch-only; do not run)
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

**Boot image (`boot.img`):** the ramdisk `build.prop` identifies stock A11 firmware `samsung/a11qnsxx/a11q:12/SP1A.210812.016/A115FXXS6CWK2:user/release-keys`. The retained ramdisk-init string dump from that image contains `selinux_setup` and `/system/bin/init`, consistent with the AOSP 12 first-stage pivot (fact 59). The ramdisk `fstab.qcom` matches the vendor fstab for `/data` (`f2fs`, `fileencryption=ice`, `latemount`, `checkpoint=fs`), `/keydata` and `/keyrefuge` (`ext4`, `fileencryption=ice,nofail`), and `/vendor/firmware_mnt` (`vfat`, `apnhlos`). The package therefore retains the stock A11 boot image and its first-stage mount table.

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

## 14. Session checkpoint — 2026-11-16 (legacy-fscrypt root cause and fix design)

### Root cause, in two parts

The shipped combination cannot initialize `/data` file-based encryption:

1. **Kernel/`vold` mismatch.** The A11 4.9.227 kernel implements only legacy fscrypt key management. It has no `FS_IOC_ADD_ENCRYPTION_KEY`, `FS_IOC_REMOVE_ENCRYPTION_KEY`, or `FS_IOC_GET_ENCRYPTION_KEY_STATUS`; v1 keys are installed with `add_key("logon", "ext4:<ref>"…)` and resolved with `request_key()`. Donor Samsung `vold` is ioctl-only: `installKey()` calls `0xC0506617`, `evictKey()` calls `0xC0406618`, and `waitForBusyFiles()` calls `0xC080661A` plus REMOVE. It imports neither `add_key` nor `keyctl`, and its recovered symbol table has no `installKeyLegacy`, `evictKeyLegacy`, `isFsKeyringSupported`, or `fscryptKeyring`. The ADD ioctl reaches the kernel and returns `ENOTTY`.

2. **Missing keyrings.** Legacy fscrypt resolves keys through the session keyring of the process doing file I/O. Every process inherits PID 1's session keyring, and `installKeyLegacy()` expects a named `fscrypt` keyring inside it. Donor Samsung `init` never creates either: no keyctl strings or symbols, no `libkeyutils.so` DT_NEEDED, and no `svc` instruction in the full `SecondStageMain`. LineageOS 22.2/23.1 still create both, so this is a Samsung removal, not an Android 15 change. A helper cannot inject a session keyring into PID 1 because `keyctl_session_to_parent()` rejects `parent->pid <= 1`.

Both parts must be fixed. Restoring only `vold`'s fallback would fail with `Unable to find device keyring`; restoring only the keyrings would leave `vold` using an ioctl the kernel does not implement.

### Leading fix design

**Part A — `/system/bin/init` wrapper (PID 1 keyring creation).**

AOSP 12 `FirstStageMain` mounts `/system` and execs `/system/bin/init` with `selinux_setup` as PID 1 before SELinux policy is loaded (`first_stage22.cpp:559-565`). The fix installs a small static wrapper at that path and moves the donor init to `/system/bin/init.real`. The wrapper creates the keyrings, preserves `argv[0]`, and execs the real init; donor `SetupSelinux` then loads policy and re-execs `/system/bin/init second_stage`, which runs the wrapper a second time. Keyring creation is idempotent, and both `/system/bin/init` and `/system/bin/init.real` are labelled `init_exec` because the second exec happens post-policy. The boot image is not modified.

- Wrapper: `keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 1)` creates PID 1's anonymous session keyring; `keyctl(KEYCTL_SEARCH, KEY_SPEC_SESSION_KEYRING, "keyring", "fscrypt", 0)` finds the named `fscrypt` keyring; if absent, `add_key("keyring", "fscrypt", NULL, 0, KEY_SPEC_SESSION_KEYRING)` creates it. This mirrors AOSP `FscryptInstallKeyring()`. On failure, log a marker to `/dev/kmsg` and continue to exec anyway (a failed wrapper must not turn an FBE failure into a dead boot).
- Exec target: the donor init, moved to `/system/bin/init.real`. Both files are set to `u:object_r:init_exec:s0`, uid/gid 0, mode 0750. The pre-policy exec would not consult a label, but the post-policy `second_stage` re-exec does.
- Markers: `A115F_FSCRYPT_WRAPPER_ENTER <argv0>`, `A115F_FSCRYPT_WRAPPER_SESSION_OK <serial>`, `_SESSION_FAIL <errno>`, `_FSCRYPT_OK <serial>`, `_FSCRYPT_FAIL <errno>`, `A115F_FSCRYPT_WRAPPER_UEVENTD_PASSTHROUGH`, `A115F_FSCRYPT_WRAPPER_EXEC /system/bin/init.real`, `_EXEC_FAIL <errno>`.
- Why `/system/bin/init` and not the boot ramdisk: it avoids unpacking, repacking, and re-signing the unverified stock A11 boot image, and it runs in PID 1 at the right point. The donor init is a regular ELF64 AArch64 binary.
- Why ueventd still works: `/system/bin/ueventd` is a symlink to `init`. The wrapper preserves `argv[0]` and skips keyring setup when the basename is `ueventd`, then execs `init.real`; the real init's basename-based ueventd mode is preserved. The mod verifies the symlink target before installing the wrapper and fails otherwise.
- Why not a service: `keyctl_session_to_parent()` explicitly rejects PID 1, and the keyring must be PID 1's. Booting a helper first and `exec`ing init does not preserve the session keyring.

**Part B — 64-bit `vold` `LD_PRELOAD` shim (legacy key translation).**

The shim intercepts `ioctl()` and calls the real `ioctl` first. It emulates only on `ENOTTY`:

- `FS_IOC_ADD_ENCRYPTION_KEY` (`0xC0506617`): if the argument is `NULL`, return `-1` with `ENOTTY`. This is the shape of AOSP's legacy-support probe, but donor `vold` contains no probe and calls ADD only with a real argument, so this branch is defensive. Otherwise require `key_spec.type == 1`, `raw_size == 64`, `key_id == 0`, and `__flags == 0`; build the 72-byte `fscrypt_key` (`mode=0`, `raw[64]`, `size=64`) and install three logon keys named `ext4:<16 hex>`, `f2fs:<16 hex>`, and `fscrypt:<16 hex>` into the `fscrypt` keyring found by searching PID 1's session keyring. Type 2 (`IDENTIFIER`), `HW_WRAPPED`, and any other raw size are rejected with `EOPNOTSUPP`/`EINVAL`; the A11 kernel has no identifier/v2 key support.
- `FS_IOC_REMOVE_ENCRYPTION_KEY` (`0xC0406618`): search and unlink the three legacy keys. Matching AOSP `evictKeyLegacy`, this is **not** idempotent: if any of the three names is missing, set `removal_status_flags=0` and return `-1` with `ENOKEY`; return 0 only when all three were found and unlinked.
- `FS_IOC_GET_ENCRYPTION_KEY_STATUS` (`0xC080661A`): search the three names and report `FSCRYPT_KEY_STATUS_PRESENT` (with `status_flags=ADDED_BY_SELF`, `user_count=1`) if any exist, `ABSENT` otherwise; `INCOMPLETELY_REMOVED` is not meaningful for legacy keys.
- Log one message per distinct result to logcat, tag `A115F_FSCRYPT`: `SHIM_LOADED`, `ADD_PROBE_ENOTTY`, `ADD_UNSUPPORTED <errno>`, `ADD_OK`, `ADD_FAIL <errno>`, `REMOVE_UNSUPPORTED <errno>`, `REMOVE_OK`, `REMOVE_FAIL <errno>`, `STATUS_UNSUPPORTED <errno>`, `STATUS_PRESENT`, `STATUS_ABSENT`, `STATUS_FAIL <errno>`. There is no literal `A115F_FSCRYPT_SHIM_LOADED` string in the binary; the tag and message are separate fields.

`LD_PRELOAD` requires the donor `vold.rc` to `setenv LD_PRELOAD /system/lib64/liba115f_fscrypt_shim.so`. The shim is 64-bit because donor `vold` is ELF64. It calls the real `ioctl` through a raw `SYS_ioctl` syscall (29) and calls `add_key`/`keyctl` through raw syscalls (217/219), so it needs neither `libdl` nor `libkeyutils`. Interposition is viable because donor `vold` imports `ioctl` and both failure sites call its normal PLT stub; `.rela.plt` slot `0x146F30` is an `R_AARCH64_JUMP_SLOT` for symbol index 105, `ioctl`.

### SELinux

No policy change is required for the intended pre-policy path. The donor `plat_sepolicy.cil` (the platform policy the ROM uses) already contains `(allow domain init (key (search)))`, `(allow domain vold (key (search)))`, `(allow domain kernel (key (search)))`, `(allow init init (key (write search setattr)))`, `(allow vold init (key (write search setattr)))`, and `(allow vold vold (key (write search setattr)))`. The donor `mapping/31.0.cil` maps the A11 vendor policy's `_31_0` types onto those donor types, and A11 `vendor_sepolicy.cil` adds `(allow vold_31_0 kernel_31_0 (key (write search)))`, `(allow domain kernel_31_0 (key (search)))`, `(allow init_31_0 kernel_31_0 (key (search)))`, and `(allow vold_31_0 init_31_0 (key (write)))`. The wrapper's keyrings are labelled `kernel` (initial SID) if created before policy load; `vold` installs and searches keys through its write/search rights on `kernel` (fallback: `vold`'s own keyring if the shim creates it).

Residual policy risks: both init files need `init_exec` for the post-policy `second_stage` re-exec. If the policy is somehow already loaded when the wrapper first runs, keyring creation in the `init` domain may be denied because the platform rule grants `write search setattr` but not `create` on the session keyring. The wrapper logs the failure and still execs the real init; the shim can create the `fscrypt` keyring in the `vold` domain as a fallback. The remaining runtime risks are a keyring label mismatch and an AVC denial on mapping the shim.

### Build integration points

- The wrapper, `init.real`, shim, and `vold.rc` change belong to the **system** image: `$WORK_DIR/system/system/bin/init`, `$WORK_DIR/system/system/bin/init.real`, `$WORK_DIR/system/system/lib64/liba115f_fscrypt_shim.so`, and `$WORK_DIR/system/system/etc/init/vold.rc`. The boot image is copied unchanged by `create_work_dir.sh` and `build_flashable_zip.sh`; no boot repack is needed and `magiskboot` is not used.
- `SET_METADATA "system" "system/bin/init" 0 0 0750 "u:object_r:init_exec:s0"`, `SET_METADATA "system" "system/bin/init.real" 0 0 0750 "u:object_r:init_exec:s0"`, and `SET_METADATA "system" "system/lib64/liba115f_fscrypt_shim.so" 0 0 0644 "u:object_r:system_lib_file:s0"`.
- The mod verifies `system/bin/init` is a regular ELF64 AArch64 file, `system/bin/ueventd` is a symlink to `init`, and `init.real` does not already exist; it fails closed otherwise.
- The shim must not link `libdl` or `libkeyutils`; it needs only bionic's `__errno` and raw syscalls.
- CI: the standalone `legacy-fscrypt-ci.yml` is local-only because the credential lacks `workflow` scope. The same compile, host-test, `readelf`, and marker checks run inside `zz_a115f_legacy_fscrypt/customize.sh`, and `build.yml` already runs all `tests/`; do not modify `publish-public.yml`.
- Host tests cover the ioctl decoder: v1 descriptor extraction, 72-byte payload layout, name formatting, probe (`arg==NULL`), ENOTTY-only fallback, and legacy install/evict/status behavior.

### Not yet done

The wrapper and shim are compiled locally and host-tested, and the branch is pushed, but the ROM is not built and no device test exists. `LD_PRELOAD` support in Samsung `vold`, the shim ABI, kernel keyring inheritance, and runtime SELinux behavior remain untested assumptions (H10). The remaining file-context question is closed for both init files: they are labelled `init_exec` via `SET_METADATA`, and the mod fails if that update fails. Readiness stays **R0**.

The detailed ABI, structs, syscall numbers, packaging, and risk list are in `FIX_LEGACY_FSCRYPT.md`. The Phase 1–5 gates, persistent task queue, and program-level risks are in `DEVELOPMENT_PLAN.md`.

## 15. Session checkpoint — 2026-11-16 (CI: 81 tests pass; build blocked; policy inputs audited)

### Registration and dispatch

The repository had reported `total_count: 0` registered workflows even though `.github/workflows/build.yml` existed at the default-branch tip. Creating a new branch and pushing an additive documentation commit did not change this. Re-asserting the repository Actions permissions with `PUT /repos/zinerom48-tech/ZineROM-a11q/actions/permissions` (`{"enabled": true, "allowed_actions": "all", "sha_pinning_required": false}`) returned HTTP 204; immediately afterward `GET /actions/workflows` reported `total_count: 2` and both workflows were `active`.

`build.yml` was dispatched with `ref=manus/a115f-legacy-fscrypt` and `audit_only=false` so the full build path would run. The run:

| Field | Value |
|---|---|
| Run / number | `43909580926` / 1 |
| Commit | `48438e6ecb9ab877c76d3bfa2a4d119134604a51` |
| Branch | `manus/a115f-legacy-fscrypt` |
| Event | `workflow_dispatch` |
| URL | <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43909580926> |

### Test result

Step 3, `python3 -m unittest discover -s tests -v`, completed successfully:

```text
Ran 69 tests in 0.214s

OK
```

The 69 tests include the 7 `test_legacy_fscrypt_vold_rc` tests for `scripts/patch_a115f_legacy_fscrypt_vold_rc.py`. This is the first CI execution of the new patcher tests and the first CI proof that the fix branch does not break the existing 62-test static/package suite.

### Exact build failure

Step 8, `Set up build environment`, failed before `make_rom.sh`:

```text
E: Unable to locate package linux-modules-extra-7.0.0-1014-aws
E: Couldn't find any package by glob 'linux-modules-extra-7.0.0-1014-aws'
##[error]Process completed with exit code 100.
```

The workflow command is:

```text
DEBIAN_FRONTEND=noninteractive sudo apt install -yq \
  p7zip-full jq attr ccache clang ffmpeg golang \
  libbrotli-dev libgtest-dev libprotobuf-dev libunwind-dev libpcre2-dev \
  libzstd-dev linux-modules-extra-$(uname -r) lld jq protobuf-compiler pcre2-utils webp \
  curl xxd patchelf flex bison libarchive-tools build-essential python3.12-venv device-tree-compiler
```

The runner kernel is `7.0.0-1014-aws`. Ubuntu's package search returns no results for `linux-modules-extra-7.0.0-1014-aws` in any suite, section, or architecture, so adding a repository or re-running the job cannot fix it. All 15 branches in `zinerom48-tech/ZineROM-a11q` carry the same `linux-modules-extra-$(uname -r)` line, so no existing ref can run the build.

### Second run — target compile and ELF verification now pass in CI

Commit `5651d76` added `tests/test_legacy_fscrypt_build.py`, which mirrors `build.sh` and verifies the wrapper/shim ELF contract in the workflow's existing test step, before the broken apt step. Run:

| Field | Value |
|---|---|
| Run / number | `43913667988` / 2 |
| Commit | `5651d76be73f075ca77f342c7cef003feee09ee5` |
| Branch | `manus/a115f-legacy-fscrypt` |
| URL | <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43913667988> |

Step 3 passed:

```text
test_dynamic_symbol_contract ... ok
test_hashes_recorded ... ok
test_host_unit_tests ... ok
test_markers_present ... ok
test_shim_is_dynamic_without_needed ... ok
test_targets_are_elf64_aarch64 ... ok
test_wrapper_is_static ... ok
Ran 76 tests in 3.028s
OK
wrapper sha256=530e2b3c1b9ddb5f66ca70ebde6ca74656428098dac2a50332d7928c25b00974 size=5296
shim sha256=1faab403b0b8c9c46c3e6f23d2b44270323e3bb3b860857295d502afc242397d size=6880
```

No test was skipped: Ubuntu clang 18 compiled both targets, `cc` compiled and ran the host unit tests, and `readelf` confirmed the dynamic-symbol contract. The CI-built hashes and sizes differ from the Zig-built artifacts because the toolchains/linkers differ; the verified ELF properties are the same.

Step 8 failed again, but on a **different kernel name**:

```text
E: Unable to locate package linux-modules-extra-7.0.0-1017-azure
##[error]Process completed with exit code 100.
```

The runner was assigned the Azure pool (`7.0.0-1017-azure`) instead of AWS (`7.0.0-1014-aws`). This confirms the `$(uname -r)` interpolation makes the missing-package name depend on the runner pool; pinning a different kernel package is not a valid fix.

### Third run — in-ROM mod-install test now passes in CI

Commit `74c8198` replaced the mod's `xxd` ELF-header reads with coreutils `od` (the Ubuntu 24.04 runner image lists `coreutils` but not `xxd` before the build step) and added `tests/test_legacy_fscrypt_mod_install.py`, which drives the real mod against a synthetic donor layout. Run:

| Field | Value |
|---|---|
| Run / number | `43921516470` / 3 |
| Commit | `74c8198a3118fb5493aafe013eaea2d1d793b644` |
| Branch | `manus/a115f-legacy-fscrypt` |
| URL | <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43921516470> |

Step 3 passed with no skips:

```text
Ran 80 tests in 6.808s

OK
```

All four new mod-install tests executed: `test_first_install`, `test_second_run_is_idempotent`, `test_non_symlink_ueventd_aborts_before_moving_init`, and `test_foreign_init_real_aborts_without_touching_init`. Together they prove that the mod compiles both targets, moves the donor init to `init.real` without changing its bytes, installs the wrapper and shim, preserves the `ueventd` symlink, writes each `fs_config`/`file_context` entry exactly once, inserts exactly one `LD_PRELOAD`, is idempotent on a second run, and aborts before touching `init` on a non-symlink `ueventd` or a foreign `init.real`. This covers the mod's own install, metadata, and `vold.rc` stages, but not the surrounding `make_rom.sh` flow.

Step 8 still failed with `E: Unable to locate package linux-modules-extra-7.0.0-1017-azure` (exit 100), so ISSUE-028 remains the gate.

### Fourth run — real `apply_modules.sh` driver path now passes in CI

Commit `3aa20d4` extends `tests/test_legacy_fscrypt_mod_install.py` with a fifth test that invokes the real `scripts/internal/apply_modules.sh` driver, rather than sourcing `customize.sh` directly. It copies the mod into a temporary `mods/` folder, adds a sentinel file under the mod's `system/` tree, and asserts that the mod runs, installs, and that `SKIPUNZIP=1` prevents the driver from copying the sentinel tree into the work dir. Run:

| Field | Value |
|---|---|
| Run / number | `43925890423` / 4 |
| Commit | `3aa20d4442ae55d2fc6ec338915589537400a215` |
| Branch | `manus/a115f-legacy-fscrypt` |
| URL | <https://github.com/zinerom48-tech/ZineROM-a11q/actions/runs/43925890423> |

Step 3 passed with no skips:

```text
test_apply_modules_driver_runs_the_mod_without_copying_a_tree ... ok
test_first_install ... ok
test_foreign_init_real_aborts_without_touching_init ... ok
test_non_symlink_ueventd_aborts_before_moving_init ... ok
test_second_run_is_idempotent ... ok
Ran 81 tests in 17.249s

OK
wrapper sha256=530e2b3c1b9ddb5f66ca70ebde6ca74656428098dac2a50332d7928c25b00974 size=5296
shim sha256=1faab403b0b8c9c46c3e6f23d2b44270323e3bb3b860857295d502afc242397d size=6880
```

All five mod-install tests executed. The new test closes the gap between the mod's own `customize.sh` checks and the way `make_rom.sh` actually invokes it, without requiring the blocked full build. Step 8 still failed with `E: Unable to locate package linux-modules-extra-7.0.0-1017-azure` (exit 100). The surrounding work-dir/image/ZIP flow remains unexercised, and no device evidence exists. R0.

### Policy-input audit (static)

Open question 6 in `FIX_LEGACY_FSCRYPT.md` was whether the assembled policy would still grant the key permissions. The gating inputs are now checked directly from the donor image and the A11 vendor policy:

| Input | Size | SHA-256 | Relevant result |
|---|---|---|---|
| donor `system/etc/selinux/plat_sepolicy.cil` | 2,237,691 | `febc700c…` | contains `vold init`, `vold vold`, `init init`, and `domain kernel` key rules |
| donor `system/etc/selinux/mapping/31.0.cil` | 141,860 | `ff34465a…` | maps `vold_31_0`→`vold`, `init_31_0`→`init`, `kernel_31_0`→`kernel` |
| A11 `vendor_sepolicy.cil` (`VE=SEPF_SM-A115F_12_0001`) | 1,803,461 | `08e2f3a2…` | `vold_31_0 kernel_31_0 (key (write search))`, `vold_31_0 init_31_0 (key (write))`, `domain kernel_31_0 (key (search))` |
| donor `system_ext_sepolicy.cil` | 488,747 | `a0a7cf4e…` | no key rules |

The donor image was identified from its `system/build.prop` as One UI 7 / Android 15 (`ro.build.id=AP3A.240905.015.A2`, `ro.build.version.sdk=35`), so the extracted `plat_sepolicy.cil` is donor policy, not A11 target policy. The `unica/patches/selinux` mod only deletes named One UI 7 types from the system_ext mapping/genfscon; it cannot remove an allow rule. This closes the policy-input question statically. It does not prove the runtime label of the keyrings or the policy-load ordering; those need the device log or a built `precompiled_sepolicy` inspection.

### Minimal fix (prepared, not applied)

```diff
-  libzstd-dev linux-modules-extra-$(uname -r) lld jq protobuf-compiler pcre2-utils webp \
+  libzstd-dev lld jq protobuf-compiler pcre2-utils webp \
```

This is the smallest change that addresses the confirmed failure. If the subsequent `sudo modprobe erofs f2fs` fails on the runner, treat that as the next distinct failure and fix it only with the new log; do not pre-emptively add `|| true` or a different kernel package without evidence. The mod’s compile, host-test, `readelf`, marker, install, metadata, and `vold.rc` checks are now covered by committed tests, but the surrounding `make_rom.sh` flow (work-dir creation, image build, ZIP packing) remains unexercised until the apt step is fixed.

### Why the edit is still pending

`GET /user` reports `X-OAuth-Scopes: gist, read:org, repo`. GitHub requires the `workflow` scope to create or update files under `.github/workflows/`, and the token does not have it. The fix therefore needs either a workflow-scoped credential or a user edit. No workaround was applied: shadowing `uname`/`sudo`, installing a dummy package with the missing name, or adding a local apt repository from the test step would hide the defect rather than fix it, and would make the build evidence untrustworthy.

### Safety boundary

`publish-public.yml` is now registered but remains `workflow_dispatch`-only. It downloads old run `38817227822` and uploads a `ZineROM_OneUI7_a52xq-V3.zip` to GoFile. It was not run and must not be run for this A115F work. The branch `build.yml` has `permissions: contents: read`, no GoFile step, no release step, and uploads only a private ZIP artifact. No release was published, no device was flashed, and no `/data` operation was performed. Readiness remains **R0**.

## 16. Change log

- 2026-10-28: Initial status created from repository, release metadata, and static package metadata. No build or device test performed.
- 2026-10-28: Corrected the source-access claim (repo is public), added the current release asset hash and the stale `BOOT_PROGRESS.md` hash discrepancy, confirmed the donor’s 32-bit support and the helper’s ELF32 dependencies, recorded the FBE v1/v2 decision path, demoted `K00R` and QSEE `0x72000206`, and identified the stock `/oem` mount and Samsung ODE partitions as separate open items. No build or device test performed.
- 2026-10-29: Closed the 32-bit helper-dependency question after listing the donor `/system/lib`; confirmed the `vdc` → `fbeEnable()` → `fscrypt_initialize_systemwide_keys()` call chain; confirmed `ro.crypto.type=file` comes from the fstab; recorded the A11 `fileencryption=ice` layout; confirmed donor Keystore2 KM4 support and the platform policy `keystore` membership; corrected the `/system/lib64` result as a truncated-image artifact; added the four FBE sub-step hypotheses and the 64-bit Keymaster/HAL verification branch. Still R0; no build or device test performed.
- 2026-10-29: Added the pre-disconnect checkpoint with the exact blocker, all session changes, build results, `/data` FBE evidence, resume offsets/hashes, and next commands; published the status repo. Still R0; no build or device test performed.
- 2026-11-03: Retracted the init import-order/deadlock theory after reading AOSP `ImportParser::EndFile()`; confirmed the barrier is correctly placed after system `class_start early_hal` and before vendor `mount_all --late`; recorded the barrier build timeline (`1103fc00…` pre-barrier, `190586c…` barrier, `23f8998f…` markers, current `f51460ef…`); confirmed FBE v1, the 64-bit KM4 compat libraries, merged system_ext, VINTF non-blocking, and the A11 vendor `vndservicemanager`; documented that `wait_for_keymaster` gates on Keystore2 registration, not KM4, and added the decisive device-log branch tree. Still R0; no build or device test performed.
- 2026-11-04: Decoded the A11 `KeymasterUtils` constructor (`QSEECom_start_app(…, "/vendor/firmware_mnt/image", "keymaster64", …)` with a `"keymaster"` retry and a `QSEECom_start_app failed` abort); established that the first `on init` KM4 start necessarily precedes the `apnhlos` mount at `on fs` and is expected to abort; confirmed AOSP `Service::Start()` clears `SVC_RESTARTING`, so the system `on late-fs` `class_start early_hal` can start a fresh KM4; confirmed the `restart_period` default of 5 s and that Android 15’s 4-crash limit does not apply; documented the `vendor.qseecomd` listener gate and `qseecom-service` registration; confirmed the KM4 rc has no `interface` declaration and therefore no lazy-start fallback; closed the Samsung ODE/UCM branch from donor `vold` disassembly (`isUcmOdeMode()` is consulted only for `wrappedkey_v0`, which A11 `/data` does not use); completed the donor `/system/lib64` listing and confirmed the full 64-bit KM4 HIDL stack; reconciled the source branches via the GitHub compare API (`23f8998f…` is 13 commits ahead of `build-once`, which is a strict ancestor; `preflight` is 2 commits ahead and 15 behind, adding stock-blob extraction and the upload-visibility guard); completed the kernel/VINTF/boot-image/installer compatibility audit (kernel 4.9.227 has FS_ENCRYPTION/ICE/XTS/PFK; no missing required framework HAL; stock A11 boot image; installer does not touch `/data`; geometry matches donor images). No build, flash, format, or new ZIP was produced. Still R0; no device test performed.
- 2026-11-16: Established the code-level root cause in two parts: the A11 4.9.227 kernel implements only legacy fscrypt (0 ADD_KEY ioctl sites; `request_key`-based v1 keys), while donor Samsung `vold` is ioctl-only and lacks every legacy fallback symbol, and donor Samsung `init` creates neither the session keyring nor the `fscrypt` keyring (no keyctl symbols, no `libkeyutils` dependency, no `svc` in `SecondStageMain`). Confirmed LineageOS 22.2/23.1 still contain the fallback, so the earlier “Android 15 removed it” claim is corrected: Samsung removed it. Confirmed SELinux already permits the intended keyring operations and that `/system/bin/ueventd` is a symlink to `init`, which rules out replacing `/system/bin/init` as the leading fix. Recorded the leading design: a pre-policy ramdisk `/init` wrapper creates both keyrings in the kernel domain and execs the real init; a 64-bit `vold` `LD_PRELOAD` shim translates ADD/REMOVE/GET_STATUS to legacy `add_key`/`keyctl` only on `ENOTTY`. Added `FIX_LEGACY_FSCRYPT.md` with the ABI and packaging design. No build, flash, format, or new ZIP was produced. Still R0; no device test performed.
- 2026-11-16 (correction): Verified the remaining design assumptions. `.rela.plt` slot `0x146F30` is an `R_AARCH64_JUMP_SLOT` for symbol index 105 = `ioctl`, so `LD_PRELOAD` interposition is viable without `libdl`. Kernel 4.9 possession permissions are additive and propagate through nested keyring search, so no `KEYCTL_SETPERM` and no `KEY_OTH_SEARCH` are needed; the default keyring mask includes `KEY_POS_WRITE`. The `fscrypt` keyring is created by AOSP `FscryptInstallKeyring()` (search-first, then `add_key("keyring", "fscrypt")`), not by `SecondStageMain`; donor init retains neither. Corrected the status text: donor `vold` has no probe (the `NULL` ADD branch is defensive only), legacy REMOVE is not idempotent (missing key returns `ENOKEY`), the real `ioctl` is reached by raw `SYS_ioctl` rather than `dlsym`, `/init.real` needed no `init_exec` under the original ramdisk design (superseded: under the `/system/bin/init` design, both init files are labelled `init_exec` for the post-policy re-exec), and the shim/`vold.rc`/metadata belong to the system image. `plat_sepolicy.cil` came from the donor One UI 7 (Android 15) system image (`system_big3.dat`/`system_full.dat`), not the A11 target; the earlier attribution is corrected in the policy-input entry below. Added `DEVELOPMENT_PLAN.md` with Phase 1–5 gates and the persistent task queue. No build, flash, format, or new ZIP was produced. Still R0; no device test performed.
- 2026-11-16 (design change): Moved the wrapper from the boot ramdisk `/init` to `/system/bin/init`, because AOSP 12 first-stage init executes `/system/bin/init selinux_setup` before loading policy. The donor init moves to `/system/bin/init.real`; both are labelled `init_exec` because the wrapper runs again for the post-policy `second_stage` re-exec. The wrapper preserves `argv[0]` and passes the `ueventd` symlink through without keyring setup. The boot image is no longer modified and `magiskboot` is no longer used. The source and CI workflow were staged at the time of this entry; they were later compiled, host-tested, and pushed (see the build entry below). Still R0.
- 2026-11-16 (build): Compiled the wrapper and shim locally with Zig 0.17.1 (clang 22.1.8 + lld) using the project flags, with no warnings. The wrapper is a 2624-byte ELF64 `ET_EXEC` with entry `0x1010848`, no `PT_INTERP`, no `.dynamic`, and 0 `DT_NEEDED`; the shim is a 6280-byte ELF64 `ET_DYN` with 0 `DT_NEEDED`, imports exactly `__android_log_print` and `__errno`, exports `ioctl`, and exports no memory functions. The first host-test run crashed with `0xC0000005` because `a115f_keyctl_fn` used `unsigned long`, which is 32-bit on Windows LLP64 and truncated a fake-keyring string pointer; switching the callback arguments to `uintptr_t` fixed the tests and left the aarch64 binaries byte-identical (same SHA-256 before and after). Host tests now print `all legacy fscrypt host tests passed`. Commits `325dcd2` (12 added files) and `48438e6` (ABI fix) are pushed to `manus/a115f-legacy-fscrypt`; the standalone CI workflow remains on `local/a115f-legacy-fscrypt-ci-workflow` at `5551184`. A later commit, `5651d76`, adds the CI compile/ELF test (see the CI compile evidence entry). Still R0; no ROM build or device test.
- 2026-11-16 (CI blocker): Attempted to dispatch `build.yml` on `manus/a115f-legacy-fscrypt` via the REST API. The request returned HTTP 404, and `GET /repos/.../actions/workflows` returned `total_count: 0` even though the default branch `oneui7-a11q-boot` contains `.github/workflows/build.yml` (blob `ffc26cf3…`, last modified 2026-10-21). Actions is enabled (`enabled: true`, `allowed_actions: all`), the runners and artifacts endpoints return HTTP 200, and the credential scopes are `gist, read:org, repo` (no `workflow`). The repository has no registered workflows. This requires user action (enable/register workflows in the GitHub UI or provide a `workflow`-scoped credential); do not rewrite or force-push the default branch. Still R0.
- 2026-11-16 (CI registration fixed): Re-asserted the repository Actions permissions with `PUT /repos/zinerom48-tech/ZineROM-a11q/actions/permissions` (`enabled: true`, `allowed_actions: all`, `sha_pinning_required: false`), which returned HTTP 204. `GET /actions/workflows` then reported `total_count: 2`: `build.yml` id `420352259` and `publish-public.yml` id `420352261`, both active. Dispatched the audited branch workflow at `manus/a115f-legacy-fscrypt@48438e6` with `audit_only=false`; run `43909580926` was created. The earlier branch-push probe (`ci/register-workflows`) did not register workflows; the permissions write did.
- 2026-11-16 (CI run): Run `43909580926` passed the full `python3 -m unittest discover -s tests -v` step (`Ran 69 tests in 0.214s`, `OK`), including all 7 new `test_legacy_fscrypt_vold_rc` tests. The job then failed before `make_rom.sh` in `Set up build environment` with `E: Unable to locate package linux-modules-extra-7.0.0-1014-aws` (exit 100). Ubuntu’s package search confirms the package does not exist in any suite or architecture; all 15 repository branches contain the same `linux-modules-extra-$(uname -r)` line. The minimal fix is to remove that package from the apt install list. The edit is pending because the token scopes are still `gist, read:org, repo` (no `workflow`); no workaround was applied. The mod’s compile/host-test/`readelf` checks did not run. No release, flash, or `/data` operation was performed. Still R0.
- 2026-11-16 (CI compile evidence): Added `tests/test_legacy_fscrypt_build.py` at `5651d76`, which mirrors `build.sh`, compiles both freestanding targets and the host tests, and verifies the ELF/dynamic-symbol contract in the workflow's existing test step. Run `43913667988` passed `Ran 76 tests in 3.028s`, `OK`, with all 7 new build tests executed (no skips). CI clang 18 produced wrapper `530e2b3c…` (5296 bytes) and shim `1faab403…` (6880 bytes); the sizes/hashes differ from the Zig artifacts but the verified ELF properties are identical. Step 8 failed again on a different runner kernel (`7.0.0-1017-azure`), confirming the workflow's `$(uname -r)` bug is pool-dependent. At that point the in-ROM install/metadata/`vold.rc` stages were not yet exercised; the next entry closes that gap. No release, flash, or `/data` operation was performed. Still R0.
- 2026-11-16 (CI mod-install evidence): Replaced the mod's `xxd` ELF-header reads with coreutils `od` because the runner image exposes `coreutils` but not `xxd` before the blocked build step, and added `tests/test_legacy_fscrypt_mod_install.py`. Commit `74c8198` is pushed to `manus/a115f-legacy-fscrypt`. Run `43921516470` passed `Ran 80 tests in 6.808s`, `OK`, with all four mod-install tests executed (no skips); they cover compile, the donor init move, wrapper/shim install, `ueventd` symlink preservation, exactly-once metadata, one `LD_PRELOAD`, idempotence, and both fail-closed layouts. Step 8 still failed on `linux-modules-extra-7.0.0-1017-azure` (ISSUE-028). The surrounding `make_rom.sh` flow remains unexercised. No release, flash, or `/data` operation was performed. Still R0.
- 2026-11-16 (CI driver-path evidence): Extended `tests/test_legacy_fscrypt_mod_install.py` with a fifth test that invokes the real `scripts/internal/apply_modules.sh` driver with a copied mod and a sentinel system tree, proving the driver runs the mod and honours `SKIPUNZIP=1`. Commit `3aa20d4` is pushed to `manus/a115f-legacy-fscrypt` (parent `74c8198`; still 14 added files because this commit modifies the existing test). Run `43925890423` passed `Ran 81 tests in 17.249s`, `OK`, with all five mod-install tests executed (no skips). Step 8 still failed on `linux-modules-extra-7.0.0-1017-azure` (ISSUE-028). The surrounding `make_rom.sh` flow remains unexercised. No release, flash, or `/data` operation was performed. Still R0.
- 2026-11-16 (policy-input audit): Wrote a read-only ext4 reader and extracted the donor policy inputs from `system_full.dat` without `7z`/`debugfs`. Confirmed the image is One UI 7 / Android 15 (`ro.build.id=AP3A.240905.015.A2`, SDK 35). Extracted `system/etc/selinux/plat_sepolicy.cil` (2,237,691 bytes, SHA-256 `febc700c…`), `mapping/31.0.cil` (141,860 bytes, SHA-256 `ff34465a…`), and `system_ext/etc/selinux/system_ext_sepolicy.cil` (488,747 bytes, SHA-256 `a0a7cf4e…`). The donor plat policy retains the legacy key allows (`vold init`, `vold vold`, `init init`, `domain kernel`); the 31.0 mapping covers `vold_31_0`/`init_31_0`/`kernel_31_0`; the system_ext policy has no key rules and the SELinux patch only removes named types, so the key allows cannot be dropped. Corrected the earlier claim that `plat_sepolicy.cil` was A11 target policy: it came from the donor image, and the earlier local copy is byte-identical. The assembled policy is still not built; the remaining risks are policy-load ordering and the runtime keyring label. No release, flash, or `/data` operation was performed. Still R0.
- 2026-11-16 (ADD-key flags evidence): Rechecked the shim's `__flags == 0` requirement against both the AOSP 15 source and the donor binary. AOSP 15 `installKey()` zero-initializes the argument, sets `raw_size` from the key, and sets flag bit 0 only when `EncryptionOptions::use_hw_wrapped_key` is true (`KeyUtil.cpp:137,164-167`); `ParseOptionsForApiLevel()` leaves that flag false unless the fstab carries `wrappedkey_v0` (`libfscrypt_fscrypt.cpp:215,227-228`), and `ice` maps to `FSCRYPT_MODE_PRIVATE` (`:58`) with v1 selected for `first_api_level <= 29` (`:212`). Donor `vold` matches: header `memset` at `0x471A4` and a conditional bit-0 store to `arg+0x4c` at `0x47288`–`0x47298`. The A11 `/data` entry has no `wrappedkey_v0`, so the shipped call keeps `__flags == 0`. Recorded as fact 58 and F21/§5.3. This closes the last static argument-layout risk; runtime behavior remains unverified. No build, flash, or `/data` operation was performed. Still R0.
- 2026-11-16 (ramdisk-init pivot evidence): Re-examined the retained A11 boot-image init string dump (`a11_init_strings.txt`, 368,325 bytes, SHA-256 `be7e2e60…`; byte-identical to `ramdisk_init_strings.txt`). It contains `selinux_setup`, `/system/bin/init`, and the Samsung `!@Boot_EBS_F: FirstStageMain Init` / `!@Boot_EBS_F: SELinux Setup` markers, and no `second_stage` code string (only `/second_stage_resources`). That matches the AOSP 12 first-stage binary that ends in `execv("/system/bin/init", {…, "selinux_setup"})`, which is the premise of the `/system/bin/init` wrapper. This is string-level evidence only: the call site was not disassembled and the boot image is no longer retained, so R1 is narrowed but not closed. Recorded as fact 59 and F22, with the §5 assumption and R1 mitigation updated. No build, flash, or `/data` operation was performed. Still R0.
