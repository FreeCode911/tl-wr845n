# TL-WR845N v4 Smart Firmware

Custom Padavan firmware for the **TP-Link TL-WR845N v4** (MT7628NN, 4 MB flash, 32 MB RAM) with a built-in **Smart WAN Watchdog**: when the internet goes down, the router fixes itself instead of waiting for someone to pull the power cable.

Based on [lucktu/padavan_TW641](https://github.com/lucktu/padavan_TW641) (padavan-ng, Linux 3.4), pinned to commit `90f694d`.

## What it does

When the internet stops working, the watchdog escalates step by step:

| Step | When | Action |
|---|---|---|
| 1 | Internet down < 3 min | Nothing. Short blips are ignored. |
| 2 | Down 3 min | Restart the WAN connection (PPPoE/DHCP). Up to 2 tries, 90 s apart. |
| 3 | Still down after WAN restarts | Reboot the router. |
| 4 | Still down after a reboot | Next reboot waits longer: +5, +10, +20, then +30 min max. No reboot loop during an ISP outage. |
| — | Internet stable for 10 min | Backoff counter resets. |

Two more protections are switched on by default:

- **Hardware CPU watchdog** (`watchdog_cpu=1`): if the router itself freezes, the MT7628 chip reboots it automatically. A script alone can't recover from a freeze; this can.
- **Built-in Internet detection** (`di_poll_mode=1`): checks 1.1.1.1, 8.8.8.8, 9.9.9.9 and others on TCP port 53, which works even where ping is blocked.

Every action is written to the system log (tag `wan_watchdog`), visible in the web UI under **Administration → System Log**.

## Settings

Change them in **Administration → Console** (or over SSH), then they persist across reboots:

```sh
nvram set wd_fail_time=300 && nvram commit
```

| Key | Default | Meaning |
|---|---|---|
| `wd_enable` | `1` | `0` turns the watchdog off |
| `wd_fail_time` | `180` | Seconds down before acting |
| `wd_wan_retries` | `2` | WAN restarts before rebooting |
| `wd_wan_wait` | `90` | Seconds to wait after each WAN restart |
| `wd_backoff_base` | `300` | Extra wait after the first unhelpful reboot (doubles each time) |
| `wd_backoff_max` | `1800` | Cap on that extra wait |
| `wd_stable_time` | `600` | Seconds of good internet before the backoff resets |
| `wd_boot_grace` | `120` | Seconds after boot before the watchdog starts checking |
| `wd_reboots` | `0` | Counter the watchdog keeps itself; don't change |

Leave **Internet detection → "Action on lost"** set to *No action*, otherwise the built-in action and the watchdog will both react.

## Build (GitHub Actions, free)

1. Create a new GitHub repository and upload everything in this folder, keeping the `.github/workflows` and `patches` folders.
2. Open the **Actions** tab → **Build TL-WR845N v4 Smart Firmware** → **Run workflow**.
3. After about 30–60 min, download the **WR845Nv4-smart-firmware** artifact. It contains `WR845Nv4_smart_watchdog.bin` and its SHA256 checksum.

The build fails on purpose if the image is larger than the 3.84 MB firmware partition.

## Before flashing: read this

- **This can brick the router.** Third-party firmware voids the warranty and isn't supported by TP-Link.
- **Get a recovery path first.** A CH341A USB programmer with an SOIC-8 clip (about ৳600–900) lets you read a full backup of the 4 MB flash chip (GigaDevice 25Q32 / Winbond W25Q32) and write it back if anything goes wrong. Save that backup somewhere safe before doing anything else.
- **Use a cable**, never Wi-Fi, and don't power off the router during flashing. A mini UPS helps.

## Flash

**Option A: stock web interface** (try first)

1. Connect a PC to a LAN port by cable.
2. Open `192.168.0.1` → **System Tools → Firmware Upgrade**, choose `WR845Nv4_smart_watchdog.bin`, upgrade.
3. Newer stock firmware may reject third-party images. If it does, use Option B. Don't try to force it.

**Option B: CH341A programmer**

Write the image at the firmware partition offset with the programmer software, keeping the bootloader (first 64 KB) and the last three 64 KB blocks (Storage, Config, Factory/radio calibration) from your own backup. The Factory block is unique to your router; never overwrite it with someone else's.

## After flashing

- Web UI: `http://192.168.1.1` (note the new IP), user `admin`, password `admin`. **Change the password immediately.**
- Set your WAN (PPPoE username/password from your ISP), Wi-Fi name and password.
- Check the watchdog is running: **Administration → Console** → `ps | grep wan_watchdog`.
- Test it: unplug the WAN cable for 5 minutes and watch **System Log**.

## Files

```
.github/workflows/build.yml          Build pipeline
patches/0001-smart-wan-watchdog.patch  Changes to the Padavan source
```

The patch adds `/usr/bin/wan_watchdog.sh`, starts it at boot from `rc.c`, and changes the defaults in `defaults.c`.
