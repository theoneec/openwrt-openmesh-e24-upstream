# Flashing, the `.bix` container, and recovery

## 1. The `.bix` container

The vendor ships firmware as a `.bix` file. Despite the extension there is
nothing exotic about it:

> **A `.bix` is a standard U-Boot legacy `uImage` with the magic word replaced
> by a board ID.**

For this board the board ID is **`0x00702202`**. (The sibling Open Mesh S24 /
Datto S24-L uses `0x00702400`; `datto_l8` uses the same mechanism.)

### Header — 64 bytes, big-endian

| Offset | Field | Notes |
|---|---|---|
| `0x00` | **magic** | board ID, `0x00702202` here — replaces the usual `0x27051956` |
| `0x04` | **header CRC** | `crc32` over the whole 64-byte header **with bytes `0x04..0x08` zeroed** |
| `0x08` | timestamp | standard `ih_time` |
| `0x0c` | **payload length** | |
| `0x10` | **load address** | |
| `0x14` | **entry point** | |
| `0x18` | **payload CRC** | `crc32` over the payload |
| `0x1c` | **`0x05050201`** | the standard `ih_os` / `ih_arch` / `ih_type` / `ih_comp` byte quartet: Linux / MIPS / kernel / gzip |
| `0x20` | version string | standard 32-byte `ih_name` |

### There is no signature

**U-Boot validates the magic word and the two CRCs. That is all.** There is no
cryptographic signature and no vendor key anywhere in the boot path.

That is the whole reason this port is possible against an unmodified stock
bootloader: an OpenWrt `uImage` stamped with the right magic is indistinguishable
from a vendor image as far as the loader is concerned.

### Stamping an OpenWrt image

You do not need a repacking tool. The image recipe does it:

```make
UIMAGE_MAGIC := 0x00702202
```

OpenWrt passes that to `mkimage` as `-M`, which overrides the `uImage` header
magic (see `include/image-commands.mk`, `Build/uImage`). The DTS declares the
same value so the kernel's own image parser agrees:

```dts
partition@0 {
	label = "firmware";
	reg = <0x0 0x1a60000>;
	compatible = "openwrt,uimage", "denx,uimage";
	openwrt,ih-magic = <0x00702202>;
};
```

Nothing else about the image is special.

## 2. Boot flow

```
Stock U-Boot 2011.12
        |
        |  reads `bootpartition` from SYSINFO (0x090000)  <-- NOT the main env
        v
   selects RUNTIME1 (0x5a0000) or RUNTIME2 (0x12d0000)
        |
        |  checks magic == 0x00702202
        |  checks header CRC, checks payload CRC
        v
   decompresses and boots the kernel
```

Two details that cost time if you do not know them:

- **The boot-slot selector is `bootpartition`, and it lives in SYSINFO
  (`u-boot-env2`, `0x090000`), not in the main environment (BDINFO,
  `0x080000`).** Setting things in the main env will not move the boot slot.
- **`boota <idx>` rewrites SYSINFO.** That is the supported way to switch slots
  from the U-Boot prompt.

The **base MAC** is the `ethaddr` variable in **BDINFO**. OpenWrt reads it
through DTS nvmem, so it survives flashing as long as you do not erase
`u-boot-env`. Don't.

### Slot layout under OpenWrt

OpenWrt `mtd-concat`s **both** `0xd30000` RUNTIME slots into one `firmware`
region of `0x1a60000`, so kernel + rootfs + overlay fit. **This gives up the
stock A/B failover** — the same trade the upstream `datto_l8` makes.
`IMAGE_SIZE` still caps the built image at one slot (`13504k`) so U-Boot can
always find a valid header at the RUNTIME1 offset.

Full partition table: `docs/HARDWARE.md`.

## 3. Installing OpenWrt

You need the serial console. There is no web-UI upgrade path onto OpenWrt.

1. **Console**: UART, **115200 8n1**.
2. **Interrupt U-Boot** during boot.
3. **TFTP an `initramfs-kernel` image into RAM** and `bootm` it. Nothing is
   written to flash at this stage, so this step is free to retry.
4. From the running initramfs, **`sysupgrade` the `squashfs-sysupgrade.bin`**.

Running from RAM first is worth the extra step: you get to confirm the board
comes up, ports enumerate and PoE binds *before* anything touches flash.

## 4. Recovery

The device is hard to brick permanently, because U-Boot itself is never
rewritten and both CRCs are checked before boot.

**If a flashed image does not boot:**

1. Console in and interrupt U-Boot.
2. TFTP a known-good `initramfs-kernel` into RAM and `bootm` it.
3. `sysupgrade` again from there.

**If the loader rejects the image** (bad magic or failed CRC), it will say so —
that is a stamping or transfer problem, not a dead board. Re-check
`UIMAGE_MAGIC` and re-transfer.

**If you still have a stock image in the other RUNTIME slot**, `boota <idx>`
rewrites SYSINFO and boots it. Note that this is only available if you have not
yet overwritten that slot — an OpenWrt install that uses the concatenated region
will eventually grow into RUNTIME2.

**Do not erase `u-boot` (`0x000000`) or `u-boot-env` (`0x080000`).** The first
is the only thing standing between you and JTAG; the second holds your MAC.

This path was exercised repeatedly during the port, in both directions.

## 5. Reboot behaviour — an honest note

On this device **`reboot -f` is reliable**, while a **plain `reboot` hung once**.
It has not been reproduced or root-caused.

While looking at it, one asymmetry in the target stood out and is recorded in
`SUBMISSION.md` for a maintainer: `rtl838x_apply_early_quirks()` clears a
flash-controller 4-byte-address bit with the comment *"if this bit is not
cleared the watchdog cannot reset the SoC"*, and **rtl839x has no equivalent
hook at all**. Whether those two facts are related is an open question, not a
diagnosis.
