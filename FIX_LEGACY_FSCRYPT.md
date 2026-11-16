# Fix Design — Legacy fscrypt for the A115F One UI 7 Port

Status: **implemented in source, compiled and host-tested locally, CI-tested including the in-ROM mod-install path; not ROM-built, not device-tested**. Readiness remains **R0**.
Last updated: 2026-11-16.
Scope: make `/data` file-based encryption (FBE v1) succeed on the Galaxy A11 (`SM-A115F`, `a11q`) with the A05s One UI 7 donor system.

This document is the implementation contract for ISSUE-024 and ISSUE-025. It records only facts that were verified from the kernel, donor binaries, AOSP source, and the A11 vendor policy. Anything not directly verified is labelled **hypothesis** or **untested assumption**.

---

## 1. Problem statement

The shipped combination cannot call `fscrypt_initialize_systemwide_keys()` successfully:

1. The A11 4.9.227 kernel implements **only legacy fscrypt key management**. It has no `FS_IOC_ADD_ENCRYPTION_KEY` (`0xC0506617`), `FS_IOC_REMOVE_ENCRYPTION_KEY` (`0xC0406618`), or `FS_IOC_GET_ENCRYPTION_KEY_STATUS` (`0xC080661A`). v1 keys are installed as `logon` keys named `ext4:<8-byte descriptor in hex>`, `f2fs:<...>`, or `fscrypt:<...>` and resolved by the kernel with `request_key()`.
2. Donor Samsung `vold` is **ioctl-only**. It calls the three ioctls above and contains no `installKeyLegacy`, `evictKeyLegacy`, `isFsKeyringSupported`, or `fscryptKeyring` code. Its ADD ioctl reaches the A11 kernel and returns `ENOTTY`.
3. Donor Samsung `init` creates **neither PID 1's session keyring nor the named `fscrypt` keyring**. Even if `vold` installed legacy keys, every process doing I/O on `/data` resolves them through PID 1's inherited session keyring, so `request_key()` would fail.

Both gaps must be fixed. A vold-only fix fails with `Unable to find device keyring`; an init-only fix leaves vold calling an unimplemented ioctl.

---

## 2. Confirmed facts and evidence

| # | Fact | Evidence |
|---|---|---|
| F1 | A11 kernel 4.9.227 has 6 `FS_IOC_SET_ENCRYPTION_POLICY` sites, 4 `FS_IOC_GET_ENCRYPTION_POLICY` sites, and **0** `FS_IOC_ADD_ENCRYPTION_KEY` sites. | `vmlinux.bin` byte scan for `0x800C6613`, `0x400C6615`, `0xC0506617`. |
| F2 | The kernel resolves v1 master keys with `request_key(&key_type_logon, "prefix:descriptor", NULL)`. It first tries `FS_KEY_DESC_PREFIX` (`fscrypt:`), then the filesystem's `key_prefix` (`ext4:` or `f2fs:`). | `a11q_keyinfo.c:79-95`, `:206-213`. |
| F3 | Donor `vold` imports `ioctl` and `syscall`, but **not** `add_key` or `keyctl`. | `donor_vold_dynsyms.txt`; no keyutils strings. |
| F4 | Donor `vold` `installKey()` is at `0x47114`; its ADD ioctl is at `0x472FC` with request `0xC0506617`. The call target is PLT stub `0x140408`; GOT slot `0x146F30` has `R_AARCH64_JUMP_SLOT` symbol index 105, whose name is `ioctl`. | AArch64 disassembly; ELF `.rela.plt` parse. |
| F5 | Donor `vold` `evictKey()` is at `0x47B74`; its REMOVE ioctl is at `0x47C30` with request `0xC0406618`, same PLT stub. `waitForBusyFiles()` is at `0x48054` and uses REMOVE plus GET_STATUS. | AArch64 disassembly. |
| F6 | Donor `vold` has no `fscrypt-provisioning`, `provisioning`, `reloadKeyFromSessionKeyring`, `keyctl_search`, `KEY_SPEC_SESSION_KEYRING`, or `Unable to find device keyring` strings. | Byte scans of `donor_vold`. |
| F7 | Donor `init` has no keyctl/keyring strings or dynsyms, no `libkeyutils.so` DT_NEEDED, and no `svc` instruction in the full `SecondStageMain` (`0x155DFC`, size `0x5574`). | `donor_init` strings/dynsyms/disassembly. |
| F8 | AOSP 12 `SecondStageMain` line 768 creates the session keyring: `keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1)`. `FscryptInstallKeyring()` creates the named `fscrypt` keyring with `add_key("keyring", "fscrypt", 0, 0, KEY_SPEC_SESSION_KEYRING)` when `mount_all` reports `FS_MGR_MNTALL_DEV_FILE_ENCRYPTED`. | `a12_init2.cpp:765-768`; `fscrypt_init_extensions.cpp:44-57`; `builtins.cpp:625-648`. |
| F9 | LineageOS 22.2 and 23.1 still contain `isFsKeyringSupported`, `installKeyLegacy`, and `evictKeyLegacy`. The removal is Samsung-specific, not an Android 15 change. | `keyutil14.cpp`; AOSP/Lineage source. |
| F10 | `/system/bin/ueventd` is a build symlink to `init`; `service ueventd` uses `seclabel u:r:ueventd:s0`. Any replacement of `/system/bin/init` must preserve `argv[0]` so the real init still enters ueventd mode, and the mod must verify the symlink target before installing the wrapper. | `init/Android.bp` symlinks; A11/donor rc; wrapper `argv[0]` check. |
| F11 | A helper cannot inject a keyring into PID 1: `keyctl_session_to_parent()` rejects `parent->pid <= 1`. PID 1 must create the keyrings itself. | Kernel `security/keys/process_keys.c`. |
| F12 | The A11 4.9 key permission model uses **possession bits additively**, and possession propagates through nested keyring search. Default key permissions are `KEY_POS_VIEW|KEY_POS_SEARCH|KEY_POS_LINK|KEY_POS_SETATTR|KEY_USR_VIEW`, plus `KEY_POS_READ` and `KEY_POS_WRITE` for keyrings. No `KEY_OTH_SEARCH` is needed. | `k49_key.c:889-900`; `k49_perm.c:30-77`; `k49_keyring.c:866-873`; `k49_process_keys.c:223-252`. |
| F13 | The donor platform policy and the A11 vendor policy grant every key permission the fix needs. Donor `plat_sepolicy.cil` (2,237,691 bytes, SHA-256 `febc700c…`, extracted from `system_full.dat`; image `AP3A.240905.015.A2`, SDK 35) contains `(allow domain init (key (search)))`, `(allow domain vold (key (search)))`, `(allow domain kernel (key (search)))`, `(allow init init (key (write search setattr)))`, `(allow vold init (key (write search setattr)))`, and `(allow vold vold (key (write search setattr)))`. Donor `mapping/31.0.cil` (SHA-256 `ff34465a…`) maps `vold_31_0`→`vold`, `init_31_0`→`init`, `kernel_31_0`→`kernel`. The A11 vendor policy (`VE=SEPF_SM-A115F_12_0001`, SHA-256 `08e2f3a2…`) adds `(allow vold_31_0 kernel_31_0 (key (write search)))`, `(allow vold_31_0 init_31_0 (key (write)))`, `(allow domain kernel_31_0 (key (search)))`, `(allow init_31_0 kernel_31_0 (key (search)))`, and `(allow vold_31_0 self (key (view read write search link setattr create)))`. Donor `system_ext_sepolicy.cil` (SHA-256 `a0a7cf4e…`) has no key rules, and the SELinux patch only deletes named types. | `plat_sepolicy.cil`, `mapping/31.0.cil`, `system_ext_sepolicy.cil`, `vendor_sepolicy.cil`; ext4 extraction and rule check. |
| F14 | The A11 v1 descriptor is 8 bytes derived by double SHA-512 of the key; the legacy name is `prefix + ":" + 16 lowercase hex chars`. The kernel compares 8 descriptor bytes. | `keyutil12.cpp:108-145`; `a11q_keyinfo.c:85-95`. |
| F15 | The legacy payload is `struct fscrypt_key { u32 mode; u8 raw[64]; u32 size; }`, 72 bytes, with `mode=0`, `raw` = 64-byte key, `size=64`. | AOSP `fillKey` (`keyutil12.cpp:121-131`); kernel `fscrypt` payload checks. |
| F16 | The shim's exact UAPI layouts are: `fscrypt_key_specifier` 40 bytes (type at 0, `u.descriptor[8]` at 8); `fscrypt_add_key_arg` header 80 bytes (`raw_size` at 40, `key_id` at 44, `__flags` at 76, `raw` at 80); `fscrypt_remove_key_arg` 64 bytes (`removal_status_flags` at 40); `fscrypt_get_key_status_arg` 128 bytes (`status` at 64, `status_flags` at 68, `user_count` at 72). | `android15_fscrypt_uapi.h:63-106`; ioctl sizes `0x50`/`0x40`/`0x80`. |
| F17 | The ADD/REMOVE/GET_STATUS call sites use the imported `ioctl` through a normal PLT entry, so `LD_PRELOAD` interposition is viable. | F4/F5. |
| F18 | `create_work_dir.sh` copies and unsigns the stock A11 `boot.img`; `build_flashable_zip.sh` copies `$WORK_DIR/kernel/*.img` unchanged. This fix does not modify the boot image. | Build scripts; checkpoint. |
| F19 | The final ZIP builds each partition image with `file_context-$PARTITION` and `fs_config-$PARTITION`; new files need `SET_METADATA`. | `build_flashable_zip.sh:655-663`; `SET_METADATA()` at `common_utils.sh:538`. |
| F20 | `magiskboot` is downloaded as the **latest** Magisk release by `external/make.sh`; its version is not pinned. This fix no longer uses it. CI compiles with Ubuntu 24.04 clang/lld. | `external_make.sh:249-268`; `build.yml:74-80`; `zz_a115f_legacy_fscrypt/customize.sh`. |

