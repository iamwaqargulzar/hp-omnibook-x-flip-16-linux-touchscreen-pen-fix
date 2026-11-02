# HP OmniBook X Flip 16 / Flip 14: Linux touchscreen and pen fix (ELAN2514)

If you run Linux on an **HP OmniBook X Flip 16, OmniBook X Flip 14 or OmniBook 7 Flip 16** (boards 8DA1, 8D9F, 8DA0; Intel Lunar Lake) and the **stylus is laggy, the pen only
updates at about 25 Hz, touch stutters, or your finger turns into a mouse pointer**, the cause is a bug in HP's ACPI firmware.
It isn't the ELAN digitizer, and it isn't the i2c-hid driver. Firmware code that Windows never runs disconnects the
touchscreen interrupt from the interrupt controller, so Linux polls the controller blindly about 630 times a second.

This repository explains the bug and gives you a check and a fix that reconnects the interrupt. After the fix, the pen and touch
behave the way they do on Windows. **New to Linux? Go straight to [Install the fix, step by step](#install-the-fix-step-by-step).**
It also contains a kernel patch for upstream Linux.

| | Before | After |
|---|---|---|
| Touchscreen interrupts while idle | ~630 per second, forever | 0 |
| Interrupt pad `PADCFG0` (GPP_E_18) | `0x80000102` (routing off) | `0x80100102` (routing on) |
| Pen | ~25 Hz, bursty, skips | smooth, Windows-like |
| Touch | stutters, sometimes becomes a pointer | normal, and suppressed while the pen is in use |

Tested on an HP OmniBook X Flip Laptop 16-as0xxx, board 8DA1, BIOS F.20, Pop!_OS 24.04 with Linux 7.2.2 (ACPI table fix) and 7.2.8 and 7.2.11 (kernel patch).

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

## Install the fix, step by step

No Linux experience needed. You copy each command below into a terminal, one at a time, and press **Enter**. It takes about
10 minutes and one reboot.

Tips before you start:

- **Paste into a terminal with `Ctrl+Shift+V`**, not `Ctrl+V`. Each grey box with a command has a copy button on GitHub.
- **`sudo` asks for your login password.** Nothing appears on screen while you type it, not even dots. That's normal. Type it and
  press Enter.
- **The fix can be undone.** See [Remove the fix](#remove-the-fix).
- **Plugged in, please.** Keep the laptop on its charger, and don't update the BIOS while you do this.

### Step 1: Open a terminal

Press the **Super key** (the Windows key), type `terminal`, and press Enter. A window opens where you can type commands.

### Step 2: Install the two programs the fix needs

Not sure which Linux you have? Run `cat /etc/os-release` and look at the `NAME=` line. Then run the block for your system.

**Ubuntu, Pop!_OS, Kubuntu, Linux Mint, Debian:**

```bash
sudo apt update
```

```bash
sudo apt install -y git python3
```

**Fedora:**

```bash
sudo dnf install -y git python3
```

**Arch, CachyOS, Manjaro, EndeavourOS:**

```bash
sudo pacman -S --needed git python
```

If it says they are already installed, that's fine.

### Step 3: Download the fix

This copies the project into a new folder in your home folder, then moves the terminal into that folder.

```bash
cd ~
```

```bash
git clone https://github.com/iamwaqargulzar/hp-omnibook-x-flip-16-linux-touchscreen-pen-fix
```

```bash
cd hp-omnibook-x-flip-16-linux-touchscreen-pen-fix
```

If you closed the terminal and come back later, run `cd ~/hp-omnibook-x-flip-16-linux-touchscreen-pen-fix` first. If `git clone`
says the folder already exists, you downloaded it earlier: run that `cd` command, then `git pull` to get the latest version.

### Step 4: Check that your laptop has this bug

**Don't touch the screen while this runs.** It takes about 2 seconds and changes nothing.

```bash
sudo ./tool/hp-touch-irq-fix check
```

You'll see something like this:

```
model   : HP OmniBook X Flip Laptop 16-as0xxx (board 8DA1)
bios    : F.20 ...
touch   : i2c-ELAN2514:00 at \_SB.PC00.I2C4.TPL1, IRQ 108
idle IRQ: 630/s (keep hands off the screen; ~600/s = affected, ~0 = fine)
pad     : ...
routing : GPIROUTIOXAPIC = 0 (0 = bug present, 1 = fine)
fix     : not active on this boot
```

Look at the `idle IRQ` line:

- **Several hundred per second:** you have this bug. Go on to step 5. The `routing` line should say `0`; if it's missing, that's
  fine, because some systems hide it.
- **About 0 per second:** you don't have this bug, or it's already fixed. Stop here.
- **`no ELAN touchscreen ... found`:** this fix isn't for your laptop. Stop here.

### Step 5: Check that your system allows the fix

```bash
cat /sys/kernel/security/lockdown
```

- **`[none]` is the word in brackets, or it says `No such file or directory`:** good. Go on to step 6.
- **`[integrity]` or `[confidentiality]` is in brackets:** Secure Boot has locked this down, and the fix can't load. Follow
  [If Secure Boot blocks the fix](#if-secure-boot-blocks-the-fix) first.

### Step 6: Install the fix

**Ubuntu, Pop!_OS, Kubuntu, Linux Mint, Debian, Fedora:** this is one command. It may take a minute.

```bash
sudo ./tool/hp-touch-irq-fix install
```

**Arch, CachyOS, Manjaro, EndeavourOS:** these need two more commands. Run all three, in order. The second adds `acpi_override` to
your boot settings (it does nothing if that's already there), and the third rebuilds your boot files. The first command prints
`NEXT STEPS` and `Done. Reboot`: don't reboot yet, because the next two commands are those steps.

```bash
sudo ./tool/hp-touch-irq-fix install
```

```bash
grep -q '^HOOKS=.*acpi_override' /etc/mkinitcpio.conf || sudo sed -i 's/^HOOKS=(base /HOOKS=(base acpi_override /' /etc/mkinitcpio.conf
```

```bash
grep '^HOOKS=' /etc/mkinitcpio.conf | grep -q acpi_override && sudo mkinitcpio -P || echo "acpi_override was not added: see Troubleshooting"
```

If you see `error:` at any point, see [Troubleshooting](#troubleshooting). Otherwise, go on to step 7.

### Step 7: Restart

Save your work first.

```bash
sudo reboot
```

### Step 8: Confirm it worked

Open a terminal again. This command works from any folder. Don't touch the screen while it runs.

```bash
sudo hp-touch-irq-fix verify
```

The last line should say **`RESULT: PASS`**. Try the pen and touch. They should now be smooth. You're done, and you can delete the
downloaded folder if you like: the fix doesn't need it any more.

### Keeping the fix

- **Kernel updates:** nothing to do. New kernels pick up the fix automatically.
- **BIOS updates:** the fix is tied to your BIOS version. Before updating the BIOS, [remove the fix](#remove-the-fix). After the
  update, repeat steps 3 to 8. If you forget, a small service removes the fix by itself at the next boot, and the problem returns
  until you install it again.

### Remove the fix

**Ubuntu, Pop!_OS, Kubuntu, Linux Mint, Debian, Fedora:**

```bash
sudo hp-touch-irq-fix uninstall
```

**Arch, CachyOS, Manjaro, EndeavourOS:** run these two, in order.

```bash
sudo sed -i 's/ acpi_override//' /etc/mkinitcpio.conf
```

```bash
sudo hp-touch-irq-fix uninstall
```

Then restart:

```bash
sudo reboot
```

### If Secure Boot blocks the fix

You have two choices.

**Choice 1: turn Secure Boot off (works on every model).**

1. If this laptop also runs Windows with BitLocker, find your BitLocker recovery key first. Windows may ask for it after this
   change.
2. Restart, and press **F10** repeatedly as soon as the screen lights up. This opens the BIOS settings.
3. Find **Secure Boot** (usually under Security or Boot Options), turn it off, then save and exit.
4. Back in Linux, run the command from step 5 again. If it now shows `[none]`, go on to step 6.

**Choice 2: keep Secure Boot on and use a DKMS module.** This is a separate project,
[testyfishy/hp-omnibook-flip16-touchscreen-fix](https://github.com/testyfishy/hp-omnibook-flip16-touchscreen-fix). Its installer
is written for Ubuntu and Debian, and it only covers boards **8DA0 and 8DA1**, so it won't help on the Flip 14 (board 8D9F). The
`check` output from step 4 shows your board. If yours matches, run these one at a time:

```bash
cd ~
```

```bash
git clone https://github.com/testyfishy/hp-omnibook-flip16-touchscreen-fix
```

```bash
cd hp-omnibook-flip16-touchscreen-fix
```

```bash
sudo ./install.sh
```

```bash
sudo reboot
```

After the restart, this should print a line containing `HP quirk`:

```bash
journalctl -b -k | grep 'HP quirk'
```

If it prints nothing, your firmware doesn't trust the module's signature yet. Run this and choose a temporary password:

```bash
sudo mokutil --import /var/lib/dkms/mok.pub
```

Then run `sudo reboot`. A blue **Perform MOK management** screen appears during the restart. Choose **Enroll MOK**, then
**Continue**, then **Yes**, type that temporary password, and choose **Reboot**.

### Troubleshooting

| What you see | What it means and what to do |
|---|---|
| `git: command not found` or `python3: not found` | Step 2 was skipped or failed. Run it again. |
| `No such file or directory` for `./tool/hp-touch-irq-fix` | The terminal is in the wrong folder. Run `cd ~/hp-omnibook-x-flip-16-linux-touchscreen-pen-fix` and try again. |
| `error: kernel lockdown is active` | Secure Boot is blocking the fix. See [If Secure Boot blocks the fix](#if-secure-boot-blocks-the-fix). |
| `error: an ACPI table override is active on this boot` | The fix, or a similar one, is already running. Run `sudo hp-touch-irq-fix verify`. To reinstall, remove the fix, restart, then repeat step 6. |
| `error: could not read the firmware table listing` | The startup log has been cleared. Restart, then run the command again straight away. |
| `error: the faulty PTPL._ON call was not found` | Your BIOS doesn't contain this exact bug, so the tool changed nothing. Please [open an issue](https://github.com/iamwaqargulzar/hp-omnibook-x-flip-16-linux-touchscreen-pen-fix/issues). |
| `sudo: hp-touch-irq-fix: command not found` | Use the full path instead: `sudo /usr/local/sbin/hp-touch-irq-fix verify` |
| `acpi_override was not added` (Arch family) | Your `HOOKS=` line has a different layout. Run `sudo nano /etc/mkinitcpio.conf`, add the word `acpi_override` right after `base` on the `HOOKS=` line, save with `Ctrl+O`, Enter, then `Ctrl+X`, and run `sudo mkinitcpio -P`. |
| `RESULT: FAIL` with `overrides applied : 0` | The fix didn't load. On the Arch family, repeat the last two commands of step 6. Then check step 5 again, restart, and rerun `verify`. |
| The laptop won't start normally | Open the boot menu: hold **Space** while it starts on Pop!_OS, or hold **Shift** or tap **Esc** on Ubuntu, Fedora and most others. Pick an older kernel entry, then [remove the fix](#remove-the-fix). On the Arch family every entry includes the fix, so ask for help in an issue. |

Still stuck? [Open an issue](https://github.com/iamwaqargulzar/hp-omnibook-x-flip-16-linux-touchscreen-pen-fix/issues) and paste
the output of step 4 and the exact error. No GitHub account? Comment on
[kernel bugzilla 220854](https://bugzilla.kernel.org/show_bug.cgi?id=220854) instead (that also needs a free account).

## How the fix works

The tool reads **your own** firmware tables and corrects one argument in the touchscreen's power-on method. That's four bytes.
It then loads the corrected table at boot through the kernel's standard
[ACPI table upgrade](https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html) mechanism. Nothing from HP is
downloaded or redistributed, and your kernel isn't modified.

**Limits:**

- **Kernel lockdown must be off.** In lockdown mode the kernel ignores ACPI table overrides. Ubuntu and Fedora turn lockdown on
  whenever Secure Boot is on; Arch does not, so Secure Boot itself can stay on there. The tool checks this and refuses to install.
- **The fix is tied to your BIOS version.** A small boot-time service removes the fix automatically when it sees a different BIOS,
  and the initramfs hook refuses to add tables built for another BIOS.
- The tool refuses to run unless it finds exactly the faulty call in the touchscreen's scope. If HP fixes the BIOS, it will say so
  and do nothing.

| Distribution | initramfs | Status |
|---|---|---|
| Pop!_OS 24.04 | initramfs-tools | corrected tables tested on hardware; `install` itself not yet run |
| Ubuntu 24.04 / 26.04 | initramfs-tools | same code path as Pop!_OS, expected to work (Secure Boot must be off) |
| Fedora | dracut | **works**: confirmed by another owner (board 8DA1, BIOS F.10, `04F3:43EF`) in [kernel bugzilla 220854](https://bugzilla.kernel.org/show_bug.cgi?id=220854) |
| Arch Linux, CachyOS, EndeavourOS | mkinitcpio | **works**: confirmed on Arch (OmniBook 7 Flip 16, board 8DA0) and on CachyOS and another mkinitcpio system (OmniBook X Flip 14, board 8D9F); later kernel updates keep the tables |

Reports from other distributions are welcome. Please open an issue with the output of `check` and `verify`.

## For kernel developers: the kernel patch

You don't need this unless you build your own kernels. [`patches/0001-HID-i2c-hid-acpi-restore-touchscreen-IRQ-routing-on-HP-OmniBook-Flip-laptops.patch`](patches/)
is patch v2 as posted upstream. It makes the `i2c-hid-acpi` driver call the firmware's own `\_SB.SGRA(\GPLI, 1)` helper after every
power-on, before the HID reset. It only applies on HP boards 8D9F/8DA0/8DA1 with an ELAN2514 touchscreen.

**Status: tested on hardware.** On board 8DA1, BIOS F.20 (Linux 7.2.8 and 7.2.11) and on board 8D9F, BIOS F.08 (Linux 7.2.9, by
another owner), the ELAN interrupt stays at 0/s while idle after boot and after s2idle suspend/resume, and pen and touch work.
Posted to the HID maintainers: [v1](https://patch.msgid.link/20260930-b4-elan2514-irq-route-v1-1-44671bf01d05@gmail.com)
(2026-09-30), [v2](https://patch.msgid.link/20261008-b4-elan2514-irq-route-v2-1-432d95eb7348@gmail.com) (2026-10-08, adds the
Flip 14 board 8D9F). See [docs/upstream.md](docs/upstream.md).

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
No. The [step-by-step install](#install-the-fix-step-by-step) works with the stock kernels of Ubuntu, Pop!_OS, Fedora and Arch, as
long as kernel lockdown is off (step 5 checks this).

### Is the ACPI table fix safe?
It changes a single argument in one method. Every other byte of your firmware tables is loaded unchanged. HP ships several tables
with identical IDs (and on 8DA1 one table twice), so the tool replaces the tables before the faulty one in their original order,
changing only their revision numbers. The layout is verified with `verify` after the reboot. If a boot ever misbehaves, see
[Troubleshooting](#troubleshooting).

### Will kernel updates remove the fix?
They shouldn't. The initramfs hook adds the corrected tables to every initramfs that gets generated, including ones for new
kernels.

### Does this fix the slow pen report rate from Launchpad bug 2142384?
On the Flip 16 (board 8DA1), yes. The slow pen was a consequence of the interrupt being disconnected. With the routing fixed, the
pen runs at full rate without any frame-salvaging patch.

### What about suspend and resume?
The fixed `_ON` runs on every resume, so the interrupt stays connected after sleep.

### Other HP models
Confirmed with the ACPI table fix (same faulty `_ON`, `verify` passes):

| Model | Board | BIOS | Digitizer | Distribution |
|---|---|---|---|---|
| OmniBook X Flip 16-as0xxx | 8DA1 | F.20 | `04F3:43F0` | Pop!_OS 24.04 |
| OmniBook X Flip 16-as0xxx | 8DA1 | F.10 | `04F3:43EF` | Fedora 44 |
| OmniBook 7 Flip 16-au0773cl | 8DA0 | F.08 | `04F3:43EF` | Arch Linux |
| OmniBook X Flip 14-fm0xxx | 8D9F | F.08 | `04F3:442A` | CachyOS |
| OmniBook X Flip 14-fm0xxx | 8D9F | F.20 | `04F3:4428` | mkinitcpio-based |

Also fixed with testyfishy's DKMS module (same IRQ storm): OmniBook X Flip 16-as0001TU (8DA1, F.10, `04F3:43F5`, Kubuntu) and
OmniBook X 16-as0043dx (8DA1, F.20, `04F3:43F0`, Manjaro).

The OmniBook X Flip 14 (board 8D9F) has the byte-identical faulty `_ON`, on the same pad (GPP_E_18). Other Flip 14 digitizers
(`04F3:442E`/`4430`) are likely the same. Run `check`: if the idle IRQ rate is in the hundreds and routing is `0`, you
have the same bug. `install` only acts when it finds the exact faulty call in the touchscreen's scope, and refuses otherwise. Please
open an issue with your `check` output either way.

## Upstream status

Last checked: 2026-11-02.

- Linux kernel: **not merged yet.** The fix is not in Linux 7.3, 7.4-rc1 or the HID `for-next` branch.
  [Patch v2 was posted 2026-10-08](https://patch.msgid.link/20261008-b4-elan2514-irq-route-v2-1-432d95eb7348@gmail.com) (boards
  8D9F, 8DA0, 8DA1) and is waiting for maintainer review; notes in [docs/upstream.md](docs/upstream.md). Two other owners have
  given `Tested-by` lines, on boards 8DA1 and 8D9F. The automated review asked whether the fix is skipped on resume when the
  touchscreen is a wakeup source; it is not one on these boards (no `_PRW`, no wake-capable interrupt, no `power/wakeup` in
  sysfs), so the fix runs on every resume. Until it is merged, use the [step-by-step install](#install-the-fix-step-by-step).
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
