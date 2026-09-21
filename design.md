# Local LLM on SUSANNA — plan (headless, SSH-only, API)

## Goal

Serve LLMs on SUSANNA's RTX 5060 Ti (16 GB VRAM) and consume them from the Raspberry Pi
over HTTP API. No UI on either box; all install/ops over the existing key-only SSH
(`bota4go@192.168.1.112`); all inference over the API.

## Success Criteria

- `ollama ls` on SUSANNA shows one 14B-class model.
- From the Pi, both `POST /api/chat` and `POST /v1/chat/completions` through the SSH
  tunnel return a completion with the model resident on GPU (`ollama ps` shows `100% GPU`).
- Nothing LLM-related runs unless requested: after boot SUSANNA is 100% games
  (no Ollama process, full VRAM free); a model is servable within ~1 min of an
  on-demand start from the Pi, and VRAM frees itself minutes after last use.

## Context And Current Facts

- Client: raspberry, Linux user `bota4go`. Server: SUSANNA, `192.168.1.112`, Windows
  (build 10.0.26200), users `bota4go`/`sorok` both in Administrators.
- Hardware (probed 2026-09-20, see `susanna_hardware.md`): i5-12400 6C/12T, 16 GB DDR5,
  RTX 5060 Ti with 16311 MiB VRAM (~12.9 GB free with desktop apps up), driver 595.79,
  CUDA 13.2, compute cap 12.0; 1 TB disk with ~307 GB free.
- SSH: key-only login verified (`whoami` → `susanna\bota4go`). Remote shell is `cmd`;
  PowerShell pipelines must be quoted.
- Official Ollama docs (inspected this run, see Sources): Windows installs via
  `irm https://ollama.com/install.ps1 | iex` (or `OllamaSetup.exe`); updates are automatic
  via the taskbar. Env config: `OLLAMA_HOST`, `OLLAMA_KEEP_ALIVE`, `OLLAMA_MODELS`,
  `OLLAMA_CONTEXT_LENGTH` (default context 4096). `ollama ps` PROCESSOR column proves
  GPU vs CPU placement. RTX 5060 Ti (compute 12.0) is on the supported list (needs
  driver 550+; ours is 595.79). Library: `qwen2.5:14b` = 9.0 GB, `qwen2.5:32b` = 20 GB.

## Constraints And Non-goals

- No GUI/RDP/browser chat. No internet exposure (LAN + SSH tunnel only).
- Inference only; no training. Single-user; Ollama has no auth, so access control =
  bind address + firewall + tunnel (local requests need no key).
- Do not fill the 16 GB system RAM with CPU-offloaded giants; VRAM is the budget.

## Key Decisions

1. **Runtime: Ollama for Windows (native CUDA).** Official install + REST + OpenAI-compat
   API in one exe; `qwen2.5` sizes verified in the library. Rejected LM Studio (UI-centric,
   against the no-UI constraint) as the default.
2. **Access: SSH tunnel by default** (`-L 11434:127.0.0.1:11434`), Ollama stays on
   loopback. Rejected LAN-bind as default (no-auth port on the LAN); kept as a documented
   opt-in.
3. **Default model: `qwen2.5:14b`** (9.0 GB — fits the ~13 GB free VRAM with headroom).
   24B-class only with GPU apps closed; 32B (20 GB) does not fit full offload.
4. **Fallback: llama.cpp `llama-server`** (OpenAI-compatible routes, `-c/--ctx-size`,
   `-ngl/--n-gpu-layers`, `--port` all verified) if a model ever needs manual split
   tuning. Not installed unless Ollama fails on a wanted model.

## Recommended Approach

Install Ollama with the official installer over SSH, confirm the binary path, pull
`qwen2.5:14b`, smoke-test both APIs through a tunnel, then wire on-demand start/stop
(a demand-started Scheduled Task plus short keep-alive) so games always win by default.
Phase-gated: each phase has a check before the next starts.

## Work Plan

**Phase 1 — install + first model** (from the Pi):

```bash
# 1a. official installer on SUSANNA
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "powershell -NoProfile -Command \"irm https://ollama.com/install.ps1 | iex\""
# 1b. confirm binary path (use the returned path in all later commands)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "where ollama"
# 1c. pull default model (download happens on SUSANNA)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "ollama pull qwen2.5:14b"
# 1d. inventory
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "ollama ls"
```

**Phase 2 — API smoke test through a tunnel** (no LAN exposure):

```bash
# serve must be up (fresh install leaves the app running). This errors if it isn't:
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "ollama ps"
# terminal A: tunnel (stays running, -N = no remote command)
ssh -N -L 11434:127.0.0.1:11434 -i ~/.ssh/id_ed25519 bota4go@192.168.1.112
# terminal B: native API, then OpenAI-compat API
curl -s http://127.0.0.1:11434/api/chat -d '{"model":"qwen2.5:14b","stream":false,"messages":[{"role":"user","content":"Reply with the word ok."}]}'
curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","messages":[{"role":"user","content":"Reply with the word ok."}]}'
# GPU proof
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "ollama ps"
```

**Phase 3 — on-demand start/stop (games win by default):**

Nothing starts at boot. Ollama runs only when the Pi asks, and weights unload after
5 idle minutes (default `OLLAMA_KEEP_ALIVE`), so VRAM returns to games automatically.
Explicitly delete any long keep-alive if one was ever set:

