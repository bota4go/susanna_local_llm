# How to use — SUSANNA local LLM (qwen2.5:14b on RTX 5060 Ti)

Tested 2026-09-22 on SUSANNA `192.168.1.112` via `bota4go@` key-only SSH. Model `qwen2.5:14b` (9 GB Q4_K_M, 14.8B) already on disk. Everything below is **LAN-only** — Ollama stays on `127.0.0.1:11434`, no public port.

---

## 0. Prereqs

- From the Pi you can `ssh -i ~/.ssh/id_ed25519 -o PasswordAuthentication=no bota4go@192.168.1.112 "whoami"` → `susanna\bota4go`
- SUSANNA has `C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe` (v0.34.2) and `C:\Users\bota4go\.ollama\models\blobs` (8.9 GB) — if `curl http://127.0.0.1:11434/api/tags` on SUSANNA doesn't list `qwen2.5:14b`, re-pull: `ollama pull qwen2.5:14b` over SSH.

Remote shell is `cmd`, not PowerShell — chain with `&`, not `|`. For PowerShell, wrap: `powershell -NoProfile -Command "..."`.

---

## 1. Start the server (once per SUSANNA boot)

The tray `ollama app.exe` **cannot** start over headless SSH (`Unable to init instance`). Use the headless `ollama serve`. A plain `Start-Process ollama serve` dies when the SSH channel closes — start **detached** so it survives disconnect:

```bash
# from the Pi:
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "powershell -NoProfile -Command \"Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe serve'} | Select-Object ProcessId,ReturnValue\""

# verify (new SSH session, proves it survived disconnect):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "tasklist | findstr /i ollama & netstat -ano | findstr :11434 & nvidia-smi --query-gpu=memory.used,memory.free --format=csv"
# expect: ollama.exe  <PID> Services,  TCP 127.0.0.1:11434 LISTENING <PID>,  ~1690 MiB used (no model yet)
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "curl.exe -s http://127.0.0.1:11434/api/tags"
# expect: {"models":[{"name":"qwen2.5:14b",...}]}
```

If `wmic` is mentioned anywhere online — it's gone on Win11 24H2, use `Invoke-CimMethod` as above. `ollama ls` / `ollama ps` **without** a running `serve` will try to auto-start the tray app and hang ~5s — ignore that, check `tasklist`/`netstat` instead.

**Stop / free VRAM** (when done or before gaming):

```bash
# unload weights but keep server up (VRAM 11757→1694, 5 min keep-alive still active):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe stop qwen2.5:14b"

# kill server entirely (no ollama process, port closed):
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "tasklist | findstr ollama"
# note the PID, then:
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "taskkill /PID <PID> /F"
# or via WMI: powershell ... "Get-Process ollama | Stop-Process -Force"
```

The detached `serve` does **not** survive a SUSANNA reboot — after reboot repeat step 1. (Phase 3 will add a `schtasks OllamaServe /run` task for on-demand start; not needed for manual tests.)

---

## 2. SSH tunnel (so the Pi can hit `127.0.0.1:11434`)

Ollama binds only to loopback on SUSANNA. Forward it:

```bash
# on the Pi — keep this running in its own shell (or with setsid):
ssh -N -L 11434:127.0.0.1:11434 -i ~/.ssh/id_ed25519 -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 bota4go@192.168.1.112

# detached alternative (survives closing the shell you started it from):
setsid -f ssh -N -L 11434:127.0.0.1:11434 -i ~/.ssh/id_ed25519 -o ExitOnForwardFailure=yes -o ServerAliveInterval=30 bota4go@192.168.1.112 </dev/null >>/tmp/ssh_tunnel.log 2>&1
ss -tlnp | grep 11434
# expect: ssh LISTEN 127.0.0.1:11434 and [::1]:11434

# verify through the tunnel:
curl -s http://127.0.0.1:11434/api/tags | jq .  # or python -m json.tool
```

Stop the tunnel: `pkill -f "ssh -N -L 11434"` or `kill <pid>` (`ps aux | grep "ssh -N -L 11434"`).

---

## 3. Call the APIs (through the tunnel on the Pi)

First call after a fresh `serve` or after 5 min idle **loads 9 GB to VRAM** — expect **~30s** (`load_duration ~28s` in the JSON). Second call with same context is **<200 ms** (prompt cached). `ollama ps` shows `100% GPU` when resident.

