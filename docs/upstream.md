# Upstream notes

## Kernel patch

File: [`patches/0001-HID-i2c-hid-acpi-restore-touchscreen-IRQ-routing-on-HP-OmniBook-X-Flip-16.patch`](../patches/)

- **What it does:** on HP boards 8D9F/8DA0/8DA1 with an `ELAN2514` device, `i2c-hid-acpi` installs a `power_up` hook. The hook reads
  `\GPLI` (the touchscreen's GPIO pad number) and calls the firmware's `\_SB.SGRA(pad, 1)` to set `GPIROUTIOXAPIC` again. The
  core calls `power_up` at probe and at resume, before the HID reset, which is right after ACPI has run the faulty `PTPL._ON`.
  Failures are logged and never fail probe or resume.
- **Status:** v1 posted 2026-09-30: https://patch.msgid.link/20260930-b4-elan2514-irq-route-v1-1-44671bf01d05@gmail.com
  v2 posted 2026-10-08 (adds 8D9F, corrects the 8DA0 description): https://patch.msgid.link/20261008-b4-elan2514-irq-route-v2-1-432d95eb7348@gmail.com
  - Builds with `W=1` with no warnings; `checkpatch.pl` reports 0 errors and 0 warnings.
  - Applies to `drivers/hid/i2c-hid/i2c-hid-acpi.c` in Linux 7.2.8.
  - Tested on board 8DA1, BIOS F.20, Linux 7.2.8: IRQ 0/s idle after boot and after s2idle resume; pen and touch normal.

### Before sending

1. Done: probe and s2idle resume tested on 8DA1. More resume cycles and a cold boot or two don't hurt.
2. Put your real name and email in the `From:` line and add `Signed-off-by:` (required by the
   [Developer's Certificate of Origin](https://docs.kernel.org/process/submitting-patches.html#sign-your-work-the-developer-s-certificate-of-origin)).
3. Add `Tested-by:` lines from other owners who tested it, with their permission.
4. Rebase on the current `hid.git` `for-next` branch.

### Where to send it

`./scripts/get_maintainer.pl` on the patch lists the current recipients. As of 2026 they are:

- To: Jiri Kosina, Benjamin Tissoires (HID maintainers)
- Cc: `linux-input@vger.kernel.org`, `linux-kernel@vger.kernel.org`, `linux-acpi@vger.kernel.org`

It should also be sent as a reply to, or with a reference to, the May 2026 linux-i2c thread about the ELAN2514 IRQ flood, and
reference Launchpad bug 2142384.

### Questions reviewers are likely to ask

- **Why not fix it in the ACPI core?** The generic fix would be to skip `_ON` when `_STA` reports a power resource already on,
  which appears to be what Windows does. That changes behaviour on every machine and needs the ACPI maintainers' agreement. The
  device quirk is the minimal fix.
- **Why call firmware methods instead of writing the pad register?** The pad and its register layout are platform details that
  the firmware already abstracts. `\_SB.SGRA` and `\GPLI` are what the firmware itself uses.
- **What about 8DA0 and 8D9F?** 8DA0 is the OmniBook 7 Flip 16 and 8D9F the OmniBook X Flip 14. Their owners confirmed the
  same faulty `_ON` and IRQ storm, and that the equivalent ACPI table correction fixes it (kernel bugzilla 220854).

## Firmware

The real fix belongs in HP's BIOS: `PTPL._ON` should write `TPIP ^ One` (as `TPL1._INI` does), and `_OFF` should write `TPIP`.
Report it through HP support with a reference to this repository, the board ID (8DA1) and the BIOS version.

## Distribution bugs

- Ubuntu: [Launchpad 2168946](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2168946) tracks this bug and carries the patch; ask for the backport there once the patch is
  merged upstream. The older [2142384](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2142384) is marked Fix Released for
  an out-of-tree workaround.
- Kernel Bugzilla: [220854](https://bugzilla.kernel.org/show_bug.cgi?id=220854).
