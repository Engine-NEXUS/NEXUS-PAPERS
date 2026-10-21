# Linux port — verification playbook and the stage preview harness

How the Linux build of NEXUS was verified (October 2026) and how to repeat it on another machine. Architecture: [`../features/58-linux-port-windows-base-and-linux-layer.md`](../features/58-linux-port-windows-base-and-linux-layer.md).

## 1. Automated suites (branch `port/windows-base`)

| Suite | Command | Result |
|---|---|---|
| Rust library | `cd src-tauri && cargo test --lib` | 1930 / 1931 pass. The one failure, `turn_detect::tests::model_loads_and_runs`, is the model's `< 1500 ms` speed assertion in an unoptimised debug build on a loaded machine; it predates the port and the model output check passes |
| Frontend | `cd frontend && npx tsc --noEmit && npx vitest run` | tsc clean; 477 / 477 |
| Worker | `cd worker && npx vitest run` | 188 / 188 (J1) |
| Production frontend build | `npx vite build`, then `grep -rlE 'eyeball\|EyeballPanel' dist/` | builds; the grep prints nothing (the preview harness is not shipped) |
| NLU data | `python server/nlu/data_foundation.py validate`; `python scripts/foundation_validation_test.py`; `python scripts/evaluation_split_test.py` | validation passes; 4 / 4; 5 / 5 |
| Linux CI (`ci.yml` on ubuntu-24.04) | every push and PR | `rust-check-linux`: three `cargo check` feature sets, dead-code ratchet (316 known, none new), 1937 lib tests + integration tests; `frontend-check` incl. the IPC contract gate; `worker-check`; `python-check` incl. the NLU Data Foundation — all pass |
| Linux package (`build-linux.yml`) | every push and PR | `NEXUS_0.1.0_amd64.deb` (84 MB) built through `scripts/build-prod.sh`, installed with apt on a clean runner; `/usr/bin/nexus` has no unresolved library; the model resources are in the package |

Known flakes, not port regressions: the live AT-SPI test (~1 in 10 on the base too), the `ghost` timing test (another team's module), `lazy_*` env tests when run with other suites in parallel.

## 2. Live checks on a Linux desktop (`tauri dev`)

Start: `tauri dev` from the repository root (logs with `RUST_LOG=info`).

| Check | How | Pass |
|---|---|---|
| Display path | log line `[GDK] …` | `Wayland session on <desktop> (layer-shell supported)` with `libgtk-layer-shell0` installed; otherwise `XWayland found — replacing …` and the install hint |
| Overlay path | `overlay_linux: stage overlay running on …` | `LayerShell` (or `fallback path` without the library) |
| Panel insets | `overlay_linux: free area WxH of MxN — panel insets …` | matches the desktop's panels (COSMIC default: top 32) |
| Microphone | `audio: device '…' accepted`, `audio capture started in …` | `default` accepted within ~3 s; no `hw:` device unless no audio server runs |
| Wake | say "Nexus" or `nexus --wake` | the orb appears under the top panel, ringed by the aura |
| Keyboard stays with the user's app | type in another app while the orb is shown | keystrokes reach that app (X11 fallback: `xprop -root _NET_ACTIVE_WINDOW` stays 0x0) |
| Clicks pass through | click anywhere but the orb | the click reaches the window underneath (X11 fallback: input shape empty when idle, only the orb when shown) |
| Sidebar drawer | `nexus --settings` | Command Hub docked right, under the panel, not in Alt-Tab; first open can take a few seconds (WebKit first load) |
| Speech to text | speak after the wake | `Groq present` in the start-up checklist; a transcript arrives |

Screenshots on COSMIC: `cosmic-screenshot --interactive=false --modal=false --notify=false --save-dir DIR` (grim is not supported there).

## 3. Diagnosing a stuck microphone

The two failure modes found on the target machine, and how they show up:

- **A hung PCM** — log: `audio: device '…' hung inside the device call for 5000ms — abandoning it`. The probe moves on; if every route hangs, attach gdb to a *child* process (yama `ptrace_scope=1` blocks attaching to the dev app): run the binary under `gdb -batch -ex run -ex 'thread apply all bt'` with `CARGO_MANIFEST_DIR` set (the wake engine finds its models there in dev) and the Vite server running, then `kill -INT` it during the stall.
- **The audio server locked out** — `default`/`pulse` stall in `snd_pcm_prepare` while `arecord -D default` works from a shell. Something in the process holds the raw `hw:` microphone (cpal's enumeration keeps every listed PCM open). `pw-dump` shows the NEXUS capture streams `suspended`.

## 4. The stage preview harness (`stage.html?eyeball`)

A dev-only page that renders the REAL stage — the WebGL orb, the aura, the pointer marker — in a normal browser with fixture states, so the look can be checked without a voice turn.

1. Start the dev server (`npm run dev` in `frontend/`, or `tauri dev`).
2. Open `http://localhost:5173/stage.html?eyeball`. Controls (bottom left) switch state, style (aura / classic / pebble), top/bottom, the pointer marker and the level; the URL follows them.
3. Parameters: `state`, `style`, `level` (0..1), `pos`, `pointer` (`1`/`0`), `panel` (top inset px, default 32).

Headless screenshots (software WebGL):

```bash
google-chrome --headless=new --user-data-dir=/tmp/eyeball-profile \
  --use-angle=swiftshader --enable-unsafe-swiftshader --hide-scrollbars \
  --window-size=1280,560 --virtual-time-budget=5000 --screenshot=speaking.png \
  "http://localhost:5173/stage.html?eyeball&state=speaking&style=aura&level=0.7"
```

It stays out of production: `stage/main.tsx` lazy-loads it only when `import.meta.env.DEV` is true and the URL has `?eyeball`, and `OrbFrame.tsx`'s harness listener is DEV-guarded; Vite folds the constant and drops both.

Results on the target (Chrome, SwiftShader): idle shows a dim rim; listening a bright multicolour halo; thinking a calm orbit around the orb's thinking morphology; speaking a mirrored rainbow ring; the orb hangs under the 32 px panel; the pointer marker draws a cyan ring, a centre dot and its label.
