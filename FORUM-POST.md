# Add support for Open Mesh S24v3 / Datto E24v3 (RTL8396M, realtek/rtl839x)

<!-- OpenWrt forum post for a "For Developers" thread. -->

Hi all — I've written and hardware-validated OpenWrt support for the
**Open Mesh S24v3** (also sold as the **Datto Networking E24v3**), a 24-port
Gigabit PoE+ switch with 2x 10G SFP+ on the Realtek **RTL8396M**. It boots from
flash under the **unmodified stock bootloader**, with all 24 copper ports and
working PoE. I'd welcome a maintainer reviewing it for import.

It's the RTL8396M sibling of the S24 / Datto S24-L I posted earlier, and the two
are built the same way.

**Code (patch + submission notes + a ready-to-PR openwrt fork):**
- Patch + full notes: https://github.com/theoneec/openwrt-openmesh-e24-upstream
- openwrt/openwrt fork with the commit (ready to PR): https://github.com/theoneec/openwrt/tree/add-openmesh-e24

## Device

- SoC: Realtek RTL8396M ("Cypress", rtl839x); 256 MB DDR3; 32 MB SPI-NOR (MX25L256xx)
- 24x GbE copper: 3x external RTL8218B over QSGMII, MAC IDs 0-23
- 2x 10G SFP+ behind an external RTL8295R (MAC ID 24 / SerDes 8 and MAC ID 36 / SerDes 12)
- 24x 802.3af/at PoE+, 30 W per port, 410 W chassis budget
- LM63 fan controller + temperature sensor
- Senao ODM — same platform family as the already-supported rtl8380 `datto_l8`
- Stock U-Boot 2011.12, loads a `.bix` container

## The change

Two patches, independently applicable:

1. `realtek: add support for Open Mesh S24v3 / Datto E24v3` —
   `target/linux/realtek/dts/rtl8391_openmesh_e24.dts` (new) and
   `target/linux/realtek/image/rtl839x.mk` (adds `Device/openmesh_e24`).
2. `realtek: openmesh_e24: program the LM63 fan controller` — a per-board
   branch in the existing
   `target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol`, plus the
   `i2c-tools` runtime dependency in the recipe.

No new driver, no new package, no new file outside the DTS, no firmware blob.

## Notes for reviewers

**PoE** is the interesting part. The transport is a **software bit-banged I2C
bus on SoC GPIO** (SCL `gpio0` 13, SDA `gpio0` 14), modelled with `i2c-gpio` —
there's no hardware I2C controller in that path. The MCU answers at **0x20**
(not 0x21; 0x21 is an *opcode*, which cost me a while) and identifies as a
Nuvoton M05xx LAN MCU fronting BCM59121, 24 ports over 3 PSE chips. Its 12-byte
frame is byte-for-byte the dialect the in-tree `realtek,pse-mcu-gen1-smbus`
driver already speaks, so it binds with no new code. It does want about 3 ms
between the write and the read.

**The PSE port map is the trap.** The PSE channel index is `port ^ 3` — each
aligned group of four ports is reversed. The vendor firmware carries that table
literally and programs the MCU's own remap feature (opcodes `0x02` then `0x1D`)
so its userspace sees 1:1. The mainline driver implements neither opcode, so the
raw XOR-3 order is what Linux sees, and the DTS compensates by attaching
`&phy<n>` to `PSE_PI(n ^ 3)`. Confirmed by moving a PD around: before the
mapping, a PD in port 1 showed up as delivering on `lan4`.
**These are mutually exclusive fixes** — if the driver ever programs the MCU
remap, the DTS XOR has to come out at the same time or they cancel. There's a
comment saying so next to the mapping. If maintainers would rather fix it in the
driver, that's a fair call and the DTS section becomes a plain 1:1 list.

**The 10G SFP+ ports are declared `disabled`** — the RTL8295R has no mainline
driver, and the vendor picks the interface mode at runtime from the detected
module, so there's no static mode to encode. I deliberately did *not* use
`fixed-link`: it would report an unconditional 10G carrier on an empty cage,
which poisons bridging and STP. Everything needed for a future RTL8295R
implementation is recorded in the DTS comments (MAC IDs, SerDes, module EEPROMs
at 0x50 on the same bit-banged bus, presence/LOS sidebands on the RTL8231). This
is the obvious follow-up and I'm happy to work on it.

