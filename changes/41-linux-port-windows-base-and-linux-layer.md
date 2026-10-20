# 41 — Linux port: Windows codebase + Linux platform layer

**Dates:** 2026-10-20 → 2026-10-21 · **Branch:** `port/windows-base` → PR into `Engine-NEXUS/linux` `main` · **Feature paper:** [`../features/58-linux-port-windows-base-and-linux-layer.md`](../features/58-linux-port-windows-base-and-linux-layer.md)

## 1. How the work was done

The coordinator (Claude) split the port into jobs, ran each in its own git worktree, routed the coding to the user's external CLI subscriptions by Artificial Analysis ranking (OpenCode MiniMax M3.1 / DeepSeek V4.1 Flash / GLM-5.3 Flash, Antigravity Gemini 3.8 Flash High, Codex GPT-5.6 Terra until its usage limit), and reviewed, built and tested every result itself before merging. Jobs that the CLIs could not finish (no edits after long runs, platform/FFI work) were done by the coordinator: J5's Rust and stage parts, J6 (the aura), J8, the microphone and focus root causes, the native shell. All edit jobs ran edit-only (no shell).

## 2. Jobs merged

| Job | Scope | Notes |
|---|---|---|
| base | `windows/main` + `linux/main` history, Windows wins conflicts; make the Windows codebase compile on Linux | keyring per target, Linux bodies for window/foreground/elevation, `/sys/class/drm` monitor size |
| T2 | screen understanding & desktop control | AT-SPI semantic tier, CDP tier bound to the focused app, identity policy gate; 9 review rounds |
| T3 | voice & audio layer | WebKitGTK media permissions, bounded audio tools, MPRIS, amixer parsing |
| T4a | app discovery, launch, honest focus | one "is it running" rule, focus by PID |
| T4b | typing, dictation, hand mode | Wayland gate, xdotool pid 0, clipboard honesty |
| T4c | GDK backend, overlay sizing, `--wake`, power commands | shutdown never powers off with apps open unless confirmed |
| T5 | packaging | `.deb`, `install.sh`, COSMIC RON hotkey `Ctrl+Super+Space`, `nexus check` |
| T6 | WebKitGTK frontend | opaque panels, Linux hotkey labels, `vad-web` 0.0.31 |
| T7, T7b | dev-run fixes, mic probe deadlines | see feature paper §4 |
| T8 | OS-native overlay | layer-shell stage, X11 fallback, input regions; plus the coordinator's fixes (non-focusable stage, main-thread GTK, weak refs, rebuild race) |
| J1 | Worker deny-by-default device gate | code only, **not deployed**; vitest 188/188 |
| J2 | `worker_request` proxy + device credential | allowlist, no token routes, legacy `user_id` supplied natively during the transition |
| J3 | renderer → Worker only through `worker_request` | Setup's OAuth calls migrated |
| J4 | screen/voice commands | locate/point, stop, repeat, point again, OCR snap |
| J5 | pointer marker + Screen pointer settings | `pointer.json`, CSP-safe marker, lifecycle tests |
| J6 | the aura | orb style `aura`, `@saystack/core@0.2.1`, shared level source |
| J7 | captions + last heard | word read-along on Edge timings; resend through Rust to the stage |
| J8 | preview harness + docs | `stage.html?eyeball`, `useRoam.ts` deleted |
| native shell | orb under the panels, sidebar drawer | safe-area measurement, Wayland monitor fallback |
| mic | never hold the raw DMIC | probe by name, reopen one device at a time |

## 3. User reports → causes → fixes

| Report | Cause | Fix |
|---|---|---|
| "nothing visual I can see" | wake thread blocked in ALSA `pipewire`; then NEXUS held the DMIC so PipeWire's own capture stalled | deadlines; reopen one device at a time |
| overlay restricts keys and clicks | tao re-enabled accept-focus; the Linux hitbox loop had no cursor and could make the window clickable | layer-shell keyboard NONE / `.focusable(false)`; input regions |
| "feel part of the OS" | stage and sidebar were ordinary toplevels | layer-shell overlay under the panels; layer-shell drawer |
| "orb inside a window" | the Windows black capsule | orb style `aura` |

