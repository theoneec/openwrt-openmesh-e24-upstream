# OpenWrt support for Open Mesh S24v3 / Datto E24v3

Upstream-ready OpenWrt device support for the **Open Mesh S24v3** (also sold as the **Datto Networking E24v3**) — a 24-port Gigabit PoE+ managed switch with 2x 10G SFP+, on the Realtek **RTL8396M** (`realtek/rtl839x` target).

This repo holds the **exact patch OpenWrt expects** for a new-device contribution, ready to `git am` onto a clone of `openwrt/openwrt` and open a Pull Request. **It is offered for upstream review** — the intent is to get this device into `openwrt/openwrt`, not to maintain a fork.

It is the RTL8396M sibling of [`openwrt-openmesh-s24-upstream`](https://github.com/theoneec/openwrt-openmesh-s24-upstream) (the rtl8382 Open Mesh S24 / Datto S24-L), and the two are built the same way.

## The patches

A two-patch series. **They are independently applicable**: patch 1 is the
device support and stands alone; patch 2 is the fan bring-up and can be taken,
deferred or rewritten separately.

**`0001` — `realtek: add support for Open Mesh S24v3 / Datto E24v3`**, 2 files:

- `target/linux/realtek/dts/rtl8391_openmesh_e24.dts` (new)
- `target/linux/realtek/image/rtl839x.mk` (adds `Device/openmesh_e24`)

**`0002` — `realtek: openmesh_e24: program the LM63 fan controller`**, 2 files:

- `target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol` (adds a
  per-board branch to the existing file, the same way
  `linksys,lgs328mpc-v2` does)
- `target/linux/realtek/image/rtl839x.mk` (adds the `i2c-tools` runtime
  dependency that branch needs)

Base: `openwrt/openwrt` master @ `19ff245b9233060bcbf55ec50cc3086d4348b72c`.

**Ready-to-PR fork branch:** <https://github.com/theoneec/openwrt/tree/add-openmesh-e24>

## What works

Validated on a real unit, booted from flash:

| Subsystem | Status |
|---|---|
| Boot from stock U-Boot (`boota`), no bootloader replacement | Works |
| 24x Gigabit copper (DSA, `lan1`..`lan24`) | Works |
| PoE+ — per-port enable/disable, class and delivered-power readout | Works |
| LM63 fan controller + temperature sensor | Works — automatic fan control, programmed by patch `0002` |
| LEDs (power, fault, LAN/PoE mode, PoE budget) | Works |
| Buttons (reset, LED mode) | Works (mode-button polarity inferred, see `docs/HARDWARE.md`) |
| Base MAC from `u-boot-env` `ethaddr` | Works |
| sysupgrade / flashing / recovery | Works, exercised repeatedly |

## What does not work

- **The two 10G SFP+ ports.** They sit behind an external **RTL8295R**, which has no mainline driver, so both are declared `status = "disabled"`. This is deliberate: a `fixed-link` would make the kernel report an unconditional 10G carrier on an empty cage. See `docs/HARDWARE.md` for everything recorded for a future RTL8295R implementation — this is the obvious follow-up contribution.
That is the only functional gap.

> **Note on fan control.** The stock bootloader leaves the LM63 in manual mode
> at PWM 0 — fans stopped — and binding the hwmon driver exposes the chip
> without programming it. **Patch `0002` fixes this**, and you want it: this is
> a 410 W PoE chassis. If you apply `0001` alone, program a fan curve yourself
> before putting the switch under load.

## Apply + build

```sh
git clone https://github.com/openwrt/openwrt.git && cd openwrt
git am /path/to/patches/*.patch          # or just 0001-*.patch for device support alone
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig   # Target System: Realtek MIPS (rtl839x); select device "OpenMesh E24"
make -j$(nproc)
```

## Flash

Short version — the long version, including the `.bix` container format and recovery, is in `docs/FLASH-AND-RECOVERY.md`.

1. Serial console, 115200 8n1; interrupt stock U-Boot.
2. TFTP an OpenWrt `initramfs-kernel` image into RAM and `bootm` it.
3. From the running initramfs, `sysupgrade` the `squashfs-sysupgrade.bin`.

The stock loader validates the uImage magic word (the board ID, `0x00702202`) and two CRCs. There is no cryptographic signature, so a correctly stamped OpenWrt image boots under the unmodified stock bootloader.

## Layout

| Path | What it is |
|---|---|
| `patches/` | The `git format-patch` output — the two-patch series, this is the contribution |
| `SUBMISSION.md` | Notes for OpenWrt maintainers: scope, exclusions, test status, open questions |
| `FORUM-POST.md` | Ready-to-paste OpenWrt forum announcement |
| `docs/HARDWARE.md` | Board map: SoC, PHYs, GPIO, LEDs, buttons, thermal, flash layout |
| `docs/POE.md` | The PoE subsystem in detail — the most reusable part for other porters |
| `docs/FLASH-AND-RECOVERY.md` | `.bix` container format, boot flow, recovery |
| `dts/`, `image/`, `base-files/` | Convenience reading copies of what the patches change |

## Licence

The DTS and image recipe are `GPL-2.0-or-later`, matching OpenWrt. See `LICENSE`.