**Fan control is patch 2.** The DTS binds the LM63 at 0x4c, but the stock
bootloader leaves it in manual mode at PWM 0 — fans stopped — and binding the
hwmon driver doesn't program it, so a curve has to be written at boot. That's a
per-board function in the existing `hwmon_fancontrol`, right next to
`linksys_lgs328mpc_v2()` which is the same LM63 + PSE MCU combination. It's a
separate patch so you can judge the two independently, not because it's
optional — patch 1 alone gives you a 410 W PoE chassis with stopped fans.
One thing worth arguing about: unlike `linksys_lgs328mpc_v2()` I write the
registers with `i2cset -f` rather than through hwmon sysfs, because the vendor's
register image isn't reachable through the sysfs attributes (PFR 31 with the SCS
bit set can't be requested via `pwm1_freq_store()`, bit 0x02 of 0x4a isn't
writable at all, and `pwm1_enable=2` is refused for a two-point LUT). I
replicate the vendor's bytes rather than interpret them — the exact
temperature/duty semantics of that curve are unconfirmed — and there's an
identity gate before any write (the PoE MCU is on the same bus), read-back
verification, and a fallback to manual full PWM if anything fails to stick.
Happy to redo it as a padded eight-point sysfs LUT instead if maintainers
prefer that.

**Boot/flash:** stock U-Boot `boota` boots a magic-stamped OpenWrt uImage
(`UIMAGE_MAGIC=0x00702202`), same mechanism as `datto_l8`. The `.bix` container
is just a uImage with the magic replaced by the board ID; there's no
cryptographic signature, only the header and payload CRCs. Stock partition
names/offsets are kept and the two 0xd30000 runtime slots are mtd-concat'd into
one firmware region.

**Naming:** I used `compatible = "openmesh,e24", "datto,e24v3"` because that's
what the validated build ran with, but it's arguably the Datto name under the
Open Mesh prefix — `openmesh,s24-v3` would be more consistent and would
disambiguate it from the rtl8382 `openmesh,s24`. Happy to rename to whatever the
maintainers want.

## Hardware validation (on a real unit)

- Boots from flash under the unmodified stock bootloader; no panic, no boot loop.
- All 24 GbE enumerate as `lan1`..`lan24`; link and traffic confirmed.
- PoE: a real PD is detected, classified and powered, with per-port
  enable/disable and delivered-power/class readout via `ethtool`, and correct
  per-port targeting once the XOR mapping is applied.
- LM63 present and answering (mfr 0x01, chip 0x41); with patch 2 applied the
  vendor table is programmed and automatic fan control is active.
- LEDs and buttons behave per the board map. Base MAC read from the u-boot-env.
- Flashing and TFTP recovery exercised repeatedly.
- **Not tested: the 10G SFP+ ports** (unsupported PHY, ports disabled).
- Minor: `reboot -f` is reliable, a plain `reboot` hung once — not reproduced or
  root-caused. While looking at it I noticed `rtl838x` has an early quirk
  clearing a flash-controller 4-byte-address bit ("if this bit is not cleared
  the watchdog cannot reset the SoC") and **rtl839x has no equivalent hook**.
  That's an observation, not a diagnosis — but someone who knows the SoC will
  know instantly whether it's relevant.

## Install

1. Serial console, 115200 8n1; interrupt stock U-Boot.
2. TFTP an OpenWrt `initramfs-kernel` image into RAM and `bootm` it.
3. From the running initramfs, `sysupgrade` the `squashfs-sysupgrade.bin`.

## DCO / licensing

Both commits are `Signed-off-by` me (DCO); the DTS is SPDX `GPL-2.0-or-later`.
Happy to rebase onto current master, rename the compatible, rework the
fan-control patch, or address any other review feedback so it can be merged.

Thanks!
