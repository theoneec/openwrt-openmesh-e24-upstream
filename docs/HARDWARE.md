# Hardware map — Open Mesh S24v3 / Datto E24v3

Everything here was established on a real unit, by reading the running hardware
and by recovering constants from the vendor firmware. Where something is
inferred rather than measured, it says so.

## Overview

| | |
|---|---|
| Product | Open Mesh S24v3, also sold as Datto Networking E24v3 |
| ODM | Senao — same platform family as the Open Mesh S8-L (`datto,l8`) and S24 |
| SoC | Realtek **RTL8396M** ("Cypress"), OpenWrt target `realtek`, subtarget `rtl839x` |
| RAM | 256 MB DDR3 |
| Flash | 32 MB SPI-NOR, Macronix MX25L256xx |
| Front ports | 24x 10/100/1000 copper with PoE+, 2x 10G SFP+ |
| PoE | 802.3af/at, 30 W per port, **410 W** chassis budget |
| Bootloader | Stock U-Boot 2011.12, loads a `.bix` container |
| Console | UART, 115200 8n1 |

The rtl839x target carries only `realtek,rtl839{1,2,3}-soc` compatibles and
reads the true part from the chip-ID register at runtime, so the DTS uses
`realtek,rtl8391-soc` and the recipe `SOC := rtl8391`, matching the closest
in-tree 24-port 839x board (`zyxel,gs1920-24hp-v2`).

## Ethernet

### Copper — 24x GbE

| Block | MDIO / MAC IDs | Front ports |
|---|---|---|
| External RTL8218B #1 | 0..7 | lan1..lan8 |
| External RTL8218B #2 | 8..15 | lan9..lan16 |
| External RTL8218B #3 | 16..23 | lan17..lan24 |

- MDIO address **equals** MAC port ID; all 24 PHYs are Clause 22 on `mdio_bus0`.
- Each octal PHY is grouped as an `ethernet-phy-package` based at MAC 0, 8 and
  16 — the rtl839x convention.
- SerDes: QSGMII on serdes 0..5, four MAC ports per serdes lane group.
- CPU / management port is the fixed rtl839x CPU port, MAC ID **52**.

### SFP+ — 2x 10G (currently disabled)

| Cage | MAC ID | SerDes | MIIM port index |
|---|---|---|---|
| lan25 | **24** | 8 | 25 |
| lan26 | **36** | 12 | 26 |

Note MAC **36**, not the 26 that the layout of the sibling boards would suggest.

Both cages sit behind an external **RTL8295R**. The vendor firmware selects the
interface mode at runtime from the detected module — 10GBase-R, with 1000BX and
100BX as fallbacks — so there is no static mode constant to encode.

**The RTL8295R has no mainline driver, so both ports are declared
`status = "disabled"`.** This is deliberate, not an oversight: a `fixed-link`
would make the kernel report an unconditional 10G carrier on an empty cage,
which poisons bridging and STP decisions and misleads userspace.

Recorded for whoever implements the RTL8295R:

- Module EEPROMs are at I2C **0x50** on the same bit-banged bus as the PoE MCU
  and the LM63 (see `docs/POE.md` for that bus).
- Presence and LOS sidebands are on the RTL8231 expander: **pins 12 and 11** for
  port 25, **pins 21 and 14** for port 26.

## GPIO

Two controllers:

- **`gpio0`** — the SoC GPIO controller. Interrupt-capable, so plain `gpio-keys`
  suffices for the buttons; they take their IRQ from `gpiod_to_irq()` and need
  no explicit `interrupts` property.
- **`gpio1`** — an external **RTL8231** GPIO/LED expander on the SoC indirect
  (aux) MDIO bus at address 0, `gpio-ranges` 0..37. Its integrated LED
  controller is left `disabled`: the 24 copper port link/act LEDs are driven by
  the RTL8396M's internal serial-LED block, not by the expander.

### SoC `gpio0` lines

| Line | Role | Polarity |
|---|---|---|
| 13 | Bit-banged I2C **SCL** | open drain |
| 14 | Bit-banged I2C **SDA** | open drain |
| 17 | LED-mode button | active low *(inferred — see below)* |
| 18 | Reset button | active low |
| 19 | PoE budget LED | active low |
| 22 | **Fan-fail input** | active low (0 = fan failed) |

### RTL8231 `gpio1` pins

| Pin | Role | Polarity |
|---|---|---|
| 2 | Fault / alarm LED | active low |
| 3 | Power LED | active low |
| 4 | LAN/PoE mode LED | active high (high = LAN mode, low = PoE mode) |
| 11, 12 | SFP+ port 25 LOS / presence | — |
| 14, 21 | SFP+ port 26 LOS / presence | — |

## LEDs

| LED | Where | Notes |
|---|---|---|
| Power | `gpio1` pin 3, active low | `led-boot` / `led-running` / `led-upgrade` |
| Fault / alarm | `gpio1` pin 2, active low | `led-failsafe`. This is the LED the vendor's fan-fail poller drives — initialised to 1 (off) and pulled to 0 on fan failure |
| LAN/PoE mode | `gpio1` pin 4, active high | High = LAN mode, low = PoE mode |
| PoE budget | `gpio0` line 19, active low | Lit = over budget. **This is the one role that does not follow the S24**, which uses line 12 |
| SYS | — | Driven by the SoC's **hardware LED engine**, not a GPIO. The DTS applies `pinmux_disable_sys_led` so it is not left blinking |
| 24x port link/act | — | RTL8396M internal serial-LED block |

