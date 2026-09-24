# The PoE subsystem — Open Mesh S24v3 / Datto E24v3

If you are porting another Senao/Realtek PoE switch, this is the part worth
reading. Almost none of it is guessable from the schematic, and one of the
findings (the port map) silently powers the wrong port if you miss it.

**Headline: PoE on this board needs no new driver and no firmware blob.** The
in-tree `realtek,pse-mcu-gen1-smbus` driver already speaks the MCU's dialect
exactly. What it does need is the right transport — which is not a hardware I2C
controller — and the right port mapping.

## 1. Transport: a software bit-banged I2C bus on SoC GPIO

There is no hardware I2C controller in this path. The SoC **bit-bangs I2C** on
two internal GPIO lines (the vendor calls it "SMI group 6"):

| Signal | Line |
|---|---|
| SCL | SoC `gpio0` line **13** |
| SDA | SoC `gpio0` line **14** |

Modelled with `i2c-gpio`:

```dts
i2c0: i2c-gpio-0 {
	compatible = "i2c-gpio";
	sda-gpios = <&gpio0 14 (GPIO_ACTIVE_HIGH | GPIO_OPEN_DRAIN)>;
	scl-gpios = <&gpio0 13 (GPIO_ACTIVE_HIGH | GPIO_OPEN_DRAIN)>;
	i2c-gpio,delay-us = <2>;
	#address-cells = <1>;
	#size-cells = <0>;
	/* ... */
};
```

Three other things live on this same bus, so do not assume it is PoE-only:

| Address | Device |
|---|---|
| **0x20** | PoE MCU |
| **0x4C** | LM63 fan controller / temperature sensor |
| **0x50** | Both SFP+ module EEPROMs |

## 2. The MCU

- **7-bit address `0x20`.**
- Identifies itself as a **Nuvoton M05xx LAN MCU** fronting **BCM59121**
  (id `0xe121`), **24 ports across 3 PSE chips**.
- **No firmware is uploaded to it.** PoE needs no binary blob and no
  cooperation from the bootloader beyond leaving the part alone.

### The 0x21 trap

Plenty of Senao-family notes claim the PoE MCU is at **0x21**. On this board it
is not. In the vendor's `custom.ko` every PoE message goes through
`poe_i2c_cmd_set()`, and all 25 call sites load **0x20** as the address.
**`0x21` is a message opcode carried in byte 0 of the command frame, not an
address.** If you probe 0x21 and get nothing, that is why.

## 3. Wire protocol

Messages are **12 bytes** in each direction:

| Byte | Meaning |
|---|---|
| 0 | opcode |
| 1 | sequence number |
| 2 | port |
| 3..10 | payload |
| 11 | checksum — the sum of bytes 0..10, `& 0xFF` |

The reply **echoes the sequence byte** and carries **0 in byte 2**.

**The MCU needs roughly a 3 ms turnaround between the write and the read.** Go
faster and you get garbage or NAKs. This is the single most common reason a
hand-rolled probe of one of these MCUs "doesn't answer".

This is byte-for-byte the frame the in-tree Realtek PSE-MCU driver already
implements, which is why no new backend is needed.

## 4. Binding it

Driver: `realtek,pse-mcu-gen1-smbus`, package `kmod-pse-realtek-mcu-i2c` —
the same transport and dialect used by `zyxel,gs1920-24hp-v2`.

```dts
pse: ethernet-pse@20 {
	compatible = "openmesh,e24-pse", "realtek,pse-mcu-gen1-smbus";
	reg = <0x20>;

	pse-pis {
		#address-cells = <1>;
		#size-cells = <0>;

		PSE_PI(0)
		PSE_PI(1)
		/* ... through PSE_PI(23) */
	};
};
```

The `PSE_PI()` node list stays a plain `0..23`. The permutation lives entirely
in the `&phy<n>` references — see the next section.

## 5. **The port map — read this one**

> **The PSE channel index is not the front-panel port index.**
> On this board, `channel = port ^ 3`: every aligned group of four ports is
> reversed.

