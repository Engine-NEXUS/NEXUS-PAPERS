# Free Vision Models for Screen Agents — clicky-windows Analysis & Ranking

**Date:** 2026-09-27
**Scope:** `Bitshank-2338/clicky-windows` (Windows screen-tutor agent), `mnfst/awesome-free-llm-apis`,
official Google/Groq/OpenRouter docs — all verified live on 2026-09-27.
**Goal:** best vision capability, $0 cost, limits sufficient for 4–5 people, with latencies.

---

## 1. clicky-windows — how it actually uses vision

| Component | Calls | Local? | Latency |
|---|---|---|---|
| `ai/hybrid_pointer.py` Tier 1: Windows UI Automation | 0 API | yes | ~5 ms, pixel-perfect |
| `ai/hybrid_pointer.py` Tier 2: RapidOCR/ONNX | 0 API | yes | ~300 ms, text-perfect |
| `ai/hybrid_pointer.py` Tier 3: vision LLM fallback | 1–2 | no | ~1–3 s, best-effort |
| `ai/universal_locator.py` two-stage grid (Set-of-Mark) | **2 per point** | no | ~1–3 s; accuracy 25–50 px @1080p |
| `ai/element_locator.py` Claude Computer-Use | 1 | no | ~5 px accuracy, **paid** |
| Main answer ("what's on screen") | **1** | no | 1 vision call |

**Quota math:** a full "answer + point" interaction ≈ **3 vision calls**.
The hybrid pointer means local UIA/OCR absorbs most pointing — API quota burns mainly on the answer call.

Providers wired in clicky: Claude (`claude-sonnet-4-6`, paid), OpenAI (`gpt-4o`, paid),
Copilot (`gpt-4o-mini`, free for students), Gemini (`gemini-2.5-flash`, free tier),
Ollama (`qwen2.5vl:3b/7b`, `llava:7b`), LM Studio.

---

## 2. Gemini 2.5 Flash-Lite deprecation — checked against official docs

Source: `ai.google.dev/gemini-api/docs/deprecations` (fetched 2026-09-27).

| Model | Status |
|---|---|
| `gemini-2.5-flash-lite` (Gemini API / AI Studio) | **No shutdown date announced — NOT deprecated, live & free** |
| `gemini-2.5-flash-lite-preview-09-2025` | **Shut down 2026-03-31** → `gemini-3.1-flash-lite` (the "deprecated" news most people saw) |
| `gemini-2.5-flash-lite` on **Vertex/Cloud** | Retirement **2026-10-20** (enterprise platform only — does not affect the free AI Studio key) |
| `gemini-3.1-flash-lite` | On deprecation schedule → shutdown **2027-05-07**, replacement `gemini-3.5-flash-lite` |
| 2.5 family access | Google "limits access to the 2.5 models to users who have actively used them" — **new keys may not see 2.5 models** |

**Verdict:** not deprecated on the free API. For new keys use **`gemini-3.5-flash-lite`** (current free
flash-lite, 338–350 tok/s measured by Artificial Analysis). Set thinking **off** for interactive agents —
reasoning modes show 7–15 s TTFT vs ~0.45 s non-reasoning.

Google no longer publishes per-model free-tier RPM/RPD; last-known / community values (verify in
AI Studio quota page): flash ≈ 15 RPM, flash-lite ≈ 30 RPM, ≈ 1,500 RPD per Google account.

---

## 3. Ranked free vision options for 4–5 people

