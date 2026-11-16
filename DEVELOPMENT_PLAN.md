# Development Plan — ZineROM One UI 7 A115F Port

Status: **active, readiness R0**.
Last updated: 2026-11-16.
Scope: Phases 1-5 from repository audit through build, device validation, and the final report for the Galaxy A11 (`SM-A115F`, `a11q`) running the A05s One UI 7 donor system.

This plan is the persistent work queue. Update task status here and in `ISSUE_REGISTER.md`; do not create a second, contradictory queue.

---

## 1. Mission, constraints, and definitions

### 1.1 Mission

Produce a genuinely bootable, usable One UI 7 ROM for `SM-A115F`, with verified recovery, without fabricating test results or claiming stability without device evidence.

### 1.2 Hard constraints

- No installation, flashing, partition change, or `/data` format without a documented recovery path and explicit user approval.
- No release publication, `main` overwrite, or destructive repository operation without approval.
- No optional feature polish while a boot-critical failure is open.
- Preserve A11 boot/kernel/DTB/vendor HALs/modem/firmware; do not blindly reuse A05s hardware components.
- Record every important modification; do not delete user files or existing source branches.
- Separate **confirmed facts**, **hypotheses**, and **untested assumptions** in every report.
- Prefer repository resources over new large downloads.

### 1.3 Readiness scale

| Level | Meaning |
|---|---|
| **R0** | Not bootable or not proven; no device evidence. Current level. |
| **R1** | Boots to the Android framework/lockscreen on the target device, with a verified recovery path. |
| **R2** | Setup Wizard completes; `/data` FBE works across reboot; core display/touch/storage stable. |
| **R3** | Core hardware matrix (audio, RIL, Wi-Fi, Bluetooth, camera, sensors, fingerprint, charging) passes with documented defects. |
| **R4** | Stability/regression target met; release candidate approved for publication. |

No level may be claimed from static analysis, package CRC checks, or a successful build alone.

### 1.4 Milestone mapping

The original M0-M9 milestones map to the five phases:

| Milestone | Phase |
|---|---|
| M0 repository/hardware audit; M1 compatibility/build environment | Phase 1 |
| M2 configuration/prerequisites; M3 images compiled; M5 installation/early boot; M6 userspace/boot animation; M7 Setup Wizard/home; M8 core hardware | Phase 2 |
| M4 installer/package validation; QA gates | Phase 3 |
| M5-M8 failure investigation; M9 stability/regression | Phase 4 |
| Final build, delivery, report | Phase 5 |

---

## 2. Current baseline (2026-11-16)

- Source: `zinerom48-tech/ZineROM-a11q`. The barrier baseline is `manus/a115f-vendor-fingerprint` at `23f8998f0200969f5748e9eb012d1dc52037a054`; the legacy-fscrypt work is on `manus/a115f-legacy-fscrypt` at `3aa20d4` (14 added files, pushed). Upstream `nova-arch008/ZineROM-a11q` returns 404; the fork is public.
- Status repo: `zinerom48-tech/ZineROM-A115F-OneUI7-port-status`, branch `audit/2026-11-03-keymaster-barrier` (head advances with each audit commit; `55e8cdb` before the current doc checkpoint). `main` must stay untouched.
- Current public asset: `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95` (1,842,649,818 bytes), statically validated with 62 tests, **not boot-tested**.
- Code-level root cause: **two-part legacy-fscrypt gap**.
  1. A11 4.9.227 kernel has no modern fscrypt key ioctls; donor `vold` is ioctl-only and has no legacy fallback.
  2. Donor `init` creates neither PID 1's session keyring nor the named `fscrypt` keyring.