Facts F12-F17 close the earlier keyring-permission, descriptor, payload, ABI-layout, and interposition questions. They do **not** prove runtime behavior on the device.

---

## 3. Architecture

Two components, both required:

```
kernel
  └─ stock A11 /init                    (boot ramdisk, unchanged)
       └─ first stage mounts /system
            └─ exec /system/bin/init selinux_setup
                 └─ new static arm64 wrapper (pre-policy, still PID 1)
                      ├─ keyctl(GET_KEYRING_ID, SESSION, create=1)
                      ├─ keyctl(SEARCH, SESSION, keyring, "fscrypt")
                      ├─ add_key("keyring", "fscrypt", NULL, 0, SESSION)   (if absent)
                      └─ execve("/system/bin/init.real", argv, envp)
                           └─ donor init: selinux_setup -> second_stage
                                └─ service vold (LD_PRELOAD=liba115f_fscrypt_shim.so)
                                     └─ ioctl(ADD/REMOVE/GET_STATUS)
                                          └─ shim: real ioctl first; on ENOTTY translate to
                                             add_key/keyctl legacy logon keys
```

Why a `/system/bin/init` wrapper instead of a ramdisk wrapper:
- AOSP 12 `FirstStageMain` execs `/system/bin/init` with `{"init", "selinux_setup"}` before loading SELinux policy (`first_stage22.cpp:559-565`). Replacing that file therefore runs our code in PID 1, pre-policy, exactly where the keyrings must be created.
- The stock A11 boot image is not unpacked, repacked, or re-signed, so the unverified ramdisk layout, compression, header version, and AVB state are no longer risks. `magiskboot` is not used.
- The donor `/system/bin/init` is a regular ELF64 AArch64 binary (confirmed), not a symlink.
- `/system/bin/ueventd` is a symlink to `init`. The wrapper preserves `argv[0]` and skips keyring setup when invoked as `ueventd`, then execs `init.real`; the real init's basename-based ueventd mode is preserved. The service's `seclabel u:r:ueventd:s0` still applies.
- The donor init is re-exec'd as `/system/bin/init second_stage` after policy load, so the wrapper runs a second time. The keyrings already exist; creation is idempotent. Both `/system/bin/init` and `/system/bin/init.real` must therefore be labelled `u:object_r:init_exec:s0`, because this second exec happens post-policy.

Why a vold shim instead of replacing vold:
- Donor `vold` is a large, Samsung-patched binary; rebuilding it is not practical.
- The failure is localized to three ioctl requests. A 64-bit `LD_PRELOAD` shim is the smallest change and preserves every other vold behavior.
- Interposition is proven possible by F17 (imported `ioctl`, normal PLT).

---

## 4. Part A — `/system/bin/init` wrapper

### 4.1 Behavior

The wrapper is a freestanding, statically linked AArch64 executable installed as `/system/bin/init`; the donor binary moves to `/system/bin/init.real`. It performs only these operations, then execs the real init with the original `argv` and `envp`:

1. If `argv[0]`'s basename is `ueventd`, log `UEVENTD_PASSTHROUGH` and exec immediately. Do not create keyrings in a ueventd process.
2. Create PID 1's session keyring:
   `keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 1)`.
3. Look up the named `fscrypt` keyring:
   `keyctl(KEYCTL_SEARCH, KEY_SPEC_SESSION_KEYRING, "keyring", "fscrypt", 0)`.
4. If absent, create it:
   `add_key("keyring", "fscrypt", NULL, 0, KEY_SPEC_SESSION_KEYRING)`.
5. Log one marker per result to `/dev/kmsg`.
6. `execve("/system/bin/init.real", argv, envp)`.

The wrapper must **never block boot**. If a keyring operation fails, it logs the error and still execs the real init. The shim creates the `fscrypt` keyring as a fallback if it is missing.

### 4.2 Syscall ABI

AArch64 generic syscall numbers:

| Operation | Number | Notes |
|---|---|---|
| `add_key` | 217 | `(type, description, payload, plen, keyring)` |
| `keyctl` | 219 | `(cmd, arg2, arg3, arg4, arg5)` |
| `execve` | 221 | `(path, argv, envp)` |
| `openat`/`open` | 56/― | use `openat(AT_FDCWD, ...)` |
| `write` | 64 | |
| `mknodat`/`mknod` | 33/― | use `mknodat(AT_FDCWD, ...)` |
| `close` | 57 | |

Constants:

| Constant | Value |
|---|---|
| `KEY_SPEC_SESSION_KEYRING` | `-3` |
| `KEYCTL_GET_KEYRING_ID` | `0` |
| `KEYCTL_SEARCH` | `10` |
| `AT_FDCWD` | `-100` |
| `O_WRONLY` | `1` |
| `O_CLOEXEC` | `0x80000` |
| `S_IFCHR` | `0x2000` |
| `S_IRUSR|S_IWUSR` | `0x180` |
| `/dev/kmsg` device | major 1, minor 11; `new_encode_dev` value `0x010B` |

`keyctl` is variadic in libc but a raw syscall takes five `unsigned long` arguments; unused arguments are zero.

### 4.3 Logging

The wrapper runs before `/dev` is populated by first-stage init. It creates `/dev/kmsg` if needed:

```c
mknodat(AT_FDCWD, "/dev/kmsg", S_IFCHR | 0600, 0x010B);
int fd = openat(AT_FDCWD, "/dev/kmsg", O_WRONLY | O_CLOEXEC);
write(fd, "<6>A115F_FSCRYPT_WRAPPER_ENTER\n", ...);
```

