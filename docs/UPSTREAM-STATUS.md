# Upstream status — there is already an effort for these devices

**Read this first.** This repository was written before we found the existing
upstream work on exactly these switches. It is not a competing port. What
follows is an honest account of what already exists, who is doing it, what we
got wrong by working in isolation, and what we think we can usefully hand over.

## The existing effort

OpenWrt forum, *For Developers*:
**[Add support for Datto L8, E24v3, E48 switches](https://forum.openwrt.org/t/add-support-for-datto-l8-e24v3-e48-switches/241657)**
— running since October 2025 and still active in April 2026.

People in it, and what they have been doing:

| Who | Role |
|---|---|
| **hmartin** (Hal Martin) | Doing the bring-up work across L8 / E8 / E24 / E48, and the source of the GPL archive below |
| **svanheule** | Realtek target maintainer; review and image-recipe guidance |
| **stevewaffler** | Testing hmartin's E24v3 images on real E24v3 hardware |
| andyboeh, janh, plappermaul, NRoach44, RaylynnKnight | U-Boot analysis, flashing, general discussion |

### What is already done there

- **The Datto L8 is supported in mainline OpenWrt** — `rtl8380_datto_l8.dts`,
  with Open Mesh S8-L as the alt-branding. hmartin's most recent L8 change,
  [openwrt/openwrt#22764](https://github.com/openwrt/openwrt/pull/22764)
  (*"realtek: fixup Datto L8 device tree"*, `ports` → `ethernet-ports` for the
  6.18 kernel), was merged on 3 April 2026.
  *(If you are reading this expecting #22764 to be the commit that first added
  the L8 — it is not; it is a later fixup. The L8 was already in tree.)*
- hmartin has a work-in-progress branch covering the whole family:
  **<https://github.com/halmartin/openwrt/tree/rtl83xx-datto>**, containing
  `rtl8380_datto_l8.dts`, `rtl8380_datto_e8.dts`,
  **`target/linux/realtek/dts/rtl8396_datto_e24.dts`** and
  `rtl8393_datto_e48.dts`.
  As of 26 March 2026 he reported copper, PoE, fans and LEDs working on the
  E24, with SFP/SFP+ still WIP.
- hmartin demonstrated installing OpenWrt on the **L8 directly from the stock
  web UI** — no UART, no TFTP.

### The GPL source archive — which we did not know about

**<https://github.com/halmartin/avalon-l2switch-realtek-rtk8382>** —
*"GPL source code for the Datto E8, E24v3, and E48 switches"*. It is also
linked from the OpenWrt wiki's GPL source archive page.

We did our reverse engineering by **disassembling the vendor binaries**, with no
idea this existed. Every board fact in `docs/HARDWARE.md` and `docs/POE.md` was
recovered the hard way — from `custom.ko` `.rodata`, from the vendor `initd`,
from `libcustom.so.0` — when a source archive was sitting in public the whole
time. That was our mistake, and anyone picking this work up should start from
the archive, not from our disassembly. (Note the archive's own caveat: the
drop is incomplete and does not build as shipped, so the disassembly-derived
facts here still have some value as corroboration.)

## Where that leaves this repository

Reframed: **a set of findings and fixes offered into hmartin's effort**, not a
port competing with it. Three things in here are, as far as we can tell, not
solved on that branch, and one is the thing the thread explicitly gave up on.

### 1. The fan-control root cause

hmartin found the symptom and the 74HC123D, probed it, hunted for a missing
GPIO, and wrote:

> *"Someone else will have to look into why PWM control of fans doesn't work on
> these models, I feel like I've wasted enough time on this."*

The symptom as he described it: from a **cold** boot, any PWM below 255 stops
the E24/E8 fans dead; after a **warm** reboot out of the Datto firmware,
partial PWM control works.

**It is not a GPIO.** The vendor firmware programs the LM63's registers and
OpenWrt does not. Recovered byte-for-byte from the vendor `initd`:

| Register | Value | Meaning |
|---|---|---|
| `0x4A` | `0x2A` | unlock the lookup table |
| `0x4D` | `0x1F` | `PWM_FREQ` — PFR 31 |
| `0x4F` | `0x0A` | LUT temp 1 |
| `0x50` | `0x0A` | LUT PWM 1 |
| `0x51` | `0x1F` | LUT temp 2 |
| `0x52` | `0x31` | LUT PWM 2 |
| `0x53` | `0x3E` | LUT max PWM |
| `0x4A` | `0x0A` | enable automatic LUT control |

**Mechanism, from the kernel driver source.** `drivers/hwmon/lm63.c` writes

```c
chip = (pwm1 * pwm1_freq * 2 + 127) / 255;
```

where `pwm1_freq` is register `0x4D` as read at probe. `pwm_highres` is only
ever set for the **LM96163**, so a plain **LM63 always takes this scaled path**.
Full scale is therefore `2 * PWM_FREQ` — and the vendor's PFR 31 gives full
scale **62 = 0x3E**, which is exactly the vendor LUT's maximum PWM value. The
vendor's table and the vendor's PWM frequency are one consistent design.

**Corroborated on live hardware:** with `pwm1 = 128` and `PWM_FREQ = 0x1F`,
register `0x4C` reads back `0x1F` — `(128 × 31 × 2 + 127) / 255 = 31`, exactly
what the formula predicts.

**Confirmed on hardware — single-variable A/B.** This is no longer a theory.
On a live unit, with the duty cycle held constant at ~50% and the clock base
held constant (`0x4A` = `0x2A` throughout, so SCS and bit `0x02` never moved),
**only register `0x4D` was varied**:

| `0x4D` | PFR | Observed |
|---|---|---|
| `0x1F` | 31 | **Fans run**, and audibly *slower* than full speed — partial PWM control working |
| `0x08` | 8 | **Fans off** |

Restoring the vendor values through the init script returned all eight
registers to the table above and the fans came back.

> **The result: LM63 register `0x4D` (`PWM_FREQ` / PFR) determines whether
> sub-100% duty drives these fans at all.** That is the thing hmartin spent
> hours hunting as a missing GPIO. It is a register, and it is this one.

**We have withdrawn our first explanation of *why*, because it was wrong.** We
had suggested the 74HC123D one-shot was timing out between edges at too low a
frequency. That is backwards. The driver reports **5806 Hz at PFR 31**, which
fits `freq = 180000 / PFR`, so PFR 8 is about **22.5 kHz** — four times
*higher*, not lower. We stopped the fans by **raising** the frequency. The
one-shot-timeout story is deleted rather than patched up; we do not have a
physical mechanism, only the causal register, and we would rather say so.

**The gap we have not closed.** We have shown that PWM frequency decides
whether partial duty works. We have **not** shown that the LM63's *power-on
default* PFR is one of the failing values. Establishing that needs a read of
`0x4D` on a genuinely cold boot, before any init script runs, and we have not
been able to do it yet. So the chain to hmartin's cold-boot-fails /
warm-boot-works symptom is **strongly suggested but not closed**.

@hmartin: that last step is one `i2cget` on a cold-booted unit, and you have
the hardware and a scope. If the cold default PFR turns out to be a low-order
value like 8, the whole symptom is explained and the fix is one register.

**Why it needs raw register writes and cannot be done through hwmon sysfs:**

- `pwm1_freq_store()` derives PFR from a frequency in Hz and chooses the SCS
  bit itself; it cannot produce **PFR 31 with SCS set**.
- Bit `0x02` of register `0x4A` is not writable through any hwmon attribute.
- `pwm1_enable=2` is refused by `lm63_lut_looks_bad()` for a two-point LUT, and
  the vendor programs two points.

Hence `i2cset -f`. Patch `0002` in this repo does it with an identity gate
(the PoE MCU is on the same bus), read-back verification, and a fallback to
manual full PWM.

### 2. PoE through the kernel PSE driver, and the port-mapping discovery

hmartin's E24 recipe uses **`realtek-poe`** (userspace). stevewaffler tested it
on a real E24v3 on 6 April 2026 and reported that PoE works, but
`ubus call poe info` shows **0 consumption** and the ports listed as
**"unknown"** while power is actually being delivered.

We drive it instead with the **in-tree kernel driver**
`realtek,pse-mcu-gen1-smbus` at **7-bit I2C address 0x20**, and it enumerates
as *"Nuvoton M05xx LAN MCU, BCM59121 (id 0xe121), 24 ports across 3 PSE
chip(s)"* — confirmed on two separate units.

And here is the discovery that we think explains his "unknown" ports:

> **The PSE channel index is the port index XOR 3.** Each aligned group of four
> ports is reversed.

The vendor firmware carries the table literally in `custom.ko` `.rodata`:

```
03 02 01 00 | 07 06 05 04 | 0b 0a 09 08 | ...
```

and then programs **the MCU's own remap feature** (opcode `0x02` to enable,
`0x1D` to push the table) so that vendor userspace sees a clean 1:1 view.
**The mainline kernel driver implements neither opcode**, so the raw XOR-3
order is what Linux sees, and the device tree has to compensate by attaching
`PSE_PI(n ^ 3)` to `&phy<n>`.

Confirmed on hardware: before the fix, a PD on physical port 1 reported as
`lan4`; after, `lan1` reads *delivering power* and `lan4` reads *searching*,
with `lan1` the only delivering port of 24.

> **Warning — these are two mutually exclusive fixes.** Patch the DTS **or**
> replicate the vendor MCU init in the driver. Do both and they cancel, and the
> mapping is scrambled again. Whoever fixes it in one place must remove the
> other in the same commit.

### 3. The LM63 and the PSE MCU share one I2C bus

hmartin's E24 DTS puts the LM63 on a separate `i2c-gpio-2`. On real hardware
there is **one adapter**, `i2c-gpio-0`, carrying both devices:
`i2cdetect -y -r 0` shows exactly `0-0020` (PSE MCU) and `0-004c` (LM63), and
nothing else.

The vendor firmware agrees independently: its **"SMI group 6"** (SCL = SoC
`gpio0` line 13, SDA = line 14) carries the PoE MCU at `0x20`, the LM63 at
`0x4C`, and **both SFP EEPROMs at 0x50**.

Offered as a correction to his DTS. It also matters for anything that writes
the LM63 — see the identity gate in patch `0002`.

### 3b. RTL8295R / SFP+ topology, confirmed on a second unit

A second E24v3's **stock U-Boot log** prints:

```
### RTL8295R config - MAC ID = 24 ###
### RTL8295R config - MAC ID = 36 ###
```

That independently confirms the SFP+ topology recorded in our DTS comments and
`docs/HARDWARE.md` — MAC ID 24 (SerDes 8) and MAC ID 36 (SerDes 12) behind an
external RTL8295R — and it matches the boot log quoted earlier in the forum
thread. Confirmed now on two units *and* by the vendor's own bootloader output,
which should be enough for whoever takes on the RTL8295R work.

### 4. A possible partition bug, raised as a question

The merged L8 DTS and hmartin's E24 DTS both declare the `u-boot-env2` /
sysinfo partition as:

```
reg = <0x90000 0x20000>;
```

`0x90000 + 0x20000 = 0xB0000`, which runs into `cfg` at `0xA0000`. The stock
firmware's own partition listing — quoted in the thread — gives SYSINFO as
`0x090000`–`0x0a0000`, i.e. **`0x10000`**. Ours uses `0x10000`.

We may well be missing something about how that region is used. Raising it as a
question for someone who knows the platform, not as an accusation.

## Forum-derived corrections to our own documentation

These came out of the thread and supersede what we had written:

- **U-Boot autoboot interrupt is `pac`** — the letters **a**, **p**, **c** in
  any order. janh found it in `abortboot` in the U-Boot source. Our docs said
  "apc"; same letters, but now with a source and a mechanism.
- **Root shell on the stock firmware.** Interrupt U-Boot, then:
  ```
  sf read $(freemem) $(flashoffset_linux) $(ssize_linux)
  setenv bootargs console=ttyS0,115200 mem=256M rdinit=/bin/sh
  bootm $(freemem)
  ```
  **Do not quote the `bootargs` value.** At the early shell, `rm /bin/cli` then
  run `/etc/rc`, and the box boots on into a root shell instead of the vendor
  CLI. This supersedes anything we wrote about the console being owned by a
  login prompt.
- **Stock default credentials after a factory reset:** `admin` / `0p3nm3$h!`
  (published in the thread).
- **Hidden vendor CLI commands** live in `libcustom.so.0`, including
  `mphiddenpoe poe power-budget <1-999>`. Relevant to the PoE budget question:
  the vendor treats the chassis budget as a **settable value with no per-unit
  storage**, which argues against our board-keyed budget table being the only
  correct model.
- **`.bix` board magic family:**

  | Device | `UIMAGE_MAGIC` |
  |---|---|
  | E24v3 | `0x00702202` |
  | E48 | `0x00702201` |
  | S24-L / L24 | `0x00702400` |

  The container is a standard U-Boot legacy uImage with the magic word
  replaced. There is **no cryptographic signature** — only the magic, the
  header CRC and the payload CRC.
- **svanheule's guidance on images**, answering hmartin directly: the **Zyxel
  GS1900 recipes** are the pattern to follow — the initramfs/factory image must
  stay within the original `0xd30000` partition, while the sysupgrade image may
  span the merged firmware partitions. That is the route to a factory image
  flashable from the stock web UI, which is how hmartin installed on the L8.
- **Open issue on E24v3, from stevewaffler (6 April 2026):** the power LED is
  **inverted** — it flashes during boot and then goes off once booted.
  `/sys/class/leds/green:sys/brightness` has to be `0` for it to be solid on.

## Known delta — our naming does not match upstream convention

See the matching section in `README.md` and `SUBMISSION.md`. Short version: the
upstream convention established by the merged L8 and followed by hmartin's E24
is `datto,<model>` as the primary compatible with Open Mesh as the alt-brand,
and our repo has it inverted. **We have deliberately not changed the code in
this pass**, because the change moves `board_name()` and therefore our
board-keyed PoE budget table and fan init, and that has to be re-validated on
hardware first.

## What we are asking for

1. hmartin — one `i2cget` of LM63 register `0x4D` on a **genuinely cold**
   E24 or E8, before anything writes to the chip. That single reading closes
   the gap between our A/B result and your cold-boot symptom.
2. A decision on where the PSE port map is corrected — DTS or driver. We are
   happy either way; it just cannot be both.
3. A second opinion on the `u-boot-env2` / sysinfo size.
4. Confirmation of the naming we should realign to, so we only re-validate on
   hardware once.
