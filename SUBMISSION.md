# PR: realtek: add support for Open Mesh S24v3 / Datto E24v3

Notes for OpenWrt maintainers reviewing this contribution.

## The series

Two patches, **independently applicable**:

1. `realtek: add support for Open Mesh S24v3 / Datto E24v3` — the device
   support. Stands alone.
2. `realtek: openmesh_e24: program the LM63 fan controller` — the fan
   bring-up. Can be taken, deferred, or rewritten without affecting patch 1.

Patch 2 is separate so the two can be judged on their own merits, **not**
because it is optional in practice: without it the board boots with stopped
fans on a 410 W PoE chassis. See "The fan patch" below.

## Base

Generated against `openwrt/openwrt` master @ `19ff245b9233060bcbf55ec50cc3086d4348b72c`.
Happy to rebase onto current master at any point.

## DCO

`Signed-off-by: James Perez-Clifton <jamesperezclifton2016@gmail.com>`

On both commits the author, the committer and the sign-off all match.

---

## What is included

Two commits, three files, **no new driver, no new package and no new file
outside the DTS**:

| Patch | File | Change |
|---|---|---|
| 1 | `target/linux/realtek/dts/rtl8391_openmesh_e24.dts` | new, 508 lines |
| 1 | `target/linux/realtek/image/rtl839x.mk` | `Device/openmesh_e24`, 13 lines |
| 2 | `target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol` | per-board branch, +165 lines |
| 2 | `target/linux/realtek/image/rtl839x.mk` | adds the `i2c-tools` runtime dependency, 1 line |

Everything the device needs at runtime is already in the tree:

- **PoE** binds the existing `realtek,pse-mcu-gen1-smbus` driver
  (`kmod-pse-realtek-mcu-i2c`). The MCU's 12-byte frame is byte-for-byte the
  dialect that driver already speaks; no new backend, no firmware blob.
- **Thermal** binds the existing `national,lm63` hwmon driver
  (`kmod-hwmon-lm63`), and patch 2 programs it from the existing
  `hwmon_fancontrol` init script — no new file.
- **Transport** to both of the above is `i2c-gpio` over two SoC GPIO lines.
- **Boot** is the stock loader, stamped-uImage mechanism already used by
  `datto_l8`; no U-Boot replacement, no new image recipe machinery.

The package list is deliberately the same shape as the in-tree
`linksys_lgs328mpc-v2`, which is the closest existing board (same LM63 + PSE
MCU pairing): `kmod-hwmon-lm63 kmod-pse-realtek-mcu-i2c`, plus `ethtool-full`
because the PSE API is the only way to drive PoE from userspace. Patch 2 adds
`i2c-tools`, which it needs at runtime.

---

## The fan patch (2/2)

**Why it exists.** The stock bootloader hands the LM63 over in manual mode
(register `0x4a` = `0x20`) with PWM value `0x00` — the fans are not driven at
all — and binding `national,lm63` only exposes the chip, it does not program
it. So patch 1 on its own produces a booting board with stopped fans, on a
chassis with a 410 W PoE budget. Patch 2 programs the vendor's lookup table at
boot so the chip runs the fans from its own temperature control.

**Why it is device support and not scope creep.**
`target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol` already carries
per-board branches for four boards, including `linksys,lgs328mpc-v2` — the same
LM63-plus-PSE-MCU pairing, with a bootloader that leaves the chip unconfigured
in the same way. Adding a branch to that existing file is exactly how this tree
does per-board fan setup.

**Honest caveats, stated in the commit message too:**

- The register values are **replicated verbatim** from the vendor firmware's
  init, in the vendor's order. The precise temperature and duty-cycle semantics
  of the resulting curve are **not confirmed** — the bytes are reproduced, not
  interpreted, and the code asserts no interpretation of them.