- Leading fix: `/system/bin/init` wrapper (pre-policy keyring creation, `argv[0]` preserved, donor init moved to `init.real`) plus a 64-bit vold `LD_PRELOAD` ioctl shim. Source compiled locally with Zig 0.17.1/clang 22.1.8, host-tested, and pushed; ROM build and device test pending. Design in `FIX_LEGACY_FSCRYPT.md`.
- Source branch `manus/a115f-legacy-fscrypt@3aa20d4` adds 14 files and is pushed. The standalone CI workflow is local-only (`local/a115f-legacy-fscrypt-ci-workflow@5551184`) because the credential lacks `workflow` scope. Repository workflows are registered (ISSUE-027 closed); `build.yml` has been dispatched four times, and the 81-test suite passes. The build stops at `Set up build environment` because the apt list contains a nonexistent versioned kernel package and omits `bc` (ISSUE-028).
- Local binaries: wrapper 2624-byte `ET_EXEC`, no `PT_INTERP`/`.dynamic`/`DT_NEEDED`; shim 6280-byte `ET_DYN`, 0 `DT_NEEDED`, imports only `__android_log_print`/`__errno`, exports `ioctl`; host tests pass.
- No physical A11, no raw log linked to the exact ZIP SHA, no verified stock restore package, unknown bootloader/AVB state, no local Android toolchain. These block device and build work.
- Readiness: **R0**.

---

## 3. Phase map

| Phase | Objective | Exit gate | Current status |
|---|---|---|---|
| **1. Repository and hardware audit** | Establish what is known, what is assumed, and what needs a device | G1 | Complete for offline evidence; device items blocked |
| **2. Port development** | Fix boot-critical gaps, then bring up hardware in dependency order | G2 | In progress: legacy-fscrypt source implemented and CI-tested; ROM build blocked by ISSUE-028 |
| **3. Automated QA** | Make every build and static claim reproducible and testable | G3 | Partial: 81-test CI suite green; image/ZIP checks pending the blocked build |
| **4. Bug investigation** | Convert device failures into minimal, evidence-backed fixes | G4 | Blocked on device |
| **5. Build, delivery, report** | Produce a candidate, verify recovery, publish only with approval, report honestly | G5 | Blocked on build/device/approval |

---

## 4. Phase 1 — Repository and hardware audit

### 4.1 Objectives

- Prove the source, donor, target, partition geometry, kernel capabilities, HAL/VINTF/SELinux compatibility, and installer behavior from artifacts.
- Enumerate every remaining unknown that requires a device.
- Do not build a "fix" on an unverified assumption.

### 4.2 Tasks

| ID | Task | Status | Evidence / next action |
|---|---|---|---|
| P1.1 | Confirm source/release availability and exact commit/asset hashes | **Done** | Fork public; `23f8998f…`; asset `f51460ef…`. |
| P1.2 | Establish donor/target identity, ABI, partition geometry | **Done** | Donor A057F/A05s; target A115F/a11q; donor declares 32-bit + 64-bit. |
| P1.3 | Audit kernel container and FBE capabilities | **Done** | A11 4.9.227; FS_ENCRYPTION/ICE/XTS/PFK; legacy-only key management. |
| P1.4 | Audit A11 vendor HALs, VINTF, SELinux | **Done (static)** | KM4 HIDL, qseecomd, vndservicemanager, key class rules verified. Runtime untested. |
| P1.5 | Audit boot image, installer, package guards | **Done (static)** | Stock boot image, BLOCK OTA, dynamic partitions, no `/data` format by installer. |
| P1.6 | Reconcile source branches and identify the only branch with the barrier | **Done** | `23f8998f…` is the base; `build-once` is an ancestor; `preflight` lacks the barrier. |
| P1.7 | Maintain the runtime-unknown register | **In progress** | H5-H10 and open questions in `PORTING_STATUS.md` / `FIX_LEGACY_FSCRYPT.md`. |
| P1.8 | Obtain exact CSC/firmware, bootloader/AVB state, verified stock restore package | **Blocked** | User input required. |

### 4.3 Gate G1

- Every offline claim has a file/offset/hash or a source citation.
- Every device-only claim is explicitly marked untested.
- No open contradiction between `PORTING_STATUS.md`, `ISSUE_REGISTER.md`, `RESUME.md`, and the design doc.

**Status: met for offline evidence.** G1 does not permit installation or a boot claim.

---

## 5. Phase 2 — Port development

### 5.1 Sequencing rule

Fix boot-critical failures first, in this order:

1. `/data` FBE / keyrings / KM4 path.
2. Reaching the Android framework and lockscreen.
3. Display and touch.
4. Storage/installer correctness.
5. Audio, RIL, Wi-Fi/Bluetooth, camera, sensors/fingerprint, power.
6. Performance and polish.

Do not spend time on optional apps, theming, or feature parity while step 1 or 2 is open.

### 5.2 Tasks

