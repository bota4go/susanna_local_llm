# Current state — local LLM on SUSANNA

Last update: 2026-09-22 ~18:40 UTC (ACTIVE — 4-model library + prompt examples)

## Where things stand

- **Phase 1 DONE (2026-09-20):** `ollama.exe` at `C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe` (v0.34.2), `qwen2.5:14b 7cdf5a0187d5 9.0 GB` on disk (`C:\Users\bota4go\.ollama\models\blobs` sha256-2049... 8,988,110,688 + 4 small). Verified via `curl.exe http://127.0.0.1:11434/api/tags`.
- **Server start FIXED:** tray `ollama app.exe` cannot init headless (`Unable to init instance`). **Detached** `Invoke-CimMethod Win32_Process Create '...ollama.exe serve'` survives SSH disconnect (PID 31440 on 2026-09-22, `LISTEN 127.0.0.1:11434`, `plan Start-Process` dies on channel close, `wmic` gone on Win11 24H2). `ollama ps` without serve still hangs — use `tasklist`/`netstat` to probe.
- **Phase 2 DONE (2026-09-22):** tunnel `setsid -f ssh -N -L 11434:127.0.0.1:11434` PID 26925 (and earlier 20327) `LISTEN 127.0.0.1:11434 + [::1]:11434`, both APIs via tunnel return `Ok`:
  - `POST /api/chat` qwen2.5:14b stream:false `{"content":"Ok"}` — first load 50.4s (load 28.9s for 9 GB), second 177 ms (cached 34/35), long 200-word essay 296 tok / 7.47s = **39.5 tok/s @ 97% GPU**.
  - `POST /v1/chat/completions` same `Ok`, `usage cached_tokens 34`.
  - `ollama ps` = `qwen2.5:14b 9.5 GB 100% GPU 4096 4 minutes` and VRAM `12071/3980` (up from baseline `1682/14369`). No `500` transient on this run.
  - Docs shipped: `how_to_use.md` (now §6 prompt examples for all 4 models), `how_to_deploy.md`, `options.md` (Pi5 `options.md` style), `roon/design.md` (Pi5 4 GB fix).
- **SUSANNA went offline mid-test:** `192.168.1.112` `No route to host` / `Timeout` / `nmap` missing `.112` (WiFi sleep), tunnel died (`bind Address already in use` + stale PID), came back via `ssh` (ping still firewalled), re-established tunnel 26925 and detached serve survived.
- **Current LIVE state (2026-09-22 18:33):** `ollama.exe 31440 Services 53 MB` `LISTEN 11434`, tunnel `26925` `LISTEN 127.0.0.1:11434`, **4-model library DONE:**
  - `ollama ls`: `qwen2.5:14b 7cdf5a0187d5 9.0 GB` (ctx 32768), `llama3.1:8b 46e0c10c039e 4.9 GB` (ctx 131072), `mistral:7b 6577803aa9a0 4.4 GB` (ctx 32768), `phi4:latest ac896e5b8b34 9.1 GB` (ctx 16384, phi3 family 14.7B Q4_K_M) — all verified via `curl /api/tags`.
  - Tests @4096 via tunnel: `llama3.1:8b` → `Ok.` 22.9s cold (load 15.5s, 6559/9492 solo, 5.3 GB 100% GPU) + OpenAI cached 15; `mistral:7b` → ` Ok.` 2.89s (load 2.78s, 5.0 GB 100% GPU, 11415/4636 together with llama = co-resident 10.3 GB); `phi4` → `Ok.` 12.4s (load 6.16s, 9.7 GB 100% GPU, 11405/4646) — all `100% GPU`.
  - After `ollama stop phi4`: `ps` empty, VRAM `1988 used / 14063 free` (baseline `1682/14369`), disk `~287 GB free` still. Engines: `llama.cpp GGUF` inside `ollama.exe` → `llama-server.exe` type C; other engines = vLLM/TensorRT-LLM/exllama/MLC-LLM (see options).

## Synthetic hardware room (2026-09-22 `num_ctx` sweep, same model)

Baseline idle (no model): **2900 used / 13151 free** (earlier minimal 1682/14369), RAM **6205 MB free**.

| ctx | ps SIZE | VRAM used/free | RAM free | GPU | load |
|---|---|---|---|---|---|
| 4096 | 9.5 GB | 12071 / 3980 | 5463 | 100% GPU | 7.4s |
| 8192 | 10 GB | 12963 / 3088 | 5185 | 100% GPU | 8.3s |
| 16384 | 11 GB | 14387 / 1664 | 5520 | 100% GPU | 8.5s |
| 32768 | 15 GB | 15686 / 365 | 3690 | 7%/93% CPU/GPU spill | 10.3s |

Headroom: 4096 comfortable (4 GB), 16384 tight (1.6 GB), 32768 OOM edge — 14B is max that stays 100% GPU with desktop open; 22–24B only with GPU apps closed; 32B 20 GB needs partial CPU.

## Next steps — multi-model library (DONE 2026-09-22 18:33)

**Goal met:** `qwen2.5:14b` default + `llama3.1:8b` + `mistral:7b` + `phi4` — all `Q4_K_M` `100% GPU @4096` on 16 GB VRAM, switch via `model` param (small models co-reside; 14B evicts in `~6s`).

| Family | Pull | Disk | Measured @4096 | Status |
|---|---|---|---|---|
| Llama | `llama3.1:8b` 4.9 GB | 4.9 | **5.3 GB 100% GPU** 6559/9492; 22.9s cold / 69 ms eval | **DONE** |
| Mistral | `mistral:7b` 4.4 GB | 4.4 | **5.0 GB 100% GPU** 11415/4636 together with llama (10.3 GB co-resident) | **DONE** |
| Phi | `phi4:latest` 9.1 GB 14.7B | 9.1 | **9.7 GB 100% GPU** 11405/4646; 12.4s cold (6.16s load) | **DONE** |

Total disk `~26.4 GB` (9+4.9+4.4+9.1) — 287 GB free. `phi4` stopped → `1988/14063` baseline; `how_to_use.md` verified: native `/api/chat` + OpenAI `/v1` + Python `OpenAI(base_url=...11434/v1)` for all 4; §6 now has haiku (qwen), Fib (llama), summary (mistral), step-by-step math & Estonia (phi4) plus multi-turn/streaming. `how_to_use.md` 170→~240 lines.
**Next:** update `options.md` table with `phi4 9.7 GB / 11405` row (llama/mistral already added), then optional long-essay tok/s sweep for phi4 like Qwen's 39.5 tok/s.

**Still pending Phase 3:** `OllamaServe` scheduled task (`schtasks /create /tn OllamaServe /tr "\"...ollama.exe\" serve" /sc onstart /ru Susanna\bota4go /disable` + `/run`/`/end`) — not yet created; current serve `31440` manual `Invoke-CimMethod`. Do when reboot-persistence needed.

## Notes

- SSH key-only `bota4go@192.168.1.112` healthy; `Invoke-CimMethod` is the only persistent headless start; `Start-Process` dies, `wmic` removed.
- Remote `cmd` rejects `|` in exec — use `&` or `powershell -Command "... | ..."`.
- `OLLAMA_KEEP_ALIVE` is default 5m (checked empty on Machine) — auto-unloads weights; `ollama stop` for instant games-back.
- WiFi sleep kills SSH/tunnel — if `No route` / `Timeout`, wake SUSANNA and re-run `Invoke-CimMethod` + `setsid -f ssh -N -L ...`.
