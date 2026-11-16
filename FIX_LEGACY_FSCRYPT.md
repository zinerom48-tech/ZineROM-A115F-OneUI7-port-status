# Fix Design — Legacy fscrypt for the A115F One UI 7 Port

Status: **design complete, not implemented, not built, not device-tested**. Readiness remains **R0**.
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
| F10 | `/system/bin/ueventd` is a build symlink to `init`; `service ueventd` uses `seclabel u:r:ueventd:s0`. Replacing `/system/bin/init` is unsafe. | `init/Android.bp` symlinks; A11/donor rc. |
| F11 | A helper cannot inject a keyring into PID 1: `keyctl_session_to_parent()` rejects `parent->pid <= 1`. PID 1 must create the keyrings itself. | Kernel `security/keys/process_keys.c`. |
| F12 | The A11 4.9 key permission model uses **possession bits additively**, and possession propagates through nested keyring search. Default key permissions are `KEY_POS_VIEW|KEY_POS_SEARCH|KEY_POS_LINK|KEY_POS_SETATTR|KEY_USR_VIEW`, plus `KEY_POS_READ` and `KEY_POS_WRITE` for keyrings. No `KEY_OTH_SEARCH` is needed. | `k49_key.c:889-900`; `k49_perm.c:30-77`; `k49_keyring.c:866-873`; `k49_process_keys.c:223-252`. |
| F13 | The A11 platform policy grants `domain` search on `init`, `vold`, and `kernel`; `init` and `vold` write/search/setattr on their own keyrings. The A11 vendor policy (`VE=SEPF_SM-A115F_12_0001`) adds `(allow vold_31_0 kernel_31_0 (key (write search)))`, `(allow domain kernel_31_0 (key (search)))`, and `(allow init_31_0 kernel_31_0 (key (search)))`. | `plat_sepolicy.cil`, `plat_pub_versioned.cil`, `vendor_sepolicy.cil`. |
| F14 | The A11 v1 descriptor is 8 bytes derived by double SHA-512 of the key; the legacy name is `prefix + ":" + 16 lowercase hex chars`. The kernel compares 8 descriptor bytes. | `keyutil12.cpp:108-145`; `a11q_keyinfo.c:85-95`. |
| F15 | The legacy payload is `struct fscrypt_key { u32 mode; u8 raw[64]; u32 size; }`, 72 bytes, with `mode=0`, `raw` = 64-byte key, `size=64`. | AOSP `fillKey` (`keyutil12.cpp:121-131`); kernel `fscrypt` payload checks. |
| F16 | The shim's exact UAPI layouts are: `fscrypt_key_specifier` 40 bytes (type at 0, `u.descriptor[8]` at 8); `fscrypt_add_key_arg` header 80 bytes (`raw_size` at 40, `key_id` at 44, `__flags` at 76, `raw` at 80); `fscrypt_remove_key_arg` 64 bytes (`removal_status_flags` at 40); `fscrypt_get_key_status_arg` 128 bytes (`status` at 64, `status_flags` at 68, `user_count` at 72). | `android15_fscrypt_uapi.h:63-106`; ioctl sizes `0x50`/`0x40`/`0x80`. |
| F17 | The ADD/REMOVE/GET_STATUS call sites use the imported `ioctl` through a normal PLT entry, so `LD_PRELOAD` interposition is viable. | F4/F5. |
| F18 | `create_work_dir.sh` copies and unsigns the stock A11 `boot.img`; `build_flashable_zip.sh` copies `$WORK_DIR/kernel/*.img` unchanged. There is no existing boot repack step. `change_fs_type.sh` proves the unpack/repack flow. | Build scripts; checkpoint. |
| F19 | The final ZIP builds each partition image with `file_context-$PARTITION` and `fs_config-$PARTITION`; new files need `SET_METADATA`. | `build_flashable_zip.sh:655-663`; `SET_METADATA()` at `common_utils.sh:538`. |
| F20 | `magiskboot` is downloaded as the **latest** Magisk release by `external/make.sh`; its version is not pinned. CI compiles with Ubuntu 24.04 clang/lld. | `external_make.sh:249-268`; `build.yml:74-80`. |

