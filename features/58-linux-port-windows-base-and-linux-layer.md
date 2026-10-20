# 58 — NEXUS on Linux: the Windows codebase with a Linux platform layer

**Date:** 2026-10-20 → 2026-10-21
**Status:** merged on branch `port/windows-base` of `Engine-NEXUS/linux` (PR: see `changes/41-…`). Verified live on Pop!_OS 24.04, COSMIC (Wayland + XWayland), PipeWire, Intel SOF DMIC, in `tauri dev`.
**Change log:** [`changes/41-linux-port-windows-base-and-linux-layer.md`](../changes/41-linux-port-windows-base-and-linux-layer.md) · **Testing:** [`testing/linux-port-verification-and-preview-harness.md`](../testing/linux-port-verification-and-preview-harness.md) · **Research:** [`research/linux-platform/`](../research/linux-platform/00-index.md)

---

## 1. Why a rebase onto the Windows codebase

The Windows and Linux repositories had drifted: of 58 Rust files the two shared, 6 were identical, and both deployed the same Cloudflare Worker. The Windows repository carried most of the product (orchestrator, memory core, voice loop, ghost mode, captions, orb design rounds); the Linux repository carried the platform work (AT-SPI/CDP perception, Wayland-aware input, PipeWire audio, COSMIC integration) and a security-hardened Worker.

Decision (user, 2026-10-20): **Windows base + Linux layer.** Branch `port/windows-base` starts at `windows/main`, merges `linux/main`'s history with the Windows side winning every conflict, then re-applies the Linux platform layer as reviewed jobs. Windows and macOS behaviour is unchanged by construction: Linux code is `#[cfg(target_os = "linux")]` or a Linux-only module.

## 2. The Linux platform layer

| Area | What runs on Linux | Where |
|---|---|---|
| Build | keyring Secret Service (`linux-native-sync-persistent`), Linux bodies for foreground title / window state / elevation, `/sys/class/drm` monitor size | `Cargo.toml`, `screen_linux.rs` |
| Screen understanding & control | AT-SPI semantic tier (foreground-scoped lookups and clicks, reading order, credential filtering), Chromium/Electron CDP tier bound to the focused app (fail-closed target choice), identity policy gate on every capture/action | `linux/*`, `policy.rs` (T2) |
| Voice & audio | WebKitGTK media permission handler on every webview, PipeWire/Pulse/ALSA capture probing with hard deadlines, amixer/wpctl volume, MPRIS media control, keep-awake | `audio_linux.rs`, `mic_permissions.rs` (T3, T7, T7b) |
| Apps & input | app discovery + launch + honest focus (focus by PID, one process scan per call), typing/dictation with a Wayland gate, hand mode | T4a, T4b |
| Display backend | `gdk_backend_linux::choose` — native Wayland + layer-shell when available, else X11/XWayland | `gdk_backend_linux.rs` (T4c, T8) |
| Overlay | layer-shell stage below the panels; layer-shell sidebar drawer; non-focusable X11 fallback | `overlay_linux.rs` (T8) |
| Packaging | `.deb` with runtime deps (incl. `libgtk-layer-shell0`), `install.sh`, COSMIC RON hotkey (`Ctrl+Super+Space` → `nexus --wake`), `nexus check` | `tauri.linux.conf.json`, `scripts/` (T5) |
| Frontend | WebKitGTK fixes, opaque panels (no blur through a transparent window), Linux hotkey labels | `platform.ts` (T6) |
| Screen/voice commands | locate/point via the semantic tier, stop speaking, repeat, point again, OCR snap | J4 |
| Pointer marker | "point at X" marker on the stage + Screen pointer settings (`pointer.json`) | J5 |
| Captions | word-by-word read-along on real Edge timings, editable "last heard" row in the sidebar | J7 |
| Aura | bare orb ringed by an audio-reactive aura (`@saystack/core`), Linux default style | J6 |
| Security | Worker deny-by-default device gate (code, not deployed), renderer → Worker only through Rust's allowlisted `worker_request` | J1, J2, J3 |

## 3. Making the overlay part of the desktop

User report: *"when it covers the entire screen it shows up like another app on top of my current app, restricting keystrokes and mouse clicks — I want it to feel like part of the OS itself."*

**Path selection.** On a Wayland session whose desktop supports `zwlr_layer_shell_v1` (COSMIC, KDE, Hyprland, Sway, river, niri, Wayfire, labwc) **and** where `libgtk-layer-shell.so.0` loads (`libloading`, no build dependency), GTK runs natively on Wayland. Otherwise X11/XWayland. `NEXUS_GDK_BACKEND` overrides; a missing library prints the install hint once.

**Layer-shell stage.** Overlay layer, anchored to all edges, keyboard interactivity NONE — not a toplevel, so never focused and never in Alt-Tab. Click-through is the Wayland input region: empty when idle, exactly the union of the stage hitboxes (the orb, spatial pins) otherwise.

**Safe area.** The stage starts with exclusive zone 0, so the compositor's first configure is the area the panels leave free. The insets are measured once (700 ms after the first map), attributed to edges from COSMIC's panel config (`com.system76.CosmicPanel*` — exclusive panels only), and the stage then goes full screen (exclusive zone -1) so screen coordinates stay exact for the pointer, the tour and the pins. The orb is placed inside the safe area (`window_manager::rect_for`). Measured on the target: `free area 1920x1048 of 1920x1080 — top: 32`.