### Native Ollama API

```bash
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "qwen2.5:14b",
  "stream": false,
  "messages": [{"role":"user","content":"Reply with the word ok."}]
}' | jq .

# with per-call context window (default 4096, costs ~1 GB per 4K at 14B):
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{
  "model": "qwen2.5:14b",
  "stream": false,
  "options": {"num_ctx": 8192},
  "messages": [{"role":"user","content":"Count to 3."}]
}' | jq .message.content
# Note: changing num_ctx reloads the model (8s) and bumps VRAM ~900 MB (4096→8192: 9.5→10 GB). Pick one and stick to it per session.
```

Streaming: set `"stream": true` and read newline-delimited JSON.

### OpenAI-compatible API

Base URL `http://127.0.0.1:11434/v1`, `api_key` is required by clients but **ignored** by Ollama — use any non-empty string (`"ollama"`).

```bash
curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "qwen2.5:14b",
  "messages": [{"role":"user","content":"Reply with the word ok."}]
}' | jq .
```

**Python (openai package):**

```python
from openai import OpenAI
client = OpenAI(base_url="http://127.0.0.1:11434/v1", api_key="ollama")
r = client.chat.completions.create(model="qwen2.5:14b", messages=[{"role":"user","content":"Reply with the word ok."}])
print(r.choices[0].message.content)  # -> Ok
print(r.usage)  # prompt_tokens, completion_tokens — second call shows cached_tokens
# per-call ctx:
# client.chat.completions.create(..., extra_body={"options": {"num_ctx": 8192}})
```

`curl -s http://127.0.0.1:11434/v1/models` also works.

### Also works directly on SUSANNA (no tunnel)

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "curl.exe -s --max-time 90 http://127.0.0.1:11434/api/chat -H \"Content-Type: application/json\" -d \"{\\\"model\\\":\\\"qwen2.5:14b\\\",\\\"stream\\\":false,\\\"messages\\\":[{\\\"role\\\":\\\"user\\\",\\\"content\\\":\\\"Reply with the word ok.\\\"}]}\""
```

---

## 4. Verify GPU residency

```bash
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe ps & nvidia-smi --query-gpu=memory.used,memory.free,utilization.gpu --format=csv"
# expect when loaded:  qwen2.5:14b  9.5–10 GB  100% GPU  4096|8192  4 minutes from now
#                      10861–11757 MiB used ( +~9 GB vs baseline 1690)
# expect when idle/stopped:  (empty ps)  1694 MiB used
```

If `PROCESSOR` shows `CPU` instead of `100% GPU`, close GPU apps (Chrome, game launchers) and retry — or update Ollama (Blackwell sm_120 needs recent builds, driver is 595.79 OK).

---

## 5. Quick health checklist

```bash
# Pi side:
curl -s http://127.0.0.1:11434/api/tags | grep -q qwen && echo "tags ok" || echo "tags FAIL"
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","stream":false,"messages":[{"role":"user","content":"ok"}]}' | grep -q '"done":true' && echo "chat ok"
curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","messages":[{"role":"user","content":"ok"}]}' | grep -q '"object":"chat.completion"' && echo "openai ok"

# SUSANNA side:
ssh -i ~/.ssh/id_ed25519 bota4go@192.168.1.112 "C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe ps"
```

---

## Troubleshooting

- **`Connection refused` on `127.0.0.1:11434`** → `serve` not running. Re-run step 1 (detached `Invoke-CimMethod`). `olllama ls/ps` alone won't start it headless — they time out with `Failed to start: Unable to init instance`.
- **First chat hangs ~30s then succeeds** → normal: 9 GB VRAM load. Don't kill it — second call is fast. A transient `500` on the very first load was seen once (2026-09-20) — retry.
- **`exec request failed on channel 0` over SSH** → you used `|` in the exec string. Use `&` to chain, or wrap PowerShell pipelines in `powershell -Command "... | ..."`.
- **VRAM OOM after `num_ctx` bump** → `ollama stop qwen2.5:14b` to unload, then retry with smaller ctx or close GPU apps.
- **Tunnel `address already in use`** → old tunnel still bound: `pkill -f "ssh -N -L 11434"` then retry.

---

## 6. Simple prompt examples — every installed model

All 4 models are `Q4_K_M` `100% GPU @4096` on SUSANNA (switch via `model`). Through the tunnel (`http://127.0.0.1:11434`). First call per model cold-loads `~3–15s`; second is `~100–200 ms` cached.

