# HP OmniBook X Flip 16: Linux touchscreen and pen fix (ELAN2514)

If you run Linux on an **HP OmniBook X Flip 16** (board 8DA0/8DA1, Intel Lunar Lake) and the **stylus is laggy, the pen only
updates at about 25 Hz, touch stutters, or your finger turns into a mouse pointer**, the cause is a bug in HP's ACPI firmware.
It isn't the ELAN digitizer, and it isn't the i2c-hid driver. Firmware code that Windows never runs disconnects the
touchscreen interrupt from the interrupt controller, so Linux polls the controller blindly about 630 times a second.

This repository explains the bug, gives you a one-command check, and a fix that reconnects the interrupt. After the fix, the
pen and touch behave the way they do on Windows. It also contains a kernel patch for upstream Linux.

| | Before | After |
|---|---|---|
| Touchscreen interrupts while idle | ~630 per second, forever | 0 |
| Interrupt pad `PADCFG0` (GPP_E_18) | `0x80000102` (routing off) | `0x80100102` (routing on) |
| Pen | ~25 Hz, bursty, skips | smooth, Windows-like |
| Touch | stutters, sometimes becomes a pointer | normal, and suppressed while the pen is in use |

Tested on an HP OmniBook X Flip Laptop 16-as0xxx, board 8DA1, BIOS F.20, Pop!_OS 24.04 with Linux 7.2.2 (ACPI table fix) and 7.2.8 (kernel patch).

## Symptoms

Any of these on Linux:

- The pen or stylus draws in jagged segments, or skips while you write (Xournal++, Krita, Write, OneNote in a browser).
- Touch scrolling stutters, touch stops working after using the pen, or a finger acts like a mouse cursor.
- `dmesg` shows lines like:
  ```
  i2c_hid_acpi i2c-ELAN2514:00: i2c_hid_get_input: IRQ triggered but there's no data
  i2c_hid_acpi i2c-ELAN2514:00: i2c_hid_get_input: incomplete report (67/7167)
  i2c_hid_acpi i2c-ELAN2514:00: i2c_hid_get_input: incomplete report (67/65280)
  ```
- The ELAN interrupt count in `/proc/interrupts` climbs by hundreds per second while you aren't touching anything:
  ```
  grep ELAN /proc/interrupts; sleep 1; grep ELAN /proc/interrupts
  ```
- One CPU core never idles, and the laptop doesn't reach deep package C-states, so battery life suffers.