If the node or open fails, it writes to fd 1 as a last resort. The `<6>` prefix is `KERN_INFO`; messages appear in `dmesg`, `logcat -b kernel`, and pstore after a crash.

Exact markers:

```
A115F_FSCRYPT_WRAPPER_ENTER <argv0>
A115F_FSCRYPT_WRAPPER_SESSION_OK <serial>
A115F_FSCRYPT_WRAPPER_SESSION_FAIL <errno>
A115F_FSCRYPT_WRAPPER_FSCRYPT_OK <serial>
A115F_FSCRYPT_WRAPPER_FSCRYPT_FAIL <errno>
A115F_FSCRYPT_WRAPPER_UEVENTD_PASSTHROUGH
A115F_FSCRYPT_WRAPPER_EXEC /system/bin/init.real
A115F_FSCRYPT_WRAPPER_EXEC_FAIL <errno>
```

### 4.4 Failure behavior

- Keyring failure: log, continue to exec. A missing keyring produces a diagnosable shim failure rather than a dead boot.
- `execve` failure: log `EXEC_FAIL`, then halt in a `wfi` loop. Do **not** reboot automatically; a boot loop destroys the evidence and may trigger Rescue Party. The only recovery is reflashing the stock system image or the last known-good port ZIP (see §11).

### 4.5 Compilation

Build with the CI clang, freestanding, no libc:

```
clang --target=aarch64-linux-gnu -nostdlib -static -no-pie \
  -ffreestanding -fno-builtin -fno-stack-protector \
  -Wl,-e,_start -Wl,--build-id=none \
  -o init_wrapper init_wrapper.c
```

The source must provide `_start` (a naked assembly stub that passes the kernel `sp` to C), `memset`, `memcpy`, and the raw syscall stubs. Do not link `libc`, `libdl`, or `libkeyutils`. Verify:

```
file init_wrapper            # ELF 64-bit LSB executable, ARM aarch64, statically linked
readelf -l init_wrapper      # no PT_INTERP
readelf -d init_wrapper      # no DT_NEEDED
```

`-no-pie` keeps the binary `ET_EXEC`; the kernel can exec either `ET_EXEC` or `ET_DYN`, but `ET_EXEC` is simpler to inspect.

### 4.6 Packaging

The wrapper and the shim live in the **system image**. The boot image is not modified:

1. Require `$WORK_DIR/system/system/bin/init` to be a regular ELF64 AArch64 file; fail if it is missing, a symlink, or the wrong architecture.
2. Require `$WORK_DIR/system/system/bin/ueventd` to be a symlink whose target is `init` or `./init`; the wrapper's ueventd passthrough depends on it.
3. Refuse to proceed if `init.real` already exists (idempotence guard).
4. `mv init init.real`, install the compiled wrapper as `init`, mode 0750, uid/gid 0.
5. `SET_METADATA "system" "system/bin/init" 0 0 0750 "u:object_r:init_exec:s0"`.
6. `SET_METADATA "system" "system/bin/init.real" 0 0 0750 "u:object_r:init_exec:s0"`.
7. Install the shim at `$WORK_DIR/system/system/lib64/liba115f_fscrypt_shim.so`, mode 0644, and label it `u:object_r:system_lib_file:s0`.
8. Patch `$WORK_DIR/system/system/etc/init/vold.rc` with the `LD_PRELOAD` line.

Do not touch `$WORK_DIR/kernel/boot.img`; do not require `magiskboot`. If a future layout moves system init into the ramdisk, this design must be re-evaluated rather than guessed.

---

## 5. Part B — 64-bit vold `LD_PRELOAD` shim

### 5.1 Interposition contract

The shim exports:

```c
int ioctl(int fd, unsigned long request, ...);
```

Algorithm:

1. Read the optional third argument with `__builtin_va_arg`.
2. Call the real ioctl with a raw `SYS_ioctl` syscall (number 29).
3. If it returns 0, return 0.
4. If it fails with anything other than `ENOTTY`, set `errno` and return `-1`.
5. On `ENOTTY`, dispatch by request:
   - `0xC0506617` → `shim_add`
   - `0xC0406618` → `shim_remove`
   - `0xC080661A` → `shim_status`
   - otherwise → restore `ENOTTY`, return `-1`.
6. Log exactly one marker per result.

The raw-syscall passthrough avoids a `libdl` dependency and cannot recurse into the shim. `dlsym(RTLD_NEXT, "ioctl")` is an acceptable alternative, but it adds a libdl dependency and is not necessary.

### 5.2 Exact UAPI layouts

All fields are little-endian AArch64.

```c
struct fscrypt_key_specifier {          // 40 bytes
    __u32 type;                         // offset 0
    __u32 __reserved;                   // offset 4
    union {
        __u8 __reserved[32];
        __u8 descriptor[8];             // offset 8
        __u8 identifier[16];            // offset 8
    } u;
};

struct fscrypt_add_key_arg {            // header 80 bytes; ioctl size 0x50
    struct fscrypt_key_specifier key_spec; // 0..39
    __u32 raw_size;                     // 40
    __u32 key_id;                       // 44
    __u32 __reserved[7];                // 48..75
    __u32 __flags;                      // 76
    __u8 raw[];                         // 80
};

struct fscrypt_remove_key_arg {         // 64 bytes; ioctl size 0x40
    struct fscrypt_key_specifier key_spec; // 0..39
    __u32 removal_status_flags;         // 40
    __u32 __reserved[5];                // 44..63
};

struct fscrypt_get_key_status_arg {     // 128 bytes; ioctl size 0x80
    struct fscrypt_key_specifier key_spec; // 0..39
    __u32 __reserved[6];                // 40..63
    __u32 status;                       // 64
    __u32 status_flags;                 // 68
    __u32 user_count;                   // 72
    __u32 __out_reserved[13];           // 76..127
};
```

Constants:

| Constant | Value |
|---|---|
| `FSCRYPT_KEY_SPEC_TYPE_DESCRIPTOR` | 1 |
| `FSCRYPT_KEY_SPEC_TYPE_IDENTIFIER` | 2 |
| `FSCRYPT_KEY_STATUS_ABSENT` | 1 |
| `FSCRYPT_KEY_STATUS_PRESENT` | 2 |
| `FSCRYPT_KEY_STATUS_INCOMPLETELY_REMOVED` | 3 |
| `FSCRYPT_KEY_STATUS_FLAG_ADDED_BY_SELF` | 1 |
| `__FSCRYPT_ADD_KEY_FLAG_HW_WRAPPED` | 1 |

### 5.3 Translation

**ADD (`0xC0506617`)**

- If `arg == NULL`, return `-1` with `ENOTTY`. This is AOSP's legacy-support probe; donor vold does not contain `isFsKeyringSupported`, but the shim must not turn a probe into success.
- Require `key_spec.type == 1`, `raw_size == 64`, `key_id == 0`, `__flags == 0`.
  - `__flags & HW_WRAPPED`: return `-1` with `EOPNOTSUPP` (legacy keys cannot be wrapped).
  - Type 2 (`IDENTIFIER`): return `-1` with `EOPNOTSUPP`; the A11 kernel has no v2 policy support.
  - Any other size: return `-1` with `EINVAL`.
- Find the `fscrypt` keyring in PID 1's session keyring. If absent, create it (fallback; the wrapper normally did this).
- Build the payload:

```c
struct fscrypt_key {
    __u32 mode;        // 0, unused by the kernel
    __u8  raw[64];     // the 64-byte v1 key
    __u32 size;        // 64
};                     // sizeof == 72
```

- Install three keys into the `fscrypt` keyring:

```
ext4:<hex8>
f2fs:<hex8>
fscrypt:<hex8>
```