Facts F12-F14 close the earlier keyring-permission and descriptor-format questions. They do **not** prove runtime behavior on the device.

---

## 3. Architecture

Two components, both required:

```
kernel
  └─ exec /init                         (new static arm64 wrapper, pre-policy)
       ├─ keyctl(GET_KEYRING_ID, SESSION, create=1)
       ├─ keyctl(SEARCH, SESSION, keyring, "fscrypt")
       ├─ add_key("keyring", "fscrypt", NULL, 0, SESSION)   (if absent)
       └─ execve("/init.real", argv, envp)
            └─ real A11 first-stage init
                 └─ exec /system/bin/init second_stage       (donor init, unchanged)
                      └─ service vold  (LD_PRELOAD=liba115f_fscrypt_shim.so)
                           └─ ioctl(ADD/REMOVE/GET_STATUS)
                                └─ shim: real ioctl first; on ENOTTY translate to
                                   add_key/keyctl legacy logon keys
```

Why a ramdisk wrapper instead of changes to `/system`:
- The session keyring must belong to PID 1 before any service or app forks.
- The wrapper runs before SELinux policy load, so its keyring creation is not subject to a domain transition and the keyrings are labelled `kernel` (the label the A11 vendor policy already allows).
- `/system/bin/init` must not be replaced because `/system/bin/ueventd` is a symlink to it and runs under `u:r:ueventd:s0`.

Why a vold shim instead of replacing vold:
- Donor `vold` is a large, Samsung-patched binary; rebuilding it is not practical.
- The failure is localized to three ioctl requests. A 64-bit `LD_PRELOAD` shim is the smallest change and preserves every other vold behavior.
- Interposition is proven possible by F17 (imported `ioctl`, normal PLT).

---

## 4. Part A — ramdisk `/init` wrapper

### 4.1 Behavior

The wrapper is a freestanding, statically linked AArch64 executable. It performs only these operations, then execs the real first-stage init:

1. Create PID 1's session keyring:
   `keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 1)`.
2. Look up the named `fscrypt` keyring:
   `keyctl(KEYCTL_SEARCH, KEY_SPEC_SESSION_KEYRING, "keyring", "fscrypt", 0)`.
3. If absent, create it:
   `add_key("keyring", "fscrypt", NULL, 0, KEY_SPEC_SESSION_KEYRING)`.
4. Log one marker per result to `/dev/kmsg`.
5. `execve("/init.real", argv, envp)`.

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
A115F_FSCRYPT_WRAPPER_ENTER
A115F_FSCRYPT_WRAPPER_SESSION_OK <serial>
A115F_FSCRYPT_WRAPPER_SESSION_FAIL <errno>
A115F_FSCRYPT_WRAPPER_FSCRYPT_OK <serial>
A115F_FSCRYPT_WRAPPER_FSCRYPT_FAIL <errno>
A115F_FSCRYPT_WRAPPER_EXEC /init.real
A115F_FSCRYPT_WRAPPER_EXEC_FAIL <errno>
```

### 4.4 Failure behavior

- Keyring failure: log, continue to exec. A missing keyring produces a diagnosable shim failure rather than a dead boot.
- `execve` failure: log `EXEC_FAIL`, then halt in a `wfi` loop. Do **not** reboot automatically; a boot loop destroys the evidence and may trigger Rescue Party. The only recovery is reflashing the stock `boot.img` (see §11).

### 4.5 Compilation

Build with the CI clang, freestanding, no libc:

```
clang --target=aarch64-linux-gnu -nostdlib -static -no-pie \
  -ffreestanding -fno-builtin -fno-stack-protector \
  -Wl,-e,_start -Wl,--build-id=none \
  -o init_wrapper init_wrapper.c
