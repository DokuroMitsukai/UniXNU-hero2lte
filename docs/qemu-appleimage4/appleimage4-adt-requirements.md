> **Cross-port note — not S7 Edge hardware evidence.** Produced by the QEMU
> AppleImage4 isolation pass in the Tab A7 (`gta4l`) workspace
> `unixnutaba72020test` on 2026-10-27. The tested kernel was a panic-screen
> diagnostic build
> (SHA-256 `72c6f4635767901506d65ac8598860d1dc06b5c72c5402914af2a8aa6f57952e`).
> Addresses and ADT values are source-workspace findings; re-derive them for
> `hero2lte` before use.

# AppleImage4 / AMFI device-tree requirements

Findings from a QEMU-only isolation pass in the source Tab A7 workspace. They are
reusable for the S7 Edge port, but the exact addresses and values must be
re-derived for the `hero2lte` kernel and ADT.

## Symptom

The kernel data-aborts before `_bsd_init` inside
`com.apple.security.AppleImage4`:

```
ldr x8, [x0]        ; x0 = 0
ESR_EL1 = 0x96000005
FAR_EL1 = 0
```

The requested key is:

```
40A0DDD2-77F8-4392-B4A3-1E7304206516:krn.zine
```

## Root cause

In the tested kernel:

| Item | Address |
| --- | --- |
| AppleImage4 `__TEXT_EXEC` | `0xfffffff004a8c000` – `0xfffffff004aa9fb4` |
| Fault instruction | `0xfffffff004a92828` |
| Caller return | `0xfffffff004a9e08c` |
| AppleImage4 site descriptor | `0xfffffff005cf84f0` (`"darwin arm kernel"`) |
| Site instance | `0xfffffff005cf83e8` |
| Null field | instance `+0x10` (`0xfffffff005cf83f8`) |

The site initializer stores the `/options` lookup result into instance `+0x10`:

```text
0x4a9859c  lookup "/options"
0x4a985ac  str  x0, [x20, #0x10]
0x4a985b0  ldr  x8, [x20, #0x08]   ; checks /chosen, not /options
0x4a985b4  cbz  x8, ...
```

Because the `/options` result is not checked, a missing `/options` node leaves
`instance+0x10` null. The later nonce-magazine property read dereferences it.

## Required device-tree inputs

These were sufficient to remove the AppleImage4 abort and the following
AMFI/IONVRAM panics in QEMU. Treat them as requirements to verify, not as S7 Edge
board data.

| Location | Property / node | Value used | Failure if missing |
| --- | --- | --- | --- |
| root | `options` node | empty node | AppleImage4 null dereference at the nonce-magazine read |
| `/chosen` | `nvram-bank-size` | `8192` (u32) | `IONVRAMCHRPHandler creation failed` |
| `/chosen` | `nvram-bank-count` | `1` (u32) | same |
| `/chosen` | `nvram-current-bank` | `0` (u32) | same |
| `/chosen` | `nvram-proxy-data` | 8192-byte empty CHRP bank | same |
| `/chosen` | `crypto-hash-method` | `sha2-384` (string) | AppleImage4 `non-sensical crypto hash method` |
| `/chosen` | `amfi-only-platform-code` | `1` (u32) | AMFI `no amfi-only-platform-code device tree node` |

The CHRP block can be produced by the existing fixture generator:

```sh
python3 src/UniXNU/scripts/qemu-entry/make-dt.py --nvram-proxy ... 
```

`--nvram-proxy` adds both the empty root `options` node and the CHRP bank under
`/chosen`. `--crypto-hash-method sha2-384` and `--amfi-platform-only` add the
other two inputs.

Do not fabricate the nonce value behind
`40A0DDD2-77F8-4392-B4A3-1E7304206516:krn.zine`. If `/options` exists but the key
is absent, expect a graceful `not found` result, not a null dereference.

## Boot arguments

The inherited QEMU configuration used:

```
amfi_get_out_of_my_way=1 cs_enforcement_disable=1
```

`cs_enforcement_disable=1` causes AMFI to panic:

```
"can't has cs_enforcement_disable" @%s:%d
```

The test removed only `cs_enforcement_disable=1`. `amfi_get_out_of_my_way=1`
remained, so the QEMU result is **not** a clean security-preserving boot. For the
S7 Edge port, prefer the `amfi-only-platform-code` input and a proper AMFI
policy instead of the bypass argument, then verify a clean run.

## Next blocker: AMFI requires PMGR

After `_bsd_init` and `_IOKitBSDInit`, AMFI panics:

```
"AMFI: No PMGR?\n" @%s:%d
```

The expected service is `ApplePMGRNub`. The prelinked `ApplePMGR` personality is:

| Key | Value |
| --- | --- |
| `IOClass` | `AppleBringUpPMGR` |
| `IONameMatch` | `pmgr1,bring-up` |
| `IOProviderClass` | `AppleARMIODevice` |

The synthetic QEMU ADT has no `pmgr1,bring-up` node and QEMU has no Apple PMGR
hardware. That is the QEMU decision point. The S7 Edge has real PMGR hardware,
but the port still needs an Apple-compatible PMGR node/provider or an
AMFI-compatible policy that does not require `ApplePMGRNub`.

## Checkpoints for the tested kernel

| Symbol | Address |
| --- | --- |
| `_bsd_init` | `0xfffffff0076ac128` |
| `_IOKitBSDInit` | `0xfffffff00791e210` |
| `_vfs_mountroot` | `0xfffffff00746bd28` |
| `_panic` | `0xfffffff007972a50` |

Observed order after the fixes:

```
_bsd_init -> _IOKitBSDInit -> AMFI "No PMGR?" panic
```

## Reusable debugging method

1. **Map the kext.** Parse `__PRELINK_INFO` from the kernel Mach-O, read each
   kext's `_PrelinkExecutableLoadAddr`, then read the embedded Mach-O header in
   `__PRELINK_TEXT` to get its `__TEXT_EXEC` range. This is how the faulting
   AppleImage4 address was mapped to the kext.
2. **Stop before the fault, not at the first shared helper.** A shared property
   helper can be called many times with valid objects. Use a conditional
   breakpoint on the null dereference, for example:

   ```gdb
   break *0xfffffff004a92828 if $x0 == 0
   ```

3. **Read the caller chain.** On the hit, record `x0`–`x30`, `sp`, `x29`, the
   saved frame, and the string at `x1`. The property name identifies the
   missing node.
4. **Walk the object.** For this failure, the chain was:

   ```
   site descriptor + 0x90
     -> [descriptor + 0x10]        ; site instance
     -> [instance + 0x10]          ; /options result (null)
     -> shared property getter
   ```

5. **Use raw RSP for late checkpoints.** QEMU's GDB stub supports hardware
   breakpoints (`Z1`) on `_bsd_init`, `_IOKitBSDInit`, `_vfs_mountroot` and
   `_panic`. This avoids the GDB batch/async issues seen with some `continue`
   sequences.
6. **Keep the kernel unchanged.** Patch only a copy of the ESP/ADT and compare
   the exact key, call chain and next checkpoint.

## Caveats

- The QEMU ADT is synthetic. Do not copy its values to the S7 Edge as board
  data.
- The tested kernel is a panic-screen diagnostic build, not the S7 Edge kernel.
- The panic screen is not proof of a working normal console.
- The QEMU run still has `amfi_get_out_of_my_way=1`; a clean security run is
  pending.
- The PMGR blocker is QEMU-specific. The S7 Edge still needs a real
  Apple-compatible PMGR provider or policy.
