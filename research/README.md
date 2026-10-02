# NEXUS Research Knowledge Base

Welcome to the **NEXUS Research Knowledge Base**. This directory organizes all technical research, benchmark studies, hardware investigations, and design analyses conducted across the NEXUS system lifecycle.

---

## 📁 Directory Structure & Categories

```
docs/research/
├── micspecification/        # Microphone hardware, DSP, dynamic AGC, and spectral profiling
├── wakeword/                # Wake word acoustic training, augmentation, and models
├── nlu-intent/              # Intent classification, slot extraction, catalogs, and STT bias
├── live-mode/               # Ultra-low latency voice dictation and live audio streaming
├── mcp-connection/          # Model Context Protocol bridges (WhatsApp, Swiggy, Amazon, etc.)
├── system-architecture/     # Core system patterns, diagnostic reviews, and UI animations
├── screen-agent-stack/      # Screen agent vision models, TTS/STT/OCR/memory latency audit
└── 2026-10-competitive-audit/  # 10-angle audit vs. the live state of the art (Oct 2026)
```

---

## 🔥 2026-10 Competitive & Technology Audit

**Start here:** [`2026-10-competitive-audit/00-executive-summary.md`](2026-10-competitive-audit/00-executive-summary.md)

Ten parallel research agents scoped to one angle each, covering what is
**shipping now** vs **announced but unusable** as of 2026-10-02. Two of the
project's own earlier recommendations were **retracted** on the evidence.

**Three findings that change the plan:**

1. **The orb architecture cannot work on Linux Wayland.** `always_on_top` is an
   empty function in GTK3; `set_position` is impossible by design; click-through
   is broken on Mutter; and `set_ignore_cursor_events` panics in current stable
   `tao`. The fullscreen transparent stage must be replaced.
2. **The 2026 design consensus is the opposite of a full-size animated orb.**
   Google, Microsoft and Apple converged on small/monochrome/collapsed. Microsoft
   removed its assistant's colour *deliberately*.
3. **Endpointing is a bigger bug than the visuals.** SRI: a fixed 500 ms gate
   causes 100% premature cut-off; **100 ms with pre-pausal acoustics drops it to
   20.3%** — and that is our documented "speech onset decapitation" bug.

**Per-area verdicts:**

| Doc | Contents |
|---|---|
| **[00. Executive Summary](2026-10-competitive-audit/00-executive-summary.md)** | Synthesis, per-area verdicts, priority order, scope changes |
| **[01. Platform Constraints](2026-10-competitive-audit/01-platform-constraints.md)** | Tauri v2 / Linux Wayland capability matrix, the unfixed `tao` panic, window-architecture alternatives |
| **[02. Ambient UI & Design Language](2026-10-competitive-audit/02-ambient-ui-design-language.md)** | Google/Microsoft/Apple presence patterns, collapse rules, M3 Expressive motion specs, documented redesign failures |
| **[03. Voice UX & Conversation](2026-10-competitive-audit/03-voice-ux-conversation.md)** | Turn-taking (+208 ms floor), latency ladder, state signalling, error recovery, confirmation, why not to build visemes |
| **[04. Voice Stack SOTA](2026-10-competitive-audit/04-voice-stack-sota.md)** | Wake word / VAD / STT / TTS state of the art with concrete upgrade paths and measured numbers |
| **[05. NLU Intent & Decision Models](2026-10-competitive-audit/05-nlu-intent-decision-models.md)** | Jev + Laya verdict with independent benchmarks; the fine-tune path; local model landscape |
| **[06. Computer-Use & Linux Gap](2026-10-competitive-audit/06-computer-use-linux-gap.md)** | The Linux moat (verified from primary vendor docs), OCR/GUI-grounding SOTA, AT-SPI, pointer patterns |
| **[07. Security & OAuth](2026-10-competitive-audit/07-security-oauth-authorization.md)** | The unauthenticated token endpoints, MCP spec violations, secure Linux storage, prompt injection |
| **[08. Distribution & Packaging](2026-10-competitive-audit/08-distribution-linux-packaging.md)** | Flathub's Generative AI policy, Tauri bundle reality, the microphone-portal gap |
| **[09. Dev-Tool UI & Design Systems](2026-10-competitive-audit/09-devtool-ui-design-systems.md)** | Token architecture, dark-mode convention, and the four AI-slop tells our CSS already ships |

**Standing scope decision (2026-10-02):** multilingual (Hindi/Telugu) is **out of
scope** for this cycle. This removes the single largest unmeasured risk in the
stack — no verified Telugu benchmark exists for any sub-6B model — and unlocks the
English-only intent-classification literature and the SiFT footprint optimisation.

---

## 📑 Research Catalog by Domain

