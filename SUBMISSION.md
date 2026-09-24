# PR: realtek: add support for Open Mesh S24v3 / Datto E24v3

Notes for OpenWrt maintainers reviewing this contribution.

## Title

`realtek: add support for Open Mesh S24v3 / Datto E24v3`

## Base

Generated against `openwrt/openwrt` master @ `19ff245b9233060bcbf55ec50cc3086d4348b72c`.
Happy to rebase onto current master at any point.

## DCO

`Signed-off-by: James Perez-Clifton <jamesperezclifton2016@gmail.com>`

The commit author and the sign-off match.

---

## What is included

One commit, two files, no new driver and no new package:

| File | Change |
|---|---|
| `target/linux/realtek/dts/rtl8391_openmesh_e24.dts` | new, 508 lines |
| `target/linux/realtek/image/rtl839x.mk` | `Device/openmesh_e24`, 13 lines |

Everything the device needs at runtime is already in the tree:

- **PoE** binds the existing `realtek,pse-mcu-gen1-smbus` driver
  (`kmod-pse-realtek-mcu-i2c`). The MCU's 12-byte frame is byte-for-byte the
  dialect that driver already speaks; no new backend, no firmware blob.
- **Thermal** binds the existing `national,lm63` hwmon driver
  (`kmod-hwmon-lm63`).
- **Transport** to both of the above is `i2c-gpio` over two SoC GPIO lines.
- **Boot** is the stock loader, stamped-uImage mechanism already used by
  `datto_l8`; no U-Boot replacement, no new image recipe machinery.

The package list is deliberately the same shape as the in-tree
`linksys_lgs328mpc-v2`, which is the closest existing board (same LM63 + PSE
MCU pairing): `kmod-hwmon-lm63 kmod-pse-realtek-mcu-i2c`, plus `ethtool-full`
because the PSE API is the only way to drive PoE from userspace.

---

## What is deliberately excluded, and why

### 1. The fan-curve init entry — the one real gap

The DTS binds the LM63 but does not program it, and **the stock bootloader hands
the chip over in manual mode with PWM 0 — the fans are not driven at all.** On a
410 W PoE chassis that matters. The fix is a per-board function in the existing
`target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol`, exactly where
`linksys_lgs328mpc_v2()` already lives in that file.

It is held out of this commit to keep the device commit to DTS + recipe, and is
offered as a second patch in the same series the moment a maintainer wants it.
If the preference is one series, say so and it will be folded in. It is flagged
here rather than left quiet because a merged board with stopped fans is a worse
outcome than a noisy review.

Two things about that entry a reviewer should know in advance: it writes the
LM63 with `i2cset -f` rather than through hwmon sysfs (the vendor's register
image is not reachable through the sysfs attributes on 6.x — PWM_FREQ PFR 31
with the SCS bit set is unreachable from `pwm1_freq_store()`, and `pwm1_enable=2`
is refused by `lm63_lut_looks_bad()` for a two-point LUT), and on any write that
does not read back it falls back to manual mode at full PWM.

### 2. The 10G SFP+ ports

Both cages are `status = "disabled"`. See "Known limitations" below. This is the
main functional gap and the obvious follow-up contribution.

### 3. PoE userspace

No CLI, no LuCI page, no UCI config in this commit. Per-port control and
class/power telemetry are reachable through the kernel PSE API via `ethtool`,
which is what upstream expects; anything friendlier is a downstream image
concern.

### 4. Debug tooling in `DEVICE_PACKAGES`

The locally validated build also shipped `i2c-tools`, `gpiod-tools` and
`tcpdump` for bring-up. They are not in the submitted recipe — none of them is
needed by anything in this commit. (`i2c-tools` becomes a real dependency if and
when the fan-curve patch above lands, and will be added with it.)

### 5. Anything derived from the vendor firmware image

No vendor binary, no vendor configuration, no extracted blob and no
unit-specific data is present in this repository or in the patch. The vendor
firmware was read only to recover facts — GPIO line numbers, the PSE port-map
table, the uImage board-ID word — and only those facts are reproduced, in
comments.