The device shows up as `ELAN2514:00 04F3:43F0` (other HP models report `04F3:4428`, `04F3:442A`, `04F3:43F5`, `04F3:43EF`;
see [Other models](#other-hp-models)).

## Check if you are affected

```
git clone https://github.com/iamwaqargulzar/hp-omnibook-x-flip-16-linux-touchscreen-pen-fix
cd hp-omnibook-x-flip-16-linux-touchscreen-pen-fix
sudo ./tool/hp-touch-irq-fix check
```

You're affected if `idle IRQ` shows several hundred per second and `routing` shows `0`.

## Fix

### Option A: ACPI table fix (works today, no custom kernel)

The tool reads **your own** firmware tables and corrects one argument in the touchscreen's power-on method. That's four bytes.
It then loads the corrected table at boot through the kernel's standard
[ACPI table upgrade](https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html) mechanism. Nothing from HP is
downloaded or redistributed, and your kernel isn't modified.

```
sudo ./tool/hp-touch-irq-fix install
sudo reboot
sudo hp-touch-irq-fix verify
```

`verify` should end with `RESULT: PASS`.

To remove it:

```
sudo hp-touch-irq-fix uninstall
sudo reboot
```

**Requirements and limits:**

- **Secure Boot must be off.** With Secure Boot on, the kernel runs in lockdown mode and ignores ACPI table overrides. The tool
  checks this and refuses to install. Use Option B instead.
- **The fix is tied to your BIOS version.** Before updating the BIOS, run `uninstall`. As a safety net, a small boot-time service
  removes the fix automatically when it sees a different BIOS, and the initramfs hook refuses to add tables built for another BIOS.
  After the update, reboot once and run `install` again.
- The tool refuses to run unless it finds exactly the faulty call in the touchscreen's scope. If HP fixes the BIOS, it will say so
  and do nothing.

| Distribution | initramfs | Status |
|---|---|---|
| Pop!_OS 24.04 | initramfs-tools | corrected tables tested on hardware; `install` itself not yet run |
| Ubuntu 24.04 / 26.04 | initramfs-tools | same code path as Pop!_OS, expected to work (Secure Boot must be off) |
| Fedora | dracut | **works**: confirmed by another owner (board 8DA1, BIOS F.10, `04F3:43EF`) in [kernel bugzilla 220854](https://bugzilla.kernel.org/show_bug.cgi?id=220854) |
| Arch Linux, CachyOS, EndeavourOS | mkinitcpio | supported, untested: add `acpi_override` to `HOOKS` in `/etc/mkinitcpio.conf`, then `sudo mkinitcpio -P` |

Reports from other distributions are welcome. Please open an issue with the output of `check` and `verify`.

### Option B: kernel patch (for Secure Boot users, and for upstream)

[`patches/0001-HID-i2c-hid-acpi-restore-touchscreen-IRQ-routing-on-HP-OmniBook-X-Flip-16.patch`](patches/) makes the
`i2c-hid-acpi` driver call the firmware's own `\_SB.SGRA(\GPLI, 1)` helper after every power-on, before the HID reset. It only
applies on HP boards 8DA0/8DA1 with an ELAN2514 touchscreen.

**Status: tested on hardware.** Built into Linux 7.2.8 (with Pop!_OS's kernel patches) on board 8DA1, BIOS F.20: the ELAN
interrupt stays at 0/s while idle after boot and after s2idle suspend/resume, and pen, touch and palm rejection work. It passes
`checkpatch.pl` and applies to Linux 7.2.8. Posted to the HID maintainers on 2026-09-30: [v1 on lore](https://patch.msgid.link/20260930-b4-elan2514-irq-route-v1-1-44671bf01d05@gmail.com). See
[docs/upstream.md](docs/upstream.md) for how it will be submitted.

## What is actually wrong

The touchscreen interrupt pin is GPIO pad **GPP_E_18** (`INTC105D:01` pin 44). It reaches the IO-APIC as IRQ 108 only while bit
20 (`GPIROUTIOXAPIC`) of the pad's `PADCFG0` register is set.

HP's SSDT defines a power resource for the touchscreen with the value backwards:

```
PowerResource (PTPL, ...)            // \_SB.PC00.I2C4.PTPL, used by TPL1 (the ELAN2514)
{
    Method (_ON)  { ... \_SB.SGRA (TPI2, TPIP) }          // TPIP = T0IP = 0  -> routing OFF
    Method (_OFF) { ... \_SB.SGRA (TPI2, TPIP ^ One) }    //                  -> routing ON
}
```

The device's own `_INI` gets it right (`SGRA (GPLI, T0IP ^ One)`), and the touchpad's power resource uses the same pattern
correctly. Linux evaluates `_ON` the first time it powers the device and after every resume. Windows doesn't evaluate it while the
resource already reports "on", so Windows never hits the bug.

With the pad disconnected, the level-triggered IO-APIC input stays asserted. Linux services the "interrupt" over and over:

- Each read takes ~1.55 ms, and the next interrupt fires ~40 µs later.
- 95 % of reads return `0xffff` (no data).
- Some reads catch the controller mid-report, which gives the `ff 1b 00 07 ...` shifted frames and the `incomplete report
  (67/7167)` messages.
- The controller spends its time answering garbage reads, so the pen drops from its native ~268 Hz (measured on Windows) to ~25 Hz.

The full analysis, with traces and measurements, is in [docs/technical-details.md](docs/technical-details.md).

## FAQ

### Is my touchscreen or pen broken?
No. The digitizer delivers ~268 pen reports per second on Windows and on Linux once the interrupt is connected. The problem is
entirely in how the firmware configures the interrupt pin.

### Why does it work on Windows?
Windows uses the same ACPI tables and the same standard HID-over-I2C driver (`hidi2c.sys`, no ELAN or HP driver). It just
doesn't run the faulty `_ON` method at boot, so the interrupt stays connected.

### Do I need a custom kernel?
Not for Option A. It works with the stock kernels of Ubuntu, Pop!_OS, Fedora and Arch, as long as Secure Boot is off.

### Is the ACPI table fix safe?
It changes a single argument in one method. Every other byte of your firmware tables is loaded unchanged. HP ships several tables
with identical IDs (and on 8DA1 one table twice), so the tool replaces the tables before the faulty one in their original order,
changing only their revision numbers. The layout is verified with `verify` after the reboot. If a boot ever misbehaves, pick
another kernel entry in the boot menu, run `uninstall`, and you're back to stock.

### Will kernel updates remove the fix?
They shouldn't. The initramfs hook adds the corrected tables to every initramfs that gets generated, including ones for new
kernels.

### Does this fix the slow pen report rate from Launchpad bug 2142384?
On the Flip 16 (board 8DA1), yes. The slow pen was a consequence of the interrupt being disconnected. With the routing fixed, the
pen runs at full rate without any frame-salvaging patch.

### What about suspend and resume?
The fixed `_ON` runs on every resume, so the interrupt stays connected after sleep.

### Other HP models
The HP OmniBook X Flip 14, OmniBook 7 Flip 16 and other OmniBook X Flip 16 variants show the same symptoms in bug reports, but
nobody has confirmed yet that their firmware has the same faulty `_ON`. Run `check`: if the idle IRQ rate is in the hundreds and
routing is `0`, you probably have the same bug. `install` only acts when it finds the exact faulty call in the touchscreen's
scope, and refuses otherwise. Please open an issue with your `check` output either way.

## Upstream status

- Linux kernel: [patch v1 posted 2026-09-30](https://patch.msgid.link/20260930-b4-elan2514-irq-route-v1-1-44671bf01d05@gmail.com) to linux-input, linux-acpi and stable; notes in [docs/upstream.md](docs/upstream.md).
- Ubuntu: [Launchpad bug 2168946](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2168946) (this firmware bug, with the patch attached); older report
  [2142384](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2142384) (pen report rate).
- Kernel Bugzilla: [bug 220854](https://bugzilla.kernel.org/show_bug.cgi?id=220854).
- linux-i2c list: [ELAN2514 IRQ flood thread, May 2026](https://ratatoskr.run/linux-i2c/2026/05/9013594/t).
- HP: the real fix is a BIOS update that makes `PTPL._ON` write `TPIP ^ One`.

## Credits

- [testyfishy/hp-omnibook-flip16-touchscreen-fix](https://github.com/testyfishy/hp-omnibook-flip16-touchscreen-fix) found the
  `PTPL._ON` / `GPIROUTIOXAPIC` problem on the same board and fixed dead touchscreens with an i2c-hid-acpi DKMS module.
- [perryFile/hp-elan2514-pen-fix](https://github.com/perryFile/hp-elan2514-pen-fix) and the reporters on Launchpad bug 2142384
  documented the misaligned-frame symptom on the Flip 14.
- This repository adds the Windows comparison traces, the proof that the routing fix also restores the full pen rate, the
  distribution-independent ACPI table fix, and the upstream patch.

## License

GPL-2.0-only. See [LICENSE](LICENSE).