- **Raw `i2cset -f`, not hwmon sysfs.** The sibling `linksys_lgs328mpc_v2()`
  uses the hwmon attributes, which is the nicer interface, but the vendor's
  register image is not reachable through it on current kernels: `0x4d`
  (`PWM_FREQ`) must hold PFR 31 while `0x4a` keeps its SCS bit set, and
  `pwm1_freq_store()` derives PFR from a frequency in Hz and picks SCS itself
  (PFR 31 is only reachable from the 180 kHz base, which clears SCS; 700/22
  rounds to 32 and 700/23 to 30); bit `0x02` of `0x4a` is not writable through
  any attribute; and `pwm1_enable=2` is refused with `-EPERM` by
  `lm63_lut_looks_bad()` unless all eight LUT points are monotonic, while the
  vendor programs two. The `lm63` driver claims `0x4c`, so the `-f` is a
  requirement, not a shortcut.
  **If you would rather have a sysfs-driven curve of our own choosing** —
  padding the LUT to eight monotonic points the way `linksys_lgs328mpc_v2()`
  does, and accepting a curve that is not byte-identical to the vendor's — that
  is a reasonable alternative and the change is small. It is written this way
  because reproducing the shipped image is the conservative choice on hardware
  whose thermal design is not documented.

**Safety properties, because it writes raw registers on a shared bus:**

- The **PoE MCU shares this bus**, so nothing is written until the address has
  identified itself as an LM63: manufacturer (`0xfe`) = `0x01`, chip (`0xff`) =
  `0x41`. Otherwise it logs and writes nothing.
- The bus number is resolved from the `lm63` driver's own binding, falling back
  to probing the adapters — adapter numbering depends on probe order.
- Every write is **read back**, up to three attempts.
- **Verified fallback**: if any write does not read back, the chip is forced
  into manual mode at full PWM. A noisy switch is an acceptable failure mode;
  an unventilated one is not.
- Every path is logged and the function always returns success, so a
  fan-control problem can never fail the boot sequence.
- Re-running rewrites the same bytes — idempotent.

---

## What is deliberately excluded, and why

### 1. The 10G SFP+ ports

Both cages are `status = "disabled"`. See "Known limitations" below. This is the
main functional gap and the obvious follow-up contribution.

### 2. PoE userspace

No CLI, no LuCI page, no UCI config in this series. Per-port control and
class/power telemetry are reachable through the kernel PSE API via `ethtool`,
which is what upstream expects; anything friendlier is a downstream image
concern.

### 3. Debug tooling in `DEVICE_PACKAGES`

The locally validated build also shipped `gpiod-tools` and `tcpdump` for
bring-up. Neither is in the submitted recipe — nothing in this series needs
them. (`i2c-tools` was in that same bring-up set, but it *is* a genuine runtime
dependency of the fan patch, so patch 2 adds it. There is precedent on this
target: `tplink_sg2008p-v3` already ships `i2c-tools`.)

### 4. Anything derived from the vendor firmware image

No vendor binary, no vendor configuration, no extracted blob and no
unit-specific data is present in this repository or in either patch. The vendor
firmware was read only to recover facts — GPIO line numbers, the PSE port-map
table, the uImage board-ID word — and only those facts are reproduced, in
comments.

### 5. `board.d`

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
| Thermal | LM63 at 0x4c ACKs; manufacturer (0xFE) = 0x01, chip (0xFF) = 0x41. With patch 2 the vendor table is programmed and automatic fan control is active. |
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
2. **The fan curve values are replicated, not understood.** Patch 2 programs
   them and automatic fan control works, but the exact temperature and
   duty-cycle semantics of the vendor LUT are unconfirmed — see "The fan
   patch". And if patch 1 is taken without patch 2, the board boots with
   stopped fans.
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

### 5. Patch 2 writes the LM63 with raw I2C, not hwmon sysfs

The sibling `linksys_lgs328mpc_v2()` in the same file uses the hwmon
attributes. Patch 2 does not, and the reasons — plus the offer to rewrite it as
a padded eight-point sysfs LUT if that is preferred — are set out in full under
"The fan patch (2/2)" above. This is the most likely thing to be argued about
in that patch, so it is called out here as well.

### 6. SFP+ `disabled` vs `fixed-link`

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

Not part of the series, but written up for whoever reviews or ports next:

- `docs/HARDWARE.md` — full board map.
- `docs/POE.md` — the PoE subsystem, the wire protocol and the port-map trap.
- `docs/FLASH-AND-RECOVERY.md` — the `.bix` container, boot flow and recovery.