| ID | Task | Priority | Status | Depends on | Next action |
|---|---|---|---|---|---|
| P2.1 | Legacy-fscrypt fix design (wrapper + shim + system-image mod) | S1 | **Done (design)** | P1.3/P1.4 | Design reflected in `FIX_LEGACY_FSCRYPT.md`. |
| P2.2 | Implement `init_wrapper.c`, shim, syscall helpers, mod, CI compile | S1 | **Done (source, pushed)** | P2.1 | 12 files pushed at `48438e6`; the standalone CI workflow is local-only. |
| P2.3 | Add host unit tests and static binary checks | S1 | **Done (local + CI)** | P2.2 | Host tests pass; `readelf` checks pass locally and in CI run `43925890423`. |
| P2.4 | Build a candidate on the new branch; record hash; do not publish | S1 | **Blocked (ISSUE-028)** | P2.2/P2.3 | Fix the workflow apt line (remove `linux-modules-extra-$(uname -r)`, add `bc`), dispatch `build.yml`, capture build log and artifact SHA. |
| P2.5 | Preflight-verify donor `system/bin/init` and `ueventd` layout | S1 | **Implemented in mod; ROM build pending** | P2.2 | Fail closed if `init` is not a regular ELF64 or `ueventd` does not resolve to `init`. |
| P2.6 | Device boot to framework/lockscreen (M5/M6) | S1 | **Blocked** | P2.4, P2.5, device, approval | Capture one SHA-linked log. |
| P2.7 | Verify `/data` FBE across reboot; Setup Wizard (M7) | S1 | **Blocked** | P2.6 | Test fresh and existing `/data` separately. |
| P2.8 | Display/touch bring-up | S1 | **Blocked** | P2.6 | Compare panel/DTB/vendor display HAL. |
| P2.9 | Storage/installer correctness (M4/M5) | S1 | **Blocked** | P2.6 | Verify super geometry, mount, recovery. |
| P2.10 | Audio bring-up | S2 | **Blocked** | P2.6 | A11 vendor audio HAL + mixer paths. |
| P2.11 | RIL/modem bring-up | S2 | **Blocked** | P2.6 | Preserve A11 modem/firmware; check IMS. |
| P2.12 | Wi-Fi/Bluetooth bring-up | S2 | **Blocked** | P2.6 | A11 vendor WLAN/BT firmware and HALs. |
| P2.13 | Camera bring-up | S2 | **Blocked** | P2.6 | A11 camera HAL/provider, donor camera app. |
| P2.14 | Sensors/fingerprint bring-up | S2 | **Blocked** | P2.6 | A11 sensor/fingerprint HALs and permissions. |
| P2.15 | Power/charging/thermal | S2 | **Blocked** | P2.6 | A11 health/power HAL, charger, thermal. |

### 5.3 Gate G2

- Device boots to the lockscreen without Rescue Party or Recovery.
- `/data` mounts with FBE v1 and survives a reboot.
- Display and touch are usable; no boot loop.
- The fix markers and the absence of `Failed to install fscrypt key to /data` are captured in one SHA-linked log.
- No open S1 boot-critical issue.

**Status: not met. R0.**

---

## 6. Phase 3 — Automated QA

### 6.1 Objectives

- Make the build reproducible enough to trust the artifact hash.
- Catch boot-critical packaging mistakes before a device test.
- Convert every device failure into a regression check.

### 6.2 Tasks

