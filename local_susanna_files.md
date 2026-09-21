# SUSANNA file layout — Ollama + SSH

Observed 2026-09-20 over SSH. Everything Ollama-related lives under `bota4go`'s profile.
No system-wide install, no services, nothing outside these paths.

## Program — `C:\Users\bota4go\AppData\Local\Programs\Ollama\`

| Entry | What |
|---|---|
| `ollama.exe` (26 MB) | CLI **and** headless server (`ollama serve`). This is what the Phase 3 task runs. |
| `ollama app.exe` (27 MB) | Tray/UI wrapper. **Cannot start over headless SSH** (`Unable to init instance`); never use it from the Pi. |
| `lib\` | CUDA / runner libraries (GPU backend). |
| `unins000.*` | Uninstaller. Run `unins000.exe` for full removal. |

Found via `where ollama` → this dir. Recorded for the Phase 3 `schtasks /tr` path.

## Models + keys — `C:\Users\bota4go\.ollama\`

This is the default `OLLAMA_MODELS` location (confirmed in server startup log).

| Entry | What |
|---|---|
| `models\blobs\` | Model weight blobs (the 9 GB `qwen2.5:14b` download lands here). |
| `models\manifests\` | Per-tag manifests pointing at blobs. |
| `id_ed25519` / `.pub` | Key Ollama generated for itself on first `serve` (signing/cloud use). Not your SSH key — leave alone. |
| `cache\` | Temp/download staging. |

## App state + logs — `C:\Users\bota4go\AppData\Local\Ollama\`

| Entry | What |
|---|---|
| `app.log` | Tray-app log (where the headless init failure was found). |
| `server.log` | Server log (0 bytes so far — `serve` logs to stdout over SSH). |
| `db.sqlite*` | Local model/history registry. |

## SSH (pre-existing, not Ollama)

| Entry | What |
|---|---|
| `C:\Users\bota4go\.ssh\authorized_keys` | Your login key ( Informational — untouched by this project). |
| `C:\ProgramData\ssh\administrators_authorized_keys` | Authoritative key file for admin users (see `README.md`). |

## Cleanup candidates (safe to delete after Phase 1)

- `%TEMP%\ollama-install.ps1` (22,627 bytes) — the downloaded official installer script.

## Disk note

C: free went 304.5 GB → 297.7 GB during the `qwen2.5:14b` pull — matches the ~9 GB model.
Plenty of headroom for a multi-model library.
