# Repository and branching decisions

Status: development integration notes. No S7 Edge hardware boot is claimed.

Audience: the agent working on the S7 Edge port. Read this before changing the
shared `UniXNU`/`Mu-Silicium` trees or merging device branches.

## 1. Canonical repositories

| Role | Repository | Branch | Revision at time of writing |
| --- | --- | --- | --- |
| Shared loader/build tree | `DokuroMitsukai/UniXNU` | `integration/hero2lte-gta4l` | `456185449295c50365c54353b0a4b385ab50a17e` |
| Shared firmware | `DokuroMitsukai/Mu-Silicium` | `integration/hero2lte-gta4l` | `e14626a3757007a2dbc902d3db323620d5568923` |
| Shared UEFI loader | `DokuroMitsukai/DarwinBoot` | `bringup/shared-video` | `3f2f304784f8fbfbf2ebfdc8eba3af3577d92967` |
| Shared CPU/GIC support | `DokuroMitsukai/ArmCpu` | `main` | `c9c4bd87cabc1fe6736e171cd2540313ac4b7883` |
| Exynos 8890 SoC tree | `DokuroMitsukai/UniXNU-exynos8890-common` | `main` | `0258950adf615983aec3c87ca65cb52b2310ec95` |
| S7 Edge device | `DokuroMitsukai/UniXNU-hero2lte` | `main` | `33ca693` |
| Tab A7 device (primary) | `DokuroMitsukai/UniXNU-gta4l` | `bringup/tab-a7-qemu-fixture` | `a42d674` at the time of writing; use the branch head |

`DokuroMitsukai/UniXNU` is a fork of `UniXNU/UniXNU`. The device repositories
are separate repositories; they are not subdirectories of the shared tree.

The Tab A7 repository is a fork of `ilovemykedi/UniXNU-gta4l`, which is itself a
standalone repository. The old `NoNameBuilder/UniXNU-gta4l` checkout is only a
reference; it is not the primary device repository.

## 2. Branch roles

| Branch | Repository | Purpose |
| --- | --- | --- |
| `integration/hero2lte-gta4l` | `UniXNU`, `Mu-Silicium` | Shared integration of compatible S7 Edge and Tab A7 changes. Build from here. |
| `bringup/platform-payload-menu` | `UniXNU` | Generic payload, menu, ramdisk and platform-input changes. |
| `bringup/tab-a7-unixnu` | `UniXNU` | Tab A7 topic branch; merged into the shared integration branch. |
| `bringup/hero2lte-mu` | `Mu-Silicium` | S7 Edge firmware changes. |
| `bringup/tab-a7-mu` | `Mu-Silicium` | Tab A7 firmware changes. |
| `bringup/shared-video` | `DarwinBoot` | Shared GOP video-base mapping fix. |
| `bringup/qemu-appleimage4-notes` | `UniXNU-hero2lte` | Cross-port AppleImage4/AMFI QEMU notes. |
| `bringup/porting-ramdisk` | `UniXNU-hero2lte` | S7 Edge porting record for the S-Boot ramdisk range. |
| `bringup/tab-a7-qemu-fixture` | `UniXNU-gta4l` | Tab A7 QEMU fixture, ADT/build fixes and shared revision pins. |
| `docs/repository-branching` | `UniXNU-hero2lte` | This document. |

Rules:

- Topic branches contain one coherent change.
- Shared, compatible changes go through the `integration/hero2lte-gta4l` branch.
- Board-specific configuration stays in the device repository (`board.toml`,
  `configs/*_defconfig`, DSC/FDF, ADT generator).
- Do not put board-only policy into the shared `UniXNU` or `Mu-Silicium` default
  configuration. Use a board PCD, DSC override, or device repository setting.
- Do not force-push shared branches.
- Push to the `DokuroMitsukai` fork. Do not open upstream pull requests unless
  the owner explicitly asks.

## 3. Why the shared branch contains both devices

The integration branch is one compatible source for both ports:

- Generic payload discovery, menu input, FAT validation and the DarwinBoot
  video-base fix are shared.
- S7 Edge and Tab A7 board changes stay in their own platform directories,
  device repositories and PCDs.
- The shared branch is a development candidate, not a hardware-qualified
  release.
- The S7 Edge and Tab A7 must both build from the same shared revision before
  it is treated as a common baseline.

If a change cannot be made compatible, keep it on a device branch and document
the conflict. Do not hide a board-specific change in a shared default.

## 4. Tab A7 primary checkout

In the Tab A7 workspace:

- `src/UniXNU-gta4l` is the primary checkout from
  `DokuroMitsukai/UniXNU-gta4l`.
- `src/UniXNU-gta4l-noname-reference` is the old NoNameBuilder checkout kept
  only for reference.
- The primary checkout has `origin` set to the `DokuroMitsukai` fork and
  `upstream` set to `ilovemykedi`.
- Its build pins are recorded in
  `src/UniXNU-gta4l/docs/build-revisions.md`.

Do not treat the NoNameBuilder checkout as a second source of truth. Its
uncommitted `board.toml` and `configs/gta4l_defconfig` contain test settings
that were deliberately not ported.

## 5. Rename incident and rule

A fork command on `NoNameBuilder/UniXNU-gta4l` followed the fork root and
renamed the existing shared `DokuroMitsukai/UniXNU` fork to
`DokuroMitsukai/UniXNU-gta4l`. It was restored to `DokuroMitsukai/UniXNU`.

Current state:

- `DokuroMitsukai/UniXNU` is the canonical shared fork.
- `DokuroMitsukai/UniXNU-gta4l` is a separate fork of
  `ilovemykedi/UniXNU-gta4l`.
- `NoNameBuilder/UniXNU-gta4l` is a fork of `Mizanla/UniXNU`; forking it again
  follows the root and does not create a board fork.

Rule: never use `gh repo fork --fork-name` on a fork-of-a-fork to try to create
a device fork. Create a standalone repository or use the existing correct fork.

## 6. Local workspace state

S7 Edge workspace shared checkouts:

- `src/UniXNU` is on `integration/hero2lte-gta4l` and tracks
  `origin/integration/hero2lte-gta4l`.
- `src/Mu-Silicium` is on `integration/hero2lte-gta4l` and tracks
  `origin/integration/hero2lte-gta4l`.
- `src/UniXNU/drivers/DarwinBoot` is at `3f2f304`.
- `src/UniXNU/drivers/ArmCpu` is at `c9c4bd8`.
- `src/Mu-Silicium/Mu_Basecore` has pre-existing modified content. Do not
  commit or reset it blindly; identify whether the change belongs to the task.

S7 Edge device repository:

- `src/UniXNU-hero2lte` is on the branch being reviewed; check `git status`
  before changing anything.
- `board.toml` currently sets `ENABLE_SECUREBOOT = "0"`. Its firmware build is
  a development image, not a security-preserving result.

Tab A7 device repository:

- The primary device branch is clean and pinned.
- The NoNameBuilder reference retains the uncommitted test settings:
  `ENABLE_SECUREBOOT=0`, `CONFIG_XNU_NATIVE_PAN=n`, the smaller payload, the
  reset probes and the `cs_enforcement_disable` argument.

## 7. Build and validation status

Host checks that pass:

- `python3 package/Scripts/test-menu-input.py`
- `PYTHONPATH=scripts python3 -m unittest unixnu.test_kernel -v`
- `python3 src/Mu-Silicium/Resources/Scripts/test_bootshim.py`
- `python3 scripts/check-gta4l.py` in the Tab A7 device repo
- `python3 scripts/qemu-smoke.py --help`
- `./unixnu status --json -D ../UniXNU-gta4l`

Firmware builds:

- Tab A7: PASS with `ENABLE_SECUREBOOT=1`.
- S7 Edge: PASS with `ENABLE_SECUREBOOT=0` from its existing board config.

Kernel build:

- Attempted for Tab A7 on Linux using the S7 workspace toolchain plus locally
  built `tcsh` and `unifdef`.
- Blocked at the corecrypto archive step because the XNU bootstrap object set
  was incomplete (`vm_kern.o`, `prng_random.o` missing).
- This is a host/build-order blocker, not a code failure. Kernel builds remain
  unvalidated for both devices.

A firmware build is not a hardware boot. QEMU checkpoints and panic-screen
output are not S7 Edge hardware evidence.

## 8. Agent workflow

Before any change:

1. Run `git status --short`, `git branch -vv`, `git remote -v` and
   `git log --oneline -5` in the shared and device repositories.
2. Run `git submodule status` and inspect dirty submodules.
3. Identify whether each dirty file is pre-existing work, generated output, or
   part of the task. Do not reset or clean blindly.
4. Decide whether the change is shared or device-specific.
5. Branch from the correct base: shared changes from the integration branch,
   device changes from the device branch.
6. Keep board-specific settings out of shared defaults.
7. Run the relevant host tests and both firmware builds where possible.
8. Commit small, coherent changes.
9. Push to the `DokuroMitsukai` fork. Do not force-push.
10. Update the device pin documents when shared revisions move.
11. Record exact commit hashes, build commands and results. Separate host,
    QEMU and hardware results.

When updating shared revisions:

- Update `UniXNU`, `Mu-Silicium`, `DarwinBoot`, `ArmCpu` and SoC pins together.
- Rebuild both device firmware images.
- Update the Tab A7 pins in `UniXNU-gta4l/docs/build-revisions.md` and the S7
  Edge record/branch as applicable.
- Do not promote a shared branch to a common baseline until both builds pass.

## 9. Known pitfalls

- Fork-of-a-fork rename: see section 5.
- Limited fetch refspec: if a branch is not visible after `git fetch origin`,
  add `+refs/heads/*:refs/remotes/origin/*` and fetch again before setting the
  upstream.
- Dirty `Mu_Basecore`: it is a shared submodule with pre-existing content; do
  not include it in a commit unless the change is intentional and reviewed.
- S7 Edge `ENABLE_SECUREBOOT=0`: do not describe the current firmware as
  security-preserving.
- Linux kernel build: `tcsh` and `unifdef` may be missing; the XNU bootstrap
  object set must be complete before the corecrypto archive step.
- QEMU AppleImage4 `/options` findings are cross-port input requirements, not
  proof that the S7 Edge hardware or ADT is correct.

## 10. Evidence and attribution

- QEMU findings live in `docs/qemu-appleimage4/` on
  `bringup/qemu-appleimage4-notes`.
- The S-Boot ramdisk porting record is on `bringup/porting-ramdisk`.
- Tab A7 pins are in `UniXNU-gta4l/docs/build-revisions.md`.
- Keep source, host-build, QEMU and hardware results clearly separated.
- Use the exact `Co-authored-by` trailers supplied in the session context.
  Never invent identities or alter the human author/committer. Preserve
  existing model trailers when amending a commit.