**Sidebar drawer.** A layer-shell surface on the top layer, keyboard ON_DEMAND, exclusive zone 0. `sidebar_geometry`'s rects become top + left/right anchors with margins inside the panel-free area; the size is the GTK size request. Measured: asked 400×1008 (right dock, margin 10, 20 below the panel), got 400×1008.

**X11 fallback.** The stage is created `.focusable(false)`, `Notification` type, skip taskbar/pager, keep-above, with the same input-shape union. Verified: WM_HINTS input False, `_NET_ACTIVE_WINDOW` stays 0x0 after `nexus --wake`, X input shape empty when idle and exactly the orb when shown.

### Root causes worth remembering

- **tao re-enables accept-focus after the first draw** for any focusable window created unfocused — that, not the window type, is why COSMIC focused the full-screen stage and swallowed keystrokes. Fix: build the stage `.focusable(false)`.
- **The Linux hitbox loop had no cursor position** (`stage::cursor_pos` is Windows-only) and could set the whole window clickable. Fix: input regions, no polling.
- **Wayland has no primary monitor** and an unmapped window has no current monitor (`Ok(None)` both): fall back to `available_monitors()`.
- **A non-resizable GtkWindow ignores the compositor's configure size**: the layer-shell stage is made resizable; the drawer is sized by size request.
- **`run_on_main_thread` lands after the caller's next synchronous step**: state the caller needs (drawer eligibility) must be decided synchronously.

## 4. The microphone

User report: *"NEXUS is running but there is nothing visual I can see."* The orb never drew because the wake-word thread never got audio.

1. **ALSA's raw `pipewire` PCM blocked forever** in `default_input_config()` (gdb: `futex_do_wait`). Every device query and probe now runs on its own thread behind a hard deadline (probe window + 3 s); a hung thread is abandoned, a late probe can never install its stream (three-state guard), and the probe order is `default`, `pulse`, hardware, raw `pipewire`, plugins, loopbacks.
2. **NEXUS held the raw DMIC open itself.** cpal 0.15's ALSA enumeration opens every listed PCM in both directions and keeps it open while the `Device` lives. Holding `hw:sofhdadsp,6` — PipeWire's own `Mic1` source — kept PipeWire from opening it, so capture through `default`/`pulse` stalled in `snd_pcm_prepare` (gdb: pipewire and pulse plugin condition waits); the probe then fell back to the raw device and took the mic from every other app. Reproduced outside the app with a 60-line cpal program. Fix: probe by name and reopen one device at a time (`audio_linux::reopen_input_device`; `default` opens lazily).

Result: `audio: device 'default' accepted`, capture started in 2.2 s, other apps keep the microphone.

## 5. The aura

User request: *"I want that aura feel — currently the orb is coming inside a window."* The orb sat inside the Windows black capsule.

- New orb style **`aura`** (Linux default; Windows keeps classic/pebble): the WebGL particle orb with no box, ringed by `@saystack/core@0.2.1`'s WebGL rim (MIT, no dependencies).
- State is mechanism, never hue: idle `paused` (dim rim), listening `active`, thinking `working` (orbit comet, ignores audio), speaking `active` + mirror.
- One energy level for orb and aura (`audio/levels.ts`) from the orb's own sources — TTS envelope while speaking, mic while listening — so the two never disagree.
- The aura's canvas parks whenever the orb is hidden (no GPU while idle). `isAdditive` must stay `false` (true makes the aura's alpha exactly 0).
- **Reduced motion calms, it does not remove.** Pop!_OS COSMIC ships with `enable-animations false`; hiding the aura there removed the look the user asked for. The renderer runs at 12% motion with no glide instead.

## 6. Performance

The wake word runs ONNX inference (tract) on every 80 ms microphone chunk above the silence gate. Unoptimised, that took 88–113 ms per chunk — the engine fell behind real time and the Rust process sat at **119% CPU while idle**. Optimising only the inference crates (`tract-*`, `ndarray`, `matrixmultiply`, `rustfft`, `num-complex`, `webrtc-vad`) brought it to **8.4 ms per chunk** and **19% idle CPU** in dev builds; in release those crates moved from `opt-level "s"` to `3` (6–10% faster per chunk, measured with the app's models), and a discarded per-chunk VAD pass was removed. A VAD gate before the embedding model would cut idle CPU further but needs a wake-word recall test first. Details: change log §6.

## 7. Known limits

- Multi-monitor: the stage and the drawer are pinned to the first monitor.
- Panel attribution is exact on COSMIC; on other desktops a gap is kept from both edges of an axis.
- The Worker gate (J1) is code only; deployment order is in the Linux repo's `docs/changes/linux-port-j1-…`.
- Windows CI on the PR: `rust-check-windows` passes (cargo check default / mock-wake / shipped features, dead-code ratchet, 1683 lib tests + integration tests); `worker-check` passes. Three jobs fail identically on `WINDOWS` main and belong to their owners: the IPC contract gate (unwired `agent_watch_*`, `get_return_review`, one-sided events), the NLU Data Foundation check (`d61c517` changed the locked dataset without a reviewed re-freeze), and the NSIS template (`RestartManager_StartSession` macro missing) — the Windows release build itself compiles.
