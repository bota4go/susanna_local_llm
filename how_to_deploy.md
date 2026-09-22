# How to deploy — SUSANNA local LLM (RTX 5060 Ti 16GB, i5-12400, Windows 10 Home)

Deploy/ops twin of `design.md` — exact commands that were proven 2026-09-20 → 2026-09-22 over `bota4go@192.168.1.112` (key `~/.ssh/id_ed25519`). Ziel: `qwen2.5:14b` servable via `http://127.0.0.1:11434` from the Pi, nothing LLM-related runs after boot unless asked.

`how_to_use.md` = daily use (tunnel + curl + Python). This file = first-time deploy + on-demand infra (Phase 1–3).

---

## 0. TL;DR checklist

- [ ] **Phase 1:** Ollama installed, `where ollama` → `C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe`, `ollama ls` shows `qwen2.5:14b 9.0 GB`
- [ ] **Phase 2:** tunnel + `POST /api/chat` and `POST /v1/chat/completions` both `Ok` + `ollama ps` = `9.5 GB 100% GPU 4096`
- [ ] **Phase 3:** `OllamaServe` task disabled-by-default, `schtasks /run` up in <60s, `ollama stop` + `schtasks /end` frees VRAM `12071→1682` (or 5m auto-unload)
- [ ] **Hardware room:** `nvidia-smi 14369 free` idle, `39.5 tok/s @ 97% GPU` at 14B/4096 (see `options.md`)

All steps use key-only SSH from the Pi; no GUI/RDP.

---

## 1. Prerequisites

```bash
ssh -i ~/.ssh/id_ed25519 -o PasswordAuthentication=no bota4go@192.168.1.112 "whoami & ver & nvidia-smi --query-gpu=name,memory.total,memory.free,driver_version --format=csv & Get-CimInstance Win32_OperatingSystem | Select TotalVisibleMemorySize,FreePhysicalMemory | Format-List"
# expect: susanna\bota4go, Windows 10.0.26200, RTX 5060 Ti 16311 MiB, ~13 GB free, 16 GB RAM, driver 595.79
```

If `No route to host` / WiFi sleep — wake SUSANNA (power, check `192.168.1.112` still leased; `nmap -sn 192.168.1.0/24` should list it).

---

## 2. Phase 1 — install + first model (from the Pi)

### 1a. Install Ollama for Windows (official installer, no `| iex`)

Windows sshd rejects `|` in exec — use the pipe-free two-liner from `current_state.md`:

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "powershell -NoProfile -Command \"irm https://ollama.com/install.ps1 -OutFile \$env:TEMP\ollama-install.ps1\""
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "powershell -NoProfile -ExecutionPolicy Bypass -File \$env:TEMP\ollama-install.ps1"
# fallback if sshd ever blocks it: run the same 2 lines once in a local PowerShell on SUSANNA, then continue headless
```

### 1b. Confirm binary path (record for Phase 3 `schtasks /tr`)

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "where ollama"
# → C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe   (26 MB)
```

### 1c. Pull default model (9 GB download on SUSANNA, ~2 min on 300 Mb/s)

```bash
# start detached serve first if 1a didn't auto-start (headless SSH cannot use tray app):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "powershell -NoProfile -Command \"Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe serve'}\""
sleep 3; ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "curl.exe -s http://127.0.0.1:11434/api/tags | findstr qwen || echo 'serve not ready'"

# then pull (or ollama pull without serve — it will auto-serve detached if the CimMethod serve is running):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe pull qwen2.5:14b"
# watch: schtasks /query /tr not needed — just wait for blob finish; C: free 304→287 GB = 9 GB landed
```

### 1d. Inventory

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "dir C:\Users\bota4go\.ollama\models\blobs & dir C:\Users\bota4go\.ollama\models\manifests\registry.ollama.ai\library\qwen2.5 & powershell -NoProfile -Command \"Get-CimInstance Win32_OperatingSystem | Select FreePhysicalMemory; Get-PSDrive C | Select Free | Format-List\" & nvidia-smi --query-gpu=memory.used,memory.free --format=csv"
# expect: sha256-2049... 8,988,110,688 + 4 small blobs, library\qwen2.5 dir, ~14369 free VRAM idle (after ollama stop)
# on-server check (through running serve):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "curl.exe -s http://127.0.0.1:11434/api/tags"
```

**Phase 1 done gate:** `curl /api/tags` lists `qwen2.5:14b` and `Get-PSDrive C` + `nvidia-smi` baselines recorded.

---

## 3. Phase 2 — API smoke test through SSH tunnel (no LAN exposure)

### serve must be detached (see 1c)

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "tasklist | findstr /i ollama & netstat -ano | findstr :11434"
# if empty: re-run the Invoke-CimMethod line above
```

### tunnel (terminal A, stays running)

```bash
# foreground (shows errors):
ssh -N -L 11434:127.0.0.1:11434 -i ~/.ssh/id_ed25519 -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 bota4go@192.168.1.112

# or detached:
setsid -f ssh -N -L 11434:127.0.0.1:11434 -i ~/.ssh/id_ed25519 -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 bota4go@192.168.1.112 </dev/null >>/tmp/ssh_tunnel.log 2>&1
ss -tlnp | grep 11434
```

### curls (terminal B, through the tunnel)

```bash
# first call loads 9 GB to VRAM — ~30s, don't abort; second call is 177 ms (cached)
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","stream":false,"messages":[{"role":"user","content":"Reply with the word ok."}]}' | jq .
curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","messages":[{"role":"user","content":"Reply with the word ok."}]}' | jq .
curl -s http://127.0.0.1:11434/api/tags | jq .
```