| ID | Task | Status | Notes |
|---|---|---|---|
| QA1 | Static validators (ZIP, fingerprint, boot chain, vendor ABI, partition layout) | **Running in CI** | The existing static validators run in the same 81-test suite; run `43925890423` passed with no skips. |
| QA2 | Host unit tests for the ioctl decoder and wrapper keyring logic | **Implemented; host tests pass** | `tests/test_legacy.c`; see `FIX_LEGACY_FSCRYPT.md` §8.1. |
| QA3 | CI compile step for wrapper/shim with AArch64 static/shared checks | **Implemented; CI green** | `tests/test_legacy_fscrypt_build.py` compiles both targets with clang 18 and verifies the ELF/symbol contract; the mod repeats the checks at ROM-build time. Do not touch `publish-public.yml`. |
| QA4 | System-image checks: `system/bin/init` is the wrapper, `init.real` is the donor init, `ueventd` resolves to `init`, `vold.rc` has `setenv`, shim present and labelled | **Implemented in mod; ROM build pending** | Fail the build if any check fails. |
| QA5 | Device log parser for `A115F_FSCRYPT_*`, `Failed to install fscrypt key`, AVC denials | **Queued** | Produce a pass/fail/branch summary. |
| QA6 | Regression matrix for FBE fresh/existing `/data`, reboot, and recovery | **Queued** | Blocked on device for execution. |
| QA7 | Pin or hash the `magiskboot` version used by `external/make.sh` | **Obsolete for this fix** | The legacy-fscrypt fix no longer modifies the boot image; keep the hardening note for other boot work. |
| QA8 | Policy-input audit: donor `plat_sepolicy.cil`/`mapping/31.0.cil`, A11 `vendor_sepolicy.cil`, donor `system_ext_sepolicy.cil` | **Done (static)** | Extracted from `system_full.dat` with a read-only ext4 reader; key allows, `_31_0` mappings, and the SELinux patch's non-interference confirmed. The assembled policy remains unbuilt. See `PORTING_STATUS.md` facts 43/57 and ISSUE-029. |

### 6.3 Gate G3

- CI is green on the candidate branch.
- All host/static tests pass; no S1 static finding is open.
- The ZIP hash, wrapper hash, `init.real` hash, and shim hash are recorded.
- The package validates without installing.
- The build fails closed on any missing `system/bin/init`, `ueventd` symlink, `vold.rc`, or shim layout.

**Status: not met.**

---

## 7. Phase 4 — Bug investigation

### 7.1 Method

For every device failure:

1. Preserve the exact log lines and the SHA of the installed ZIP.
2. Identify the **earliest confirmed failure**, not the loudest later symptom.
3. Separate confirmed facts, hypotheses, and untested assumptions.
4. Make the smallest justified change; do not stack unrelated fixes.
5. Record the change, the expected effect, the observed effect, and the rollback.
6. Update `ISSUE_REGISTER.md` and `PORTING_STATUS.md` in the same session.

### 7.2 Tasks

| ID | Task | Status | Notes |
|---|---|---|---|
| P4.1 | Device log intake template and branch tree | **Done** | `NEXT_DEVICE_TEST.md`. |
| P4.2 | Issue register maintenance | **Ongoing** | ISSUE-001…029. |
| P4.3 | Triage the first SHA-linked FBE log | **Blocked** | Needs device + approved flash. |
| P4.4 | Fix display/touch/audio/RIL/Wi-Fi/BT/camera/sensors/power defects | **Blocked** | Needs P2.6. |
| P4.5 | Stability/regression soak (M9) | **Blocked** | Define target with user; no claim without evidence. |

### 7.3 Gate G4

- Every S1 boot-critical issue is closed or mitigated with device evidence.
- The hardware matrix is filled with pass/fail/known-defect entries.
- No known data-loss or boot-loop defect remains.
- Recovery from a failed boot is demonstrated, not assumed.

**Status: not met.**

---

## 8. Phase 5 — Build, delivery, and final report

### 8.1 Deliverables

- Candidate ZIP with SHA-256 and a build-provenance record (branch, commit, CI run, tool versions).
- Boot image hash and a description of the wrapper/shim changes.
- Verified stock restore instructions for the user's exact CSC/firmware.
- Device test report with raw logs and the exact artifact hash.
- Final report separating static/offline work from device-verified behavior.
- Updated `PORTING_STATUS.md`, `ISSUE_REGISTER.md`, `RESUME.md`, `NEXT_DEVICE_TEST.md`.

### 8.2 Tasks

| ID | Task | Status | Gate |
|---|---|---|---|
| P5.1 | Build candidate on the new branch; record all hashes | **Blocked** | P2.4 |
| P5.2 | Verify stock restore path on the exact device/CSC | **Blocked** | P1.8 |
| P5.3 | Obtain explicit user approval to install | **Blocked** | P5.2 |
| P5.4 | Device validation and log capture | **Blocked** | P5.3 |
| P5.5 | Publish release | **Blocked** | P5.4 + explicit approval |
| P5.6 | Final report | **Queued (offline portion can start)** | P5.4/P5.5 |

### 8.3 Gate G5

