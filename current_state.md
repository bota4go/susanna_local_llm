# Current state — local LLM on SUSANNA

Last update: 2026-09-20 ~17:50 UTC (SHUT DOWN — SUSANNA clean)

## Where things stand

- **Phase 1a DONE.** `ollama.exe` at
  `C:\Users\bota4go\AppData\Local\Programs\Ollama\ollama.exe` (v0.34.2).
- **Server start resolved:** the tray/UI wrapper (`ollama app.exe`) cannot init over
  headless SSH (`Unable to init instance`), but pure `ollama serve` runs fine —
  listening on 127.0.0.1:11434, GPU discovered via CUDA (15.9 GiB total). Phase 3
  task must run `ollama serve`, never the app.
- **Phase 1c VERIFIED:** `ollama ls` on-device shows
  `qwen2.5:14b  7cdf5a0187d5  9.0 GB`. Phase 1 complete.
- **Phase 2 IN PROGRESS:** tunnel open, both API curls sent; first answers pending.
  `ollama ps` currently empty (model not yet resident), VRAM still baseline
  (2393 MiB used / 13658 MiB free) — consistent with 9 GB first-load still in flight.
- **Phase 2 retry RUNNING (17:40 UTC):** root cause of the 17:17–17:30 failures found —
  a single transient 500 on the server at 17:17:14 (visible in `serve` log); chat
  POSTs never reached it. The server itself is healthy: ONE `ollama.exe` (PID 10984,
  the headless `serve` under sshd), port 11434 LISTENING, pull verified complete.
  Both API curls re-sent with status codes; first 9 GB GPU load takes minutes.
- **17:45 UTC probe:** inventory intact, but `ollama ps` STILL empty and VRAM still
  baseline (2648/13403) — no inference has run anywhere yet. Either the retry curls
  are still waiting on first load, or they died the same way. Verdict comes from the
  retry session the moment it lands.
- **SHUT DOWN 17:50 UTC (user request):** `ollama stop qwen2.5:14b` (nothing loaded),
  `taskkill /F ollama.exe` killed PID 10984, `ollama app.exe` was never running.
  Verified: no `ollama.exe` processes, no `OllamaServe` task (boot-clean — Phase 3
  never ran), VRAM back at desktop baseline (2648/13403). Local tunnel closed too.
  Retry curls confirmed dead (exit 52). Model stays on disk (`qwen2.5:14b 9.0 GB`)
  for next time. Only leftover: `%TEMP%\ollama-install.ps1` not found (already gone).
- **Next step (when resumed):** `schtasks /run`-style start was never wired; next
  session should start `ollama serve`, re-run Phase 2 curls, and watch for the
  ~17:17-type transient (single 500) before trusting a failure.
- **Next step:** when curls return `ok` + `ollama ps` shows `100% GPU`, Phase 2 is
  done → Phase 3 (demand-start task, then `ollama stop` + `schtasks /end` to hand
  VRAM back to games). If curls fail again, suspect the tray app fighting for the
  port and check `tasklist` for a second instance.
- **Lesson learned:** Windows sshd rejects exec requests containing `|` (`exec request
  failed on channel 0`). Workaround used: download script with `irm ... -OutFile`
  (no pipe), then run with `powershell -ExecutionPolicy Bypass -File`. `design.md`
  Phase 1a already reflects the working single-quote pattern; pipe-free commands only.
- **Not started:** Phase 1b (`where ollama`), 1c (`ollama pull qwen2.5:14b`, ~9 GB),
  1d (`ollama ls`), Phase 2 tunnel smoke test, Phase 3 on-demand task.

## Next steps (resume path — Phase 1 done, no reinstall needed)

1. Start server: `ssh bota4go@192.168.1.112 "ollama serve"` (headless; tray app can't
   init over SSH — never use it). Confirm `Listening on 127.0.0.1:11434` + CUDA GPU
   discovered in its log.
2. Phase 2 retry: open tunnel (`ssh -N -L 11434:127.0.0.1:11434 ...`), both API curls
   with status codes, `ollama ps` must show `100% GPU`. First 9 GB load takes minutes;
   distrust any instant empty failure — check the serve log for a transient 500 first.
3. If curls fail while the server is healthy: suspect tunnel, rebuild it; decisive
   split-test is on-box inference (`ollama run qwen2.5:14b "say ok"` over SSH).
4. Phase 3 (games-first): create disabled `OllamaServe` task with the recorded exe path,
   `schtasks /run` to serve, `ollama stop` + `schtasks /end` to free VRAM.
5. Recreate the 5-minute watcher if a long unattended run needs tracking.

## Notes

- SSH key-only auth to `bota4go@192.168.1.112` healthy throughout.
- Each SSH probe from the helper shell needs one-time sandbox approval; interactive
  user terminal needs none.
- 5-minute watcher cron (`7051f003`) deleted 17:55 UTC after shutdown; recreate on resume.