### GPU proof (on SUSANNA)

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe ps & nvidia-smi --query-gpu=memory.used,memory.free,utilization.gpu --format=csv"
# expect after first chat: qwen2.5:14b  9.5 GB  100% GPU  4096  4 minutes from now ; 12071/3980 MiB ; 39.5 tok/s for 200-word essay
```

**Phase 2 done gate:** both curls `Ok` + `ollama ps` = `100% GPU`. If `CPU` instead, close Chrome/launchers and retry; if `Connection refused`, `serve` died (re-run detached `Invoke-CimMethod`).

---

## 4. Phase 3 — on-demand start/stop (games win by default)

Deploy once; after that SUSANNA boots with **no** Ollama.

### 3a. One-time task + env check

```bash
# create disabled on-demand task from the Pi-measured path (runs as bota4go so it sees C:\Users\bota4go\.ollama)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "schtasks /create /tn OllamaServe /tr \"'C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe' serve\" /sc onstart /ru Susanna\bota4go /f"
# Note: /ru without /rp prompts for password via schtasks UI — if it fails, run the same line once in an Admin PowerShell on SUSANNA and answer the prompt, or use:
# schtasks /create /tn OllamaServe /tr "..." /sc onstart /ru Susanna\bota4go /rp <PASSWORD> /f   (not logged in repo)

ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "schtasks /change /tn OllamaServe /disable"

# keep-alive must be default 5m (empty = auto-unload):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "powershell -NoProfile -Command \"[Environment]::GetEnvironmentVariable('OLLAMA_KEEP_ALIVE','Machine')\""
# if it prints a value: powershell ... "[Environment]::SetEnvironmentVariable('OLLAMA_KEEP_ALIVE',\$null,'Machine')"

# remove any manual detached serve from phases 1–2 before testing the task path:
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "tasklist | findstr ollama & taskkill /F /IM ollama.exe 2>nul & timeout /t 2 >nul & tasklist | findstr ollama || echo 'clean'"
```

### 3b. Verify the task path

```bash
# cold start via task (should be <60s to first Ok):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "schtasks /run /tn OllamaServe & timeout /t 5 >nul & netstat -ano | findstr :11434 & tasklist | findstr ollama"
curl -s http://127.0.0.1:11434/api/tags | jq .   # through tunnel from 3a? re-open tunnel if you killed it
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","stream":false,"messages":[{"role":"user","content":"ok"}]}' | jq .

# instant VRAM reclaim before gaming (no 5m wait):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe stop qwen2.5:14b & schtasks /end /tn OllamaServe & nvidia-smi --query-gpu=memory.free --format=csv"
# expect: memory.free ≈ reboot baseline (14369 MiB), ps empty

# reboot check (when you can): after reboot, without /run, ollama ps must be empty and mem free ≈ 14 GB
```

**Phase 3 done gate:** after reboot `ps` empty → `schtasks /run` → Phase 2 curls pass → `stop + /end` → free `14369`. Rollback: `schtasks /delete /tn OllamaServe /f` (+ `taskkill /F /IM ollama.exe`).

---

## 5. Phase 4 — daily driver

Point any OpenAI client at `http://127.0.0.1:11434/v1` (`api_key="ollama"`). Per-call ctx via native API: `"options":{"num_ctx":8192}` (see `options.md` — 4096 = 3980 free, 8192 = 3088, 16384 = 1664, **32768 = 365 + 7% CPU spill**).

---

## 6. Validation matrix

| Phase | Command | Expect |
|---|---|---|
| 1 | `curl.exe -s http://127.0.0.1:11434/api/tags` on SUSANNA | `qwen2.5:14b` 8.9 GB |
| 1 | `nvidia-smi` idle after `ollama stop` | `~1682 used / 14369 free` |
| 2 | `curl http://127.0.0.1:11434/api/chat` via tunnel | `Ok`, `load ~8s` cold, `177 ms` warm |
| 2 | `curl .../v1/chat/completions` via tunnel | `chat.completion` `Ok` |
| 2 | `ollama ps` | `9.5 GB 100% GPU 4096` |
| 3 | reboot → `ollama ps` | empty, `~14369 free` |
| 3 | `schtasks /run` → curls | same as Phase 2, <60s |
| 3 | `ollama stop + schtasks /end` | `ps empty`, `free 14369` |

---

## 7. Troubleshooting

- `wmic` not found → Win11 24H2 removed it, use `Invoke-CimMethod` as above.
- `Failed to start: Unable to init instance` / `timed out waiting for server` → tray app tried to start headless — kill it and use detached `serve`.
- `bind [127.0.0.1]:11434: Address already in use` → old tunnel holds it: `pkill -f "ssh -N -L 11434"`.
- `Timeout, server 192.168.1.112 not responding` → SUSANNA WiFi sleep/off — wake it, `nmap -sn 192.168.1.0/24` to find it, `ssh ... "whoami"` to confirm.
- `exec request failed on channel 0` → you used `|` in SSH exec — use `&` or PowerShell quoting.
- `7%/93% CPU/GPU` at `ps` → VRAM OOM (e.g., `num_ctx 32768` → `15 GB`); `ollama stop` + smaller `num_ctx` or close GPU apps.

---

## 8. Files & sources

- SUSANNA paths: `C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe`, `C:\Users\bota4go\.ollama\models\blobs`, `C:\Users\bota4go\AppData\Local\Ollama\server.log`
- `susanna_hardware.md` / `options.md` for headroom (VRAM per `ctx`, `39.5 tok/s @ 97% GPU`)
- Sources: `design.md:155` — ollama README/FAQ/API/gpu, qwen2.5 library, llama.cpp server.