### qwen2.5:14b — default, balanced chat/reasoning (9.0 GB, 14.8B)

```bash
# native
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","stream":false,"messages":[{"role":"user","content":"Write a haiku about the Baltic sea."}]}' | jq .message.content

# longer context
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","stream":false,"options":{"num_ctx":8192},"messages":[{"role":"user","content":"Summarize this article in 3 bullets: ...paste text..."}]}' | jq .

# OpenAI compat + Python
curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","messages":[{"role":"user","content":"Explain why the sky is blue in one sentence."}]}' | jq .choices[0].message.content
```
```python
from openai import OpenAI
c = OpenAI(base_url="http://127.0.0.1:11434/v1", api_key="ollama")
print(c.chat.completions.create(model="qwen2.5:14b", messages=[{"role":"user","content":"Reply with the word ok."}]).choices[0].message.content)
```

### llama3.1:8b — Meta, great for code/instruct (4.9 GB, 8.0B)

```bash
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"llama3.1:8b","stream":false,"messages":[{"role":"user","content":"Write a Python function that returns the nth Fibonacci number."}]}' | jq .message.content

curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"llama3.1:8b","messages":[{"role":"user","content":"Reply with the word ok."}]}' | jq .
```
```python
print(c.chat.completions.create(model="llama3.1:8b", messages=[{"role":"user","content":"Explain recursion like I am 10."}]).choices[0].message.content)
```

### mistral:7b — Mistral, fast, good for summarization (4.4 GB, 7.2B)

```bash
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"mistral:7b","stream":false,"messages":[{"role":"user","content":"Summarize in one line: The quick brown fox jumps over the lazy dog."}]}' | jq .message.content

curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"mistral:7b","messages":[{"role":"user","content":"Reply with the word ok."}]}' | jq .
```
```python
print(c.chat.completions.create(model="mistral:7b", messages=[{"role":"user","content":"Give me 3 ideas for a weekend hike."}]).choices[0].message.content)
```

### phi4:latest — Microsoft, small but strong reasoning (9.1 GB, 14.7B, phi3 family)

```bash
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"phi4","stream":false,"messages":[{"role":"user","content":"Solve step by step: If a train travels 60 km/h for 2.5 hours, how far does it go?"}]}' | jq .message.content

curl -s http://127.0.0.1:11434/v1/chat/completions -H "Content-Type: application/json" -d '{"model":"phi4","messages":[{"role":"user","content":"Reply with the word ok."}]}' | jq .
```
```python
print(c.chat.completions.create(model="phi4", messages=[{"role":"user","content":"What is the capital of Estonia? Answer in one word."}]).choices[0].message.content)
```

### Multi-turn & streaming

```bash
# multi-turn (same model param, history in messages)
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"llama3.1:8b","stream":false,"messages":[{"role":"user","content":"My name is Anna."},{"role":"assistant","content":"Nice to meet you Anna!"},{"role":"user","content":"What is my name?"}]}' | jq .message.content
# -> Anna

# streaming (newline JSON, partial tokens)
curl -s http://127.0.0.1:11434/api/chat -H "Content-Type: application/json" -d '{"model":"qwen2.5:14b","stream":true,"messages":[{"role":"user","content":"Count to 3 slowly."}]}'
```

Switch `model` = `qwen2.5:14b` | `llama3.1:8b` | `mistral:7b` | `phi4` — that's it. Use `phi4:latest` also works (same as `phi4`). List all: `curl -s http://127.0.0.1:11434/api/tags | jq .models[].name`.

---

## Files & paths

- Binary: `C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe`
- Models: `C:\Users\bota4go\.ollama\models\blobs` / `manifests/registry.ollama.ai/library/{qwen2.5,llama3.1,mistral,phi4}`
- No `OllamaServe` scheduled task yet — server is manual-detached until Phase 3 (`schtasks /create /tn OllamaServe ... /disable` + `schtasks /run /end`).