### 6. `board.d`

None required.

- `01_leds`: the realtek target's `01_leds` has an entirely empty `case` — every
  realtek board declares its LEDs in the DTS, as this one does.
- `05_compat-version`: gates sysupgrade across breaking changes; a new board
  starts at the current version.
- `02_network`: the generic DSA default produces a correct config. The base MAC
  is read from the `u-boot-env` `ethaddr` through DTS nvmem, and the device
  comes up with its factory MAC. If per-port or label-derived MACs are wanted, a
  MAC-assignment entry matching the sibling 24-port boards can be added — a
  reviewer preference, not a functional gap.

---

## Test status per subsystem

Validated on a real unit running an OpenWrt build of this patch from flash.

| Subsystem | Result |
|---|---|
| Boot | Boots from flash under the **unmodified stock U-Boot 2011.12** via `boota`. No panic, no boot loop. |
| Flash / sysupgrade / recovery | Exercised repeatedly, including TFTP-to-RAM recovery. Stock partition table and U-Boot environment intact afterwards. |
| 24x Gigabit copper | All 24 enumerate as DSA user ports `lan1`..`lan24`; link and traffic confirmed. |
| PoE — transport | `i2c-gpio` bus comes up; MCU ACKs at 0x20 and answers the identify command. |
| PoE — delivery | A real PD is detected, classified and powered; per-port enable/disable works; delivered power and class read back via `ethtool`. |
| PoE — port mapping | Confirmed by moving a PD between ports: the raw channel order is port ^ 3, and the DTS mapping corrects it (see below). |
| Thermal | LM63 at 0x4c ACKs; manufacturer (0xFE) = 0x01, chip (0xFF) = 0x41. Fan control active once the init entry programs the curve. |
| LEDs | Power, fault, LAN/PoE-mode and PoE-budget behave per the board map. |
| Buttons | Reset works. LED-mode button works; its `GPIO_ACTIVE_LOW` polarity is inferred from the vendor handler, not bench-measured. |
| MAC address | Correct factory MAC from `u-boot-env` `ethaddr` via nvmem. |
| **10G SFP+** | **Not tested — unsupported PHY, ports disabled.** |

### dtc

Clean. The compiled DTS produces exactly four warnings — `unit_address_vs_reg`,
`avoid_unnecessary_addr_size`, `interrupt_provider`, `interrupt_map` — all four
originating in `rtl839x.dtsi`, and the in-tree `rtl8391_zyxel_gs1920-24hp-v2`
produces the identical set. Nothing is introduced by this board.

---

## Known limitations

1. **10G SFP+ ports are disabled.** Both cages sit behind an external
   **RTL8295R**, which has no mainline driver. The vendor firmware picks the
   interface mode at runtime from the detected module (10GBase-R, falling back
   to 1000BX/100BX), so there is no static mode to encode.
   They are left `disabled` rather than given a `fixed-link` on purpose: a
   `fixed-link` reports carrier unconditionally, so an empty cage would look
   like a live 10G link and poison bridging and STP decisions.
   Everything needed for a future RTL8295R implementation is recorded in the
   DTS and in `docs/HARDWARE.md`: MAC ID 24 (SerDes 8) and MAC ID 36
   (SerDes 12), MIIM port indices 25/26, module EEPROMs at I2C 0x50 on the
   shared bit-banged bus, and the presence/LOS sidebands on the RTL8231
   expander.
2. **Fan curve not programmed by this commit** — see exclusion 1.
3. **A/B failover is given up.** The two stock `RUNTIME` slots are
   `mtd-concat`'d into one firmware region so kernel + rootfs + overlay fit,
   as the upstream `datto_l8` sibling also does.
4. **LED-mode button polarity is inferred.** Documented as such in the DTS.

---

## Reviewer discussion points (anticipated)

### 1. Naming and `compatible`

