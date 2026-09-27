# NEXUS Screen Agent — Implementation Summary (Route C, Phases 1–4+5a)

**Date:** 2026-09-28
**Companion to:** `01-free-vision-models-for-screen-agents.md`,
`02-agent-latency-audit-tts-stt-ocr-memory.md` (same directory).
**Code:** `Engine-NEXUS/linux` branch `merge-origin-linux-preview` (PR #1).

## What was built

Clicky's screen-tutor concepts ported into NEXUS (Tauri/Rust) instead of
forked: screenshot Q&A + conditional pointer, point-only v1.

## Measured results (not estimates)

| Claim | Measurement |
|---|---|
| Portal capture (Wayland) | 1920×1080 → 1280×720 JPEG, ~1s first run, instant after |
| RapidOCR live | 0.98 confidence, 977ms warm on CPU |
| OCR snap accuracy | **96.7% over 150 synthetic samples** (`scripts/vision_accuracy.py`) |
| Test suite | 543+ Rust serial, tsc, 28 vitest, vite build, CI green |
| VLM end-to-end | **NOT measured** (no API key on test machines) |

## Bugs found by measurement (all fixed)

1. **Over-merging OCR lines** (3× line-height gap fused adjacent rows):
   fixed to 1× line height + tight vertical overlap → harness 75% → 91.7%.
2. **OCR drops inter-word spaces** ("Closewindow" scored 0.0): added
   space-insensitive containment (0.75) to the shared scorer → 96.7%.
   The shared scorer improved the UIA tier for free.
3. **Unfair harness**: 8px default-font text + overlapping placements.
   Fixed with 22px DejaVu + rejection sampling (harness must model real UI).
4. **Stale std-MutexGuard across .await** (!Send future): clone out first.
5. **CI referenced deleted dirs** (`server/sidecar`, `server/n8n`): dropped
   dead steps; data-foundation validation now runs and passes.

## Architecture decisions worth recording

- **Marker in main stage, not a new window**: the stage is never natively
  hidden — zero window/RAM/latency cost, CSS positioning is Wayland-safe.
- **No new Rust crates for capture**: GDI + portal/zbus + screencapture
  (xcap needed missing pipewire headers).
- **Speak-first, snap-after**: TTS never waits for OCR (~1s hidden in speech).
- **Shared fuzzy scorer across UIA + OCR tiers**: one threshold (≥0.5),
  consistent behavior, single test surface.
- **Privacy gate before pixels**: exclusion refuses capture AND UIA, not
  just the marker (Windows-enforced; other OSes lack fg-title APIs).
- **Push-protection incident**: a revoked `ghu_` token in old history
  blocked the branch push; redacted via surgical rebase (filter-repo had
  duplicated 39 shared commits — detected by subject comparison, redone).

## Open verification (needs a keyed machine + Windows/macOS)

VLM answer path, visual marker, UIA locate-first, TCC/portal first-run UX.
