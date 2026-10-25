# UniXNU for the Galaxy S7 Edge

Samsung Galaxy S7 Edge (SM-G935F, `hero2lte`) on Exynos 8890. Phase 1 is a
Mu-Silicium UEFI firmware with display and keypad input. XNU needs GICv2
support in ArmCpu and native storage, display, input and power drivers.

## Status

- The shared SoC tree describes eight cores, PSCI 0.2, GIC-400 and the 26 MHz
  timer. Only `cpu0` is enabled for the first bring-up.
- The Mu-Silicium firmware builds and its BootShim and keypad host tests pass.
- Hardware boot is not validated. `ArmCpu` registers GICv3 only, so no XNU
  kernel can start on this board yet.
- EFS and CPEFS have no verified backup. Do not wipe or repartition.

See [porting.toml](porting.toml) for the per-area record and current limits.

## Build

From the UniXNU checkout:

```sh
./unixnu defconfig hero2lte_defconfig -D ../UniXNU-hero2lte
./unixnu status --json -D ../UniXNU-hero2lte
python3 scripts/porting-record.py check --device ../UniXNU-hero2lte
```

The device selects the iPadOS 17.7.11 build 21H461 profile, the only profile in
this UniXNU revision. The firmware is the Mu-Silicium fork at
`Platforms/Samsung/hero2ltePkg`.

## Inputs

| Item | Value |
| --- | --- |
| SoC | Samsung Exynos 8890 (S5E8890) |
| RAM | `0x80000000` + `0x7e400000`, upper `0x880000000` + `0x80000000` |
| Framebuffer | `0xe2a00000`, `0xe10000` |
| UEFI FD | `0xc9000000`, `0x200000` |
| Boot image | v0, base `0x10000000`, kernel `0x10008000`, tags `0x10000100`, page 2048 |
| Display | s6e3hf4 `SVC_OCTA`, 1440x2560 |
| Keypad | `GPA0-3`, `GPA0-4`, `GPA0-5`, `GPA2-4`, active low |

Follow [Port a device](https://unixnu.com/porting/) and
[device repository](https://unixnu.com/porting/device-repository) when changing
the port. Record hardware results separately from host builds.
