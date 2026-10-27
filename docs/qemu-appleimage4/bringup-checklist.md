> **Cross-port note — not S7 Edge hardware evidence.** This checklist is derived
> from the QEMU AppleImage4 isolation pass in the Tab A7 (`gta4l`) workspace
> `unixnutaba72020test` on 2026-10-27. The addresses and ADT values in the
> companion notes are source-workspace findings; re-derive them for `hero2lte`.

# S7 Edge bring-up checklist

Use this as a working checklist. It is based on QEMU-only findings and has not
been verified on S7 Edge hardware.

## 0. Identify the target

- [ ] Confirm the target: this repository is SM-G935F `hero2lte` (Exynos 8890).
      If a Snapdragon `hero2qlte` variant is ever added, keep its ADT and inputs
      separate.
- [ ] Record the exact model, bootloader, firmware and kernel used.
- [ ] Keep the S7 Edge kernel bytes unchanged for diagnostic runs; use a copy of
      the ESP/ADT.

## 1. Device-tree / ADT inputs

- [ ] Add an `/options` node. An empty node is enough to make the lookup
      non-null; it is not a place to fabricate nonce values.
- [ ] Add the CHRP NVRAM proxy under `/chosen`:
      - [ ] `nvram-bank-size = 8192`
      - [ ] `nvram-bank-count = 1`
      - [ ] `nvram-current-bank = 0`
      - [ ] `nvram-proxy-data = <8192-byte empty CHRP bank>`
- [ ] Add `/chosen/crypto-hash-method = "sha2-384"` (or the value the target
      Image4 build actually expects).
- [ ] Add `/chosen/amfi-only-platform-code = 1` if AMFI is expected to use the
      platform-code policy.
- [ ] Confirm the S7 Edge ADT generator emits these nodes, rather than patching
      a binary ADT by hand.

## 2. Security and boot arguments

- [ ] Do not pass `cs_enforcement_disable=1`; AMFI panics with
      `can't has cs_enforcement_disable`.
- [ ] Do not rely on `amfi_get_out_of_my_way=1` for the final configuration.
      The QEMU pass still had it, so the QEMU result is not a clean
      security-preserving boot.
- [ ] Decide the AMFI policy: platform code, signatures, or a documented
      development-only exception. Record the decision.
- [ ] Re-run with the clean policy and confirm AMFI reaches `_IOKitBSDInit`
      without a bypass argument.

## 3. PMGR / platform provider

- [ ] Provide an `ApplePMGRNub` service, or an AMFI-compatible policy that does
      not require it.
- [ ] The prelinked `ApplePMGR` personality matches:
      - `IOClass = AppleBringUpPMGR`
      - `IONameMatch = pmgr1,bring-up`
      - `IOProviderClass = AppleARMIODevice`
- [ ] If using a device-tree node, confirm the S7 Edge PMGR node is under the
      `arm-io` provider and has the resources the driver expects.
- [ ] Confirm AMFI no longer panics with `"AMFI: No PMGR?\n"`.
- [ ] If PMGR requires extensive Apple platform emulation, stop and return to
      board-specific work; do not fake a nonce or skip the AMFI check.

## 4. Checkpoints

Verify each checkpoint on the S7 Edge kernel (addresses will differ):

- [ ] `_bsd_init`
- [ ] `_IOKitBSDInit`
- [ ] `_vfs_mountroot`
- [ ] first user-space process / launchd (later stage)

For QEMU reference, the tested kernel reached:

```
_bsd_init (0xfffffff0076ac128)
  -> _IOKitBSDInit (0xfffffff00791e210)
  -> AMFI "No PMGR?" panic
```

## 5. AppleImage4 nonce path

- [ ] Confirm AppleImage4 reads the nonce-magazine key
      `40A0DDD2-77F8-4392-B4A3-1E7304206516:krn.zine` from `/options`.
- [ ] If the key is missing, confirm the read returns a normal `not found`
      result instead of a null dereference.
- [ ] Do not invent boot nonces, manifest hashes or trust evaluations. Keep the
      original security result.

## 6. Evidence and artifacts

- [ ] Save the exact kernel SHA-256, ADT SHA-256 and boot arguments.
- [ ] Save the ESP/ADT as a separate diagnostic artifact; do not overwrite the
      tablet boot image.
- [ ] Record the fault PC, registers and call chain for any new abort.
- [ ] Distinguish a shared-helper hit with a valid object from the actual null
      hit.

## 7. Video / console

- [ ] Test the normal video console separately from the panic screen.
- [ ] Treat panic-screen output as diagnostic only.
- [ ] If normal console is blank, check console mapping and output suppression
      before assuming the boot failed.

## 8. Decision gate

- [ ] If progress requires Apple PMGR/AES/GPU hardware emulation, stop QEMU work
      and return to S7 Edge board-specific bring-up.
- [ ] Document the blocker and the last reached checkpoint.
- [ ] Do not weaken security checks just to pass a QEMU checkpoint.