where `<hex8>` is the 16-character lowercase hex of `key_spec.u.descriptor[0..7]`.
- If any `add_key` fails, set `errno` and return `-1`. AOSP installs all three and fails on the first error.
- Zero the local payload before returning.
- Return 0.

**REMOVE (`0xC0406618`)**

- Require `key_spec.type == 1`; otherwise `EOPNOTSUPP`.
- For each of the three names, `keyctl_search(device_keyring, "logon", name, 0)`.
- If found, `keyctl_unlink(serial, device_keyring)`.
- Match AOSP `evictKeyLegacy`: if any name is missing, set `removal_status_flags = 0` and return `-1` with `ENOKEY`; otherwise set `removal_status_flags = 0` and return 0. (A modern kernel also returns `ENOKEY` for an absent key; do not silently claim success.)
- Do not revoke or invalidate; unlink only.

**GET_STATUS (`0xC080661A`)**

- Require `key_spec.type == 1`; otherwise `EOPNOTSUPP`.
- Search the three names.
- If any exists: `status = PRESENT` (2), `status_flags = ADDED_BY_SELF` (1), `user_count = 1`.
- Otherwise: `status = ABSENT` (1), `status_flags = 0`, `user_count = 0`.
- Leave the reserved fields zero.
- Return 0.

`INCOMPLETELY_REMOVED` is not meaningful for legacy keys; do not report it.

### 5.4 Logging

The shim logs to logcat with `__android_log_print(A115F_ANDROID_LOG_INFO, "A115F_FSCRYPT", …)`. Emit one message per distinct result:

```
SHIM_LOADED
ADD_PROBE_ENOTTY
ADD_UNSUPPORTED <errno>
ADD_OK
ADD_FAIL <errno>
REMOVE_UNSUPPORTED <errno>
REMOVE_OK
REMOVE_FAIL <errno>
STATUS_UNSUPPORTED <errno>
STATUS_PRESENT
STATUS_ABSENT
STATUS_FAIL <errno>
```

The wrapper writes to `/dev/kmsg` (falling back to `/dev/console`, then fd 2) with prefix `A115F_FSCRYPT_WRAPPER: `:

```
A115F_FSCRYPT_WRAPPER: ENTER
A115F_FSCRYPT_WRAPPER: SESSION_OK <serial>
A115F_FSCRYPT_WRAPPER: SESSION_FAIL <errno>
A115F_FSCRYPT_WRAPPER: FSCRYPT_OK <serial>
A115F_FSCRYPT_WRAPPER: FSCRYPT_FAIL <errno>
A115F_FSCRYPT_WRAPPER: UEVENTD_PASSTHROUGH
A115F_FSCRYPT_WRAPPER: EXEC /system/bin/init.real
A115F_FSCRYPT_WRAPPER: EXEC_FAIL <errno>
```

### 5.5 Compilation

```
clang --target=aarch64-linux-gnu -shared -fPIC -nostdlib \
  -ffreestanding -fno-builtin -fno-stack-protector \
  -Wl,-soname,liba115f_fscrypt_shim.so \
  -o liba115f_fscrypt_shim.so fscrypt_legacy_shim.c
```

The shim declares `extern int *__errno(void);` and sets `*__errno() = err` on failure. Bionic exports `__errno`; vold already links libc. Do not link `libdl` or `libkeyutils`.

A constructor logs `SHIM_LOADED`; keyring fallback creation happens lazily inside `a115f_legacy_install()` when the `fscrypt` keyring search fails with `ENOKEY`:

```c
__attribute__((constructor)) static void shim_init(void) { ... }
```

Verify:

```
file liba115f_fscrypt_shim.so   # ELF 64-bit LSB shared object, ARM aarch64
readelf -d liba115f_fscrypt_shim.so
readelf -Ws liba115f_fscrypt_shim.so | grep ' ioctl$'
```

### 5.6 Loading

Patch the donor `/system/etc/init/vold.rc` service block:

```
service vold /system/bin/vold \
    --blkid_context=... \
    --fsck_context=...
    setenv LD_PRELOAD /system/lib64/liba115f_fscrypt_shim.so
    class core
    ...
```

Insert the `setenv` line **after** the complete multiline `service vold` command, not after the first backslash-continued line. Place it before `class core`. The file has exactly one `service vold` block and one `class core` line.

`LD_PRELOAD` is honored for vold because:
- vold is started by init as uid 0, not setuid; no `AT_SECURE` transition.
- vold is in the default linker namespace and `/system/lib64` is on its search path.
- SELinux `system_lib_file` is readable/mappable by vold under the stock policy.

If `LD_PRELOAD` is ignored (no `SHIM_LOADED` marker), the fallback is a `/system/bin/vold` wrapper that `execv`s a relabeled `vold.real` with `LD_PRELOAD` set. Do not implement the fallback until the primary mechanism is tested and fails.

---

## 6. SELinux and keyring permissions

### 6.1 Labels

| Object | Created by | Label |
|---|---|---|
| PID 1 session keyring | wrapper, pre-policy | `kernel` (initial SID) |
| Named `fscrypt` keyring | wrapper, pre-policy | `kernel` (initial SID) |
| Logon keys `ext4:`/`f2fs:`/`fscrypt:` | vold (shim), after policy load | `vold` |

If the policy is already loaded before the wrapper runs (not expected: `selinux_setup` is the policy loader), the keyrings would be labelled `init`, but the platform rule grants `write search setattr` and not `create` on the session keyring, so creation may be denied. The wrapper logs the failure and still execs the real init; the shim can create the `fscrypt` keyring in the `vold` domain as a fallback. If the shim creates the `fscrypt` keyring, that keyring is labelled `vold`; the `(allow vold vold (key (write search setattr)))` rule covers it.

### 6.2 Required operations and rules

| Operation | Subject | Object | Permission | Rule source |
|---|---|---|---|---|
| create/link `fscrypt` keyring | wrapper (`init` domain) | session keyring (`kernel`) | pre-policy; no check | — |
| search session keyring | vold | `kernel` | `search` | vendor `vold_31_0 kernel_31_0 (key (write search))` |
| add logon key | vold | `fscrypt` (`kernel`) | `write` | same vendor rule |
| search `fscrypt` keyring | vold | `kernel` | `search` | same vendor rule |
| create logon keys | vold | new key (`vold`) | no separate check on `add_key` | — |
| search logon key | vold / any process | `vold` | `search` | plat `(allow domain vold (key (search)))` |
| search `fscrypt` keyring | any process | `kernel` | `search` | plat `(allow domain kernel (key (search)))` / vendor `(allow domain kernel_31_0 (key (search)))` |
| unlink logon key | vold | `fscrypt` (`kernel` or `vold`) | `write` | vendor `vold_31_0 kernel_31_0 (key (write search))` or `vold_31_0 self (key (... write ...))` |
| view/read logon key | kernel `request_key` on behalf of any process | `vold` | `view` via possession | `KEY_POS_VIEW` default permission; plat `(allow domain vold (key (search)))` |

If the wrapper runs after policy load (not expected), substitute `init` for `kernel` in the session-keyring and `fscrypt`-keyring rows; the `vold init`/`init init` rules still cover vold's search and write, but keyring creation in `init` may be denied because the platform rule does not grant `create`. The wrapper logs and continues; the shim fallback creates the keyring in the `vold` domain.

### 6.3 Why no `KEYCTL_SETPERM` is needed

Kernel 4.9 `key_task_permission()` uses the possession bits (`key->perm >> 24`) **additively** whenever `is_key_possessed(key_ref)` is true. `keyring_search_aux()` sets `ctx->possessed = is_key_possessed(keyring_ref)` and propagates it to every nested key it returns. The default permissions from `key_create_or_update()` include `KEY_POS_VIEW|KEY_POS_SEARCH|KEY_POS_LINK|KEY_POS_SETATTR`, and keyrings also get `KEY_POS_READ|KEY_POS_WRITE`.