The unit is branded **Open Mesh S24v3** and **Datto Networking E24v3**; the DTS
uses `compatible = "openmesh,e24", "datto,e24v3"`, `model = "Open Mesh E24v3"`
and the recipe uses `DEVICE_VENDOR := OpenMesh` / `DEVICE_MODEL := E24` with
`DEVICE_ALT0_*` for the Datto branding.

`openmesh,e24` is the string the hardware-validated build ran with, which is why
it is what is submitted, but it is arguably the Datto name under the Open Mesh
vendor prefix. The consistent alternative is `openmesh,s24-v3` — which also
disambiguates it from the rtl8382 `openmesh,s24`. **Happy to rename to whatever
the maintainers prefer; it is a pure string change.** Flagging it rather than
guessing.

### 2. `SOC := rtl8391` for an RTL8396M

The RTL8396M has no dedicated in-tree SoC token. The true part is read from the
chip-ID register at runtime, so `rtl8391` is used because it matches the closest
24-port 839x board (`zyxel,gs1920-24hp-v2`) and drives
`DEVICE_DTS = rtl8391_openmesh_e24`. If a maintainer would rather add an
`rtl8396` token, that is a separate and welcome change.

### 3. The PSE port-map XOR — the one that needs a decision

The PSE channel index on this board is `port ^ 3`: each aligned group of four
ports is reversed. The vendor firmware carries that table literally in
`custom.ko` .rodata (`03 02 01 00 | 07 06 05 04 | ...`) and programs the MCU's
own remap feature (opcode `0x02` to enable, then `0x1D` to push the table) so
its userspace sees a 1:1 view.

**The mainline driver implements neither opcode**, so the raw XOR-3 order is
what Linux sees, and this DTS compensates by attaching `&phy<n>` to
`PSE_PI(n ^ 3)`. Confirmed on hardware: before the mapping, a PD in front-panel
port 1 was powered but reported as delivering on `lan4`.

**These are two mutually exclusive fixes.** If the driver ever gains opcodes
`0x02` + `0x1D` and programs the MCU remap itself, this XOR **must** be removed
or the two corrections cancel and re-scramble the mapping. The DTS carries that
warning in a comment next to the mapping. If the maintainers would rather fix it
in the driver, that is a reasonable call and this DTS section becomes a plain
1:1 list.

### 4. The extra `compatible` on the PSE node

The PSE node is `compatible = "openmesh,e24-pse", "realtek,pse-mcu-gen1-smbus"`.
The first string is not in any binding; it binds via the fallback. Trivially
dropped if preferred.

### 5. SFP+ `disabled` vs `fixed-link`

Reviewers of the sibling S24 asked about the `fixed-link` idiom. The reasoning
here is the opposite and is set out under "Known limitations" — on this board a
`fixed-link` would be actively wrong, because the cages are real 10G cages that
are genuinely empty most of the time.

---

## One observation for a maintainer's eye (not a diagnosis)

On this device `reboot -f` is reliable, while a plain `reboot` hung once.

While looking at that, one asymmetry stood out in the target itself. In
`target/linux/realtek/patches-6.18/300-02-enhance-realtek-board-setup.patch`,
`rtl838x_apply_early_quirks()` clears `BIT(30)` of `RTL838X_PLL_CML_CTRL` with
the comment:

> *"Disable 4 byte address mode of flash controller. If this bit is not cleared
> the watchdog cannot reset the SoC."*

`rtl838x` is the only SoC family in that file with an `.apply_quirks` hook —
**rtl839x has no equivalent.** Whether rtl839x has the same flash-controller bit,
whether it matters there, and whether it has anything at all to do with the
`reboot` hang, are all open questions this contribution is not in a position to
answer. It is raised purely because a maintainer who knows the SoC will resolve
it in a minute and it would be wasteful not to mention it.

---

## Supporting documentation

Not part of the patch, but written up for whoever reviews or ports next:

- `docs/HARDWARE.md` — full board map.
- `docs/POE.md` — the PoE subsystem, the wire protocol and the port-map trap.
- `docs/FLASH-AND-RECOVERY.md` — the `.bix` container, boot flow and recovery.
