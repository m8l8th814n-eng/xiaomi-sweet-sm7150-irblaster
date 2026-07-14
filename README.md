# IR blaster — Xiaomi Redmi Note 10 Pro (sweet), mainline / postmarketOS

Enables the consumer-IR transmitter LED on the mainline kernel. It is **not**
enabled by default. After this patch it appears as `/dev/lirc0` and transmits
with `ir-ctl` (v4l-utils). TX only (no receiver in hardware).

## Scope: device-tree only — no code
This is a **pure device-tree change**. Nothing in C needs to be written:
- the `ir-spi` driver already exists in mainline (`drivers/media/rc/ir-spi.c`);
- the required config is already set in the pmOS sm7150 kernel:
  `CONFIG_IR_SPI=m  CONFIG_RC_CORE=m  CONFIG_LIRC=y  CONFIG_RC_DEVICES=y  CONFIG_SPI_QCOM_GENI=y`.

So the only thing missing is the DT node — this patch.

## Hardware
The IR LED sits on **QUPv3 SE8 SPI** (`spi@a88000`, mainline label `spi8`),
matching downstream `sweet-sdmmagpie.dtsi` (`&qupv3_se8_spi` / `irled@0`).
`uart8` shares SE8 and is already disabled on sweet, so the bus is free.

## The one gotcha
This kernel's `ir-spi` driver matches **`compatible = "ir-spi-led"`**, not the
classic `"ir-spi"`. With `"ir-spi"` the SPI device is created but stays
**unbound** — `/dev/lirc0` never appears and a manual bind returns `ENODEV`.
Confirm the driver's match:

    modinfo ir-spi | grep alias      # -> of:N*T*Cir-spi-led

## Apply
Add the patch to the kernel package (`source=` + `sha512sums`) and rebuild, or
apply it directly to `arch/arm64/boot/dts/qcom/sm7150-xiaomi-sweet.dts`, then
flash the kernel/DTB.

## Verify
    ls /dev/lirc0
    sudo apk add v4l-utils
    ir-ctl -d /dev/lirc0 --scancode=nec:0x0408

Point another phone's front camera at the top-edge LED — a faint purple/white
flicker means it transmits.

## Sending IR codes
`/dev/lirc0` only transmits — you supply the codes. Ready-made codes for most
TVs, ACs, etc. are in the community **Flipper-IRDB**:
<https://github.com/Lucaslhm/Flipper-IRDB>

Send a scancode for a known protocol:

    ir-ctl -d /dev/lirc0 --scancode=nec:0x0408

Or send a raw pulse/space file (ir-ctl format), e.g. `power.txt`:

    carrier 38000
    pulse 4500
    space 4500
    pulse 560
    space 1690
    ...

    ir-ctl -d /dev/lirc0 --send=power.txt

A Flipper `.ir` "raw" signal converts 1:1: its `data:` line is alternating
durations in microseconds (pulse, space, pulse, space ...) — just prefix each
with `pulse`/`space` and add `carrier <frequency>`.

## Status across mainline sm7150 Xiaomi devices
Verified in the sm7150-mainline tree (v7.1_rc3):

| Device | IR in mainline | Note |
|--------|----------------|------|
| toco | yes | already has the `ir-spi-led` node on `spi8` |
| tucana | yes | already has it |
| sweet (Redmi Note 10 Pro) | no | **this patch** |
| davinci (Mi 9T) | no | has an IR LED in hardware; not yet enabled — same DT-only fix, confirm the SPI node from downstream |
| surya (POCO X3 NFC) | no | same as davinci |
| sunfish (Pixel 4a) | n/a | no IR (uart8 is the serial console) |

The `ir-spi-led` compatible gotcha is the same across current mainline.