Therefore:
- A process that inherits PID 1's session keyring possesses the `fscrypt` keyring and its logon keys.
- Search and view succeed through the possession bits; `KEY_OTH_SEARCH` is not required.
- vold's `add_key`/`keyctl_unlink` needs write on the keyring; `KEY_POS_WRITE` is present for keyrings.

This matches AOSP Android 9-12 legacy behavior and the existing A11 policy. Do not add `KEYCTL_SETPERM` unless a device log shows `EACCES` from `add_key`/`request_key` with the possession path already active.

### 6.4 Residual SELinux risks

- If the shim is loaded but `add_key` returns `EACCES`, capture the AVC denial. Do not assume the policy is wrong; the most likely cause is a keyring label mismatch.
- `/system/bin/init.real` **does** need `u:object_r:init_exec:s0`: the donor init is re-exec'd as `/system/bin/init second_stage` after policy load. `/system/bin/init` (the wrapper) needs the same label. Both are set by `SET_METADATA`; the mod fails if either update fails.
- `/system/bin/ueventd` is a symlink to `init`, so the ueventd service now executes the wrapper. The wrapper detects `ueventd` from `argv[0]`, skips keyring setup, and execs `init.real`. The service `seclabel u:r:ueventd:s0` still labels the process. If ueventd fails to start with an exec denial, capture the AVC before changing labels; do not unlink or replace the symlink.
- The shim is a `/system/lib64` file labelled `system_lib_file`. If vold is denied `read`/`map`/`execute`, capture the denial before adding any rule.

---

## 7. Build integration

### 7.1 Branch and files

Base branch: `manus/a115f-vendor-fingerprint` at `23f8998f0200969f5748e9eb012d1dc52037a054`. Do **not** build from `build-once` or `preflight`.

New files on `manus/a115f-legacy-fscrypt` (head `3aa20d4`, based on `manus/a115f-vendor-fingerprint@23f8998f0200969f5748e9eb012d1dc52037a054`):

```
prebuilts/a115f_legacy_fscrypt/
  build.sh
  src/a115f_syscalls.h
  src/freestanding.c
  src/fscrypt_legacy.h
  src/fscrypt_legacy.c
  src/init_wrapper.c
  src/fscrypt_legacy_shim.c
  tests/test_legacy.c
scripts/patch_a115f_legacy_fscrypt_vold_rc.py
tests/test_legacy_fscrypt_vold_rc.py
tests/test_legacy_fscrypt_build.py
tests/test_legacy_fscrypt_mod_install.py
.github/workflows/legacy-fscrypt-ci.yml   (local-only; not pushed)
unica/mods/zz_a115f_legacy_fscrypt/
  customize.sh
  module.prop
```

The workflow file is preserved on local branch `local/a115f-legacy-fscrypt-ci-workflow` at `5551184`, because the push credential lacks GitHub's `workflow` scope. All other files are pushed as `3aa20d4` (14 files; `74c8198` added the in-ROM mod-install test, `5651d76` added the CI compile/ELF test, and `48438e6` is the pointer-size ABI fix on top of `325dcd2`).

Compiled outputs (not committed):

```
prebuilts/a115f_legacy_fscrypt/out/init.a115f_legacy_fscrypt
prebuilts/a115f_legacy_fscrypt/out/liba115f_fscrypt_shim.so
```

### 7.2 CI compile step

`legacy-fscrypt-ci.yml` is not pushed and is not registered in the repository. The same checks run in two places that are pushed:

1. `build.yml` runs `python3 -m unittest discover -s tests -v`, which includes `tests/test_legacy_fscrypt_vold_rc.py`, `tests/test_legacy_fscrypt_build.py`, and `tests/test_legacy_fscrypt_mod_install.py`. The build test mirrors `build.sh`, compiles both targets, compiles/runs the host tests, and verifies the ELF/dynamic-symbol contract; it skips only when no target compiler is present. The mod-install test drives the real `customize.sh`, `SET_METADATA`, and `vold.rc` patcher against a synthetic donor tree, and one of its tests invokes the real `scripts/internal/apply_modules.sh` driver to prove `SKIPUNZIP=1` suppresses the automatic system-tree copy.
2. `zz_a115f_legacy_fscrypt/customize.sh` compiles both targets with `build.sh`, runs `build.sh --host-test`, runs the patcher tests, verifies the donor init/ueventd layout, checks the ELF type/architecture/`PT_INTERP`/`.dynamic`/`DT_NEEDED`, verifies the shim exports `ioctl` and no memory functions and imports only `__android_log_print`/`__errno`, and checks the wrapper/shim markers.

Local equivalent results (Zig 0.17.1 / clang 22.1.8 + lld) are recorded in `PORTING_STATUS.md` fact 45: both binaries compile without warnings; the wrapper is a 2624-byte `ET_EXEC` and the shim a 6280-byte `ET_DYN`, each with 0 `DT_NEEDED`; host tests pass.

CI registration is fixed (ISSUE-027): `build.yml` is active (id `420352259`). Run `43909580926` at `48438e6` passed 69 tests and then failed at `Set up build environment` with `E: Unable to locate package linux-modules-extra-7.0.0-1014-aws` (ISSUE-028). After adding `tests/test_legacy_fscrypt_build.py`, run `43913667988` at `5651d76` passed 76 tests with no skips: Ubuntu clang 18 compiled both targets and `readelf` verified the symbol contract. CI hashes: wrapper `530e2b3c…` (5296 bytes), shim `1faab403…` (6880 bytes). After adding `tests/test_legacy_fscrypt_mod_install.py` and replacing the mod's `xxd` reads with coreutils `od`, run `43921516470` at `74c8198` passed 80 tests with no skips, including the four mod-install tests. After adding the real `apply_modules.sh` driver test, run `43925890423` at `3aa20d4` passed 81 tests with no skips, including all five mod-install tests. Runs 2, 3, and 4 then failed at `Set up build environment` with `linux-modules-extra-7.0.0-1017-azure`, confirming the package name depends on the runner pool. The full `make_rom.sh` flow (work-dir creation, image build, ZIP packing) still has not run; the mod's own compile/install/metadata/`vold.rc` stages and the driver-originated invocation are now covered by the committed tests. After the workflow line is fixed, re-dispatch and record the step logs. Do not modify `publish-public.yml`.

### 7.3 Mod execution order

`make_rom.sh` order:

1. `create_work_dir.sh` — copies and unsigns the stock `boot.img`.
2. patches.
3. `apply_modules.sh "$SRC_DIR/unica/patches"`.
4. `apply_modules.sh "$SRC_DIR/target/$TARGET_CODENAME/patches"`.
5. `apply_modules.sh "$SRC_DIR/unica/mods"` — this runs `zz_a115f_legacy_fscrypt`.
6. APK build.
7. `build_flashable_zip.sh` — copies `$WORK_DIR/kernel/*.img` unchanged and packs the modified system image.

The mod therefore edits `$WORK_DIR/system/system` in step 5. The boot image stays byte-identical to the stock unsigned image produced by `create_work_dir.sh`.

### 7.4 Mod behavior

`unica/mods/zz_a115f_legacy_fscrypt/customize.sh` must:

1. Skip unless `TARGET_CODENAME == a115f`.
2. Require `$WORK_DIR/system/system/bin/init` to be a regular ELF64 AArch64 file (the donor init).
3. Require `$WORK_DIR/system/system/bin/ueventd` to be a symlink to `init` or `./init`.
4. Fail if `$WORK_DIR/system/system/bin/init.real` already exists.
5. Compile both binaries with `prebuilts/a115f_legacy_fscrypt/build.sh`; fail on any compile or architecture check error.
6. Move donor `init` to `init.real`; install the wrapper as `init`; install the shim into `$WORK_DIR/system/system/lib64`.
7. `SET_METADATA "system" "system/bin/init" 0 0 0750 "u:object_r:init_exec:s0"`.
8. `SET_METADATA "system" "system/bin/init.real" 0 0 0750 "u:object_r:init_exec:s0"`.
9. `SET_METADATA "system" "system/lib64/liba115f_fscrypt_shim.so" 0 0 0644 "u:object_r:system_lib_file:s0"`.
10. Patch `$WORK_DIR/system/system/etc/init/vold.rc` to add the `setenv` line exactly once, inside the `service vold` block. Use `scripts/patch_a115f_legacy_fscrypt_vold_rc.py`; it is idempotent and rejects a different pre-existing `LD_PRELOAD`.
11. Verify the final `vold.rc` contains `service vold`, `setenv LD_PRELOAD`, and the shim path.
12. Verify the final `init` is the wrapper (contains the `A115F_FSCRYPT_WRAPPER` marker), `init.real` is the donor init, and `ueventd` still resolves to `init`.
13. Do not modify or repack `boot.img`; do not call `magiskboot`.

Use `SKIPUNZIP=1` in `customize.sh` so `APPLY_MODULE` does not auto-copy a `system/` tree.

### 7.5 Required failure conditions

The build must fail loudly if:

- `$WORK_DIR/system/system/bin/init` is missing, a symlink, or not ELF64 AArch64.
- `$WORK_DIR/system/system/bin/ueventd` is missing or does not resolve to `init`.
- `init.real` already exists.
- Compilation of the wrapper or shim fails.
- The wrapper is not `ET_EXEC`/no `PT_INTERP`, or the shim is not `ET_DYN`/exports `ioctl`; either has a `DT_NEEDED` dependency.
- `vold.rc` is missing or has no `service vold` block.
- The `setenv` patch fails or produces a different `LD_PRELOAD`.
- `SET_METADATA` fails for either init file or the shim.
- The final wrapper/shim markers are absent.

Do not silently skip the fix and ship a ZIP that cannot pass FBE.

---

## 8. Verification plan

### 8.1 Host/static checks (no device)

- Compile-time `_Static_assert`s for every struct size and offset in §5.2.
- Unit test the descriptor → name formatting: descriptor `00 11 22 33 44 55 66 77` → `ext4:0011223344556677`, `f2fs:...`, `fscrypt:...`.
- Unit test the 72-byte payload: `mode == 0`, `size == 64`, `raw` copied exactly.
- Unit test the probe: `arg == NULL` → `ENOTTY`, not success.
- Unit test ENOTTY-only fallback: raw ioctl success → no translation; raw ioctl `EINVAL` → no translation; raw ioctl `ENOTTY` → translation.
- Verify wrapper: ELF64 AArch64, `ET_EXEC`, no `PT_INTERP`, no `DT_NEEDED`.
- Verify shim: ELF64 AArch64, `ET_DYN`, exports `ioctl`, no `DT_NEEDED`, and only `__android_log_print`/`__errno` undefined.
- Verify the mod's system-image checks against the actual donor `system/bin/init` and `ueventd` symlink at build time; fail closed on any layout mismatch.
- The mod greps the compiled wrapper for `A115F_FSCRYPT_WRAPPER` and the shim for `SHIM_LOADED`; it performs the `readelf`/`awk` dependency and export checks.
- Unit-test the `vold.rc` patcher against single-line, multiline, CRLF, missing-service, and already-patched inputs.
- **Local result (2026-11-16):** all checks pass with Zig 0.17.1/clang 22.1.8: wrapper `ET_EXEC`, 2624 bytes, no `PT_INTERP`/`.dynamic`, 0 `DT_NEEDED`; shim `ET_DYN`, 6280 bytes, 0 `DT_NEEDED`, imports exactly `__android_log_print __errno`, exports `ioctl`, no memory exports; host tests print `all legacy fscrypt host tests passed`. The host tests initially crashed on Windows because the callback arguments used `unsigned long` (32-bit on LLP64); switching to `uintptr_t` fixed the host-only truncation and left the aarch64 binaries byte-identical.
- **CI result (2026-11-16):** run `43913667988` at `5651d76` executed `tests/test_legacy_fscrypt_build.py` with no skips. Ubuntu clang 18 compiled both freestanding targets, `cc` compiled and ran the host tests, and `readelf` verified the ELF/dynamic-symbol contract above. CI-built SHA-256: wrapper `530e2b3c1b9ddb5f66ca70ebde6ca74656428098dac2a50332d7928c25b00974` (5296 bytes), shim `1faab403b0b8c9c46c3e6f23d2b44270323e3bb3b860857295d502afc242397d` (6880 bytes). The size/hash difference from the Zig build is expected from the different toolchain/linker; the verified properties match. This is compile/test-step evidence, not a ROM build. Run `43921516470` at `74c8198` passed all 80 tests, including `tests/test_legacy_fscrypt_mod_install.py` (4 tests), which exercises the real mod's compile, donor init move, wrapper/shim install, `ueventd` symlink preservation, exactly-once metadata, single `LD_PRELOAD`, idempotence, and fail-closed layouts against a synthetic donor tree. Run `43925890423` at `3aa20d4` passed all 81 tests, including the fifth mod-install test that invokes the real `scripts/internal/apply_modules.sh` driver and proves `SKIPUNZIP=1` suppresses the automatic system-tree copy. These tests do not cover the full `make_rom.sh` flow.

### 8.2 Device test matrix

Prerequisites: unlocked bootloader, verified stock `A115FXXS6CWK2` restore package, user approval, and a documented recovery path.

| Case | `/data` state | Expected result |
|---|---|---|
| D1 | Freshly formatted by the ROM installer | wrapper markers; shim ADD_OK ×3; `/data` mounts; zygote/system_server; lockscreen |
| D2 | Existing `/data` from a previous port attempt | wrapper/shim markers; either mounts (same key) or a clear `fscrypt` key mismatch; no boot loop |
| D3 | Stock-encrypted `/data` (if preserved) | mounts only if the stock key is retrievable; otherwise a documented `cryptfs` error, not a silent reboot |
| D4 | No SIM / no network | boot to lockscreen independent of RIL/Wi-Fi |

Do not format `/data` for a test without explicit user approval. Do not test on a device without a verified restore path.

### 8.3 Log capture

One boot, one file, SHA-linked to the exact ZIP:

```
adb shell dmesg > dmesg.txt
adb logcat -b all -d > logcat.txt
adb shell getprop > getprop.txt
adb shell ls -l /data > data_ls.txt 2>&1
```

Grep:

```
A115F_FSCRYPT
Failed to install fscrypt key
Unable to find device keyring
fscrypt
keymaster
keystore2
avc: denied
```

### 8.4 Interpretation

| Observation | Meaning | Next action |
|---|---|---|
| `WRAPPER_ENTER` missing | `/system/bin/init` is not the wrapper, or first-stage init did not exec it | Verify the installed system image contains the wrapper; restore the previous port ZIP or stock system |
| `UEVENTD_PASSTHROUGH` missing for ueventd | `argv[0]` handling or the symlink was changed | Check the ueventd symlink and service definition; do not replace the symlink |
| `SESSION_FAIL` / `FSCRYPT_FAIL` | Keyring syscall failed | Check `CONFIG_KEYS`, seccomp, kernel log |
| `SHIM_LOADED` missing | `LD_PRELOAD` ignored or rc patch absent | Verify `vold.rc`, linker namespace, `AT_SECURE` |
| `ADD_PROBE_ENOTTY` then no `ADD_OK` | vold probed but did not install | Check the following vold error; capture full log |
| `ADD_OK` ×3 but `/data` fails | `request_key` cannot find the keys | Check session keyring inheritance, keyring label, AVC denials, `CONFIG_KEYS` |
| `ADD_FAIL EACCES` | SELinux key permission or label mismatch | Capture AVC; compare labels; do not add a blind rule |
| `ADD_FAIL ENOKEY` | `fscrypt` keyring missing | Check wrapper/shim keyring creation |
| `Unexpected ioctl` | New request number (e.g., ALL_USERS) | Add handling only after confirming the caller and semantics |
| No markers, boot loop | Wrapper/init.real problem in the system image | Reflash the previous port ZIP or stock system; do not iterate blindly |

