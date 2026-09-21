# SUSANNA security scan — background-process assessment

Date: 2026-09-20 ~18:00 UTC. Method: **strictly read-only** over SSH
(`tasklist /v`, `tasklist` service view via PID correlation, `netstat -ano`,
`nvidia-smi`, `schtasks /query /fo LIST`). Nothing was started, stopped, or modified.
Scope/limits: point-in-time userland survey only — no memory forensics, no AV scan was
triggered, encrypted payloads can't be ruled out from names/traffic shape alone.

## Verdict: CLEAN — no malware or miner indicators

No unknown processes, no compute-only GPU workload, no strange listeners, no rogue
persistence. The box looks exactly like what it is: a lived-in gaming PC
(~9.5 days uptime) running Steam, Epic, EA Desktop, Wargaming, Discord, and Roblox,
with ASUS/ROG + Gigabyte + Logitech vendor software on top.

## Processes (~260 total, all attributed)

- **System/services:** normal Windows set (svchost groups, lsass, dwm, audiodg,
  SearchIndexer, spooler). Defender stack active: `MsMpEng.exe`, `NisSrv.exe`,
  `MpDefenderCoreService.exe`, `SecurityHealth*`.
- **Vendor (signed, expected on this board):** ASUS Armoury Crate family
  (`ArmouryCrate.Service`, `asus_framework` ×7, `atkexComSvc`, `LightingService`,
  `AsusFanControlService`, `ROGLiveService`), Gigabyte `GCC.exe`, Logitech G Hub
  (`lghub_agent`, `lghub_system_tray`), NVIDIA containers/overlay/app.
- **Gaming:** `steam.exe` + helpers, `EpicGamesLauncher`, `EADesktop` + EA services,
  Wargaming Game Center (`wgc.exe`, `wgc_renderer_host`), `Discord.exe` ×6,
  `RobloxPlayerBeta.exe`, EasyAntiCheat (`EACefSubProcess`, `EABackgroundService`),
  `EOSOverlayRenderer` (Epic overlay), `XboxPcAppFT`, EA background services.
- **Ours:** `sshd.exe` (SYSTEM listener + per-connection instance as `bota4go`),
  plus one `cmd.exe`/`conhost.exe` pair owned by `Susanna\bota4go` — that's this very
  scan's SSH exec channel, not a finding. `Taskmgr.exe` is open in the `sorok`
  console session (someone was looking at Task Manager, or left it open).
- **Not present:** no `xmrig`/`nicehash`/`claymore`/`nbminer`/any miner names, no
  AnyDesk/TeamViewer/RustDesk/VNC, no `mshta`/`wscript`/`cscript`/`certutil`/`bitsadmin`,
  no PowerShell processes at all, no random-name exes out of Temp/AppData.

## Resource anomalies reviewed (all benign)

- `wgc_renderer_host.exe` 12:49 CPU-hours and `AsusFanControlService.exe` 1:26 — heavy
  but vendor-signed bloatware/renderers, not miners (near-zero GPU compute, see below).
- `EACefSubProcess.exe` 3:18 CPU-hours — EA anti-cheat Chromium, expected on this box.
- `audiodg.exe` 1:17 — typical with audio enhancements on.

## Network (`netstat -ano`)

- Inbound listeners: only expected ones — 22 (sshd, our entry), 135/445 (Windows),
  5040/7680 (Delivery Optimization), 27036 (Steam); everything else is loopback IPC
  (ASUS 13030–13032/50100, Armoury sockets, EA 3215–3217/63588+, LG Hub 9010/45654).
  No unknown 0.0.0.0 listener.
- Outbound: overwhelmingly 443 to Microsoft/AWS/Cloudflare/CDN endpoints (updates,
  telemetry, game backends). Notable non-443, all attributed: Steam `103.10.125.22:27018`
  (game traffic), EA `98.85.38.9:8095` + `100.50.20.250:9000` (that 100.x is CGNAT space
  — odd-looking but tied to the EA Desktop process, 443 sibling connections to EA).
  No stratum/mining-pool ports (3333/4444/14444-style), no long-lived odd-port sessions.

## GPU (`nvidia-smi`)

Idle-gaming profile: 43 °C, 23 W / 180 W, 8% util, 2392 MiB used. All 24 GPU processes
are type C+G (graphics) — game launchers, overlays, dwm, browsers. **Zero compute-only
(C) processes**: a GPU miner would show as type C with near-100% util and high wattage.
Absent.

## Persistence (`schtasks`)

Only Microsoft/Windows, ASUS, Edge/Google updater, NVIDIA App SelfUpdate, OneDrive, and
Xbox/Spotlight (`GCC`, `XblGameSaveTask`, `SoftLanding*`) tasks. No third-party or
obscure auto-start task. (Our `OllamaServe` was never created — Phase 3 didn't run.)

## Recommendations (optional hardening, not findings)

1. Uninstall or disable what isn't used — Armoury Crate + GCC + Logitech + NVIDIA App
   all run background services; less surface, less CPU (that fan service).
2. The Java updater (`jusched`/`jucheck`) runs for a JRE of unknown need; remove Java
   if nothing uses it.
3. Keep Defender signatures on (active now) and run a full scan occasionally —
   `Start-MpScan -ScanType FullScan` from an Admin PowerShell when the box is idle.

## Re-scan (read-only, copy-paste from the Pi)

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "tasklist /v"
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "netstat -ano & nvidia-smi & schtasks /query /fo LIST"
```

(Remote shell rejects `|` in exec commands — use `&` chaining, never pipes.)
