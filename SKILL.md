---
name: barrier-config
description: Configure and troubleshoot Barrier KVM (Windows server + macOS/Linux clients). Covers autostart, keyboard layout sync (ABNT2<->US Intl), clipboard lag, edge jitter, macOS permissions, and server debug logging. Use when setting up or fixing Barrier across machines.
---

# Barrier Config & Troubleshooting

Barrier shares keyboard/mouse over LAN (port `24800/tcp`). One **server** (physical keyboard) + N **clients**.

## Core facts
- Server **injects** events; client **interprets with its own keyboard layout**. No character remapping across machines — only modifier remapping (per-screen in `section: screens`).
- This install runs **`--disable-crypto`** on all sides. Mismatched TLS = drops. Do not re-enable crypto without aligning all sides.
- Current build does **not** support `clipboardSharingSize` (error: `unknown argument "clipboardSharingSize"`). Only the boolean `clipboardSharing` works.

## Real topology (reference)
| Machine | Role | IP | Screen |
|---------|------|----|--------|
| Windows | Server | 192.168.100.94 | `desktop` |
| Mac | Client | 192.168.100.83 | `macbook` |
| Linux | Client | 192.168.100.102 | `lab` |

Screen order L→R: `macbook` | `desktop` | `lab`. Server starts with `--name desktop`.

## Barrier.conf (Windows server)
`C:\Users\<user>\AppData\Local\Barrier\Barrier.conf`:
```ini
section: screens
	Desktop:
	macbook:
		ctrl = ctrl
		super = alt
		alt = meta
	lab:
end
section: links
	desktop:
		left = macbook
		right = lab
	macbook:
		right = desktop
	lab:
		left = desktop
end
section: options
	clipboardSharing = false
end
```
- Modifier remap is **per screen**. Never use `key: X = Y` (config error).
- Screen name of server comes from `--name`, must match `links` entry.

## Windows server autostart (Scheduled Task — preferred)
```powershell
$action = New-ScheduledTaskAction -Execute "C:\Program Files\Barrier\barriers.exe" -Argument "--name desktop --disable-crypto -c C:\Users\$env:USERNAME\AppData\Local\Barrier\Barrier.conf"
$trigger = New-ScheduledTaskTrigger -AtLogOn -User $env:USERNAME
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0
Register-ScheduledTask -TaskName "BarrierServer" -Action $action -Trigger $trigger -Settings $settings -Force
```
For pre-login start: `-AtStartup` + `-Delay "00:00:30"`. Do NOT use a `.cmd`/normal shortcut — `barriers` is a console app; closing its window kills the server.

## Mac client
- Manual: `/Applications/Barrier.app/Contents/MacOS/barrierc -f --no-tray --name macbook --disable-crypto 192.168.100.94:24800`
- launchd plist `~/Library/LaunchAgents/com.barrier.client.plist`: Label `com.barrier.client`, ProgramArguments `barrierc -f --no-tray --name macbook --disable-crypto 192.168.100.94:24800`, `RunAtLoad true`, `KeepAlive true`.
- **Permissions (manual, non-automatable)** in System Settings → Privacy & Security: **Local Network**, **Accessibility**, **Input Monitoring**. Missing Local Network → `No route to host`.

## Linux client
- systemd user `~/.config/systemd/user/barrier-client.service`: `ExecStart=/usr/bin/barrierc -f --no-tray --name lab --disable-crypto 192.168.100.94:24800`, `Restart=always`, `WantedBy=default.target`. Then `systemctl --user daemon-reload && systemctl --user enable --now barrier-client.service`.
- **Auto-layout (ABNT2 ↔ US Intl)** when server is US International and Linux is ABNT2: loop script `~/bin/barrier-layout.sh` that `setxkbmap -model abnt2 -layout us -variant intl` while `pgrep` finds barrier/barrierc/barriers, else `setxkbmap -model abnt2 -layout br -option compose:menu`. Exports `DISPLAY=:0` and `XAUTHORITY=$HOME/.Xauthority`. Run via `~/.config/systemd/user/barrier-layout.service` (`ExecStart=%h/bin/barrier-layout.sh`). On **Wayland** replace `setxkbmap` with `gsettings set org.gnome.desktop.input-sources sources "[('xkb','us+intl')]"`.

## Symptom → Cause → Fix
| Symptom | Cause | Fix |
|---------|-------|-----|
| `No route to host` (Mac) | missing Local Network perm | grant on Mac (manual) |
| input dead on Mac | missing Accessibility/Input Monitoring | grant on Mac (manual) |
| `server already has a connected client with our name` | orphan client | `pkill -f barrierc` (Linux) / `killall barrierc` (Mac) |
| closed window kills server | started via .cmd | Scheduled Task / hidden VBScript |
| short freezes on screen switch | clipboard ≥6 MB | `clipboardSharing = false` |
| cursor "vibrates" at edge / boundary lag | edge jitter | offset Mac screen in Configure Server; or Lock to screen (Scroll Lock) |
| `unknown argument "clipboardSharingSize"` | build lacks size limit | use `clipboardSharing = false` |
| wrong accents/ç on Linux | layout ≠ server | auto-layout script |
| intermittent Mac drops | TLS/crypto mismatch | `--disable-crypto` both sides |
| diagnose server | need log | start with `-d DEBUG -l logfile` |

## Server debug logging (diagnosis)
```powershell
Stop-Process -Name barriers -Force
Start-Process -FilePath "C:\Program Files\Barrier\barriers.exe" -ArgumentList "--name","desktop","--disable-crypto","-c","$env:USERPROFILE\AppData\Local\Barrier\Barrier.conf","-d","DEBUG","-l","$env:USERPROFILE\barrier_server.log" -WindowStyle Hidden
```
Read `barrier_server.log`; grep for the client screen, `dead`, `slow`, `clipboard ... size=`, `switch from/to`. Revert to normal instance (no `-d/-l`) after.

## Gotchas
- `barrier` (GUI) vs `barriers` (server daemon) vs `barrierc` (client). Running `barriers.exe` directly is more predictable for autostart.
- Mac client has slightly higher input latency by nature; use a notarized build.
- `setxkbmap` needs the graphical session (DISPLAY/XAUTHORITY); fails over bare SSH without those vars.