- User-approved installation completed.
- Device boots and passes the agreed core matrix.
- Recovery path verified on the same device.
- Final report published; no unsupported stability or boot claim.
- Readiness level matches evidence.

**Status: not met. R0.**

---

## 9. Persistent task queue (ordered)

| Order | ID | Action | Owner | Status |
|---|---|---|---|---|
| 1 | P2.2 | Commit and push the 12-file legacy-fscrypt source without altering the donor tree | Agent | **Done** (`48438e6`) |
| 2 | P2.3 | Compile locally, run host tests and ELF/symbol checks | Agent | **Done** |
| 3 | P2.4a | Unblock GitHub Actions workflow registration (repository currently reports zero workflows) | User | **Blocked** (ISSUE-027) |
| 4 | P2.4b | Dispatch `build.yml`; build candidate; record hashes; do not publish | Agent | Queued (waiting on P2.4a) |
| 5 | P2.5 | Inspect the built system image (`init`, `init.real`, `ueventd`, `vold.rc`, shim metadata) | Agent | Queued (waiting on P2.4b) |
| 6 | — | Update docs and issue register with build/static results | Agent | **Done** (this checkpoint) |
| 7 | P1.8 | Obtain exact CSC/firmware, unlock/AVB state, restore package | User | Blocked |
| 8 | P5.3 | Obtain explicit approval to install | User | Blocked |
| 9 | P2.6 | Device boot + one SHA-linked log | User/Agent | Blocked |
| 10 | P2.7 | Verify FBE across reboot + Setup Wizard | User/Agent | Blocked |
| 11 | P2.8-P2.15 | Hardware bring-up in dependency order | User/Agent | Blocked |
| 12 | P4.5 | Stability/regression soak | User/Agent | Blocked |
| 13 | P5.5 | Publish only after approval and device evidence | User | Blocked |
| 14 | P5.6 | Final report | Agent | Queued |

---

## 10. Gates summary

| Gate | Meaning | Current |
|---|---|---|
| G0 | Baseline established: source, asset, donor/target, root cause, recovery constraints documented | **Met** |
| G1 | Offline audit complete; device unknowns enumerated | **Met for offline evidence** |
| G2 | Boot-critical fix implemented, built, and statically validated | **Not met** |
| G3 | Candidate build passes QA and has reproducible hashes | **Not met** |
| G4 | Device boots to framework, FBE stable, core matrix defined | **Not met** |
| G5 | Release approved, published, and final report complete | **Not met** |

---

## 11. Risk register (program level)

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | No physical A11 | Cannot verify boot/hardware | Keep R0; do not claim success; prepare device-test package. |
| R2 | No verified stock restore package/CSC | Brick/data-loss risk | Block installation until P1.8 is complete. |
| R3 | Donor `system/bin/init` layout differs or `ueventd` is not a symlink | Build fails or ueventd breaks | Mod verifies both and fails closed; restore previous system image. |
| R4 | `LD_PRELOAD` ignored or shim not loaded | FBE still fails | Marker check; fallback vold wrapper only after evidence. |
| R5 | SELinux denial on keyring operations | FBE fails | Capture AVC; targeted rule only with evidence. |
| R6 | Keymaster/Keystore2 fails earlier than key installation | Fix insufficient | Separate ISSUE-019/020/021; capture key retrieval lines. |
| R7 | Existing `/data` key mismatch | Cannot decrypt user data | Document; format only with explicit approval. |
| R8 | CI network/toolchain dependency | Non-reproducible build | Record tool versions and artifact hashes; pin where practical. |
| R9 | Donor system incompatibility beyond FBE | Later boot/hardware failures | Phase 2/4 hardware matrix; do not declare victory at FBE. |
| R10 | Release asset unavailable upstream | Cannot re-download the 1.84 GB ZIP | Preserve local hashes; do not rely on the 404 upstream. |

---

## 12. Evidence and reporting rules

- Tag every statement as **[verified]**, **[hypothesis]**, or **[untested]** when the distinction matters.
- A device claim requires: exact ZIP SHA, boot log, device model/CSC, and timestamp.
- A build claim requires: branch, commit, CI run, tool versions, artifact SHA.
- A static claim requires: file path, offset/hash, and the command used.
- Never say "boot fixed", "stable", or "working" without device evidence.
- Preserve failed logs; do not paraphrase the earliest failure.
- Do not silently skip a failed check; record it in `ISSUE_REGISTER.md`.