### 🎙️ 1. Microphone Hardware & DSP (`micspecification/`)
Deep-dive investigations into laptop internal mic arrays, spectral noise gates, dynamic AGC, and hardware calibration:
- **[Apex Wake Word & Laptop Mic Hardware Adaptation](micspecification/apex-wake-word-and-laptop-mic-hardware-adaptation.md)**:
  Analysis of laptop mic arrays (Intel Smart Sound Technology, Realtek HD Audio), fan resonance peaks (113.3 Hz), automatic high-pass filtering (128.3 Hz), dynamic hardware pre-gain scaling, and impulsive noise rejection (coughs, key clicks).

### ⚡ 2. Wake Word Research (`wakeword/`)
Production plans and benchmarks for real-time edge wake word detection:
- **[Wake Word Training Production Plan](wakeword/wake-word-training-production-plan-2026-09-14.md)**:
  Architecture and deployment specs for openWakeWord embedding classifiers, false alarm suppression, and Google Colab / local PyTorch training workflows.

### 🧠 3. NLU & Intent Engine (`nlu-intent/`)
Taxonomy, benchmark datasets, STT conditioning, and model evaluation:
- **[Command Phrasing Catalog](nlu-intent/command-phrasing-catalog-2026-09-11.md)**:
  Reference phrasing corpus across all 55+ production intent families.
- **[External NLU Data Sources & Acquisition Plan](nlu-intent/external-nlu-data-sources-and-acquisition-plan-2026-09-13.md)**:
  Strategic plan for external dataset acquisition (MASSIVE, CLINC150, SLURP) and zero-poisoning curation.
- **[NLU Model & Dataset Deep Audit](nlu-intent/nlu-model-and-dataset-deep-audit-2026-09-14.md)**:
  Dataset integrity verification, slot consistency checks, and train/test leakage audit.
- **[NLU Model Data Audit Latest](nlu-intent/nlu-model-data-audit-latest.md)**:
  Latest verification benchmarks for BERT-Mini ONNX model accuracy and OOS rejection.
- **[NLU & Wake Word Accuracy Gap Analysis](nlu-intent/nlu-wakeword-code-vs-research-accuracy-gap-2026-09-14.md)**:
  Comparative analysis between research expectations and live code performance.
- **[STT Vocabulary Biasing & Conditioning](nlu-intent/stt-vocabulary-bias-2026-09-22.md)**:
  Whisper STT prompt biasing, temperature stabilization (0.0), and multilingual hallucination suppression.
- **[List PRs Tolerance, Collect Categories & Sidebar Minimalism](nlu-intent/list-prs-tolerance-collect-categories-sidebar-minimalism-2026-09-18.md)**:
  Deterministic regex relaxation, interactive voice collector category tree, and minimal UI overhaul.

### 🚀 4. Live Mode & Audio Streaming (`live-mode/`)
Real-time audio streaming and continuous dictation:
- **[Live Mode Feasibility Study](live-mode/live-mode-feasibility-2026-09-11.md)**:
  Feasibility and latency budgeting for live desktop voice streaming.
- **[Live Mode Source Analysis](live-mode/live-mode-source-analysis-2026-09-11.md)**:
  Deep-dive analysis of open-source streaming STT engines and WebSocket protocols.

### 🔌 5. MCP (Model Context Protocol) Bridges (`mcp-connection/`)
Comprehensive multi-part research series on MCP client connections, OAuth security, and real-time pairing:
- **[01. Industry Patterns & Architecture](mcp-connection/01-industry-patterns.md)**
- **[02. OAuth 2.1 Specification & PKCE Flows](mcp-connection/02-oauth-spec.md)**
- **[03. WhatsApp MCP Bridge & Session Rotation](mcp-connection/03-whatsapp.md)**
- **[04. Swiggy Food Delivery Bridge & Spec-OAuth](mcp-connection/04-swiggy.md)**
- **[05. Amazon & Long-Tail Commerce Bridges](mcp-connection/05-amazon-and-long-tail.md)**
- **[06. Shared Bridge Infrastructure & Fallback](mcp-connection/06-shared-infrastructure.md)**
- **[07. Comparison & Verdict Matrix](mcp-connection/07-comparison-verdict.md)**
- **[08. Gap Analysis & Live Upgrade Plan](mcp-connection/08-gap-analysis-and-upgrades.md)**

### 🏗️ 6. System Architecture & Core Diagnostics (`system-architecture/`)
Cross-cutting architectural evaluations and debugging studies:
- **[05. Enterprise Integration Patterns](system-architecture/05-enterprise-patterns-research.md)**
- **[06. Implementation Deep Research](system-architecture/06-implementation-deep-research.md)**
- **[Diagnostic Review & Performance Audit](system-architecture/diagnostic-review-2026-09-12.md)**
- **[Orb Stuck Animation Root Cause Analysis](system-architecture/orb-stuck-animation-root-cause-2026-09-18.md)**
- **[PR Analyse Button & Sidecar Event Flow](system-architecture/pr-analyse-button-flow-2026-09-18.md)**
