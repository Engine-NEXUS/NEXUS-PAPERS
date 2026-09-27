# Agent Stack Latency Audit — TTS, STT, OCR, Memory & Optimization Patterns

**Date:** 2026-09-27
**Scope:** free/locally-runnable options for a 4–5 person team, latency numbers verified from
official docs/benchmarks, plus how NEXUS and clicky-windows optimize each stage.
Companion to `01-free-vision-models-for-screen-agents.md`.

---

## 1. TTS comparison (latency = time-to-first-audio)

### Local (unlimited, offline, $0)

| Engine | Size | TTFA / first audio | Real-time factor | Quality | Notes |
|---|---|---|---|---|---|
| **Kokoro-82M** | ~330 MB | 47–49 ms (CUDA), 361–668 ms (CPU) [tts-bench] | 0.02–0.03 GPU, 0.124 CPU | Excellent (top open quality) | NEXUS lazy-loads (~1.7 s first call, then cached; ~350 MB RAM) |
| **Piper (ONNX)** | ~60–200 MB | **62 ms** CPU [tts-bench linux-default] | ~0.04 (Pi5: 0.53 s/sentence) | Good | NEXUS default via `piper-rs`; instant, tiny |
| **MeloTTS** | ~200 MB | 130 ms (GPU) | 0.06 | Good, multilingual | |
| **espeak-ng** | tiny | ~100 ms | instant | Robotic (last-resort fallback) | Picovoice pipeline measured ~1.4 s (harness-inflated); direct synth is near-instant |
| **MOSS-TTS Nano** (tts.ai) | 100M | free-tier API, CPU-capable | — | New, edge-oriented | 2 GB VRAM, voice cloning |

### Free cloud (API)

| Service | Free quota | TTFA | Notes |
|---|---|---|---|
| **Microsoft Edge TTS** | unlimited via unofficial endpoint (Azure F0 = 500k chars/mo official) | ~500–1,000 ms network | 400+ voices, 140+ langs, commercial OK; **clicky default**; NEXUS merged build uses it |
| **Google Cloud TTS** | **1M chars/month** (billing account required) | ~300–600 ms | 380+ voices; commercial OK |
| **ElevenLabs free** | **10k chars/mo** (~12–15 min), attribution, **no commercial use** | Flash v2.5: 75 ms model inference, **p90 TTFA 135–150 ms** | Best quality; paid $5+/mo for real use |
| **AWS Polly** | 5M chars (12-mo trial) | ~300–800 ms | Billing required |

**Reference TTFA ranking (Cartesia 100-request p90):** ElevenLabs 135–150 ms → OpenAI TTS ~200 ms.
OpenAI TTS hallucination rate ~10% vs ElevenLabs 5%.

**Voice-assistant end-to-end note (Picovoice VART, incl. LLM token gen):** Orca 204 ms → streaming
ElevenLabs 504 ms → espeak 1,504 ms → Piper 1,587 ms → Azure 1,656 ms → OpenAI 2,925 ms → Kokoro 3,000 ms.
These include LLM latency — TTFA alone (table above) is the cleaner engine comparison.

**Recommended:** Piper (NEXUS default, 62 ms CPU) for instant local replies; Kokoro for quality when
latency budget allows; Edge TTS when network is up and voice quality matters.

---

## 2. STT comparison

### Local