---

## 13. Immediate next actions

1. **Unblock GitHub Actions workflow registration (user action).** The repository reports zero registered workflows: dispatch returns HTTP 404 and `GET /actions/workflows` returns `total_count: 0`, although `build.yml` exists on `oneui7-a11q-boot`, Actions is enabled, and the token has `repo` scope. Enable/register workflows in the GitHub UI or supply a `workflow`-scoped credential. Do not rewrite the default branch.
2. **Dispatch `build.yml` on `manus/a115f-legacy-fscrypt`.** Watch the unit-test step, then the mod's compile/host-test/`readelf` checks; fix failures before producing a ZIP.
3. **Build and record.** Record the ZIP, wrapper, `init.real`, and shim SHA-256; do not publish.
4. **Inspect the built system image:** `system/bin/init` wrapper, `system/bin/init.real` donor init, `ueventd` symlink, one `setenv LD_PRELOAD`, shim label/presence.
5. **Do not device-test** until the exact CSC/firmware, unlock/AVB state, stock restore package, and explicit approval exist; then capture one SHA-linked log and do not format `/data`.
6. **Keep the docs current** with build results and any CI failure output.
7. **Request from the user:** exact CSC/firmware, bootloader/AVB state, verified stock restore package, and explicit approval before any flash.
8. **When a device is available:** capture one full SHA-linked log and triage the earliest confirmed failure.
9. Keep readiness R0 until a device reaches the framework.

---

## 14. Change log

- 2026-11-16: Initial Phase 1-5 plan. Records the two-part FBE root cause, the wrapper/shim design gate, the persistent task queue, G0-G5 gates, program risks, and the evidence rules. No build, flash, format, or new ZIP. R0.
- 2026-11-16 (design change): Wrapper moved from the boot ramdisk `/init` to `/system/bin/init`; donor init becomes `/system/bin/init.real`, both labelled `init_exec`. `argv[0]` is preserved for the `ueventd` symlink. Boot image and `magiskboot` are no longer part of this fix. Source and CI workflow exist; compile, ROM build, and device test are still pending. R0.
- 2026-11-16 (local build): Compiled the wrapper and shim with Zig 0.17.1/clang 22.1.8 + lld using the project flags, no warnings. Wrapper `ET_EXEC`, 2624 bytes, entry `0x1010848`, no `PT_INTERP`/`.dynamic`, 0 `DT_NEEDED`; shim `ET_DYN`, 6280 bytes, 0 `DT_NEEDED`, imports exactly `__android_log_print __errno`, exports `ioctl`, no memory exports. Host tests initially crashed with `0xC0000005` because `a115f_keyctl_fn` used 32-bit `unsigned long` on Windows LLP64; switching to `uintptr_t` fixed the host tests and left the aarch64 binaries byte-identical. Commits `325dcd2` (12 files) and `48438e6` (ABI fix) pushed. R0; no ROM build or device test.
- 2026-11-16 (CI blocker): `build.yml` dispatch returns HTTP 404 and the repository lists zero workflows, although the default branch `oneui7-a11q-boot` contains `build.yml`, Actions is enabled, and the token has `repo` scope. Requires user action to register workflows or a `workflow`-scoped credential (ISSUE-027). Do not rewrite the default branch. R0.
- 2026-11-16 (FUS/workflow preflight): Verified both pinned firmware tuples are the current latest on Samsung FUS with samloader-rs 2.2.0 (`SM-A115F/MWD` = `A115FXXS6CWK2/A115FOJM6CWK1/A115FXXS6CWK1/A115FXXS6CWK2`; `SM-A057F/TUR` = `A057FXXUFDZH2/A057FOJMFDZH2/A057FXXUFDZH2/A057FXXUFDZH2`), so the build's FUS download path is not blocked by unavailable firmware. Found a second workflow apt defect: `make_rom.sh` sources `build_utils.sh`, which requires `bc` at source time, but neither the workflow list nor the Ubuntu 24.04 image manifest provides it. ISSUE-028 now requires removing `linux-modules-extra-$(uname -r)` and adding `bc` in the same edit; added failed approach 21. Updated the phase/task status to the 81-test CI checkpoint. R0; no build, flash, or `/data` operation.
