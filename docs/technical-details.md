# Technical details

Machine: HP OmniBook X Flip Laptop 16-as0xxx, board 8DA1, Insyde BIOS F.20 (2026-04-27), Intel Core Ultra 9 288V (Lunar Lake).
Digitizer: ACPI `ELAN2514` (`_CID PNP0C50`), HID `04F3:43F0`, on `\_SB.PC00.I2C4.TPL1`, bus speed 400 kHz, IRQ 108 (IO-APIC,
level, active low). Linux drivers: `i2c_hid_acpi` + `hid-multitouch`.

## 1. How the interrupt is wired

`TPL1._CRS` has two variants, chosen by the BIOS variable `TPLM`:

- `TPLM == 0`: a `GpioInt()` through the Intel GPIO controller.
- `TPLM == 1` (this machine): a plain `Interrupt (ResourceConsumer, Level, ActiveLow, Exclusive)`. Its vector is patched in
  `_INI` with `INUM (GPLI)`. The pad is routed straight to the IO-APIC by the `GPIROUTIOXAPIC` bit (bit 20) of its `PADCFG0`
  register.

`TPL1._INI` (DSDT) sets that bit correctly:

```
If ((TPLM == One)) {
    Local0 = (T0IP ^ One)
    SGRA (GPLI, Local0)      // routing := 1 when T0IP == 0
    SGII (GPLI, Zero)        // no RX inversion
    GRXE (GPLI, Zero)        // level
}
```

`SGRA` is HP/Intel's helper that writes bit 20 of the pad's `PADCFG0`:

```
Method (SGRA, 2, Serialized) {
    Local2 = (GADR (Arg0, One) + (GNMB (Arg0) * 0x10))
    OperationRegion (PDW0, SystemMemory, Local2, 0x04)
    Field (PDW0, AnyAcc, NoLock, Preserve) { , 20, TEMP, 1, Offset (0x04) }
    TEMP = Arg1
}
```

## 2. The bug

An SSDT (`SSDT25` on BIOS F.20) defines the touchscreen power resource that `TPL1` depends on. It's only created if `TPLS == One`:

```
Scope (\_SB.PC00.I2C4) {
    TPIP = T0IP
    TPI2 = GPLI
    If ((TPLT != Zero)) { If ((TPLS == One)) {
        PowerResource (PTPL, 0x00, 0x0000) {
            Method (_ON) {
                \_SB.SGOV (TPPE, TPEP)          // power enable
                Sleep (0x02)
                \_SB.SGOV (TPPR, TPRP)          // release reset
                ONTM = Timer
                \_SB.SGRA (TPI2, TPIP)          // routing := T0IP = 0   <-- wrong
            }
            Method (_OFF) {
                Local0 = (TPIP ^ One)
                \_SB.SGRA (TPI2, Local0)        // routing := 1          <-- inverted too
                ...
            }
        }
    }}
}
```

The touchpad's power resource in the same table uses `SGRA (GPDI, PPDI)` on `_ON` and `PPDI ^ One` on `_OFF`, and it works
because `PPDI` is 1. For the touchscreen, `T0IP` is 0, so `_ON` disconnects the interrupt and `_OFF` connects it.

Linux evaluates `_ON` in `acpi_power_on_unlocked()` whenever a power resource's reference count goes from 0 to 1. It doesn't
check `_STA` first. That happens at device enumeration and on every resume. Windows evidently doesn't evaluate `_ON` while
`_STA` already reports the resource on, so on Windows the routing set by `_INI` survives.

Confirmed on the live system (`/sys/kernel/debug/pinctrl/INTC105D:01/pins`):

```
pin 44 (GPP_E_18) 44:INTC105D:01 GPIO 0x80000102 0x0000006c 0x00000000   <- stock: bit 20 = 0, RXSTATE = 1 (line idle high)
pin 44 (GPP_E_18) 44:INTC105D:01 GPIO 0x80100102 0x0000006c 0x00000000   <- fixed: bit 20 = 1
```

`RXSTATE = 1` while the interrupt storms shows the ELAN controller is *not* asserting its interrupt line. The IO-APIC input is
simply disconnected and reads as asserted.

## 3. What the storm does

A 42-second `i2c_hid` transport trace on the stock driver (tracepoint added to `i2c_hid_get_input()`):