## Buttons

| Button | Line | Code | Notes |
|---|---|---|---|
| Reset | `gpio0` 18, active low | `KEY_RESTART` | Recovered from the vendor's `board_reset_init()`, which builds gpioId `0x00020002`; the vendor encodes a line as `group * 8 + pin`, so group 2 pin 2 = SoC `gpio0` line 18 |
| LED mode | `gpio0` 17, active low | `KEY_LIGHTS_TOGGLE` | ~50 ms debounce, matching the vendor handler |

**Both lines are specific to this board** — neither is inherited from the S24
sibling, and the S24's reset line 11 is referenced by nothing at all in this
device's vendor firmware.

**The LED-mode button's `GPIO_ACTIVE_LOW` polarity is inferred, not measured.**
The vendor handler acts on level 1, which is consistent with an active-low
button acting on the release edge. Bench-verify before relying on it.

## Fan and thermal

An **LM63** fan controller and remote/local temperature sensor sits at I2C
**0x4C on the same software bit-banged bus as the PoE MCU** (SoC `gpio0` 13/14).
Confirmed on live hardware: 0x4C ACKs and reports manufacturer (register 0xFE)
= 0x01, chip (register 0xFF) = 0x41.

**The stock bootloader leaves the chip in manual mode (register 0x4A = 0x20)
with PWM value 0x00 — the fans are not driven at all.** The vendor's userspace
programs a lookup table at boot. Binding `national,lm63` exposes the chip but
does not program it, so **a port must program the curve or the chassis is
unventilated under PoE load.** On a 410 W PoE switch that is a thermal hazard,
not a comfort issue.

**Patch `0002` in this repo does exactly that** — a per-board branch in the
existing `target/linux/realtek/base-files/etc/init.d/hwmon_fancontrol`, gated on
the chip identifying itself as an LM63 (the PoE MCU is on the same bus), with
read-back verification on every write and a fallback to manual full PWM if
anything fails to stick.

Fan failure is reported on SoC `gpio0` line **22**, active low. That line is
deliberately left **unclaimed** by every node in the DTS so it stays readable
from userspace with the gpiod tools (`gpioget <chip> 22`). Modelling it as a
`gpio-key` would hand the line to the input layer, which claims it exclusively
and makes it unreadable — and a fan failure is a level, not a keypress.

### Why the fan curve is written with raw I2C rather than hwmon sysfs

The vendor's register image cannot be reproduced through the hwmon sysfs
attributes on current kernels:

- Register 0x4D (`PWM_FREQ`) must hold 0x1F (PFR 31) while register 0x4A keeps
  its SCS bit (0x08) set. `pwm1_freq_store()` takes a frequency in Hz, derives
  PFR with `DIV_ROUND_CLOSEST(base, hz)` and chooses SCS itself. PFR 31 is only
  reachable from the 180 kHz base, which clears SCS — and 700/22 rounds to 32
  while 700/23 rounds to 30, so PFR 31 *with* SCS set is unreachable.
- Bit 0x02 of register 0x4A is not writable through any hwmon attribute.
- `pwm1_enable=2` is refused with `-EPERM` by `lm63_lut_looks_bad()` unless all
  eight LUT points are monotonic, and the vendor programs only two.

The `lm63` driver claims 0x4C, so `i2c-dev` refuses the address unless it is
forced — `i2cset -f` is a requirement here, not a shortcut. Writing behind the
driver leaves its cached copy stale until its next `update_interval` refresh,
which changes what sysfs reports, not what the chip does.

## Flash layout

32 MB SPI-NOR. Confirmed from U-Boot's own partition table and `printenv`.
Stock names and offsets are kept so the stock loader and factory data stay
intact.

| Stock name | OpenWrt label | Offset | Size |
|---|---|---|---|
| LOADER | `u-boot` (read-only) | `0x000000` | `0x080000` |
| BDINFO | `u-boot-env` | `0x080000` | `0x010000` |
| SYSINFO | `u-boot-env2` | `0x090000` | `0x010000` (`0x1000` of usable env) |
| JFFS2_CFG | `cfg` | `0x0a0000` | `0x400000` |
| JFFS2_LOG | `log` | `0x4a0000` | `0x100000` |
| RUNTIME1 | `runtime` (`fwconcat0`) | `0x5a0000` | `0xd30000` |
| RUNTIME2 | `runtime2` (`fwconcat1`) | `0x12d0000` | `0xd30000` |

- The **base MAC** is the `ethaddr` variable in **BDINFO**, read through DTS
  nvmem (`u-boot,env` layout).
- The **boot partition selector** is `bootpartition` in **SYSINFO**, *not* in
  the main environment. `boota <idx>` rewrites SYSINFO.
- The two `0xd30000` RUNTIME slots are **`mtd-concat`'d** into a single
  `firmware` region of `0x1a60000` so kernel + rootfs + overlay fit. This gives
  up the stock A/B failover, as the upstream `datto_l8` sibling also does.
  `IMAGE_SIZE` still caps the built image at one slot (`13504k`).

See `docs/FLASH-AND-RECOVERY.md` for the container format and the boot flow.
