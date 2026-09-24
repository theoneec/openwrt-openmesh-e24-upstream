# OpenWrt support for Open Mesh S24v3 / Datto E24v3

OpenWrt device support for the **Open Mesh S24v3** (also sold as the **Datto Networking E24v3**) — a 24-port Gigabit PoE+ managed switch with 2x 10G SFP+, on the Realtek **RTL8396M** (`realtek/rtl839x` target).

> ## Read this before anything else: there is already an upstream effort
>
> There is an active, year-old OpenWrt effort covering exactly these devices,
> which we were unaware of while doing this work:
> **[Add support for Datto L8, E24v3, E48 switches](https://forum.openwrt.org/t/add-support-for-datto-l8-e24v3-e48-switches/241657)**
> (*For Developers*, Oct 2025 – Apr 2026), driven by **hmartin** with review
> from **svanheule** (Realtek target maintainer) and hardware testing from
> **stevewaffler**.
>
> - The **Datto L8 is already in mainline**; hmartin's latest L8 change,
>   [openwrt/openwrt#22764](https://github.com/openwrt/openwrt/pull/22764),
>   merged 3 Apr 2026.
> - hmartin has a WIP branch covering E8 / E24 / E48 —
>   <https://github.com/halmartin/openwrt/tree/rtl83xx-datto> — including
>   `target/linux/realtek/dts/rtl8396_datto_e24.dts`. As of 26 Mar 2026 he
>   reported copper, PoE, fans and LEDs working, SFP/SFP+ still WIP.
> - A **GPL source archive for our exact devices already exists**:
>   <https://github.com/halmartin/avalon-l2switch-realtek-rtk8382> — *"GPL
>   source code for the Datto E8, E24v3, and E48 switches"*, also linked from
>   the OpenWrt wiki GPL archive page. **We did our reverse engineering by
>   disassembling vendor binaries without knowing it existed.**
>   **But the archive is incomplete** — it has U-Boot and a near-vanilla
>   kernel, and *not* the fan init, the PoE stack or the board configuration.
>   For those three, disassembly is still the only source. See
>   [`docs/UPSTREAM-STATUS.md`](docs/UPSTREAM-STATUS.md) before you clone
>   553 MB.
>
> **So this repo is not a competing port.** It is a set of findings and fixes
> offered *into* that effort. The full picture — who is doing what, what we got
> wrong, what we are asking for — is in **[`docs/UPSTREAM-STATUS.md`](docs/UPSTREAM-STATUS.md)**.

It is the RTL8396M sibling of [`openwrt-openmesh-s24-upstream`](https://github.com/theoneec/openwrt-openmesh-s24-upstream) (the rtl8382 Open Mesh S24 / Datto S24-L), and the two are built the same way.

## What we have that the upstream effort does not

Lead with these three; everything else in this repo is supporting material.

**1. The root cause of the fan-control problem the thread gave up on.**
hmartin found the symptom (from a cold boot any PWM below 255 stops the fans;
after a warm reboot out of the Datto firmware partial control works), found the
74HC123D monostable, hunted a missing GPIO, and wrote: *"Someone else will have
to look into why PWM control of fans doesn't work on these models, I feel like
I've wasted enough time on this."*
**It is not a GPIO — it is LM63 register `0x4D`, and that is now an
experimental result rather than a theory.** With duty held at ~50% and the
clock base untouched (`0x4A` = `0x2A` throughout), varying **only** `0x4D` on a
live unit gave: PFR 31 (`0x1F`) → fans run, audibly slower than full speed,
i.e. partial PWM control working; PFR 8 (`0x08`) → fans off. Restoring the
vendor values brought them back. **Register `0x4D` (`PWM_FREQ` / PFR)
determines whether sub-100% duty drives these fans at all.**
We do *not* claim a physical mechanism — our first attempt at one was wrong and
has been withdrawn (see below) — and we have not yet read `0x4D` on a genuinely
cold boot, so the last link to hmartin's cold-boot symptom is open. Full detail
in [`docs/UPSTREAM-STATUS.md`](docs/UPSTREAM-STATUS.md) and patch `0002`.

**2. PoE through the in-tree kernel PSE driver, plus the port-mapping
discovery.** hmartin's E24 recipe uses userspace `realtek-poe`; stevewaffler
reported on 6 Apr 2026 that it delivers power but shows 0 consumption and
"unknown" ports. We bind `realtek,pse-mcu-gen1-smbus` at 7-bit address **0x20**
and it enumerates as *Nuvoton M05xx LAN MCU, BCM59121 (id 0xe121), 24 ports
across 3 PSE chip(s)* on two separate units. The likely explanation for those
"unknown" ports: **the PSE channel index is the port index XOR 3**. See below
and `docs/POE.md`.

**Withdrawn:** we previously explained the fan behaviour as the 74HC123D
one-shot timing out between edges at too *low* a frequency. That is backwards —
the driver reports 5806 Hz at PFR 31, fitting `freq = 180000 / PFR`, so the
failing PFR 8 is ~22.5 kHz, four times *higher*. The fans stopped when the
frequency was **raised**. The explanation is deleted rather than repaired; the
causal register stands on its own evidence.

**3. The LM63 and the PSE MCU are on one I2C bus, not two.** hmartin's E24 DTS
puts the LM63 on a separate `i2c-gpio-2`; on real hardware there is only
`i2c-gpio-0`, carrying `0-0020` and `0-004c` and nothing else. Offered as a
correction.

**4. The LED Mode button — what it actually does, at register level.**
hmartin asked svanheule about this button in the thread and the answer was
that `BTN_0` is not wired up by any standard script and that port LEDs via the
hardware peripheral are not really supported. The semantics have never been
written down. We recovered them from the vendor firmware: a press only toggles
a flag, and a 1 Hz thread re-asserts the software-LED enable bits from it —
LAN mode hands all three LED entities back to the hardware scan engine, PoE
mode takes software control of entities 0 and 1 and shows **green steady =
delivering, amber steady = fault, dark = searching/disabled**. The registers
(`LED_SW_P_EN_CTRL`, `LED_SW_P_CTRL`, `LED_SW_CTRL`), the blink encoding and
the vendor ioctls are all in
[`docs/UPSTREAM-STATUS.md`](docs/UPSTREAM-STATUS.md).
**We are not offering a patch for this — we have not implemented it.** It is
written down so that whoever does, can.

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

## Known delta / naming — our convention is wrong

**Upstream convention is `datto,<model>` primary, Open Mesh as the alt-brand.
Ours is inverted. This is a known delta, not an oversight we are defending.**

The merged L8 uses:

```
compatible = "datto,l8", "realtek,rtl838x-soc";
DEVICE_VENDOR := Datto
DEVICE_MODEL  := L8
DEVICE_ALT0_VENDOR := Open Mesh
DEVICE_ALT0_MODEL  := S8-L
```

hmartin's E24 follows it:

| | hmartin / upstream | This repo |
|---|---|---|
| DTS filename | `rtl8396_datto_e24.dts` | `rtl8391_openmesh_e24.dts` |
| `compatible` | `"datto,e24v3", "realtek,rtl8396-soc"` | `"openmesh,e24", "datto,e24v3", ...` |
| `model` | `Datto E24v3` | `Open Mesh E24v3` |
| Recipe | `Device/datto_e24`, `SOC := rtl8396` | `Device/openmesh_e24`, `SOC := rtl8391` |
| `DEVICE_MODEL` | `E24` | `E24` |
| Alt branding | `Open Mesh` / `S24v3` | Datto in `DEVICE_ALT0_*` (inverted) |
| `UIMAGE_MAGIC` | `0x00702202` | `0x00702202` (same) |
| `IMAGE_SIZE` | `13504k` | `13504k` (same) |

Note the two values that already agree — `UIMAGE_MAGIC` and `IMAGE_SIZE` — were
arrived at independently, which is a useful cross-check on both.

**Ours predates our discovery of that convention.** Realignment is *pending*,
not refused. It is not done in this pass because changing the compatible
changes `board_name()`, and `board_name()` keys both our PoE budget table and
our fan init — so the rename is a functional change that **must be re-validated
on hardware before submission**, not a search-and-replace. We would rather
document the delta honestly than ship an untested rename.

## What works

Validated on a real unit, booted from flash:

| Subsystem | Status |
|---|---|
| Boot from stock U-Boot (`boota`), no bootloader replacement | Works |
| 24x Gigabit copper (DSA, `lan1`..`lan24`) | Works |
| PoE+ — per-port enable/disable, class and delivered-power readout | Works, via the in-tree kernel PSE driver |
| LM63 fan controller + temperature sensor | Works — automatic fan control, programmed by patch `0002` |
| LEDs (power, fault, LAN/PoE mode, PoE budget) | Works — but see the inverted power LED below |
| Buttons (reset, LED mode) | Works (mode-button polarity inferred, see `docs/HARDWARE.md`) |
| Base MAC from `u-boot-env` `ethaddr` | Works |
| sysupgrade / flashing / recovery | Works, exercised repeatedly |

## What does not work / known issues

- **The two 10G SFP+ ports.** They sit behind an external **RTL8295R**, which
  has no mainline driver, so both are declared `status = "disabled"`. This is
  deliberate: a `fixed-link` would make the kernel report an unconditional 10G
  carrier on an empty cage. See `docs/HARDWARE.md` for everything recorded for
  a future RTL8295R implementation — this is the obvious follow-up
  contribution. The topology is well attested: a second unit's stock U-Boot log
  prints `### RTL8295R config - MAC ID = 24 ###` and
  `### RTL8295R config - MAC ID = 36 ###`, matching our port table, the boot
  log quoted in the forum thread, and both units.
- **Our generated network config is wrong, and this is our bug.** Because the
  SFP+ ports are disabled, `lan25` and `lan26` **do not exist as netdevs** on a
  live unit — yet the config we generate still lists them in the bridge VLAN:
  `network.lan_vlan.ports='lan1 ... lan24 lan25 lan26'`. A bridge VLAN that
  references non-existent ports is simply wrong and will generate errors.
  **Known issue, to fix.** (Not fixed in this pass — this pass is documentation
  only.)
- **Inverted power LED**, reported by stevewaffler on a real E24v3 (6 Apr
  2026): it flashes during boot and then goes *off* once booted;
  `/sys/class/leds/green:sys/brightness` must be `0` for solid-on. Open issue,
  not yet addressed here.
- **`u-boot-env2` / sysinfo partition size** — we use `0x10000`; the merged L8
  and hmartin's E24 use `0x20000`, which appears to overlap `cfg` at `0xa0000`.
  Raised as a question in `docs/UPSTREAM-STATUS.md`, not as an assertion.

> **Note on fan control.** The stock bootloader leaves the LM63 in manual mode
> at PWM 0 — fans stopped — and binding the hwmon driver exposes the chip
> without programming it. **Patch `0002` fixes this**, and you want it: this is
> a 410 W PoE chassis. If you apply `0001` alone, program a fan curve yourself
> before putting the switch under load.

## The PSE port map, in one paragraph

The PSE channel index is `port ^ 3` — each aligned group of four ports is
reversed. The vendor firmware carries the table literally in `custom.ko`
`.rodata` and programs the MCU's own remap feature (opcode `0x02` then `0x1D`)
so its userspace sees 1:1. The mainline driver implements **neither** opcode,
so the raw XOR-3 order is exposed and the DTS compensates by attaching
`PSE_PI(n ^ 3)` to `&phy<n>`. Confirmed on hardware: before the fix a PD on
physical port 1 reported as `lan4`; after, `lan1` reads *delivering power* and
`lan4` *searching*, with `lan1` the only delivering port of 24.
**These are two mutually exclusive fixes** — patch the DTS *or* replicate the
vendor MCU init in the driver. Doing both cancels out.

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

1. Serial console, 115200 8n1; interrupt stock U-Boot with **`pac`** — the
   letters `a`, `p`, `c` in any order (janh traced it to `abortboot` in the
   U-Boot source).
2. TFTP an OpenWrt `initramfs-kernel` image into RAM and `bootm` it.
3. From the running initramfs, `sysupgrade` the `squashfs-sysupgrade.bin`.

The stock loader **definitely validates the header CRC and the payload CRC**,
and there is **no cryptographic signature** anywhere in the boot or TFTP-upgrade
path — which is why a correctly built OpenWrt image boots under the unmodified
stock bootloader. The magic word is a per-board identifier which we set to
match; **whether the shipped loader actually enforces it is unverified** (the
GPL source compiles `image_check_magic()` out by default, and we have never
deliberately flashed a wrong one). The family:

| Device | `UIMAGE_MAGIC` |
|---|---|
| E24v3 | `0x00702202` |
| E48 | `0x00702201` |
| S24-L / L24 | `0x00702400` |

> **Warning, and it is not a small one:** `boota` **erases 4 KB — the image
> header — from a partition that fails to boot**, and flips the
> active-partition selector. That is confirmed in the GPL source
> (`common/cmd_bootm.c:1660-1663`). One failed attempt destroys that slot's
> image. Know this before experimenting.

We cannot explain the magic *encoding*: the vendor U-Boot documents the field
as a Chip/Vendor/Product bitfield, and our values do not fit it. The values are
read from real flash dumps and corroborated by hmartin's hexdump in the thread;
the scheme behind them is Senao's and we are not going to guess at it.

**A stock-web-UI-flashable factory image should be possible** and is the better
install path. svanheule pointed hmartin at the **Zyxel GS1900 recipes** as the
pattern: the initramfs/factory image must stay within the original `0xd30000`
partition, while the sysupgrade image may span the merged firmware partitions.
hmartin has already demonstrated installing OpenWrt on the L8 straight from the
stock web UI, no UART needed. Not implemented here yet.

## Getting a root shell on the stock firmware

Interrupt U-Boot, then:

```
sf read $(freemem) $(flashoffset_linux) $(ssize_linux)
setenv bootargs console=ttyS0,115200 mem=256M rdinit=/bin/sh
bootm $(freemem)
```

**Do not quote the `bootargs` value.** At the early shell, `rm /bin/cli` and
then run `/etc/rc` — the box boots on into a root shell instead of the vendor
CLI. Stock credentials after a factory reset are `admin` / `0p3nm3$h!` (per the
forum thread). Hidden vendor CLI commands live in `libcustom.so.0`, including
`mphiddenpoe poe power-budget <1-999>`.

## Layout

| Path | What it is |
|---|---|
| `patches/` | The `git format-patch` output — the two-patch series, this is the contribution |
| `docs/UPSTREAM-STATUS.md` | **The existing upstream effort, what we got wrong, what we are offering** |
| `SUBMISSION.md` | Notes for OpenWrt maintainers: scope, exclusions, test status, open questions |
| `FORUM-POST.md` | Our reply into the existing forum thread |
| `docs/HARDWARE.md` | Board map: SoC, PHYs, GPIO, LEDs, buttons, thermal, flash layout |
| `docs/POE.md` | The PoE subsystem in detail — the most reusable part for other porters |
| `docs/FLASH-AND-RECOVERY.md` | `.bix` container format, boot flow, recovery |
| `dts/`, `image/`, `base-files/` | Convenience reading copies of what the patches change |

## Credit

The fan symptom characterisation, the 74HC123D identification, the E8/E24/E48
bring-up branch and the GPL source archive are **hmartin's**. The image-recipe
guidance is **svanheule's**. The E24v3 PoE and power-LED field reports are
**stevewaffler's**. The `abortboot` finding is **janh's**. Where this repo uses
their work or their answers, it says so.

## Licence

The DTS and image recipe are `GPL-2.0-or-later`, matching OpenWrt. See `LICENSE`.
