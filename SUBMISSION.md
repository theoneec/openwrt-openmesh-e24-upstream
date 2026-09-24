# PR: realtek: add support for Open Mesh S24v3 / Datto E24v3

Notes for OpenWrt maintainers reviewing this contribution.

## Before anything else: this is not a standalone port

There is an active, year-old upstream effort covering exactly these devices,
which we were unaware of while doing this work:
**[Add support for Datto L8, E24v3, E48 switches](https://forum.openwrt.org/t/add-support-for-datto-l8-e24v3-e48-switches/241657)**
(*For Developers*, Oct 2025 - Apr 2026), driven by **hmartin**, with review
from **svanheule** and hardware testing from **stevewaffler**.

- The **Datto L8 is already in mainline**. hmartin's latest L8 change,
  [openwrt/openwrt#22764](https://github.com/openwrt/openwrt/pull/22764)
  (*"realtek: fixup Datto L8 device tree"*), merged 3 Apr 2026.
- hmartin has a WIP branch covering E8 / E24 / E48:
  <https://github.com/halmartin/openwrt/tree/rtl83xx-datto>, including
  `target/linux/realtek/dts/rtl8396_datto_e24.dts`. As of 26 Mar 2026 he
  reported copper, PoE, fans and LEDs working, SFP/SFP+ still WIP.
- A **GPL source archive for these exact devices already exists**:
  <https://github.com/halmartin/avalon-l2switch-realtek-rtk8382> - *"GPL source
  code for the Datto E8, E24v3, and E48 switches"*, also linked from the
  OpenWrt wiki GPL archive page. **Our reverse engineering was done by
  disassembling vendor binaries without knowing it existed.** The archive is
  incomplete, though — it carries U-Boot and a near-vanilla kernel, but not the
  fan init, the PoE stack or the board configuration, so for those three the
  disassembly is still the only source. What it *does* independently confirm
  (and the two claims of ours it forced us to correct) is set out in
  `docs/UPSTREAM-STATUS.md`.

**Treat this series as findings and fixes offered into that effort**, not as a
competing port. If a maintainer would rather see these changes land in
hmartin's branch than as a separate device commit, that is a good outcome and
we will do the work to get them there.

Three things here appear not to be solved on that branch - the fan-control root
cause (which the thread explicitly gave up on), the PSE port-map XOR, and the
single-I2C-bus correction. They are set out below and in
`docs/UPSTREAM-STATUS.md`.

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

**One value in that sequence is now experimentally confirmed as causal.**
With duty held at ~50% and `0x4A` held at `0x2A` (so SCS and bit `0x02` never
moved), varying **only** `0x4D` on a live unit gave PFR 31 (`0x1F`) → fans run,
audibly slower than full speed, i.e. partial PWM control working, and PFR 8
(`0x08`) → fans off; restoring the vendor values brought them back. So
`0x4D` (`PWM_FREQ` / PFR) determines whether sub-100% duty drives these fans at
all. We do **not** offer a physical mechanism for that — an earlier explanation
involving the 74HC123D one-shot timing out at too low a frequency was wrong and
is withdrawn, since the failing PFR 8 is ~22.5 kHz against PFR 31's reported
5806 Hz, i.e. four times *higher*. And we have not yet read `0x4D` on a
genuinely cold boot, so we cannot yet say the power-on default is a failing
value.

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
   expander. The topology is well attested: a second unit's stock U-Boot
   log prints `### RTL8295R config - MAC ID = 24 ###` and
   `### RTL8295R config - MAC ID = 36 ###`, matching our table, the boot log
   quoted in the forum thread, and both units. The vendor GPL source makes
   it more than a coincidence: the sibling 1G-fibre board file
   `rtl8382m_8218b_intphy_8218b_2fib_1g_demo_board.c` puts its two fibre ports
   at mac_id **24 and 26** (matching the L24 / S24-L), so the E24v3's **24 and
   36** is a genuine per-board difference rather than a transcription error.
   Note also that this SDK snapshot has **no 10G support at all** — the SerDes
   mode enum tops out at 5G/QSGMII/HiSGMII and `RTL8295R` gets zero hits
   archive-wide — so the GPL drop is not a research route for these cages.
2. **The fan curve values are replicated, not understood.** Patch 2 programs
   them and automatic fan control works, but the exact temperature and
   duty-cycle semantics of the vendor LUT are unconfirmed — see "The fan
   patch". And if patch 1 is taken without patch 2, the board boots with
   stopped fans.
3. **A/B failover is given up.** The two stock `RUNTIME` slots are
   `mtd-concat`'d into one firmware region so kernel + rootfs + overlay fit,
   as the upstream `datto_l8` sibling also does.
4. **LED-mode button polarity is inferred.** Documented as such in the DTS.
5. **Our generated network config references non-existent ports - our bug.**
   Because the SFP+ ports are disabled, `lan25` and `lan26` do not exist as
   netdevs on a live unit, yet the config we generate still lists them in the
   bridge VLAN:
   `network.lan_vlan.ports='lan1 ... lan24 lan25 lan26'`.
   A bridge VLAN referencing ports that do not exist is wrong and will generate
   errors. **Known issue, to be fixed before submission.** Not fixed in this
   pass, which is documentation only.
6. **Inverted power LED**, reported by stevewaffler on a real E24v3 on 6 Apr
   2026: it flashes during boot and goes off once booted;
   `/sys/class/leds/green:sys/brightness` must be `0` for solid on. Open, not
   addressed here.
7. **No factory image flashable from the stock web UI.** svanheule pointed
   hmartin at the Zyxel GS1900 recipes as the pattern - the initramfs/factory
   image must stay within the original `0xd30000` partition while the
   sysupgrade image may span the merged firmware partitions. hmartin has
   demonstrated installing OpenWrt on the L8 straight from the stock web UI
   with no UART. That is a much better install path than the TFTP procedure
   documented here, and it is not implemented in this series.

---

## Reviewer discussion points (anticipated)

### 1. KNOWN DELTA - naming: ours does not match upstream convention

**This is a known delta, not a position we are defending.** The convention
established by the merged L8 and followed by hmartin's E24 is `datto,<model>`
as the primary compatible with Open Mesh as the alt-brand. Ours is inverted.

The merged L8:

```
compatible = "datto,l8", "realtek,rtl838x-soc";
DEVICE_VENDOR      := Datto
DEVICE_MODEL       := L8
DEVICE_ALT0_VENDOR := Open Mesh
DEVICE_ALT0_MODEL  := S8-L
```

hmartin's E24, and this series, side by side:

| | hmartin / upstream convention | This series |
|---|---|---|
| DTS filename | `rtl8396_datto_e24.dts` | `rtl8391_openmesh_e24.dts` |
| `compatible` | `"datto,e24v3", "realtek,rtl8396-soc"` | `"openmesh,e24", "datto,e24v3", ...` |
| `model` | `Datto E24v3` | `Open Mesh E24v3` |
| Recipe | `Device/datto_e24` | `Device/openmesh_e24` |
| `SOC` | `rtl8396` | `rtl8391` |
| `DEVICE_MODEL` | `E24` | `E24` |
| Alt branding | `DEVICE_ALT0_VENDOR := Open Mesh` / `DEVICE_ALT0_MODEL := S24v3` | Datto in `DEVICE_ALT0_*` (inverted) |
| `UIMAGE_MAGIC` | `0x00702202` | `0x00702202` (same) |
| `IMAGE_SIZE` | `13504k` | `13504k` (same) |

Two values already agree - `UIMAGE_MAGIC` and `IMAGE_SIZE` - and were reached
independently, which is a useful cross-check on both.

**Ours predates our discovery of that convention.** Realignment to
`datto,e24v3` + `SOC := rtl8396` + the `rtl8396_datto_e24.dts` filename is
**pending, not refused**.

It is deliberately *not* done in this pass, for one reason: the compatible
string is what `board_name()` returns, and `board_name()` keys both the
board-specific **PoE budget table** and the **fan init branch** added by patch
2. Renaming is therefore a functional change to two runtime paths, and it
**must be re-validated on hardware before submission**. We would rather
document the delta honestly than ship an untested rename and have a reviewer
discover it.

Reviewers should assume the final submission carries the upstream names, and
review the *substance* rather than the strings.

### 2. `SOC := rtl8391` for an RTL8396M

The RTL8396M has no dedicated in-tree SoC token, and `rtl8391` was chosen here
because it matches the closest 24-port 839x board
(`zyxel,gs1920-24hp-v2`) and drives `DEVICE_DTS = rtl8391_openmesh_e24`.

**hmartin's branch uses `SOC := rtl8396` and
`compatible = ..., "realtek,rtl8396-soc"`,** which is the better answer and is
what we will realign to - subject to the same `board_name()` re-validation
described above.

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

### 3a. Two corrections we owe you, from mining the vendor GPL source

Recorded here because both make our own earlier statements weaker:

1. **We should not have said the bootloader "validates the magic word".**
   `include/image.h:192-196` and `:485-492` compile `image_check_magic()` out
   unless `CONFIG_ENABLE_IH_MAGIC_NUMBER_CHK` is defined, and that symbol
   appears nowhere in the archive — so as shipped in that source the check is a
   no-op returning 1. What is definitely validated is the **header CRC and the
   data CRC**; the magic is a per-board identifier we set to match, and whether
   the shipped loader enforces it is **unverified**. `UIMAGE_MAGIC` stays as it
   is either way, since matching costs nothing.
2. **We are not going to explain the magic's encoding.** `image.h:178-190`
   documents it as [b31..b12] Chip ID / [b11..b04] Vendor ID / [b03..b00]
   Product ID, set from `CONFIG_IH_MAGIC_NUMBER` (the archive's one real
   example is `83800000`, the RTL8380 chip ID). Our values do not fit: E24v3
   `0x00702202` would give a chip ID of `0x00702`, which is not a Realtek chip
   ID and is identical across two SoC families. Senao evidently repurposed the
   field. The **values** are solid — `0x00702202` read from an E24v3 flash dump,
   `0x00702201` from hmartin's posted hexdump of the oms48 image — but the
   scheme is not something we can account for.

And one operational fact worth a reviewer's attention, now confirmed from
source rather than inferred: `common/cmd_bootm.c:1660-1663` erases 4 KB (the
image header) from a partition that fails to boot, and flips the
active-partition selector. A single failed boot destroys that slot's image.

### 3b. `u-boot-env2` / sysinfo size - a question, not an accusation

The merged L8 DTS and hmartin's E24 both declare the `u-boot-env2` / sysinfo
partition as `reg = <0x90000 0x20000>`. That runs to `0xb0000` and therefore
overlaps `cfg` at `0xa0000`. The stock firmware's own partition listing, quoted
in the forum thread, gives SYSINFO as `0x090000`-`0x0a0000`, i.e. **`0x10000`**.

This series uses `0x10000`.

We may be missing something about how that region is used, so this is raised as
a question for someone who knows the platform rather than as a bug report.

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

## Two things we noticed and are explicitly NOT claiming are related

Recorded for completeness, not as a bug report. Neither is a claim.

**A warm-reset hang we saw once and cannot reproduce.** On one unit a plain
`reboot` hung once, while `reboot -f` was reliable. We have since tested a
second, independent E24v3: a plain `reboot` on it came back cleanly and fast —
the kernel printed `reboot: Restarting system`, the U-Boot banner appeared
**1.6 s later**, and it reached userspace in about 37 s. The running tally
across both units is **one hang in seven warm resets**, not reproducible, and
on the second unit the exact method that failed once works fine. That is not
enough to attribute anything to anything.

**An unrelated code asymmetry we noticed while looking at it.** In
`target/linux/realtek/patches-6.18/300-02-enhance-realtek-board-setup.patch`,
`rtl838x_apply_early_quirks()` clears `BIT(30)` of `RTL838X_PLL_CML_CTRL` with
the comment *"Disable 4 byte address mode of flash controller. If this bit is
not cleared the watchdog cannot reset the SoC."* `rtl838x` is the only SoC
family in that file with an `.apply_quirks` hook; rtl839x has no equivalent.

**We have no evidence connecting these two paragraphs**, and we are not
asserting that the missing rtl839x hook causes anything. We are not asking for
either to be actioned. They are written down only so that neither is a
surprise to anyone who later encounters them.

## Supporting documentation

Not part of the series, but written up for whoever reviews or ports next:

- `docs/UPSTREAM-STATUS.md` — **the existing upstream effort, what we got
  wrong by working in isolation, and the three findings we are offering into
  it.** Read this one first.
- `docs/HARDWARE.md` — full board map.
- `docs/POE.md` — the PoE subsystem, the wire protocol and the port-map trap.
- `docs/FLASH-AND-RECOVERY.md` — the `.bix` container, boot flow and recovery.