```

The source must provide `_start`, `memset`, `memcpy`, and the raw syscall stubs. Do not link `libc`, `libdl`, or `libkeyutils`. Verify:

```
file init_wrapper            # ELF 64-bit LSB executable, ARM aarch64, statically linked
readelf -l init_wrapper      # no PT_INTERP
readelf -d init_wrapper      # no DT_NEEDED
```

`-no-pie` keeps the binary `ET_EXEC`; the kernel can exec either `ET_EXEC` or `ET_DYN`, but `ET_EXEC` is simpler to inspect.

### 4.6 Packaging

The wrapper replaces `/init` in the **boot image ramdisk**, not in `/system`:

1. `cp -a "$WORK_DIR/kernel/boot.img" "$TMP/boot.img"`.
2. `magiskboot unpack -h "$TMP/boot.img"`.
3. Require `ramdisk.cpio` and `magiskboot cpio ramdisk.cpio "exists init"`.
4. Refuse to proceed if `init.real` already exists (idempotence guard).
5. `magiskboot cpio ramdisk.cpio "mv init init.real"`.
6. `magiskboot cpio ramdisk.cpio "add 0750 init <compiled wrapper>"`.
7. Verify both entries exist.
8. `magiskboot repack "$TMP/boot.img" "$TMP/boot_new.img"`.
9. Replace `$WORK_DIR/kernel/boot.img` only after every step succeeds.

If `vendor_boot.img` exists, the mod must **fail the build** with a clear message: the first-stage ramdisk location and `/init` path have not been verified for that layout. Do not guess.

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

Write to fd 2 (init redirects service stderr to kmsg by default) or `/dev/kmsg`:

```
A115F_FSCRYPT_SHIM_LOADED
A115F_FSCRYPT_SHIM_ADD_PROBE_ENOTTY
A115F_FSCRYPT_SHIM_ADD_OK <name>
A115F_FSCRYPT_SHIM_ADD_FAIL <errno>
A115F_FSCRYPT_SHIM_REMOVE_OK
A115F_FSCRYPT_SHIM_REMOVE_FAIL <errno>
A115F_FSCRYPT_SHIM_STATUS_PRESENT
A115F_FSCRYPT_SHIM_STATUS_ABSENT
A115F_FSCRYPT_SHIM_UNEXPECTED <request>
```

### 5.5 Compilation

```
clang --target=aarch64-linux-gnu -shared -fPIC -nostdlib \
  -ffreestanding -fno-builtin -fno-stack-protector \
  -Wl,-soname,liba115f_fscrypt_shim.so \
  -o liba115f_fscrypt_shim.so fscrypt_legacy_shim.c
