# LLM Options for SUSANNA (RTX 5060 Ti 16GB + i5-12400, 16GB RAM)

## Hardware Constraints

- **GPU:** NVIDIA GeForce RTX 5060 Ti **16311 MiB VRAM (16 GB)**, Blackwell `sm_120`, driver 595.79, CUDA 13.2, WDDM (display-attached)
- **VRAM free (observed 2026-09-22):** ~**13–14 GB** with desktop idle (`2900 MiB` used baseline, `1682 MiB` minimal after `ollama stop`), ~**10.8–12 GB** with Chrome/Discord/Wallpaper/Steam open. Baseline matters — close GPU apps to gain ~1.5 GB.
- **CPU:** Intel i5-12400 — 6 cores / 12 threads @ 2.5 GHz (no E-cores), ~49% idle with bloatware (Armoury/GCC/Logitech/NVIDIA App). Good for partial CPU offload and CPU-only fallback.
- **RAM:** 16 GB DDR5 (16487856 KB visible) — **6.2 GB free** idle, **5.3 GB** with `qwen2.5:14b` @ 4096 resident, **3.6 GB** at 32768 spill. Large CPU offload + big `num_ctx` will pressure RAM.
- **Disk:** T-FORCE 1 TB — **287 GB free** (used 735 GB). Room for a large model library (30× 9 GB models).
- **OS:** Windows 10 Home 64-bit build 26200, **headless over SSH** (`bota4go@192.168.1.112` key-only, `cmd` shell, `|` must be quoted). No RDP/GUI needed.
- **Binding constraint = VRAM.** System RAM is 16 GB, so don't over-spill.

---

## Recommended Runtimes

### 1. Ollama for Windows (native CUDA) — **Best overall choice**

- One exe (`C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe` v0.34.2) = `ollama serve` + `pull/ls/ps/stop` + REST + OpenAI-compat `/v1`
- `Q4_K_M` quantizations auto-selected, recent Blackwell kernels (`sm_120`) included — **driver 550+ required, we have 595.79**
- `OLLAMA_KEEP_ALIVE=5m` default auto-unloads weights; `ollama ps` `PROCESSOR` column is GPU proof
- **Install:** `powershell -Command "irm https://ollama.com/install.ps1 | iex"` over SSH (or `OllamaSetup.exe` locally)

### 2. llama.cpp `llama-server` (CUDA build) — **Manual tuning fallback**

- Use when Ollama can't fit a wanted model with full offload — control `-c/--ctx-size` and `--n-gpu-layers` / `--port`
- OpenAI-compatible routes verified in docs, good for 30B+ partial offload experiments
- **Install:** only if Ollama fails on a target model; build CUDA 13.x with `sm_120` support

### 3. LM Studio — **GUI alternative**

- Nice per-model GPU slider if you want a desktop UI on SUSANNA itself
- Rejected for default (no-UI constraint, SSH-only), but fine for ad-hoc tests on the console session

**Not needed:** WSL2 (extra VRAM overhead), MLC-LLM (ARM-focused, not for Blackwell).

---

## Recommended Models (fitting in 16GB VRAM, Q4_K_M)

Measured 2026-09-22 on `qwen2.5:14b` (14.8B) — load ~7–10s cold, **39.5 tok/s @ 97% GPU** for 296-token 200-word essay at 4096. Others are library/Jama estimates scaled to same scaling law.

| Model | Size (Q4_K_M) | VRAM @ 4096 | VRAM @ 8192 | VRAM @ 16384 | Fit | Notes |
|---|---|---|---|---|---|---|
| **Qwen2.5 7B** | ~4.7 GB | ~6 GB | ~6.5 GB | ~7.5 GB | ✅ 100% GPU + 6 GB headroom | Fastest 7B, good reasoning |
| **Qwen2.5 14B** *(default)* | **9.0 GB** | **9.5 GB / 3980 free** | **10 GB / 3088** | **11 GB / 1664** | **✅ 100% GPU** | **Tested sweet spot — 39 tok/s** |
| Qwen2.5 14B @ 32768 | 9.0 GB blob | **15 GB / 365 free** | — | — | ⚠️ **7%/93% CPU/GPU spill** | Needs RAM, `Free 3690 MB`; avoid unless you need 24k tokens |
| **Llama 3.1 8B** | ~4.9 GB | ~6 GB | ~6.8 GB | ~8 GB | ✅ 100% GPU | Good general, slightly behind Qwen 14B |
| **Gemma 2 9B** | ~5.8 GB | ~7 GB | ~7.5 GB | ~8.5 GB | ✅ 100% GPU | Fast, good for chat |
| **DeepSeek-R1 14B** (distill) | ~9 GB | ~9.5 GB | ~10 GB | ~11 GB | ✅ 100% GPU | Strong reasoning, similar to Qwen 14B |
| **Mistral Small 22B** | ~13.5 GB | ~14 GB | ~14.5 GB | ~15.5 GB | ⚠️ **Tight** — close GPU apps | Only with desktop closed; no 16K without spill |
| **Qwen2.5 32B** | **20 GB** | ~20 GB | ~21 GB | — | ❌ **No full GPU** — partial CPU | Needs `~30% CPU` offload, RAM 8+ GB; slow |
| **Llama 3.1 70B** | ~40 GB | ~40 GB | — | — | ❌ No | Out — needs 2× VRAM |

