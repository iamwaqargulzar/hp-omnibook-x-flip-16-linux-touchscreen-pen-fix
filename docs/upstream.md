# Upstream notes

## Kernel patch

File: [`patches/0001-HID-i2c-hid-acpi-restore-touchscreen-IRQ-routing-on-HP-OmniBook-X-Flip-16.patch`](../patches/)

- **What it does:** on HP boards 8DA0/8DA1 with an `ELAN2514` device, `i2c-hid-acpi` installs a `power_up` hook. The hook reads
  `\GPLI` (the touchscreen's GPIO pad number) and calls the firmware's `\_SB.SGRA(pad, 1)` to set `GPIROUTIOXAPIC` again. The
  core calls `power_up` at probe and at resume, before the HID reset, which is right after ACPI has run the faulty `PTPL._ON`.
  Failures are logged and never fail probe or resume.
- **Status:** tested, not yet posted.
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
- **What about 8DA0?** It's the same platform family and was reported with the same symptom by the author of
  `testyfishy/hp-omnibook-flip16-touchscreen-fix`. It still needs to be confirmed on hardware before submission.

## Firmware

The real fix belongs in HP's BIOS: `PTPL._ON` should write `TPIP ^ One` (as `TPL1._INI` does), and `_OFF` should write `TPIP`.
Report it through HP support with a reference to this repository, the board ID (8DA1) and the BIOS version.

## Distribution bugs

- Ubuntu: [Launchpad 2142384](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2142384). Add the finding that the slow pen
  report rate on board 8DA1 is fixed by restoring the IRQ routing.
- Kernel Bugzilla: [220854](https://bugzilla.kernel.org/show_bug.cgi?id=220854).
