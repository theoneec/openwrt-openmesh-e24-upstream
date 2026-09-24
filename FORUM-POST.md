# Reply to: "Add support for Datto L8, E24v3, E48 switches"

<!--
  Ready-to-paste REPLY into the existing OpenWrt forum thread:
  https://forum.openwrt.org/t/add-support-for-datto-l8-e24v3-e48-switches/241657
  For Developers. Not a new thread.
-->

@hmartin @svanheule @stevewaffler @janh — I came to these switches from the
Open Mesh side (S24v3 = E24v3, S24 = S24-L) and, embarrassingly, did the whole
thing without finding this thread first. I disassembled the vendor binaries to
recover the board facts, when @hmartin's GPL drop
(https://github.com/halmartin/avalon-l2switch-realtek-rtk8382) was public the
entire time. My fault, and I'm not going to pretend otherwise.

So rather than post a competing port, here are the few things I have that this
thread doesn't — starting with the two it seems stuck on. Everything below is
from two real E24v3 units.

Full write-ups: https://github.com/theoneec/openwrt-openmesh-e24-upstream
(`docs/UPSTREAM-STATUS.md` is the one aimed at this thread).

---

## 1. The fans: it isn't a GPIO, it's the LM63's PWM frequency

@hmartin, on the cold-boot-any-PWM-below-255-kills-the-fans problem you wrote:

> Someone else will have to look into why PWM control of fans doesn't work on
> these models, I feel like I've wasted enough time on this.

I think this is it. The vendor firmware **programs the LM63's registers** and
OpenWrt doesn't. Recovered byte-for-byte from the vendor `initd`:

```
0x4A = 0x2A   # unlock LUT
0x4D = 0x1F   # PWM_FREQ -> PFR 31
0x4F = 0x0A   # LUT temp 1
0x50 = 0x0A   # LUT pwm  1
0x51 = 0x1F   # LUT temp 2
0x52 = 0x31   # LUT pwm  2
0x53 = 0x3E   # LUT max pwm
0x4A = 0x0A   # enable automatic LUT control
```

The mechanism is in `drivers/hwmon/lm63.c`:

```c
chip = (pwm1 * pwm1_freq * 2 + 127) / 255;
```

`pwm1_freq` is register `0x4D` as read at probe, and `pwm_highres` is only ever
set for the **LM96163** — so a plain **LM63 always takes this scaled path**.
Full scale is `2 * PWM_FREQ`. The vendor's PFR 31 therefore gives full scale
**62 = 0x3E**, which is exactly the LUT's max-PWM byte above. The vendor's
table and the vendor's PWM frequency are one consistent design; take `0x4D`
away and the table stops meaning what it says.

Corroborated on a live unit: with `pwm1 = 128` and `PWM_FREQ = 0x1F`, register
`0x4C` reads back `0x1F` — `(128 × 31 × 2 + 127) / 255 = 31`, exactly as the
formula predicts.

**And unlike last time I wrote this up, it's now an experiment rather than a
hypothesis.** Single variable, on a live unit: duty held at ~50%, clock base
held constant (`0x4A` = `0x2A` throughout, so SCS and bit `0x02` never moved),
**only `0x4D` changed**:

```
0x4D = 0x1F  (PFR 31)  ->  fans RUN, and audibly slower than full speed
0x4D = 0x08  (PFR 8)   ->  fans OFF
```

Restoring the vendor values through the init script brought all eight registers
back and the fans with them. So: **`0x4D` decides whether sub-100% duty drives
these fans at all.** That's the thing you were hunting as a missing GPIO.

**I owe you a correction on the *why*, though.** In an earlier draft I claimed
the 74HC123D one-shot was timing out between edges because the frequency was
too low. That's backwards, and I'm withdrawing it rather than patching it up.
The driver reports **5806 Hz at PFR 31**, which fits `freq = 180000 / PFR`, so
PFR 8 is about **22.5 kHz** — four times *higher*. We stopped the fans by
**raising** the frequency, not lowering it. I have the causal register; I do
not have the physical mechanism, and I'm not going to invent a second story to
replace the first one.

**The gap I haven't closed:** I've shown frequency matters, but I have *not*
shown that the LM63's power-on default PFR is one of the failing values. That
needs a read of `0x4D` on a genuinely cold boot, before anything writes to the
chip, and I haven't managed it. So the link to your cold-boot-fails /
warm-boot-works symptom is strongly suggested but not proven. One `i2cget` on a
cold unit would settle it — and you have the hardware and the scope.

One practical note: **this can't be done through hwmon sysfs.**
`pwm1_freq_store()` derives PFR from Hz and chooses SCS itself, so it can't
produce PFR 31 with SCS set; bit `0x02` of `0x4A` isn't writable through any
attribute; and `pwm1_enable=2` is refused by `lm63_lut_looks_bad()` for a
two-point LUT. So it's `i2cset -f`, with an identity gate (the PoE MCU is on
the same bus), read-back verification, and a fallback to manual full PWM.

## 2. PoE: the channel index is `port ^ 3`

@stevewaffler — your 6 Apr report of PoE delivering power while
`ubus call poe info` shows 0 consumption and ports as "unknown" looks like this
to me.

I'm driving PoE with the **in-tree kernel PSE driver**,
`realtek,pse-mcu-gen1-smbus`, at 7-bit address **0x20** (not 0x21 — 0x21 is an
*opcode*, which cost me a while). It enumerates as:

```
Nuvoton M05xx LAN MCU, BCM59121 (id 0xe121), 24 ports across 3 PSE chip(s)
```

on both units. And the mapping:

> **The PSE channel index is the port index XOR 3** — each aligned group of
> four ports is reversed.

The vendor firmware carries the table literally in `custom.ko` `.rodata`:

```
03 02 01 00 | 07 06 05 04 | 0b 0a 09 08 | ...
```

…and then programs **the MCU's own remap feature** (opcode `0x02` to enable,
`0x1D` to push the table) so vendor userspace sees a clean 1:1 view. The
mainline driver implements **neither opcode**, so what Linux sees is the raw
XOR-3 order. My DTS compensates by attaching `PSE_PI(n ^ 3)` to `&phy<n>`.

Confirmed by moving a PD around: before the fix, a PD on physical port 1
reported as delivering on `lan4`; after, `lan1` reads *delivering power*,
`lan4` reads *searching*, and `lan1` is the only delivering port of 24.

**These are two mutually exclusive fixes.** Either the DTS carries the XOR, or
the driver learns opcodes `0x02`/`0x1D` and programs the MCU remap — do both
and they cancel and the mapping is scrambled again. I genuinely don't mind
which way it goes; if a maintainer would rather fix it in the driver, my DTS
section collapses to a plain 1:1 list. It just has to be one or the other, in
one commit.

## 3. One I2C bus, not two

@hmartin, your E24 DTS puts the LM63 on a separate `i2c-gpio-2`. On my
hardware there's only one adapter. `i2cdetect -y -r 0` on `i2c-gpio-0` shows
exactly `0x20` (PSE MCU) and `0x4c` (LM63) and nothing else.

The vendor firmware agrees independently: its "SMI group 6" — SCL on SoC
`gpio0` line 13, SDA on line 14 — carries the PoE MCU at `0x20`, the LM63 at
`0x4C`, **and both SFP EEPROMs at 0x50**. That last one may be useful to you
for the SFP work.

This also matters for anything writing the LM63: the PSE MCU is on the same
bus, hence the identity gate before any register write.

## 4. A question about `u-boot-env2` / sysinfo

The merged L8 DTS and your E24 both declare it as `reg = <0x90000 0x20000>`.
That runs to `0xb0000` and overlaps `cfg` at `0xa0000`, and the stock
partition listing quoted earlier in this thread gives SYSINFO as
`0x090000`–`0x0a0000`, i.e. `0x10000`. I've used `0x10000`.

I might well be missing something about how that region is used — raising it as
a question, not a claim.

## 5. Naming — I have it wrong and I know it

Mine is `rtl8391_openmesh_e24.dts` with
`compatible = "openmesh,e24", "datto,e24v3", ...`, `SOC := rtl8391`, and Open
Mesh as the primary vendor with Datto in `DEVICE_ALT0_*`. Yours is
`rtl8396_datto_e24.dts`, `compatible = "datto,e24v3", "realtek,rtl8396-soc"`,
`Device/datto_e24`, `SOC := rtl8396`, Open Mesh as the alt — which matches the
merged L8's `datto,l8` + `Open Mesh` / `S8-L`, so yours is the right one.

I'm realigning to `datto,e24v3` / `rtl8396`. Not in this pass, because it moves
`board_name()`, and `board_name()` keys both my PoE budget table and the fan
init — so it's a functional change I want to re-validate on hardware rather
than a sed. Flagging it so nobody wastes time reviewing the old strings.

Two things that already agree between us and were reached independently, which
I take as a decent cross-check on both:
`UIMAGE_MAGIC := 0x00702202` and `IMAGE_SIZE := 13504k`.

## 6. Smaller things, and corrections to my own notes from this thread

- @janh's `abortboot` finding: the autoboot interrupt string is **`pac`** — the
  letters `a`, `p`, `c` in any order. My docs said "apc"; same letters, now
  with a source.
- The root-shell recipe from this thread (`rdinit=/bin/sh`, unquoted
  `bootargs`, then `rm /bin/cli` and `/etc/rc`) supersedes what I'd written
  about the console being owned by a login prompt. Thank you.
- `.bix` magic family, for the record: E24v3 `0x00702202`, E48 `0x00702201`,
  S24-L/L24 `0x00702400`. Standard U-Boot legacy uImage with the magic word
  replaced; no cryptographic signature, just magic + header CRC + payload CRC.
- A second unit's stock U-Boot log prints
  `### RTL8295R config - MAC ID = 24 ###` and
  `### RTL8295R config - MAC ID = 36 ###`, which matches the boot log quoted
  earlier in this thread and the MAC ID / SerDes table in my DTS comments. So
  that topology is confirmed on two units plus the vendor bootloader, if it
  helps the SFP+ work.
\1, including
  `mphiddenpoe poe power-budget <1-999>` — relevant to the budget discussion,
  since it shows the vendor treats the chassis budget as a settable value with
  no per-unit storage.
- @svanheule — thanks for the GS1900 pointer; that's the route I'll take for a
  stock-web-UI-flashable factory image (initramfs/factory inside the original
  `0xd30000`, sysupgrade spanning the merged firmware partitions). @hmartin
  already showed this works on the L8 with no UART at all, which is a much
  better install story than the TFTP dance I've been documenting.
- @stevewaffler's inverted power LED on the E24v3 reproduces here in effect —
  `/sys/class/leds/green:sys/brightness` has to be `0` for solid on. Not fixed
  in my tree yet either.

## One defect of my own, for completeness

On a live unit running my image, `lan25`/`lan26` don't exist as netdevs — I
deliberately disable the SFP+ ports because the RTL8295R has no mainline driver
— **yet my generated network config still lists them in the bridge VLAN**
(`network.lan_vlan.ports='lan1 ... lan24 lan25 lan26'`). A bridge VLAN
referencing non-existent ports is wrong and throws errors. Known, on my list.

---

Happy to rebase, rename, split, or drop anything. @hmartin, your branch is
further along than mine on the family as a whole — I'd rather these findings
land in yours than maintain a parallel tree. Tell me what shape you want them
in.