```

The shim declares `extern int *__errno(void);` and sets `*__errno() = err` on failure. Bionic exports `__errno`; vold already links libc. Do not link `libdl` or `libkeyutils`.

A constructor writes `A115F_FSCRYPT_SHIM_LOADED` and performs a fallback keyring check:

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
| PID 1 session keyring | wrapper, before policy load | `kernel` |
| Named `fscrypt` keyring | wrapper, before policy load | `kernel` |
| Logon keys `ext4:`/`f2fs:`/`fscrypt:` | vold (shim), after policy load | `vold` |

If the shim creates the `fscrypt` keyring as a fallback, that keyring is labelled `vold`; the rules below cover both cases.

### 6.2 Required operations and rules

| Operation | Subject | Object | Permission | Rule source |
|---|---|---|---|---|
| create/link `fscrypt` keyring | wrapper | session keyring (`kernel`) | pre-policy; no check | — |
| search session keyring | vold | `kernel` | `search` | vendor `vold_31_0 kernel_31_0 (key (write search))` |
| add logon key | vold | `fscrypt` (`kernel`) | `write` | same vendor rule |
| search `fscrypt` keyring | vold | `kernel` | `search` | same vendor rule |
| create logon keys | vold | new key (`vold`) | no separate check on `add_key` | — |
| search logon key | vold / any process | `vold` | `search` | plat `(allow domain vold (key (search)))` |
| search `fscrypt` keyring | any process | `kernel` | `search` | plat `(allow domain kernel (key (search)))` / vendor `(allow domain kernel_31_0 (key (search)))` |
| unlink logon key | vold | `fscrypt` (`kernel` or `vold`) | `write` | vendor `vold_31_0 kernel_31_0 (key (write search))` or `vold_31_0 self (key (... write ...))` |
| view/read logon key | kernel `request_key` on behalf of any process | `vold` | `view` via possession | `KEY_POS_VIEW` default permission; plat `(allow domain vold (key (search)))` |

### 6.3 Why no `KEYCTL_SETPERM` is needed

Kernel 4.9 `key_task_permission()` uses the possession bits (`key->perm >> 24`) **additively** whenever `is_key_possessed(key_ref)` is true. `keyring_search_aux()` sets `ctx->possessed = is_key_possessed(keyring_ref)` and propagates it to every nested key it returns. The default permissions from `key_create_or_update()` include `KEY_POS_VIEW|KEY_POS_SEARCH|KEY_POS_LINK|KEY_POS_SETATTR`, and keyrings also get `KEY_POS_READ|KEY_POS_WRITE`.

Therefore:
- A process that inherits PID 1's session keyring possesses the `fscrypt` keyring and its logon keys.
- Search and view succeed through the possession bits; `KEY_OTH_SEARCH` is not required.
- vold's `add_key`/`keyctl_unlink` needs write on the keyring; `KEY_POS_WRITE` is present for keyrings.

This matches AOSP Android 9-12 legacy behavior and the existing A11 policy. Do not add `KEYCTL_SETPERM` unless a device log shows `EACCES` from `add_key`/`request_key` with the possession path already active.

### 6.4 Residual SELinux risks

- If the shim is loaded but `add_key` returns `EACCES`, capture the AVC denial. Do not assume the policy is wrong; the most likely cause is a keyring label mismatch.
- The `/init.real` file does not need `init_exec` for the pre-policy `execve`, because SELinux policy is not loaded yet. Add a file-contexts entry only if a future design re-execs `/init.real` after policy load.
- The shim is a `/system/lib64` file labelled `system_lib_file`. If vold is denied `read`/`map`/`execute`, capture the denial before adding any rule.

---

## 7. Build integration

### 7.1 Branch and files

Base branch: `manus/a115f-vendor-fingerprint` at `23f8998f0200969f5748e9eb012d1dc52037a054`. Do **not** build from `build-once` or `preflight`.

New files:

```
prebuilts/a115f_legacy_fscrypt/
  build.sh
  src/init_wrapper.c
  src/fscrypt_legacy_shim.c
  src/syscalls.h
  tests/test_uapi.c
  module.prop
unica/mods/zz_a115f_legacy_fscrypt/
  customize.sh
  module.prop
```

Compiled outputs (CI, not committed unless reproducibility requires it):

```
prebuilts/a115f_legacy_fscrypt/out/init_wrapper
prebuilts/a115f_legacy_fscrypt/out/liba115f_fscrypt_shim.so
```

### 7.2 CI compile step

Add a step before `Build ROM` in `.github/workflows/build.yml`:

```yaml
- name: Build A115F legacy-fscrypt components
  run: ./prebuilts/a115f_legacy_fscrypt/build.sh
