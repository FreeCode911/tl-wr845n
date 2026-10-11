# Iqbal Mahmud Smart Router Firmware: TL-WR845N v4

Custom Padavan firmware for the **TP-Link TL-WR845N v4** (MT7628NN, 4 MB flash, 32 MB RAM), by **[Iqbal Mahmud](https://iqbalmahmud.com)**.

- **Smart WAN Watchdog**: the router fixes its own internet drops.
- **Speed Control**: chosen devices get full speed, everyone else gets the limit you set.
- **Iqbal Mahmud branding**: logo, page title, footer, SSH banner, default Wi-Fi name.

Based on [lucktu/padavan_TW641](https://github.com/lucktu/padavan_TW641) (padavan-ng, Linux 3.4), pinned to commit `90f694d`. GPL v2.

## Version 2 changes

| | v1 | v2 |
|---|---|---|
| Smart WAN Watchdog | ✅ | ✅ |
| Hardware CPU watchdog | ✅ | ✅ |
| Speed Control (VIP / per-device / shared pool) | ❌ | ✅ |
| Traffic control (`tc`, HTB) | ❌ | ✅ |
| IPSet | ✅ | ❌ removed to make room |
| Branding | Padavan | Iqbal Mahmud |
| Default Wi-Fi name | `TL_WR845N-V4_%s` (bug) | `IqbalMahmud_WiFi` |

Upgrading from v1 keeps your settings (Wi-Fi, PPPoE, password). Take a **Settings Backup** first anyway.

## Smart WAN Watchdog

| Step | When | Action |
|---|---|---|
| 1 | Internet down < 3 min | Nothing. Short blips are ignored. |
| 2 | Down 3 min | Restart the WAN connection, up to 2 tries 90 s apart. |
| 3 | Still down | Reboot the router. |
| 4 | Still down after a reboot | Each next reboot waits longer (+5, +10, +20, max +30 min). |
| — | Stable 10 min | Backoff resets. |

Settings are nvram keys, e.g. `nvram set wd_fail_time=300 && nvram commit`:
`wd_enable`, `wd_fail_time`, `wd_wan_retries`, `wd_wan_wait`, `wd_backoff_base`, `wd_backoff_max`, `wd_stable_time`, `wd_boot_grace`.

## Speed Control

Three kinds of device:

- **VIP**: full speed, and **priority**. When a VIP device needs bandwidth it gets up to 90 % of your line first.
- **Custom**: a device with its own download/upload limit.
- **Others**: everyone else. Two modes:
  - `each`: every device gets its own limit (e.g. 5 Mbps each, however many connect).
  - `pool`: all of them **share** one limit (e.g. 10 Mbps together).

Devices are matched by **MAC address**, so rules keep working when IPs change. New devices are picked up within about 15 s.

### Commands

Run them in **Administration → Console** (or SSH). Speeds are in **Mbps** (`5`, `1.5`) or kbps with `k` (`512k`). `0` = unlimited.

```sh
speedctl list                          # connected devices, MAC, name, current rule
speedctl line 30 30                    # your real ISP plan: down / up  (do this first)
speedctl others each 5 1               # every other device: 5 Mbps down / 1 Mbps up each
speedctl others pool 10 2              # ...or all others share 10 down / 2 up
speedctl vip add aa:bb:cc:dd:ee:ff     # full speed + priority
speedctl vip del aa:bb:cc:dd:ee:ff
speedctl limit aa:bb:cc:dd:ee:ff 2 0.5 # custom: 2 Mbps down / 500 kbps up
speedctl unlimit aa:bb:cc:dd:ee:ff
speedctl on                            # enable
speedctl off                           # disable (everything back to normal)
speedctl status                        # show settings
```

Rules are saved and survive reboots. Changes apply immediately.

### First-time setup example

```sh
speedctl list                              # find your laptop/phone MAC
speedctl line 30 30                        # plan speed
speedctl vip add 11:22:33:44:55:66         # your device
speedctl others each 5 1                   # everyone else
speedctl on
```

### Notes

- Set `speedctl line` to your **real** plan speed. Too high and VIP priority gets weaker; too low and everyone is capped below what you pay for.
- `speedctl on` switches off Software Offload (SFE), which would otherwise skip the limits. If it says so, reboot once.
- Traffic between devices on your own network (e.g. file copy laptop → PC) is never limited, only internet traffic.
- Activity is logged under **Administration → System Log** with the tag `speedctl`.

## Build (GitHub Actions)

1. Put this folder in a GitHub repository, keeping `.github/workflows` and `patches`.
2. **Actions → Build Iqbal Mahmud Smart Firmware → Run workflow**.
3. After 30–60 min download the **IqbalMahmud-WR845Nv4-v2** artifact: `IqbalMahmud_WR845Nv4_v2.bin` + `SHA256SUMS.txt`.

The build fails on purpose if the image is larger than the 3.84 MB firmware partition.

## Flash

**From Padavan (v1 → v2):** **Administration → Firmware Upgrade**, choose the `.bin`, wait for the reboot. Use a cable, don't power off.

**From TP-Link stock:** System Tools → Firmware Upgrade on `192.168.0.1`. If the stock firmware rejects it, use a CH341A programmer. Always keep a full 4 MB flash backup.

## Defaults (fresh install or after reset)

- Web UI: `http://192.168.0.1`, user `admin`, password `admin`
- Wi-Fi: `IqbalMahmud_WiFi`, password `1234567890`

**Change both immediately.**

## Disclaimer

Use at your own risk. No warranty. Third-party firmware voids the TP-Link warranty and a failed flash can brick the router. For **TL-WR845N v4 only**; do not flash other models or versions.

The upstream footer notice ("Firmware distribution is prohibited / Non-Commercial Use Only") from the Padavan fork authors is kept unchanged in the web UI. Respect it when sharing builds.

## Files

```
.github/workflows/build.yml                   Build pipeline
patches/0001-iqbal-mahmud-smart-firmware.patch All changes to the Padavan source
```