```bash
# one-time setup: demand-startable task (disabled schedule; started only via /run).
# uses the Phase-1-confirmed path; runs as bota4go so it sees the same OLLAMA_MODELS
# library (schtasks will ask for the account password via /rp)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 'schtasks /create /tn OllamaServe /tr "\"<PATH-FROM-PHASE-1B>\" serve" /sc onstart /ru Susanna\bota4go'
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "schtasks /change /tn OllamaServe /disable"
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 'powershell -NoProfile -Command "[Environment]::GetEnvironmentVariable(\"OLLAMA_KEEP_ALIVE\", \"Machine\")"'
# ^ must print nothing (default 5m idle unload). If it prints a value, remove it:
# ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 'powershell -NoProfile -Command "[Environment]::SetEnvironmentVariable(\"OLLAMA_KEEP_ALIVE\", $null, \"Machine\")"'

# start serving (from the Pi, whenever a model is needed)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "schtasks /run /tn OllamaServe"

# instant VRAM release before gaming (frees GPU immediately, no 5m wait)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "ollama stop qwen2.5:14b"
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "schtasks /end /tn OllamaServe"
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "nvidia-smi --query-gpu=memory.free --format=csv"
```

**Phase 4 — daily driver:** point one real client at `http://127.0.0.1:11434/v1`
(OpenAI client: `base_url="http://127.0.0.1:11434/v1"`, `api_key="ollama"`).
Per-call context via the native API: `"options": {"num_ctx": 8192}` (default is 4096;
8–16K costs ~1–2 GB extra VRAM at 14B).

## Validation Plan

- Phase 1: `ollama ls` shows `qwen2.5:14b`; `nvidia-smi --query-gpu=memory.total,
  memory.free --format=csv` baseline recorded.
- Phase 2: both curls return `ok`; `ollama ps` shows `100% GPU` (per official FAQ,
  this column is the GPU-placement proof). `curl -s http://127.0.0.1:11434/api/tags`
  lists the model.
- Phase 3: after a SUSANNA reboot, `ollama ps` (over SSH) shows nothing and
  `nvidia-smi` free ≈ 16 GB (games untouched). Then `schtasks /run`, re-run Phase 2
  checks — must pass. Then `ollama stop` + `schtasks /end`, and free VRAM returns
  to the reboot baseline.
- Highest-risk check: Phase 2 GPU proof. If PROCESSOR shows CPU, update Ollama first
  (automatic updates via taskbar) and close GPU apps (Chrome/game launchers) before
  changing anything else.

## Risks / Rollback

- Installer over non-interactive SSH may need a logged-in session; if Phase 1a fails,
  run the same `irm ... | iex` line once in a local PowerShell, then continue headless.
- Scheduled-task-as-SYSTEM would look in the wrong model dir; mitigated by running as
  `bota4go`. Rollback: `schtasks /delete /tn OllamaServe /f` removes the launcher entirely.
- Game conflict: if a game stutters while a model is resident, `ollama stop` + 5m idle
  unload is the fix path — no config change needed. Start-serve while a heavy game holds
  VRAM may fail to offload to GPU; mitigation: stop the game or accept partial CPU
  offload (`ollama ps` shows the split).
- A 24B experiment can OOM VRAM; mitigation: `ollama ps` watch + `ollama stop <model>`;
  disk is not a risk (307 GB free).
- No secrets involved (no API keys; SSH key auth already in place). Nothing to rotate.

## Open Questions

None. Model choice, runtime, access method, and persistence are all decided above;
remaining unknowns (exact `ollama.exe` path) are resolved by Phase 1b measurement.

## Sources

- [https://raw.githubusercontent.com/ollama/ollama/main/README.md](https://raw.githubusercontent.com/ollama/ollama/main/README.md) — Windows install (`irm ...install.ps1 | iex`), `/api/chat` usage.
- [https://docs.ollama.com/faq](https://docs.ollama.com/faq) — `OLLAMA_HOST`/`OLLAMA_KEEP_ALIVE`/`OLLAMA_CONTEXT_LENGTH`/`OLLAMA_MODELS`, `ollama ps` GPU proof, automatic updates.
- [https://docs.ollama.com/api](https://docs.ollama.com/api) — base URLs `http://localhost:11434/api` and `/v1`, local requests need no key.
- [https://ollama.com/library/qwen2.5](https://ollama.com/library/qwen2.5) — `qwen2.5:14b` 9.0 GB / 32K context, `32b` 20 GB.
- [https://docs.ollama.com/gpu](https://docs.ollama.com/gpu) — RTX 5060 Ti compute 12.0 supported, driver 550+.
- [https://raw.githubusercontent.com/ggml-org/llama.cpp/master/tools/server/README.md](https://raw.githubusercontent.com/ggml-org/llama.cpp/master/tools/server/README.md) — `llama-server` OpenAI-compat routes, `-c`, `-ngl`, `--port`.
- [https://docs.ollama.com/api/openai-compatibility](https://docs.ollama.com/api/openai-compatibility) — local `POST /v1/chat/completions`, `api_key='ollama'` required but ignored.
- [https://docs.ollama.com/cli](https://docs.ollama.com/cli) — `ollama pull/ls/rm/ps/stop/serve`.