| Measurement | Value |
|---|---|
| I2C transactions | 26,155 in 42 s (~623/s), including 1,224 in the first 2 s with nobody touching the screen |
| `0xffff` "no data" reads | 95.4 % |
| valid reports | 4.4 % (touch `0x01` 694, pen `0x07` 403, vendor `0x22` 66) |
| malformed (`ff 1b 00 07 ...`, shifted by one byte) | 0.16 % |
| time from end of read to next IRQ | median 36 µs, max 317 µs; never longer in 42 s |
| read duration | ~1.55 ms (67 bytes at 400 kHz) |
| touch reports in a "touch after pen" phase | 0 |

The I2C bus is busy ~97 % of the time with reads the controller has nothing for. The pen drops to 25–27 Hz with 30–100 ms gaps.
Touch stalls, the controller sometimes wedges until a cold power cycle, and occasional stale stylus proximity makes libinput
treat a finger as a pointer.

### Windows, same machine

Windows 11 uses only the in-box stack: `mshidkmdf` → `hidi2c` → `ACPI`. There's no ELAN or HP driver and no vendor
initialisation (the boot trace shows only `SET_POWER ON` 0x0800 and `RESET` 0x0100).

| Measurement | Windows |
|---|---|
| pen reports | 1,701 in 6.345 s = **267.9 Hz**, median 3.40 ms, p99 6.38 ms |
| reads per interrupt | exactly one (ETW `Microsoft-Windows-SPB-HIDI2C`) |
| time from read completion to next IRQ | median 1.4 ms, 1st percentile 138 µs |

So on Windows the line goes inactive after every read. On Linux it never does.

## 4. The fix

Anything that sets bit 20 again after `_ON` fixes it:

- **ACPI table fix** (`tool/hp-touch-irq-fix`): change `\_SB.SGRA (TPI2, TPIP)` in `PTPL._ON` to `\_SB.SGRA (TPI2, TPLS)`.
  `TPLS` is always 1 there, because `PTPL` only exists inside `If ((TPLS == One))`. The name has the same length, so only four
  bytes change (plus the checksum and revision). The corrected table is loaded from the initramfs.
- **Kernel quirk** (`patches/`): `i2c-hid-acpi` calls `\_SB.SGRA (\GPLI, 1)` from its `power_up` hook, which runs at probe and on
  resume before the HID reset.

Result on board 8DA1 with the table fix: idle interrupts 0/s, `PADCFG0 = 0x80100102`. The pen tracks smoothly with pressure and
tilt, touch works, and touch is suppressed while the pen is in range (libinput's normal pen/touch arbitration; the touchscreen and
stylus share a libinput device group).

## 5. Why the table fix replaces more than one table

The kernel's initrd table upgrade (`drivers/acpi/tables.c`) replaces a firmware table with the first unused initrd table that
has the same signature, OEM ID and OEM table ID and a higher OEM revision. On this BIOS **all 31 SSDTs share
`HPQOEM`/`8DA1`**. So a single corrected table would replace the first SSDT, not the touchscreen one.

The tool therefore supplies copies of every SSDT up to the faulty one, in firmware order, each with its revision raised by one,
and changes only the faulty table's content.

One more quirk: **the XSDT lists SSDT1 twice** (entries 1 and 26, at different addresses, byte-identical). ACPICA normally skips
the second copy as a duplicate. Once entry 1 has been replaced, the copy no longer matches and would load twice
(`AE_ALREADY_EXISTS` for `\_SB.ACM`). The tool detects later duplicates from the boot log's table listing and replaces them with
an empty table of the same ID, so every table's content is loaded exactly once, in the original order.

## 6. Things that did not help (so you don't have to try them)

- Firmware, BIOS and linux-firmware updates. Kernels 7.0.11 to 7.2.2 were tried; changelogs up to 7.2.8 show no related fix
  as of September 2026.
- Switching the IRQ to edge-triggered at runtime.
- Polling the controller on a 3.7 ms timer with the IRQ masked. The pen reaches ~270 Hz, but it breaks touch hand-off and
  proximity, because it works around the symptom.
- Waiting for the IRQ line to settle after each read. The x86 IO-APIC `irq_get_irqchip_state()` only supports
  `IRQCHIP_STATE_ACTIVE`, so the line level can't be read, and the line never settles anyway.
- Salvaging misaligned `0xff`-prefixed frames. It recovers more reports but keeps the storm.
- HP's old EzTouchFilter package, runtime PM changes, and Intel THC/QuickI2C (not used on this platform).