## 4. Verification

Live on the target: `[GDK] Wayland session on COSMIC (layer-shell supported)`, `stage overlay running on LayerShell`, `free area 1920x1048 of 1920x1080 — top: 32`, `audio: device 'default' accepted`, `capture started in 2.2s`; drawer asked 400×1008 → got 400×1008; X11 fallback input shape empty when idle, only the orb when shown; aura in all four states through the preview harness. Suites: see [`../testing/linux-port-verification-and-preview-harness.md`](../testing/linux-port-verification-and-preview-harness.md).

## 5. Security notes

- No secret is in the branch (scanned for Groq/OpenAI/Google/GitHub/Slack/AWS key patterns and private keys before the push). The user's Groq key lives only in the OS keyring (`com.nexus.assistant` / `nexus-apikey-groq`).
- The Worker gate (J1) is code only. Deploy order: migrate D1 (`0002`), deploy the gated Worker with the transition window, ship clients that send the device bearer, then flip `REMOVE_LEGACY_USER_ID_AFTER_GATED_WORKER_DEPLOYMENT`.

## 6. Performance

All numbers measured on the target laptop (8 threads, 7.6 GB RAM).

**Where the time went.** Idle (no one speaking) the Rust process used **119% CPU** in `tauri dev`; one thread, `cpal_alsa_in`, was at ~100%. That thread runs the wake word: mel spectrogram → embedding → classifier (tract ONNX) on every 80 ms chunk that passes the silence gate — and a laptop room is above the gate most of the time (RMS 0.010–0.016 against 0.003). A temporary timer around `engine.process()` showed **88–113 ms of work per 80 ms chunk** (max 521 ms): the engine could not keep up with the microphone.

**Fix 1 — dev builds optimise the inference crates** (`[profile.dev.package.<crate>] opt-level = 3` for `tract-*`, `ndarray`, `matrixmultiply`, `rustfft`, `num-complex`, `webrtc-vad`; NEXUS's own crate stays at 0 for fast rebuilds). Result: **8.4 ms per chunk average, 13.7 ms max** (13× faster); idle CPU of the Rust process **119% → 19%** (audio thread 12%, tokio workers ~1% each). Optimising *every* dependency instead forces a full-tree rebuild that exhausted the laptop's memory — scope matters.

**Fix 2 — release builds: speed for the inference crates, size for the rest.** A standalone benchmark with the app's own models (tract 0.23.4, same shapes) at the release profile's `opt-level = "s"` vs `3`, six interleaved runs each:

| opt-level | best | median | per-chunk breakdown (median run) |
|---|---|---|---|
| `"s"` (release before) | 13.4 ms | 15.4 ms | mel ~0.8 + embedding ~14 + classifier ~0.3 |
| `3` (inference crates now) | 12.0 ms | 14.5 ms | |

6–10% faster; the embedding model is >90% of the cost. The rest of the binary stays size-optimised. (The Windows installer profile `dist` is unchanged.)

**Fix 3 — dead work removed.** A WebRTC VAD pass ran on every chunk (four 20 ms frames, an `i16` copy each) and its result was discarded (`let _ = voiced`). Removed; behaviour unchanged.

**Not done, on purpose: a VAD gate before the embedding model.** It would cut idle CPU in noisy rooms to ~1%, but it changes which audio the wake model sees, and the recall set (2,298 files, 92.3% recall) is not on the target machine. It needs that validation first.

**Memory.** Fresh dev run, all processes: ~610 MB (Rust 297 MB with debug info, stage WebKit 205 MB, network process 55 MB, Python STT sidecar 49 MB). Earlier, lower readings were taken while the system was deep in swap and under-report RSS.

**Startup.** Audio capture starts 2.1–2.2 s after launch; panels measured ~2.5 s after the stage maps.