**Quantization:** `Q4_K_M` is the sweet spot for 16 GB. `Q3_K_M` shaves ~1.5 GB but costs quality; `Q5/Q6/Q8` gains little and quickly OOMs. Stick to `Q4_K_M` unless testing.

**Recommended:** **`qwen2.5:14b` @ `num_ctx=4096` (or `8192` if you need 6k words) via Ollama.** `14B` is the largest that stays `100% GPU` with desktop open; `22–24B` only with GPU apps closed; `30B+` = partial offload.

---

## Performance Expectations (RTX 5060 Ti, headless `ollama serve`)

- **Cold load:** `7.4s` @ 4096, `8.3s` @ 8192, `8.5s` @ 16384, `10.3s` @ 32768 (9 GB → VRAM). Second call same `ctx` = **177 ms** (prompt cached 34/35 tok).
- **Decode:** **~39–45 tok/s** for 14B Q4_K_M @ 4096 100% GPU (`296 tok / 7.47s` in 200-word essay, `97% GPU`). `8192–16384` similar; `32768` spill drops to ~25 tok/s + higher RAM.
- **VRAM headroom after load:** `3980 MB` @ 4096, `3088` @ 8192, `1664` @ 16384, `365` @ 32768. Keep **>1 GB free** for stability — don't run games concurrently.
- **RAM after load:** `5463 MB` @ 4096, `5185` @ 8192, `3690` @ 32768 (from `6205 MB` idle). Close Chrome if `Free < 4 GB`.

Pi 5 comparison (same prompt family): `1B ~10–15 tok/s CPU`, `3B ~4–7 tok/s CPU` — SUSANNA is **~5–10× faster** at 14B with GPU.

---

## Tips (SUSANNA-specific)

- **Start detached:** `Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe serve'}` — `Start-Process` dies when SSH closes, `wmic` is gone on Win11 24H2.
- **Tunnel by default:** `ssh -N -L 11434:127.0.0.1:11434 ...` — Ollama stays on loopback (no LAN auth). `setsid -f ... >>/tmp/ssh_tunnel.log 2>&1` to detach.
- **Context:** `options.num_ctx` reloads the model — pick one (`4096` for chat, `8192` for docs) and keep it. Change = 8s stall.
- **Games-first:** `ollama stop qwen2.5:14b` frees VRAM `12071→1682` instantly; `OLLAMA_KEEP_ALIVE` default `5m` auto-unloads anyway. No reboot needed.
- **Scheduled task (Phase 3):** `schtasks /create /tn OllamaServe /tr "\"C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe\" serve" /sc onstart /ru Susanna\bota4go /disable` + `schtasks /run`/`/end` — on-demand, no boot cost.
- **Blackwell gotcha:** `sm_120` needs CUDA 13.x + recent Ollama/llama.cpp — we have it (595.79/13.2). Old binaries = `CPU` fallback.
- **SSH:** remote shell is `cmd` — chain with `&`, not `|`; wrap PowerShell pipes in `powershell -Command "... | ..."`; `ssh -vvv` for debug.
- **Monitoring:** `nvidia-smi --query-gpu=memory.used,memory.free --format=csv` + `Get-CimInstance Win32_OperatingSystem | Select FreePhysicalMemory` + `ollama ps` (look for `100% GPU`).
- **NVMe not needed:** SUSANNA has 1 TB NVMe already — load is VRAM PCIe, not disk.
