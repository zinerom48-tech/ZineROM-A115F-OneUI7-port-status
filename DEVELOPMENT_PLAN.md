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

- Source: `zinerom48-tech/ZineROM-a11q`, branch `manus/a115f-vendor-fingerprint` at `23f8998f0200969f5748e9eb012d1dc52037a054`. Upstream `nova-arch008/ZineROM-a11q` returns 404; the fork is public.
- Status repo: `zinerom48-tech/ZineROM-A115F-OneUI7-port-status`, branch `audit/2026-11-03-keymaster-barrier` at `3fb26b8` (plus the current doc checkpoint). `main` remains at `1d2d0ad` and must stay untouched.
- Current public asset: `f51460eff376466d24c12d39902e954e6037e5de22b3a7aef68088346f26fa95` (1,842,649,818 bytes), statically validated with 62 tests, **not boot-tested**.
- Code-level root cause: **two-part legacy-fscrypt gap**.
  1. A11 4.9.227 kernel has no modern fscrypt key ioctls; donor `vold` is ioctl-only and has no legacy fallback.
  2. Donor `init` creates neither PID 1's session keyring nor the named `fscrypt` keyring.
- Leading fix: pre-policy ramdisk `/init` wrapper plus a 64-bit vold `LD_PRELOAD` ioctl shim. Design complete in `FIX_LEGACY_FSCRYPT.md`; not implemented or built.
- No physical A11, no raw log linked to the exact ZIP SHA, no verified stock restore package, unknown bootloader/AVB state, no local Android toolchain. These block device and build work.
- Readiness: **R0**.

---

## 3. Phase map

| Phase | Objective | Exit gate | Current status |
|---|---|---|---|
| **1. Repository and hardware audit** | Establish what is known, what is assumed, and what needs a device | G1 | Complete for offline evidence; device items blocked |
| **2. Port development** | Fix boot-critical gaps, then bring up hardware in dependency order | G2 | In progress: legacy-fscrypt design done; implementation next |
| **3. Automated QA** | Make every build and static claim reproducible and testable | G3 | Partial: prior 62 static tests; new host/CI tests queued |
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
| P2.1 | Legacy-fscrypt fix design (wrapper + shim + boot patcher) | S1 | **Done (design)** | P1.3/P1.4 | Implement in P2.2. |
| P2.2 | Implement `init_wrapper.c`, shim, syscall helpers, boot patcher, mod, CI compile | S1 | **Queued (next)** | P2.1 | Create branch from `23f8998f…`; write source and tests. |
| P2.3 | Add host unit tests and static binary checks | S1 | **Queued** | P2.2 | Struct offsets, names, payload, probe, ENOTTY-only. |
| P2.4 | Build a candidate on the new branch; record hash; do not publish | S1 | **Queued** | P2.2/P2.3 | Run CI, capture build log and artifact SHA. |
| P2.5 | Preflight-verify stock A11 `boot.img` layout (`/init`, compression, header) | S1 | **Queued** | P2.2 | Fail closed if layout differs. |
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
| QA1 | Static validators (ZIP, fingerprint, boot chain, vendor ABI, partition layout) | **Partial** | Existing 62 tests; not re-run this session. |
| QA2 | Host unit tests for the ioctl decoder and wrapper keyring logic | **Queued** | See `FIX_LEGACY_FSCRYPT.md` §8.1. |
| QA3 | CI compile step for wrapper/shim with AArch64 static/shared checks | **Queued** | Add before `Build ROM`; do not touch `publish-public.yml`. |
| QA4 | Image/package checks: boot ramdisk contains `init` + `init.real`; `vold.rc` contains `setenv`; shim present and labelled | **Queued** | Fail the build if any check fails. |
| QA5 | Device log parser for `A115F_FSCRYPT_*`, `Failed to install fscrypt key`, AVC denials | **Queued** | Produce a pass/fail/branch summary. |
| QA6 | Regression matrix for FBE fresh/existing `/data`, reboot, and recovery | **Queued** | Blocked on device for execution. |
| QA7 | Pin or hash the `magiskboot` version used by `external/make.sh` | **Queued (hardening)** | Current build downloads "latest"; record risk R10. |