```

`build.sh` uses `clang`/`ld.lld` from the existing build environment and fails on any compile or static-check error. Do not modify `publish-public.yml`.

### 7.3 Mod execution order

`make_rom.sh` order:

1. `create_work_dir.sh` — copies and unsigns the stock `boot.img`.
2. patches.
3. `apply_modules.sh "$SRC_DIR/unica/patches"`.
4. `apply_modules.sh "$SRC_DIR/target/$TARGET_CODENAME/patches"`.
5. `apply_modules.sh "$SRC_DIR/unica/mods"` — this runs `zz_a115f_legacy_fscrypt`.
6. APK build.
7. `build_flashable_zip.sh` — copies `$WORK_DIR/kernel/*.img` unchanged.

The mod must therefore patch `$WORK_DIR/kernel/boot.img` in step 5.

### 7.4 Mod behavior

`unica/mods/zz_a115f_legacy_fscrypt/customize.sh` must:

1. Skip unless `TARGET_CODENAME == a115f`.
2. Require `$WORK_DIR/kernel/boot.img`.
3. Fail if `$WORK_DIR/kernel/vendor_boot.img` exists (layout not verified).
4. Unpack, verify `/init`, rename to `/init.real`, add the wrapper, repack, replace the image.
5. Copy the shim to `$WORK_DIR/system/system/lib64/liba115f_fscrypt_shim.so`.
6. `SET_METADATA "system" "system/lib64/liba115f_fscrypt_shim.so" 0 0 0644 "u:object_r:system_lib_file:s0"`.
7. Patch `$WORK_DIR/system/system/etc/init/vold.rc` to add the `setenv` line exactly once, inside the `service vold` block.
8. Verify the final `vold.rc` contains `service vold`, `setenv LD_PRELOAD`, and the shim path.
9. Verify the repacked boot image still reports an Android boot header and contains both `init` and `init.real`.
10. Clean up temporary files; never leave a temp directory under `$WORK_DIR` (the ZIP builder scans `$WORK_DIR` top-level directories).

Use `SKIPUNZIP=1` in `customize.sh` so `APPLY_MODULE` does not auto-copy a `system/` tree.

### 7.5 Required failure conditions

The build must fail loudly if:

- `boot.img` is missing or unpack fails.
- `ramdisk.cpio` is missing.
- `/init` is absent or `init.real` already exists.
- `magiskboot repack` fails or does not produce the new image.
- `vold.rc` is missing or has no `service vold` block.
- The shim or wrapper binary is missing, wrong architecture, or dynamically linked.
- `SET_METADATA` fails.

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
- Verify shim: ELF64 AArch64, `ET_DYN`, exports `ioctl`, no `DT_NEEDED` on `libdl`/`libkeyutils`.
- Verify the mod's boot patcher against a **local synthetic cpio** (no device image) to catch `magiskboot cpio` syntax errors.
- Add a CI check that greps the final `vold.rc` and boot image contents.

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
| `WRAPPER_ENTER` missing | Wrapper not in the boot ramdisk or kernel did not exec it | Verify boot image contents and header; reflash stock boot |
| `SESSION_FAIL` / `FSCRYPT_FAIL` | Keyring syscall failed | Check `CONFIG_KEYS`, seccomp, kernel log |
| `SHIM_LOADED` missing | `LD_PRELOAD` ignored or rc patch absent | Verify `vold.rc`, linker namespace, `AT_SECURE` |
| `ADD_PROBE_ENOTTY` then no `ADD_OK` | vold probed but did not install | Check the following vold error; capture full log |
| `ADD_OK` ×3 but `/data` fails | `request_key` cannot find the keys | Check session keyring inheritance, keyring label, AVC denials, `CONFIG_KEYS` |
| `ADD_FAIL EACCES` | SELinux key permission or label mismatch | Capture AVC; compare labels; do not add a blind rule |
| `ADD_FAIL ENOKEY` | `fscrypt` keyring missing | Check wrapper/shim keyring creation |
| `Unexpected ioctl` | New request number (e.g., ALL_USERS) | Add handling only after confirming the caller and semantics |
| No markers, boot loop | Wrapper or boot image problem | Reflash stock boot; do not iterate blindly |

---

## 9. Risks and mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Stock A11 boot image has no `/init` or uses a different layout | Medium | Build fails | Preflight check; fail closed |
| R2 | `magiskboot` version changes and `cpio mv/add` syntax differs | Medium | Build fails | Pin/verify in CI; fallback to `cpio` extraction |
| R3 | `LD_PRELOAD` ignored by vold | Low | Fix ineffective | Check `SHIM_LOADED`; fallback vold wrapper |
| R4 | SELinux denial on key operations | Medium | FBE fails | Capture AVC; targeted rule only after evidence |
| R5 | Keyring not inherited by zygote/apps | Low | FBE mounts but app I/O fails | Verify session keyring inheritance; do not override in services |
| R6 | Wrapper hangs or breaks first-stage init | Low | Dead boot | Wrapper only does keyring syscalls + exec; stock boot restore |
| R7 | `/data` key mismatch from prior failed boot | Medium | Cannot decrypt existing data | Document; format only with explicit approval |
| R8 | Metadata encryption / wrapped keys engage | Low | Different key path | Check fstab and vold logs; do not assume |
| R9 | `ioctl` interposition breaks non-fscrypt vold calls | Low | vold misbehaves | Raw passthrough for all non-target requests |
| R10 | Build uses unpinned `magiskboot` | Medium | Non-reproducible build | Record version/hash; pin in a follow-up |

---

## 10. What this design does not do

- It does not change the FBE policy version, fstab, `/data` format, or key derivation.
- It does not touch `/keydata`, `/keyrefuge`, or Samsung ODE.
- It does not fix or bypass Keymaster/Keystore2; it assumes the existing barrier and KM4 path are healthy enough to retrieve the key. If step 2 of `fscrypt_initialize_systemwide_keys()` fails, that is a separate issue.
- It does not guarantee that the donor system's framework will boot after `/data` mounts. Display, audio, RIL, and other hardware work remain separate phases.
- It does not authorize flashing, formatting, or publishing.

---

## 11. Rollback and safety

- The wrapper lives only in the boot image ramdisk. Reflashing the stock A11 `boot.img` restores the original first-stage init.
- The shim and `vold.rc` change live in the system image. Reflashing the stock system (or the previous port ZIP) restores them.
- A failed wrapper should not loop: it execs the real init even when keyring creation fails. If it cannot exec, it halts and logs; the device must be recovered with the stock boot image.
- Never flash without:
  1. the exact device CSC/firmware,
  2. a verified stock `A115FXXS6CWK2` (or matching) restore package,
  3. confirmed bootloader unlock/AVB state,
  4. explicit user approval.
- Never format `/data` as a diagnostic shortcut. It destroys user data and does not prove the fix.

---

## 12. Open questions

1. Does the stock A11 `boot.img` contain `/init` at the ramdisk root, and what compression/header version does it use? (F18 says no repack step exists; `change_fs_type.sh` implies `ramdisk.cpio`.)
2. Is the stock `/init` a real ELF or a symlink? (Must be a real binary for a pre-`/system` exec.)
3. Does vold run with `AT_SECURE` unset? (Expected; vold is root-started, not setuid.)
4. Is `/system/lib64` on vold's linker namespace search path? (Expected default namespace.)
5. Does the final assembled policy preserve the A11 vendor `vold_31_0 kernel_31_0 (key (write search))` rule? (Must verify on device or by inspecting the built `vendor_sepolicy`/precompiled policy.)
6. Does `request_key()` from zygote/app processes find the `fscrypt` keyring? (Depends on session keyring inheritance; verify with markers/AVC logs.)
7. Does the A11 kernel have `CONFIG_KEYS`, `CONFIG_ENCRYPTED_KEYS`, and the `logon` key type enabled? (Legacy fscrypt requires them; verify in `/proc/config.gz` or the kernel build config.)

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
- 2026-11-16 (correction): Verified `ioctl` interposition from `.rela.plt` (slot `0x146F30`, `R_AARCH64_JUMP_SLOT`, symbol 105 = `ioctl`); confirmed kernel 4.9 possession permissions are additive and propagate through nested keyring search, so no `KEYCTL_SETPERM` is required; confirmed AOSP `FscryptInstallKeyring()` searches first and donor init retains neither the session keyring nor the `fscrypt` keyring. Clarified that `plat_sepolicy.cil` is the A11 target policy (API 31, from `system_big3.dat`), not donor policy. Aligned the status-doc wording on the defensive `NULL` probe, non-idempotent REMOVE, raw `SYS_ioctl`, pre-policy `/init.real` label, and system-image patch set. Still not implemented, built, or device-tested. R0.