| Engine | Size | Latency (first token / per short clip) | WER | Notes |
|---|---|---|---|---|
| **Moonshine v2 Tiny streaming** | 34M | **50 ms** | 12.0% | NEXUS option (ultra-low RAM) |
| **Moonshine v2 Small streaming** | 123M | **148 ms** (M3), 165 ms CPU | 7.84% | 13.1× faster than Whisper Small @ same WER |
| **Moonshine v2 Medium streaming** | 245M | **258 ms** (M3), 269 ms CPU | **6.65%** | **NEXUS default** (~400 MB RAM) |
| **Whisper Small** (faster-whisper) | 244M | 1,940 ms | 8.59% | |
| **Whisper Large v3** | 1,550M | 11,286 ms (M3) / ~17 s (linux x86) | 7.44% | |
| **whisper.cpp turbo q5_0** | 1.6 GB | 1.95 s/sample avg (CPU), quant WER +0.33% | — | 3–5× faster than HF Transformers (clicky/whisper.cpp #1526) |
| **Parakeet TDT** (MLX) | 600M | 0.18 s | competitive EN | Apple Silicon only |
| **Vosk** | small | ~50 ms partials | much worse than Whisper | Not recommended |

### Cloud (free tiers)

| Service | Free quota | Latency | Notes |
|---|---|---|---|
| **Groq `whisper-large-v3-turbo`** | 20 RPM, 2,000 RPD, 28,800 audio s/day | ~247 ms (LPU) | **NEXUS primary STT**; free |
| **Deepgram** | trial credits ($200k-style promos vary) | ~200–400 ms | clicky uses it; paid after credits |
| **OpenAI Whisper API** | none (paid $0.006/min) | ~300–600 ms | paid |

**Recommended:** NEXUS's existing cascade is already optimal — Groq cloud (~247 ms, free) primary,
Moonshine Medium (269 ms, 6.65% WER) local fallback; `localSttOnly` for privacy. For clicky, add
Moonshine/faster-whisper base local before Deepgram to save quota.

---

## 3. OCR comparison (screen text extraction)

| Engine | Accuracy | Latency | License/Cost | Notes |
|---|---|---|---|---|
| **PaddleOCR v3.7** | **96.3%** OmniDocBench | sub-second CPU (1.5M edge models) | Apache 2.0 | **Best open-source**; tables + layout |
| **RapidOCR (PP-OCRv4 ONNX)** | high on UI text | **~100–300 ms** CPU | Apache 2.0 | **clicky tier-2 pointer**; pure ONNX, no framework |
| **Tesseract 5** | CER 2.8% EN / 6.2% Hindi | 0.31 s on clean screenshots (ocr_compare); 3–8 s/page in fastocr bench (complex docs) | Apache 2.0 | No layout/tables; ~75% on complex docs (Paddle study) |
| **EasyOCR** | similar to Tesseract | **1.84 s** (5.9× slower than Tesseract) | Apache 2.0 | Too slow for real-time |
| **Surya (VLM-style)** | 83.3% olmOCR benchmark | 5 pages/s GPU (650M) | permissive | Layout + 90 langs; needs GPU |
| **VLM OCR (GPT-4o/Gemini/Claude)** | CER 3.5–3.8% EN — **2–5× worse than dedicated OCR** on raw transcription (fastocr 2026) | 1–3 s + cost | paid/free-quota | Use for *understanding* messy UI, not transcription |
| **Google Cloud Vision** | CER 1.0% | 1–3 s | $1.50/1k pages | paid |

**Recommended:** keep clicky's tiering — UIA (5 ms) → RapidOCR/PaddleOCR (~300 ms) → vision LLM (1–3 s).
Never use a VLM for plain text transcription when an ONNX OCR runs 10× faster and more accurately.

---

## 4. Agent memory comparison

| System | Search latency | Model/quality | Cost | Notes |
|---|---|---|---|---|
| **SQLite + SM-2 spaced repetition** (clicky `memory/journal*`) | **<1 ms** | n/a (rules) | $0, single file | clicky's actual choice — journal + review scheduling |
| **SQLite + sqlite-vec / FAISS local** | **5–20 ms** (vector) + ~10 ms embedding (bge-small CPU) | good | $0 | True Memory paper: single-SQLite approach **93.0% LoCoMo vs Mem0 61.4%** |
| **Mem0** (63k stars) | +100–500 ms (async LLM extraction) | good | hosted paid / self-host LLM cost | extraction call adds latency + token cost per turn |
| **Letta, LangMem, Cognee, Zep/Graphiti** | 100 ms–2 s (KG/LLM passes) | strong for long-horizon | mostly paid/hosted | Zep needs Neo4j; overkill for desktop agent |
| **Cloudflare KV / D1** (NEXUS worker) | 1–5 ms edge | n/a | free tier | cross-device sync, already available |

**Recommended:** local single-file SQLite (journal + FTS5 + optional sqlite-vec) — sub-ms reads, no
network, no per-query LLM cost. That is exactly what clicky does and what True Memory validated.
Add CF KV only for cross-device sync of preferences.

---

## 5. Agent latency optimization — patterns ranked by impact

### End-to-end voice-agent budget (NEXUS path)

| Stage | Best (local-first) | Typical | Worst (cloud-only) |
|---|---|---|---|
| Wake → STT result | 150 ms (Moonshine tiny) | 250 ms (Groq) | 2 s (cold Whisper cloud) |
| Intent parse | **<1 ms** regex | 10–30 ms BERT-Mini ONNX | 300–800 ms LLM classify |
| LLM answer | 242 ms (9Router Cerebras→Groq) | 600 ms | 2,100 ms (Worker path) |
| TTS first audio | 62 ms (Piper CPU) | 300 ms | 1,000 ms (Edge network) |
| **Total** | **~0.5 s** | **~1.2 s** | **~4 s** |

Screen-agent budget (clicky): screenshot ~100 ms + VLM TTFT ~450 ms + gen ~1 s + TTS ~500 ms ≈ **2 s**.

### Optimizations (both codebases)

1. **Tiered cascade — cheap/local first, expensive/cloud last** (biggest win):
   NEXUS intent: regex <1 ms → BERT ONNX → Qwen brain; 9Router: Cerebras → Groq → Gemini → Worker.
   clicky pointer: UIA 5 ms → OCR 300 ms → VLM 1–3 s.
2. **Streaming:** TTS from first LLM token (word-boundary); STT partials; never wait for full LLM completion before speaking.
3. **Lazy load + keep-alive tradeoff:** NEXUS Kokoro lazy-loads (−350 MB idle, +1.7 s first speak), STT keep-alive avoids 10–15 s cold loads.
4. **Parallelize independent calls:** NEXUS architect `tokio::join!` (600–1000 ms saved), MCP state probes parallelized 25 s → 5 s.
5. **Cache aggressively:** clicky 30-day model-list cache; NEXUS Cloudflare KV edge cache; KV D1 fallback.
6. **Reduce call count:** clicky two-stage grid = 2 VLM calls/point → single-pass SoM or UIA-first would halve it.
7. **Free-tier quota sharding:** one API key **per person** multiplies aggregate RPM/RPD (5× for a 5-person team).
8. **Thinking/reasoning off** for interactive turns: Gemini ~0.45 s TTFT vs 7–15 s with reasoning.

## Sources

- NEXUS docs: `docs/features/46-*` (TTS), STT architecture section, `src-tauri/src/tts.rs`, `router.rs` (9Router)
- clicky-windows: `ai/hybrid_pointer.py`, `ai/universal_locator.py`, `companion_manager.py`
- Moonshine benchmark table (husky-moor/moonshine site), whisper.cpp issue #1526, faster-whisper README
- tts-bench (5uck1ess/tts-bench), MeloTTS README, Pi5 Piper latency issue
- ElevenLabs docs (latency page, free tier), Cartesia TTFA/WER comparisons, undetectr/audexum free-tier surveys (2026)
- fastocr.org OCR benchmarks 2026, PaddleOCR study (75% Tesseract on complex docs), ocr_compare
- PingCAP "Local-First with SQLite" + True Memory paper (LoCoMo), Mem0 docs, arxiv 2602.14038 (adaptive memory for agents)