### 6.3 Gate G3

- CI is green on the candidate branch.
- All host/static tests pass; no S1 static finding is open.
- The ZIP hash, boot image hash, wrapper hash, and shim hash are recorded.
- The package validates without installing.
- The build fails closed on any missing boot/ramdisk/vold layout.

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
| P4.2 | Issue register maintenance | **Ongoing** | ISSUE-001…025. |
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
| 1 | P2.2 | Create branch from `23f8998f…`; implement wrapper, shim, syscall helpers | Agent | Queued |
| 2 | P2.2 | Implement `zz_a115f_legacy_fscrypt` mod + boot patcher + `vold.rc` patch + metadata | Agent | Queued |
| 3 | P2.3 | Add host tests and static binary checks | Agent | Queued |
| 4 | QA3/QA4 | Add CI compile step and image/package checks | Agent | Queued |
| 5 | P2.5 | Preflight the stock boot image layout; fail closed | Agent | Queued |
| 6 | P2.4 | Build candidate; record hashes; do not publish | Agent | Queued |
| 7 | — | Update docs and issue register with build/static results | Agent | Queued |
| 8 | P1.8 | Obtain exact CSC/firmware, unlock/AVB state, restore package | User | Blocked |
| 9 | P5.3 | Obtain explicit approval to install | User | Blocked |
| 10 | P2.6 | Device boot + one SHA-linked log | User/Agent | Blocked |
| 11 | P2.7 | Verify FBE across reboot + Setup Wizard | User/Agent | Blocked |
| 12 | P2.8-P2.15 | Hardware bring-up in dependency order | User/Agent | Blocked |
| 13 | P4.5 | Stability/regression soak | User/Agent | Blocked |
| 14 | P5.5 | Publish only after approval and device evidence | User | Blocked |
| 15 | P5.6 | Final report | Agent | Queued |

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
| R3 | Boot image layout differs from the assumption | Build fails or dead boot | Preflight check; fail closed; reflash stock boot. |
| R4 | `LD_PRELOAD` ignored or shim not loaded | FBE still fails | Marker check; fallback vold wrapper only after evidence. |
| R5 | SELinux denial on keyring operations | FBE fails | Capture AVC; targeted rule only with evidence. |
| R6 | Keymaster/Keystore2 fails earlier than key installation | Fix insufficient | Separate ISSUE-019/020/021; capture key retrieval lines. |
| R7 | Existing `/data` key mismatch | Cannot decrypt user data | Document; format only with explicit approval. |
| R8 | Unpinned `magiskboot` / CI network dependency | Non-reproducible build | Record version/hash; pin in QA7. |
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

1. Create a new source branch from `23f8998f0200969f5748e9eb012d1dc52037a054`; do not touch `main`.
2. Implement `prebuilts/a115f_legacy_fscrypt/` (wrapper, shim, syscall helpers, build script, tests).
3. Implement `unica/mods/zz_a115f_legacy_fscrypt/` (boot patcher, shim copy, metadata, `vold.rc` patch, fail-closed checks).
4. Add the CI compile step and image/package checks.
5. Preflight the stock A11 `boot.img`: `/init` present, ramdisk compression/header, `vendor_boot` absent.
6. Build the candidate; record the ZIP, boot image, wrapper, and shim hashes; do not publish.
7. Update `PORTING_STATUS.md`, `ISSUE_REGISTER.md`, `RESUME.md`, and `NEXT_DEVICE_TEST.md` with the build results.
8. Request from the user: exact CSC/firmware, bootloader/AVB state, verified stock restore package, and explicit approval before any flash.
9. When a device is available: capture one full SHA-linked log and triage the earliest confirmed failure.
10. Keep readiness R0 until a device reaches the framework.

---

## 14. Change log

- 2026-11-16: Initial Phase 1-5 plan. Records the two-part FBE root cause, the wrapper/shim design gate, the persistent task queue, G0-G5 gates, program risks, and the evidence rules. No build, flash, format, or new ZIP. R0.
