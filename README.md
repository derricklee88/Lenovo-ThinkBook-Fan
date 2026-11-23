# thinkbook-fan-modes

Thermal-mode switching and fan-speed readout for the **Lenovo ThinkBook 15p IMH
(`20V3`)** on Linux, packaged for Arch.

The reverse-engineered Windows tool
[lhzlhz419/ThinkBookFanControl](https://github.com/lhzlhz419/ThinkBookFanControl)
is C#/WPF and cannot run here. This package talks to the same class of Lenovo
WMI interface, from user space, without a custom kernel module.

## What your firmware actually allows

Decoded from this machine's DSDT (`iasl -d`) and verified live:

| Call | Meaning |
|---|---|
| `\_SB.GZFD.WMAA(0, 0x08, 0)` | fan 1 RPM |
| `\_SB.GZFD.WMAA(0, 0x09, 0)` | fan 2 RPM |
| `\_SB.GZFD.WMAA(0, 0x0A, 0)` | nominal maximum RPM (3400; fan 2 reaches 3800) |
| `\_SB.GZFD.WMAA(0, 0x2C, N)` | set thermal mode (N = 1, 2, 3) |
| `\_SB.GZFD.WMAA(0, 0x2D, 0)` | current thermal mode |
| `.../ideapad_acpi/VPC2004:00/fan_mode` | EC fan mode; `2` bursts both fans to ~5100/5200 rpm |

`0x2C` is the same path the Fn+Q hotkey uses; it writes EC register `FCMO`.

**There is no fan curve or RPM target on this firmware.** No `WMLA`, no `FANG`,
and `FANS` is read-only in every OEM code path — the embedded controller owns
the curve, so this tool offers three modes and telemetry, not a curve.

What the EC does have is its fan-mode channel (the one `ideapad_laptop` exposes
as `fan_mode`). Writing `2` there — the documented *Dust Cleaning* state — spins
both fans to **~5100/5200 rpm within two seconds**, well above any thermal mode.
That is what the GUI's **Full speed turbo** button uses; see below.

The in-kernel `lenovo_wmi_gamezone` driver matches this device's GUID but fails
to probe: it expects the WMI method to return a 4-byte buffer, while this 2020
firmware returns a bare integer. Hence the `acpi_call` dependency.

## Firmware quirks this tool works around

The DSDT declares an object named `PABD` as external, reads and writes it in
the mode paths, and **no ACPI table on this machine defines it** (7 references
in the DSDT binary, 0 in all 15 SSDTs, with a 1-bit hole in the `ERAX` field
where it should live). Consequences, all confirmed on hardware:

- `0x2C <mode>` stores `FCMO`, then aborts with `AE_NOT_FOUND` on `PABD`. The
  mode change still lands, so `thinkbook-fan mode` treats that specific abort as
  benign and verifies by read-back. You will not see the error.
- `0x2D` aborts with `AE_NOT_FOUND` when `FCMO` is `0`, because that branch
  reads `PABD` before returning `2`. That abort *is* the `FCMO == 0` reading,
  which is the aggressive curve.
- On battery, `auto` (`0x2C 3`) stores the same `FCMO` value as `performance`,
  and the bit that distinguished them was `PABD`. So on battery the two are
  reported identically as `performance`; that is the firmware, not a bug here.

Any other ACPI error is reported and not swallowed.

## Install

```sh
cd ~/Projects/thinkbook-fan-modes
makepkg -si
```

This installs `/usr/bin/thinkbook-fan`, `/usr/bin/thinkbook-fan-gui`, the
Python package under `/usr/lib/thinkbook-fan`, the root helper and its polkit
action, a desktop entry, an optional `thinkbook-fan-mode.service`, and
`/etc/thinkbook-fan-modes.conf`. The `acpi_call` module is autoloaded by its own
`modules-load.d` entry, shipped by `acpi_call-dkms`.

The boot unit is **not** enabled by the package. To reapply a mode at every
boot:

```sh
sudoedit /etc/thinkbook-fan-modes.conf     # set MODE=quiet|auto|performance
sudo systemctl enable --now thinkbook-fan-mode.service
```

## Use

```sh
sudo thinkbook-fan status         # mode, power source, both fan RPMs, CPU temperature
sudo thinkbook-fan watch          # one line per second
thinkbook-fan modes               # list the three modes (no root needed)
sudo thinkbook-fan mode quiet     # set mode, verified by read-back
sudo thinkbook-fan mode auto
sudo thinkbook-fan mode performance
sudo thinkbook-fan raw 0x0a 0     # arbitrary WMAA command, prints hex
```

`/proc/acpi/call` is mode `0200`, so reads need root as well as writes. If you
want unprivileged telemetry, run `thinkbook-fan watch` as a systemd unit that
writes to `/run`; the package does not do that yet.

Writes are serialised with a lock in `/run`, so `watch` cannot interleave with
a boot-time mode change. Every write is read back; if the firmware keeps a
different mode, the previous mode is restored and the command exits 1.

## GUI

`thinkbook-fan-gui` is a PyQt6 window, also in the application menu as
**ThinkBook Fan** (red fan icon, installed as `thinkbook-fan.svg`). It shows the
three modes, both fan RPMs, CPU temperature and
power source, has the "reapply this mode at every boot" checkbox, and plots the last
two minutes of both fans (rpm) and CPU temperature (°C) on a small custom-painted
graph, each against its own vertical scale. The graph is drawn with `QPainter`
rather than QtCharts, so the package needs no extra dependency.

The window is unprivileged. On launch it starts
`/usr/lib/thinkbook-fan/thinkbook-fan-helper` through `pkexec` and keeps it for
the life of the window, so you authenticate once instead of once per click.
The helper is mode `0700` and the shipped polkit action
(`org.thinkbookfan.helper`) uses `auth_admin_keep`. Requests are JSON lines over
pipes, so no display or other environment is handed to the root process.

Closing or minimising the window hides it to the tray instead of quitting —
the helper keeps running, so an armed turbo latch is not silently cancelled.
The tray menu has **Show window**, **Full speed turbo**, and **Quit**; quitting
is the only thing that stops the helper. If no system tray is available,
closing really closes.

If you dismiss the password dialog the window stays open, says why, and shows
**Reconnect**. It does not re-prompt on a timer. The same helper runs directly
when the GUI is started as root, which is why both paths work.

The CLI above is unchanged and remains the way to script things; the GUI is
just the same three modes with the readout attached.

## Full speed turbo

The GUI's **Full speed turbo** button writes `2` to the EC's `fan_mode`
attribute and re-writes it every 20 s while it is on. Measured on this machine:

| Point | fan 1 | fan 2 |
|---|---|---|
| before | 2400 | 2500 |
| `2` written, +2 s | 5100 | 5200 |
| +4 s to +60 s | ~5000 (brief self-pulses to 2100) | ~5500 (same pulses) |
| `0` written, +2 s | 1800 | 1900 |

Any non-zero value does this: `1`, `2`, `4`, and `5` (the last via the OEM's
`VPC0.VPCW` method, which `ideapad_laptop` refuses) all select the same maximum
program. `0` is the only other program, and it is a quiet *cap* — ~2000 rpm at
idle, no ramp with load: 94 °C package at 2500 rpm under full load.

That is the uncomfortable part, and the window says it: **once `fan_mode` is
written, the EC leaves its automatic curve, and only a reboot brings it back.**
Turning turbo off writes `0`; that is right at idle and wrong under sustained
load. If you use turbo for a game, either leave it armed while playing or
reboot afterwards rather than sitting in quiet mode.

Read-back of `fan_mode` is unreliable here (`0` reads back `133`, non-zero reads
back `3`), so the tool never trusts it — it writes and watches RPM only.

## Limitations

- Verified only on DMI product `20V3`. On anything else, writes are refused;
  reads still work. Adding a model means decompiling its DSDT and confirming the
  same `WMAA` commands before extending `SUPPORTED_PRODUCTS`.
- Degrades gracefully but does not integrate with `sensors`: fan RPM is not a
  hwmon channel. That needs a kernel driver with a buffer-returning shim.
- Mode names describe measured behaviour, not Lenovo's adjectives. Under full
  load at 94 °C on this machine, `quiet` (`0x2C 1`, `FCMO 0x02`) settles at
  2900/3100 rpm while `performance` (`0x2C 2`, `FCMO 0x00`) reaches
  3400/3800 rpm. The firmware's own 1/2/3 numbering is unchanged in `raw`, so
  `raw 0x2c 1` is still the low-fan mode.
- Turbo is a latch, not a level: the EC still owns the speed, it dips to
  ~2100 rpm for a second or two every ~10 s while armed, and the EC's automatic
  curve does not come back until a reboot.

## Remove

```sh
sudo systemctl disable --now thinkbook-fan-mode.service
sudo pacman -Rns thinkbook-fan-modes
```

## Development

```sh
python3 -m unittest discover -s tests
```

Design notes: `docs/superpowers/specs/2026-11-22-thinkbook-fan-modes-design.md`.

## Does the mode actually change fan speed?

Yes, measured. At idle every mode sits at the fan floor (~2100 rpm), so a mode
change on a 45 °C package looks like a no-op. Under load the difference is large
and, on this firmware, points the opposite way from Lenovo's adjectives.

`tools/measure.sh` sets a mode, runs `nproc-1` busy loops, and samples both fans
plus the package temperature every 2 s:

```sh
sudo ./tools/measure.sh quiet 60
sleep 120
sudo ./tools/measure.sh performance 60
```

Measured on `20V3` (AC, PL1 60 W / PL2 90 W, unchanged by mode):

| mode | `0x2C` arg | `FCMO` | fan 1 max | fan 2 max | temp |
|---|---|---|---|---|---|
| `quiet` | 1 | `0x02` | 2900 | 3100 | 94 °C |
| `performance` | 2 | `0x00` | 3600 | 3900 | 94 °C |

The low-fan mode was repeated for 90 s after a 120 s cooldown and still
plateaued at 2900/3100, so this is a curve difference, not just a slower ramp.
`performance` was still climbing at the 60 s mark (2500 → 3600 rpm), so a longer
run reads higher still.

One caveat worth stating: both modes sat at a 94 °C package during the load, so
neither cooled better in these runs — the higher fan did not buy headroom on a
15 W-class chassis. If your goal is a cooler machine under sustained load rather
than a louder fan, capping package power does more than any fan mode.
