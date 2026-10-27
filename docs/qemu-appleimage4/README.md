> **Cross-port note — not S7 Edge hardware evidence.** These files were produced
> by the QEMU AppleImage4 isolation pass in the Galaxy Tab A7 (`gta4l`) workspace
> `unixnutaba72020test` on 2026-10-27. The tested kernel was a panic-screen
> diagnostic build (SHA-256 `72c6f4635767901506d65ac8598860d1dc06b5c72c5402914af2a8aa6f57952e`).
> Paths and addresses refer to that source workspace. Re-derive them for
> `hero2lte` before use.

# S7 Edge port notes — cross-port QEMU findings

These notes were imported from the Tab A7 QEMU pass. They are not `hero2lte`
hardware results and have not been verified on the S7 Edge.

## Target

| Item | Value |
| --- | --- |
| Device | Samsung Galaxy S7 Edge (SM-G935F, `hero2lte`) |
| SoC | Samsung Exynos 8890 (S5E8890) |
| Current stage | Phase 1 UEFI bring-up; QEMU cross-port findings only |

## Status

The QEMU isolation pass in the source Tab A7 workspace removed the
`com.apple.security.AppleImage4` null dereference and reached `_bsd_init` and
`_IOKitBSDInit`. The next blocker is
AMFI requiring the `ApplePMGRNub` service, which the synthetic QEMU platform
does not provide. That is the QEMU decision point; the S7 Edge has real PMGR
hardware but still needs an Apple-compatible PMGR provider or an AMFI-compatible
policy.

Normal video console was not qualified. The panic renderer works; that is not
proof of a normal console.

## Key findings

1. AppleImage4 initializes from the device tree. A missing `/options` node
   becomes a null object and causes a data abort at `ldr x8, [x0]`.
2. The `/options` result is stored and then not checked; a missing node reaches
   the nonce-magazine property read.
3. `/chosen` needs a CHRP NVRAM proxy or `IONVRAMCHRPHandler` creation fails.
4. `/chosen/crypto-hash-method` must be valid (`sha2-384` in the QEMU test) or
   AppleImage4 panics.
5. Do not use `cs_enforcement_disable=1`; AMFI panics with
   `can't has cs_enforcement_disable`.
6. AMFI needs `amfi-only-platform-code` in the device tree when the bypass is
   removed.
7. AMFI later needs `ApplePMGRNub` (`pmgr1,bring-up` on `AppleARMIODevice`).

## Files

- [`appleimage4-adt-requirements.md`](appleimage4-adt-requirements.md) — detailed
  root cause, required ADT inputs, addresses and reusable debugging method.
- [`bringup-checklist.md`](bringup-checklist.md) — actionable checklist for the
  S7 Edge port.
- [`qemu-result.md`](qemu-result.md) — exact QEMU test record, provenance,
  changes, checkpoints and reproduction command.

## Evidence

- Kernel tested: SHA-256
  `72c6f4635767901506d65ac8598860d1dc06b5c72c5402914af2a8aa6f57952e`
  (panic-screen diagnostic build; kernel bytes unchanged).
- Test ESP (source workspace): `unixnutaba72020test/out/qemu/options-test/esp`.
- Original ESP (source workspace): `unixnutaba72020test/out/qemu/defaults-test/esp`
  (untouched).
- QEMU checkpoints reached: `_bsd_init`, `_IOKitBSDInit`.
- Next stop: AMFI `"AMFI: No PMGR?\n"`.

## Constraints used

- No S7 Edge tablet flashing or hardware changes; the QEMU pass used a separate
  Tab A7 workspace.
- No source-kernel overwrites; kernel bytes unchanged.
- ADT and boot-argument changes only in a copied test ESP.
- Security checks were not bypassed intentionally; the QEMU run still contains
  the inherited `amfi_get_out_of_my_way=1`, so a clean security-preserving run
  remains to be done.