| # | Option | Free limits | Latency (TTFT / speed) | Cost | Verdict |
|---|---|---|---|---|---|
| **1** | **Google Gemini API free** — `gemini-3.5-flash-lite` / `gemini-2.5-flash` | ~15–30 RPM, ~1,500 RPD **per Google account**; 5 people = 5 keys → 75–150 RPM aggregate | **~0.45 s TTFT, 204–346 tok/s** (Artificial Analysis) | $0 (list: $0.10–0.30/M in) | **Best quality + speed.** One key per person; shared single key is snug (~300 queries/user/day) |
| **2** | **Groq free** — `qwen/qwen3.6-27b` (vision ✅, image = 2048 tok) | 30 RPM, 1,000 RPD | LPU: **sub-second TTFT** (fastest first-byte class) | $0 (list $0.60/$3.60) | Best failover. *awesome-list shows Groq as text-only — Groq vision docs confirm image input* |
| **3** | **Local Ollama** — `qwen2.5vl:7b` (5 GB) / `:3b` (3 GB) | Unlimited, offline, private | GPU: ~2–6 s TTFT, 20–50 tok/s; M4: 47 tok/s but ~10 s image prefill; **CPU-only 30–90 s = unusable** | $0 + hardware | Only where each user has ≥8 GB VRAM GPU |
| **4** | **GitHub Copilot** (students) | quota-based; clicky auto-prioritizes free models | fast (`gpt-4o-mini`) | $0 | Student-only; already built into clicky |
| **5** | **Cloudflare Workers AI** | 10K neurons/day (shared across models) | ~1–2 s typical | $0 | NEXUS already runs on CF; vision: `gemma-4-26b-a4b-it`, `llama-4-scout`. Neurons burn fast on images |
| **6** | **Kilo gateway** — `stepfun/step-3.7-flash:free`, `nemotron-nano-omni:free` | **200 req/hr/IP, no key** | varies (rotating pool) | $0 | Generous but volatile pool; prompts may be logged |
| **7** | **OpenRouter `:free`** (live 2026-09-27: `qwen3.8-27b`, `gemma-4-26b/31b`, `inkling`+small, `dots-3`, `nemotron-nano-omni`) | 50 RPD **per model**, 20 RPM (1,000 RPD after $10 top-up) | varies by host | $0 | Too thin for 5 users unless rotating across the 8 free vision models |
| **8** | **Mistral free mode** — Mistral Small 4, Ministral 3 8B/14B | $10/mo credits per account | good | $0 | Solid, but credits shared across all Mistral products |
| **9** | **Z AI `GLM-4.6V-Flash`** | free but **1 concurrent request** | — | $0 | Serializes 5 users — bottleneck |
| **10** | Cohere trial (1,000 calls/**month**), OVHcloud anonymous (2 RPM/IP, Qwen2.5-VL-72B), HF ($0.10/mo) | too thin | — | $0 | Not viable for 4–5 people |

**Excluded (not free):** Claude Computer-Use (~$0.006/call), GPT-4o, Gemini 2.5 Pro (50 RPD).

**Paid-equivalent cost per screenshot query** (~2k vision tokens, for budgeting when free quota runs out):
Gemini 2.5 Flash-Lite ≈ $0.0003 · 2.5 Flash ≈ $0.0007 · GPT-4o ≈ $0.004 · Claude Sonnet ≈ $0.006.

---

## 4. Recommended $0 stack for 4–5 people

1. **Primary:** Gemini free — one key **per person** (multiplies quota 5×), `gemini-3.5-flash-lite`, thinking off.
2. **Failover:** Groq free `qwen/qwen3.6-27b` (vision, LPU fast).
3. **Offline/privacy:** Ollama `qwen2.5vl:7b` where a GPU exists.
4. Keep clicky's hybrid pointer tiers (UIA → OCR → vision) — API quota burns only when local tiers miss.

## Sources

- clicky-windows repo + `ai/hybrid_pointer.py`, `ai/universal_locator.py`, `ai/element_locator.py`, `ai/model_registry.py`
- https://ai.google.dev/gemini-api/docs/deprecations (live fetch 2026-09-27)
- https://ai.google.dev/gemini-api/docs/rate-limits, https://console.groq.com/docs/vision
- https://github.com/mnfst/awesome-free-llm-apis (README + footnotes)
- OpenRouter `GET /api/v1/models` (live, 2026-09-27: 17 free, 8 with image input)
- Artificial Analysis model pages (Gemini 3.5/3.6 Flash, Gemini 2.5 Flash)
