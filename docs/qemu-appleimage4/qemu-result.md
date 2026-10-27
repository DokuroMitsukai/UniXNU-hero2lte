> **Cross-port note — not S7 Edge hardware evidence.** This is the QEMU test
> record from the Tab A7 (`gta4l`) workspace `unixnutaba72020test`, 2026-10-27.
> It is not a `hero2lte` hardware result. Paths below refer to that source
> workspace.

# QEMU AppleImage4 isolation — result

Date: 2026-10-27

## Provenance

| Item | Value |
| --- | --- |
| Kernel | `unixnutaba72020test/out/qemu/options-test/esp/EFI/UniXNU/Payloads/kernel.macho` |
| Kernel SHA-256 | `72c6f4635767901506d65ac8598860d1dc06b5c72c5402914af2a8aa6f57952e` |
| Kernel bytes changed | no |
| Original ESP | `unixnutaba72020test/out/qemu/defaults-test/esp` (untouched) |
| Test ESP | `unixnutaba72020test/out/qemu/options-test/esp` (copy) |
| Original ADT SHA-256 | `ccddf0c16206dbba24b7b4b9c834d9c70a3ce8179f0442ba31adb8995d304db0` |
| Final ADT SHA-256 | `304b8e22def099a84550a2c80f29496198795c7dd0c710b17632db64a430b672` |

The tested kernel is the panic-screen diagnostic build, not the S7 Edge tablet kernel. The AppleImage4 and AMFI code paths are the original kext code.

## Baseline

Data abort in `com.apple.security.AppleImage4`:

- fault PC `0xfffffff004a92828`, instruction `ldr x8, [x0]`
- `x0 = 0`
- `ESR_EL1 = 0x96000005`, `FAR_EL1 = 0`
- requested key: `40A0DDD2-77F8-4392-B4A3-1E7304206516:krn.zine`
- caller return `0xfffffff004a9e08c`

The null object was `AppleImage4` site state at `0xfffffff005cf83e8`, field `+0x10`
(`0xfffffff005cf83f8`). Initialization stores the `/options` lookup result into
that field at `0x4a985ac`, then checks the `/chosen` field at `+0x08` instead of
the `/options` field.

## Inputs tested

All changes are in the copied ESP/ADT; kernel bytes are unchanged.

| Order | Change | Result |
| --- | --- | --- |
| 0 | baseline | AppleImage4 null dereference at `0x4a92828` |
| 1 | add empty root `/options` node | AppleImage4 abort gone; `IONVRAMCHRPHandler creation failed` |
| 2 | add CHRP NVRAM proxy properties to `/chosen` | IONVRAM panic gone; AMFI `can't has cs_enforcement_disable` |
| 3 | remove `cs_enforcement_disable=1` from boot args | AMFI `no amfi-only-platform-code device tree node` |
| 4 | add `/chosen/amfi-only-platform-code = 1` | `_bsd_init` and `_IOKitBSDInit` reached |

The final boot args in the test copy are:

```
-v serial=3 debug=0x14e keepsym=1 amfi_get_out_of_my_way=1
```

`amfi_get_out_of_my_way=1` remains. A clean security-preserving run is still
pending; do not treat this result as a security-preserving boot.

## Checkpoints reached

| Symbol | Address | Reached |
| --- | --- | --- |
| `_bsd_init` | `0xfffffff0076ac128` | yes |
| `_IOKitBSDInit` | `0xfffffff00791e210` | yes |
| `_vfs_mountroot` | `0xfffffff00746bd28` | not reached |
| `_panic` | `0xfffffff007972a50` | next stop |

## Next blocker

After `_IOKitBSDInit`, AMFI panics:

```
"AMFI: No PMGR?\n" @%s:%d
```

`ApplePMGR` personality:

- `IOClass = AppleBringUpPMGR`
- `IONameMatch = pmgr1,bring-up`
- `IOProviderClass = AppleARMIODevice`

AMFI requires the `ApplePMGRNub` service. The synthetic QEMU ADT has no
`pmgr1,bring-up` node and QEMU has no Apple PMGR hardware. This is the QEMU
decision point; continuing would require Apple PMGR platform emulation.

## Reproduction

The test used QEMU `virt` with the copied ESP and `ramfb`. Run the command from
the source workspace root `unixnutaba72020test/`:

```sh
qemu-system-aarch64 -M virt,virtualization=off,secure=off,gic-version=3 \
  -accel tcg -cpu cortex-a710 -m 1024 -smp 1 \
  -display none -monitor none -nic none -no-reboot \
  -serial file:/tmp/opencode/options-run/serial.log -device ramfb \
  -gdb tcp:127.0.0.1:12360 -S \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/aarch64/QEMU_EFI.fd \
  -drive if=pflash,format=raw,file=/tmp/opencode/options-run/vars.fd \
  -drive if=none,id=esp,format=raw,readonly=on,file=fat:out/qemu/options-test/esp \
  -device virtio-blk-pci,drive=esp
```

Late checkpoints can be observed with raw RSP hardware breakpoints:

```text
Z1,fffffff0076ac128,4   # _bsd_init
Z1,fffffff00791e210,4   # _IOKitBSDInit
Z1,fffffff00746bd28,4   # _vfs_mountroot
Z1,fffffff007972a50,4   # _panic
```

## Caveats

- The test ADT is synthetic QEMU input, not S7 Edge board data.
- Normal video console was not qualified. The panic renderer works; that is not
  proof of a normal console.
- The tested kernel is a panic-screen diagnostic build.
- The security bypass argument `amfi_get_out_of_my_way=1` was still present.
- The PMGR blocker is QEMU-specific; the S7 Edge has real PMGR hardware but still
  needs an Apple-compatible PMGR provider or AMFI policy.
