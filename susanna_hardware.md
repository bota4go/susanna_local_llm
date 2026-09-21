# SUSANNA — hardware survey for local LLM

Probed 2026-09-20 over ssh (`bota4go@192.168.1.112`, key auth). All values are observed, not specs-sheet.

## Raw specs

| Part | Observed |
|---|---|
| Host / OS | `Susanna`, Windows 10 Home 64-bit, build 10.0.26200 |
| CPU | 12th Gen Intel i5-12400 — 6 cores / 12 threads, 2.5 GHz base |
| RAM | 16 GB (2× 8 GB TeamGroup, 4000 MT/s, SMBIOS type 34 = DDR5), total 16883564544 B |
| GPU | NVIDIA GeForce RTX 5060 Ti, **16311 MiB VRAM (16 GB)**, driver 595.79, CUDA 13.2, compute cap **12.0 (Blackwell, sm_120)**, WDDM, display-attached |
| Disk | T-FORCE TM8FPP001T 1 TB (`1024203640320` B); C: ~715 GB used / ~307 GB free |
| SSH | key-only works: `ssh -i ~/.ssh/id_ed25519 -o PasswordAuthentication=no bota4go@192.168.1.112 "whoami"` → `susanna\bota4go` |

At probe time 3154 MiB VRAM was already in use (dwm, Chrome, Explorer, game/EAC processes); **12897 MiB free**.

## What this means for local LLMs

- VRAM is the binding constraint: ~13 GB free with the desktop up, ~15–16 GB if GPU apps are closed.
- Sensible full-offload targets (GGUF Q4_K_M, rough): ≤14B (~9 GB) comfortable; 24B (~14–15 GB) tight but doable with apps closed; 30B/32B (~18–20 GB) does **not** fit — needs partial CPU offload; 70B is out for GPU (needs ~40 GB).
- System RAM is 16 GB, so large CPU-offloaded layers + big contexts will pressure RAM. Close Chrome/game launchers before long runs.
- Blackwell (sm_120) is a new arch: use **recent** builds (Ollama / llama.cpp / LM Studio current releases, CUDA 13.x) — old binaries may lack sm_120 kernels. Driver 595.79 + CUDA 13.2 are already new enough.
- 307 GB free disk = room for a large model library; no concern.
- i5-12400 (6 P-cores, no E-cores) is fine for partial offload and fast for small GGUFs on CPU.

## Suggested stack

1. **Ollama for Windows (native, CUDA)** — simplest: `ollama run qwen2.5:14b` / `deepseek-r1:14b` class; 24B only with GPU apps closed.
2. **LM Studio** — if a GUI + per-model GPU-offload slider is wanted.
3. **llama.cpp (CUDA build)** — if partial offload tuning is needed (`-ngl` split for 30B+ experiments).
4. Skip WSL2 for serving unless a Linux-only tool is required — native CUDA avoids the WSL VRAM/virtualization overhead.

## Re-probe commands

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "nvidia-smi --query-gpu=name,memory.total,memory.free,driver_version,compute_cap --format=csv"
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 'powershell -NoProfile -Command "Get-CimInstance Win32_DiskDrive | Select-Object Model,MediaType,Size"'
```

Note: over ssh the remote shell is `cmd` — chain with `&`, and `|` pipes belong to cmd, so wrap PowerShell pipelines in quotes as above.
