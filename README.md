# UniXNU for the Galaxy S7 Edge

Samsung Galaxy S7 Edge (SM-G935F, `hero2lte`) on Exynos 8890. Phase 1 is a
Mu-Silicium UEFI firmware with display and keypad input. XNU uses ArmCpu's
GICv2 path and still needs native storage, display, input and power drivers.

## Status

- The shared SoC tree describes eight cores, PSCI 0.2, GIC-400 and the 26 MHz
  timer. Only `cpu0` is enabled for the first bring-up.
- The Mu-Silicium firmware builds and its BootShim and keypad host tests pass.
- The standalone Mu-Silicium firmware boots on hardware: the safemem image
  reached the UEFI UI, and the bring-up probe confirmed the timer, keypad and
  counter. See `evidence/uefi-bringup-20261026.txt`.
- The UniXNU-integrated firmware builds with the loader, RAM disk and DarwinBoot
  in the FD. Its payload and hardware boot are not validated.
- `ArmCpu` now maps the GIC-400 distributor and boot CPU interface, uses PPI 27
  for the virtual timer, and sends SGI 0 through `GICD_SGIR`. Its QEMU GICv2
  self-test passes; the XNU handoff and device boot are not yet validated.
  GICv2 secondary-core bring-up is not implemented.
- EFS and CPEFS are captured and integrity-checked; see
  `evidence/efs-cpefs-backup-20261026.txt`. Do not wipe or repartition.

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

The GICv2 commits are not in `UniXNU/ArmCpu` yet, so the `drivers/ArmCpu`
submodule points at the fork commit `c9c4bd8`. Run `git submodule update
--init --recursive` after pulling and check `unixnu status`:

```sh
./unixnu status --json -D ../UniXNU-hero2lte | python3 -c \
  'import json,sys; print([m for m in json.load(sys.stdin)["modules"] if m["name"] == "ArmCpu"])'
```

`status` must report revision `c9c4bd8`. If your checkout still has the
upstream submodule pointer, either update it or use the gitignored `local.toml`
override with a sibling fork clone:

```sh
git clone https://github.com/DokuroMitsukai/ArmCpu.git ../ArmCpu
git -C ../ArmCpu checkout c9c4bd8
cat > ../UniXNU-hero2lte/local.toml <<'EOF'
[[source]]
kind = "driver"
name = "ArmCpu"
local = "../ArmCpu"
EOF
```

The override reports `"local": true` and `"origin": "local (feeds)"`; without
the GICv2 code the kernel cannot map the hero2lte GIC.

## Inputs

| Item | Value |
| --- | --- |
| SoC | Samsung Exynos 8890 (S5E8890) |
| RAM | `0x80000000` + `0x7e400000`, upper `0x880000000` + `0x80000000` |
| Framebuffer | `0xe2a00000`, `0xe10000` |
| UEFI FD | `0xb1200000`, `0x200000` |
| Boot image | v0, base `0x10000000`, kernel `0x10008000`, tags `0x10000100`, page 2048 |
| Display | s6e3hf4 `SVC_OCTA`, 1440x2560 |
| Keypad | `GPA0-3`, `GPA0-4`, `GPA0-5`, `GPA2-4`, active low |

Follow [Port a device](https://unixnu.com/porting/) and
[device repository](https://unixnu.com/porting/device-repository) when changing
the port. Record hardware results separately from host builds.
