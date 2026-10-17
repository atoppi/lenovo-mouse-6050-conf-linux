# lenovo-mouse-6050-conf-linux

Configure the **Lenovo Wireless Multi-Mode Pro Plus Mouse 6050** on Linux over Bluetooth LE, with no Windows app needed.

It covers the same settings as Lenovo's Windows tool (Lenovo Accessories and Display Manager): DPI stages, wheel acceleration, left/right scroll speed, swapping the primary buttons, and restoring the defaults.

## Requirements

- Linux with BlueZ and systemd (the script uses `busctl` to talk to BlueZ)
- Python 3. No extra packages are needed.
- The mouse paired and connected **over Bluetooth**. The 2.4 GHz USB receiver is not supported.

No root is needed: the script talks to BlueZ over D-Bus as your user.

## Install

```sh
curl -o ~/bin/lenovo-mouse https://raw.githubusercontent.com/atoppi/lenovo-mouse-6050-conf-linux/main/lenovo-mouse
chmod +x ~/bin/lenovo-mouse
```

Or clone the repo and run `./lenovo-mouse`.

## Usage

```sh
lenovo-mouse status                          # battery, DPI, wheel, buttons, timers (read-only)
lenovo-mouse dpi 800                         # single DPI level
lenovo-mouse dpi 1600 2400 4000 --active 2   # 1-3 stages, 100-4000 in steps of 100
lenovo-mouse wheel-accel off|low|medium|high
lenovo-mouse hscroll-speed low|medium|high   # tilt-wheel (left/right) scroll speed
lenovo-mouse swap-buttons on|off             # left-handed
lenovo-mouse restore                         # write back factory defaults and verify
lenovo-mouse factory-reset                   # the app's own "Restore Default" command
lenovo-mouse help [command]
```

Add `-v` to any command to print the raw frames, or `--mac XX:XX:...` if more than one mouse is connected.

Factory defaults: DPI 1600 / 2400 / 4000 with DPI 2 active, wheel acceleration medium, left/right scroll speed medium, standard button layout.

## Protocol

The mouse has a vendor GATT service `c1d02500-2d1f-400a-95d2-6a2f7bca0c25`. Requests are written to characteristic `…02501`, and replies arrive on `…02502` (notify/read).

```
request: 00 <payload len> <group> <cmd> 00 <seq> <payload…>
reply:   02 <payload len> <group> <cmd> 00 <seq> <payload…>
```

A "get" is the "set" command with the high bit set (`cmd | 0x80`).

| Function | Set | Get | Payload |
|---|---|---|---|
| DPI stages | <code>04&nbsp;03</code> | <code>04&nbsp;83</code> | `00`, count (1-3), active stage (0-based), then X and Y (big-endian u16) per stage |
| Wheel acceleration | <code>04&nbsp;0a</code> | <code>04&nbsp;8a</code> | enabled (0/1), level (0-2). Off = <code>00&nbsp;01</code>, low = <code>01&nbsp;00</code>, medium = <code>01&nbsp;01</code>, high = <code>01&nbsp;02</code> |
| Button list | – | <code>02&nbsp;81</code> | `00`, count, <code>00&nbsp;00</code>, then the button ids |
| Button actions | <code>02&nbsp;04</code> | <code>02&nbsp;84</code> | 7 button ids (zero-padded), then 7 bytes per id: `00` type param x speed <code>00&nbsp;00</code> |
| Restore defaults | <code>00&nbsp;09</code> | – | `02` |
| Battery | – | <code>07&nbsp;82</code> | `00`, percent, percent |
| Dim / sleep / off timers | – | <code>07&nbsp;83</code> | read only |

Button action types: `01 n` = mouse button n, `04 ±1` = vertical scroll, `07 ±1 <speed>` = horizontal scroll (low/medium/high = 50/100/200), `0f` = cycle DPI.

The protocol was worked out from Bluetooth captures of the Windows app (BTVS + Wireshark). The script sends only the set commands the Windows app sends, and it refuses firmware-update commands (group `09`).

## Tested firmware

Only firmware **V1.0.03** has been tested. `lenovo-mouse status` shows your firmware version on its first line. If yours is different, please [open an issue](https://github.com/atoppi/lenovo-mouse-6050-conf-linux/issues) with the output of `lenovo-mouse -v status`, whether it works or not.

## Disclaimer

Not affiliated with or endorsed by Lenovo. Use at your own risk.