---

## 9. Risks and mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | A11 stock first stage does not exec `/system/bin/init selinux_setup`, or the donor init layout differs | Low | Wrapper never runs / build fails | Verified AOSP 12 flow; mod fails closed if `system/bin/init` is not a regular ELF64 binary |
| R2 | Donor init is re-exec'd post-policy and `init.real` label is wrong | Low | Second-stage exec denial | Both files labelled `init_exec` via `SET_METADATA`; capture AVC if it still fails |
| R3 | `LD_PRELOAD` ignored by vold | Low | Fix ineffective | Check `SHIM_LOADED`; fallback vold wrapper only after evidence |
| R4 | SELinux denial on key operations | Medium | FBE fails | Capture AVC; targeted rule only after evidence |
| R5 | Keyring not inherited by zygote/apps | Low | FBE mounts but app I/O fails | Verify session keyring inheritance; do not override in services |
| R6 | Wrapper breaks init or ueventd | Low | Dead boot / no device nodes | Wrapper only does keyring syscalls + exec; ueventd passthrough; restore previous system image |
| R7 | `/data` key mismatch from prior failed boot | Medium | Cannot decrypt existing data | Document; format only with explicit approval |
| R8 | Metadata encryption / wrapped keys engage | Low | Different key path | Check fstab and vold logs; do not assume |
| R9 | `ioctl` interposition breaks non-fscrypt vold calls | Low | vold misbehaves | Raw passthrough for all non-target requests |
| R10 | Wrapper keyring creation fails silently | Low | Shim falls back; FBE may still fail | Wrapper logs every result; shim creates keyring on `ENOKEY` |
| R11 | Recovery now requires reflashing the system image, not just boot | Medium | Harder recovery | Document a verified stock-system/previous-ZIP restore before any flash |

---

## 10. What this design does not do

- It does not change the FBE policy version, fstab, `/data` format, or key derivation.
- It does not touch `/keydata`, `/keyrefuge`, or Samsung ODE.
- It does not fix or bypass Keymaster/Keystore2; it assumes the existing barrier and KM4 path are healthy enough to retrieve the key. If step 2 of `fscrypt_initialize_systemwide_keys()` fails, that is a separate issue.
- It does not guarantee that the donor system's framework will boot after `/data` mounts. Display, audio, RIL, and other hardware work remain separate phases.
- It does not authorize flashing, formatting, or publishing.

---

## 11. Rollback and safety

- The wrapper, `init.real`, the shim, and the `vold.rc` change all live in the system image. Reflashing the stock A11 system (or the previous known-good port ZIP) restores them.
- The boot image is untouched. Reflashing the stock `boot.img` is still a valid diagnostic/rollback step for boot issues, but it will not remove this fix.
- A failed wrapper should not loop: it execs the real init even when keyring creation fails. If it cannot exec, it halts and logs; the device must be recovered with the previous system image.
- Never flash without:
  1. the exact device CSC/firmware,
  2. a verified stock `A115FXXS6CWK2` (or matching) restore package,
  3. confirmed bootloader unlock/AVB state,
  4. a verified previous port ZIP or stock system image,
  5. explicit user approval.
- Never format `/data` as a diagnostic shortcut. It destroys user data and does not prove the fix.

---

## 12. Open questions

1. Does the A11 stock first-stage `/init` (Android 12) exec `/system/bin/init selinux_setup` before loading policy, as AOSP 12 does? The donor init's `selinux_setup` path is assumed to match. If the stock first stage instead loads policy itself, the wrapper would create keyrings in `init` context; the existing `vold init`/`init init` key rules cover that, but the "pre-policy" claim would be wrong. Verify from the stock ramdisk/source or a device log.
2. Is `/system/bin/init` a regular ELF64 AArch64 binary in the donor system? (Confirmed for the donor `system` image used here; the mod re-checks at build time.)
3. Does the donor init exec `/system/bin/init second_stage` after policy load? (AOSP 12 does; verify on the donor init if a device log shows a second exec.)
4. Does vold run with `AT_SECURE` unset? (Expected; vold is root-started, not setuid.)
5. Is `/system/lib64` on vold's linker namespace search path? (Expected default namespace.)
6. ~~Does the final assembled policy preserve the A11 vendor `vold_31_0 kernel_31_0 (key (write search))` rule?~~ **Closed statically (2026-11-16).** The donor `plat_sepolicy.cil` retains the legacy key allows, the donor `mapping/31.0.cil` maps the A11 vendor `_31_0` types, and the A11 `vendor_sepolicy.cil` retains `(allow vold_31_0 kernel_31_0 (key (write search)))` and `(allow vold_31_0 init_31_0 (key (write)))`; the donor `system_ext_sepolicy.cil` has no key rules and the SELinux patch only removes named types. The assembled binary policy is still not built, so verify the runtime keyring label/ordering on device (AVC log) or by inspecting a built `precompiled_sepolicy`.
7. Does `request_key()` from zygote/app processes find the `fscrypt` keyring? (Depends on session keyring inheritance; verify with markers/AVC logs.)
8. Does the A11 kernel have `CONFIG_KEYS`, `CONFIG_ENCRYPTED_KEYS`, and the `logon` key type enabled? (Legacy fscrypt requires them; verify in `/proc/config.gz` or the kernel build config.)

None of these questions may be answered by assumption in the status docs; each needs a build artifact, a device log, or a source check.

---

## 13. References

Local evidence (paths under `C:\Users\claymi5433\AppData\Local\Temp\opencode` unless noted):

- `donor_vold`, `donor_vold_dynsyms.txt`, `donor_init`, `donor_init_strings*.txt`
- `keyutil12.cpp`, `keyutil14.cpp`, `a12_init2.cpp`, `core12src/init/fscrypt_init_extensions.cpp`, `core12src/init/builtins.cpp`
- `android15_fscrypt_uapi.h`
- `a11q_keyinfo.c`, `a11q_fscrypt_h2.txt`
- `plat_sepolicy.cil`, `plat_pub_versioned.cil`, `vendor_sepolicy.cil`, `vendor_sepolicy_version.txt`
- `k49_key.c`, `k49_perm.c`, `k49_keyring.c`, `k49_process_keys.c`
- `a11q_build/change_fs_type.sh`, `a11q_build/common_utils.sh`, `a11q_build/build_flashable_zip.sh`, `a11q_build/external_make.sh`, `a11q_build/build.yml`

AOSP references:

- `system/core/init/init.cpp` (Android 12): session keyring at line 768.
- `system/core/init/fscrypt_init_extensions.cpp`: `FscryptInstallKeyring()`.
- `system/core/init/builtins.cpp`: `FscryptInstallKeyring()` on `FS_MGR_MNTALL_DEV_FILE_ENCRYPTED`.
- `system/vold/KeyUtil.cpp` (Android 12/14/15): `isFsKeyringSupportedImpl`, `installKeyLegacy`, `evictKeyLegacy`, `fscryptKeyring`.

Kernel references:

- `security/keys/key.c`: default permissions at `key_create_or_update`.
- `security/keys/permission.c`: possession bits and LSM hook.
- `security/keys/keyring.c`: possession propagation in `keyring_search_aux`.
- `security/keys/process_keys.c`: anonymous session keyring permissions.
- `fs/crypto/keyinfo.c` (A11): `find_and_lock_process_key` / `request_key`.

---

## 14. Change log

- 2026-11-16: Initial design. Records the two-part root cause, exact UAPI layouts, legacy key names/payload, wrapper/shim ABI, SELinux rules, build integration, verification matrix, risks, and open questions. Not implemented, not built, not device-tested. R0.
- 2026-11-16 (correction): Verified `ioctl` interposition from `.rela.plt` (slot `0x146F30`, `R_AARCH64_JUMP_SLOT`, symbol 105 = `ioctl`); confirmed kernel 4.9 possession permissions are additive and propagate through nested keyring search, so no `KEYCTL_SETPERM` is required; confirmed AOSP `FscryptInstallKeyring()` searches first and donor init retains neither the session keyring nor the `fscrypt` keyring. Clarified that `plat_sepolicy.cil` is the A11 target policy (API 31, from `system_big3.dat`), not donor policy. Aligned the status-doc wording on the defensive `NULL` probe, non-idempotent REMOVE, raw `SYS_ioctl`, and system-image patch set. Still not implemented, built, or device-tested. R0.
- 2026-11-16 (design change): Moved the wrapper from the boot ramdisk `/init` to `/system/bin/init`, because AOSP 12 first-stage init executes `/system/bin/init selinux_setup` before policy load. The donor init moves to `/system/bin/init.real`; both files are labelled `init_exec` because the wrapper is re-executed for `second_stage`. The wrapper preserves `argv[0]` and passes `ueventd` through. The boot image is no longer modified, and `magiskboot` is no longer part of the fix. Source implementation exists on `manus/a115f-legacy-fscrypt`; it is not yet compiled, ROM-built, or device-tested at this point. Superseded by the build entry below. R0.
- 2026-11-16 (build): Compiled both targets locally with Zig 0.17.1 (clang 22.1.8 + lld) using the project flags, no warnings. Wrapper: 2624-byte ELF64 `ET_EXEC`, entry `0x1010848`, no `PT_INTERP`, no `.dynamic`, 0 `DT_NEEDED`. Shim: 6280-byte ELF64 `ET_DYN`, 0 `DT_NEEDED`, imports exactly `__android_log_print` and `__errno`, exports `ioctl`, no memory exports. The first host-test run crashed with `0xC0000005` because `a115f_keyctl_fn` used 32-bit `unsigned long` on Windows LLP64 and truncated a fake-keyring string pointer; switching the callback arguments to `uintptr_t` fixed the tests and left the aarch64 binaries byte-identical (same SHA-256 before and after). Host tests now print `all legacy fscrypt host tests passed`. Pushed as `325dcd2` (12 files) and `48438e6` (ABI fix); the CI workflow remains local-only (`local/a115f-legacy-fscrypt-ci-workflow@5551184`). R0; no ROM build or device test.
- 2026-11-16 (CI blocker): `POST .../actions/workflows/build.yml/dispatches` returns HTTP 404 and the workflow list is empty (`total_count: 0`), although the default branch `oneui7-a11q-boot` contains `.github/workflows/build.yml` (blob `ffc26cf3…`, last modified 2026-10-21), Actions is enabled, and the credential has `repo` scope. The repository has no registered workflows; this requires user action (enable/register workflows in the GitHub UI or provide a `workflow`-scoped credential). Do not rewrite the default branch. R0.
- 2026-11-16 (CI registration and first run): Re-asserted `PUT /repos/zinerom48-tech/ZineROM-a11q/actions/permissions` (`enabled: true`, `allowed_actions: all`, `sha_pinning_required: false`); HTTP 204. `GET /actions/workflows` then reported `total_count: 2` (`build.yml` id `420352259`, `publish-public.yml` id `420352261`, both active). Dispatched `build.yml` at `manus/a115f-legacy-fscrypt@48438e6` with `audit_only=false`; run `43909580926`. The test step passed all 69 tests (`Ran 69 tests in 0.214s`, `OK`), including the 7 patcher tests. The build job then failed before `make_rom.sh` in `Set up build environment` with `E: Unable to locate package linux-modules-extra-7.0.0-1014-aws` (exit 100); the package does not exist in any Ubuntu suite/architecture and all 15 branches carry the same line. The mod’s compile/host-test/`readelf` checks did not run. The minimal fix is to remove `linux-modules-extra-$(uname -r)` from the apt install list; it is pending because the token scopes are `gist, read:org, repo` (no `workflow`). No release, flash, or `/data` operation performed. R0.
- 2026-11-16 (CI compile evidence): Added `tests/test_legacy_fscrypt_build.py` at `5651d76`; run `43913667988` passed all 76 tests (`OK`, 3.028 s) with no skips. Ubuntu clang 18 compiled the wrapper and shim, the host tests compiled and ran, and `readelf` verified the dynamic-symbol contract. CI hashes: wrapper `530e2b3c…` (5296 bytes), shim `1faab403…` (6880 bytes). The run still failed at `Set up build environment`, now on Azure kernel `7.0.0-1017-azure`, proving the workflow's `$(uname -r)` bug is pool-dependent; the in-ROM compile/install inside `make_rom.sh` remains unexercised. No release, flash, or `/data` operation performed. R0.
- 2026-11-16 (CI mod-install evidence): Replaced the mod's `xxd` ELF-header reads with coreutils `od` and added `tests/test_legacy_fscrypt_mod_install.py` at `74c8198`. Run `43921516470` passed all 80 tests (`OK`, 6.808 s) with no skips; the four mod-install tests exercise the real `customize.sh`, `SET_METADATA`, and `vold.rc` patcher against a synthetic donor tree, including idempotence and both fail-closed layouts. Step 8 still failed on `linux-modules-extra-7.0.0-1017-azure`; the surrounding `make_rom.sh` flow remains unexercised. No release, flash, or `/data` operation performed. R0.
- 2026-11-16 (CI driver-path evidence): Extended `tests/test_legacy_fscrypt_mod_install.py` with a fifth test that invokes the real `scripts/internal/apply_modules.sh` driver with a copied mod and a sentinel system tree, proving the mod runs and `SKIPUNZIP=1` suppresses the automatic tree copy. Commit `3aa20d4` is pushed to `manus/a115f-legacy-fscrypt`. Run `43925890423` passed all 81 tests (`OK`, 17.249 s) with no skips; step 8 still failed on `linux-modules-extra-7.0.0-1017-azure`. The driver-originated invocation is now covered; the surrounding work-dir/image/ZIP flow remains unexercised. No release, flash, or `/data` operation performed. R0.
- 2026-11-16 (policy-input audit): Extracted the donor policy inputs from `system_full.dat` with a read-only ext4 reader, because no `7z`/`debugfs` is available. The image is One UI 7 / Android 15 (`AP3A.240905.015.A2`, SDK 35). Donor `plat_sepolicy.cil` (2,237,691 bytes, SHA-256 `febc700c…`) retains `vold init`, `vold vold`, `init init`, and `domain kernel` key allows; donor `mapping/31.0.cil` (SHA-256 `ff34465a…`) maps the A11 vendor `_31_0` types; A11 `vendor_sepolicy.cil` (SHA-256 `08e2f3a2…`) supplies the versioned `kernel`/`init` key writes; donor `system_ext_sepolicy.cil` (SHA-256 `a0a7cf4e…`) has no key rules and the SELinux patch only deletes named types. Corrected F13's provenance: the file is donor policy, not A11 target policy. Closed open question 6 statically; the assembled policy and runtime keyring label/ordering remain unverified. R0.
