# OpenWrt support for Open Mesh S24v3 / Datto E24v3

Upstream-ready OpenWrt device support for the **Open Mesh S24v3** (also sold as the **Datto Networking E24v3**) — a 24-port Gigabit PoE+ managed switch with 2x 10G SFP+, on the Realtek **RTL8396M** (`realtek/rtl839x` target).

This repo holds the **exact patch OpenWrt expects** for a new-device contribution, ready to `git am` onto a clone of `openwrt/openwrt` and open a Pull Request. **It is offered for upstream review** — the intent is to get this device into `openwrt/openwrt`, not to maintain a fork.

It is the RTL8396M sibling of [`openwrt-openmesh-s24-upstream`](https://github.com/theoneec/openwrt-openmesh-s24-upstream) (the rtl8382 Open Mesh S24 / Datto S24-L), and the two are built the same way.

## The commit

`realtek: add support for Open Mesh S24v3 / Datto E24v3` — exactly 2 files:

- `target/linux/realtek/dts/rtl8391_openmesh_e24.dts`
- `target/linux/realtek/image/rtl839x.mk` (adds `Device/openmesh_e24`)

Base: `openwrt/openwrt` master @ `19ff245b9233060bcbf55ec50cc3086d4348b72c`.

**Ready-to-PR fork branch:** <https://github.com/theoneec/openwrt/tree/add-openmesh-e24>

## What works

Validated on a real unit, booted from flash:

| Subsystem | Status |
|---|---|
| Boot from stock U-Boot (`boota`), no bootloader replacement | Works |
| 24x Gigabit copper (DSA, `lan1`..`lan24`) | Works |
| PoE+ — per-port enable/disable, class and delivered-power readout | Works |
| LM63 fan controller + temperature sensor | Works (needs the init script, see below) |
| LEDs (power, fault, LAN/PoE mode, PoE budget) | Works |
| Buttons (reset, LED mode) | Works (mode-button polarity inferred, see `docs/HARDWARE.md`) |
| Base MAC from `u-boot-env` `ethaddr` | Works |
| sysupgrade / flashing / recovery | Works, exercised repeatedly |

## What does not work

- **The two 10G SFP+ ports.** They sit behind an external **RTL8295R**, which has no mainline driver, so both are declared `status = "disabled"`. This is deliberate: a `fixed-link` would make the kernel report an unconditional 10G carrier on an empty cage. See `docs/HARDWARE.md` for everything recorded for a future RTL8295R implementation — this is the obvious follow-up contribution.
- **Fan control is not in this commit.** The DTS binds the LM63, but the stock bootloader leaves the chip in manual mode at PWM 0 (fans stopped), so a fan curve must be programmed at boot. That belongs in `target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol` and is held back as a separate follow-up patch — see `SUBMISSION.md`. **Do not run this device under PoE load without it.**

## Apply + build

```sh
git clone https://github.com/openwrt/openwrt.git && cd openwrt
git am /path/to/patches/0001-realtek-add-support-for-Open-Mesh-S24v3-Datto-E24v3.patch
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
| `patches/` | The `git format-patch` output — this is the contribution |
| `SUBMISSION.md` | Notes for OpenWrt maintainers: scope, exclusions, test status, open questions |
| `FORUM-POST.md` | Ready-to-paste OpenWrt forum announcement |
| `docs/HARDWARE.md` | Board map: SoC, PHYs, GPIO, LEDs, buttons, thermal, flash layout |
| `docs/POE.md` | The PoE subsystem in detail — the most reusable part for other porters |
| `docs/FLASH-AND-RECOVERY.md` | `.bix` container format, boot flow, recovery |
| `dts/`, `image/` | Convenience copies of the two files the patch contains |

## Licence

The DTS and image recipe are `GPL-2.0-or-later`, matching OpenWrt. See `LICENSE`.