### What the vendor does

The vendor firmware carries the table literally in `custom.ko` `.rodata` at
offset `0x46c`:

```
03 02 01 00   07 06 05 04   0b 0a 09 08
0f 0e 0d 0c   13 12 11 10   17 16 15 14
```

i.e. `table[i] == i ^ 3`. (A second, plain-identity array follows it.)

`board_poe_init()` then programs **the MCU's own built-in remap feature**:
opcode **`0x02`** to enable the remap, then opcode **`0x1D`** to push that
24-entry table. From that point the vendor's userspace sees a clean 1:1 view and
never has to think about it again.

### What mainline does

**The in-tree driver implements neither opcode `0x02` nor opcode `0x1D`.** The
remap is therefore never programmed, and the **raw XOR-3 channel order is what
the MCU exposes to Linux**.

Confirmed on hardware: with a straight 1:1 mapping, a PD plugged into
front-panel **port 1** (`phy0`) is correctly detected, classified and powered —
but "delivering power" is reported on **`lan4`**, i.e. phy index 3. And
`0 ^ 3 == 3`, exactly as the vendor table predicts.

### The fix used here

The DTS compensates: `&phy<n>` attaches to `PSE_PI(n ^ 3)`.

```dts
&phy0  { pses = <&pse_pi3>; };
&phy1  { pses = <&pse_pi2>; };
&phy2  { pses = <&pse_pi1>; };
&phy3  { pses = <&pse_pi0>; };
&phy4  { pses = <&pse_pi7>; };
/* ... and so on, in reversed groups of four, through &phy23 { pses = <&pse_pi20>; }; */
```

### ⚠ The two fixes are mutually exclusive

There are exactly two correct ways to solve this, and **applying both is worse
than applying neither**:

1. **Permute in the device tree** (what this port does), leaving the driver
   alone; or
2. **Program the MCU remap in the driver** (implement opcodes `0x02` and `0x1D`)
   and leave the device tree 1:1.

If the mainline driver ever gains the remap opcodes, **the XOR in this DTS must
be removed at the same time.** Otherwise the two corrections cancel and the
mapping is scrambled again — and it will look like it "used to work", which is
the worst kind of regression to chase. The DTS carries this warning in a comment
next to the mapping; please keep it there.

### Porting note

Do not assume `^ 3` on a different board. Assume *some* permutation until you
have proved otherwise. The cheap empirical test is the one used here: plug a
single PD into physical port 1 and see which interface reports delivering
power. If it is not `lan1`, you have a port map to work out — the vendor's
`.rodata` table is usually sitting right there.

## 6. Using it

PoE is exposed through the kernel PSE API, so `ethtool` drives it. Include
`ethtool-full`; busybox's applet does not carry the PSE commands.

```sh
# show PoE state for a port
ethtool --show-pse lan1

# enable / disable power on a port
ethtool --set-pse lan1 c33-pse-admin-control enable
ethtool --set-pse lan1 c33-pse-admin-control disable

# per-port available power limit (802.3at, 30 W)
ethtool --set-pse lan1 c33-pse-avail-pw-limit 30000
```

Delivered power and detected class read back through `--show-pse`.

Chassis budget is **410 W** across 24 ports at up to 30 W each, so the full
24 x 30 W is oversubscribed by design. The `gpio0` line 19 LED is the vendor's
over-budget indicator.

## 7. Thermal is part of the PoE story

The **LM63 at 0x4C shares this bus**, and the stock bootloader leaves it in
manual mode at PWM 0 — **fans stopped**. Binding the hwmon driver exposes the
chip; it does not program it. Under real PoE load an unventilated 410 W chassis
is a hazard, so a port must program the fan curve at boot.

Details, including why the vendor's register image cannot be reproduced through
hwmon sysfs on current kernels, are in `docs/HARDWARE.md`.

Fan failure is a level on SoC `gpio0` line **22** (active low), left unclaimed
in the DTS so it stays readable with `gpioget`.
