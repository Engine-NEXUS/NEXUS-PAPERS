# Hey Siri (macOS) — Reverse-Engineering Findings via REA

**Date:** 2026-11-06
**Tool:** REA — "Reverse Engineer Anything" v6.8.0 (`rea-agents`, npx), engine-free static mode
**Host:** macOS 27.2 (build 26B123), Apple Silicon (arm64e)
**Scope:** Static, read-only analysis of the on-device Hey Siri wake-word stack on *this* Mac, for NEXUS wake-word interoperability research. No runtime instrumentation, no network, no modification.
**Second target (`hitadoma.in`):** does not exist — see [§8](#8-second-target--hitadomain).
**Status:** Complete for the engine-free pass. Decompilation (pseudocode) not performed — see [§7 Limitations](#7-limitations).

---

## 1. Why this is in scope

REA's own disclaimer scopes it to "lawful research, education, interoperability, software preservation, and analysis of software you own or are authorized to examine." This pass is:

- **On our own hardware** — the Mac this runs on.
- **Read-only** — only `otool`/`codesign`/`plutil`/`lipo` and REA's Mach-O/Obj-C parsers touched the files. Nothing was patched, launched, hooked, or debugged. SIP stayed on.
- **Interoperability-motivated** — NEXUS ships its own wake-word (openWakeWord/BERT-Mini pipeline). Understanding how Apple structures a shipping, always-on wake-word detector informs our own front-end, two-pass design, and false-trigger suppression.

It is **not** circumvention, piracy, or exploitation. No DRM was bypassed; no secrets extracted.

---

## 2. Method & tooling

| Item | Detail |
|---|---|
| Driver | REA v6.8.0 via `npx -y rea-agents@latest <cmd>` |
| Engines present | none (no Hopper / Ghidra / IDA / JADX) → **no pseudocode**; Mach-O/Obj-C/asset/string analysis only |
| Node | v26.13.0 (meets REA's ≥26.12.1) |
| Xcode CLT | present (`otool`, `codesign`, `plutil`, `lipo`, `nm`) |
| Commands used | `doctor`, `inspect-macho`, `inspect-signature`, `inspect-native-dispatch-metadata` |

**Key macOS reality:** on modern macOS the *framework* binaries (e.g. `CoreSpeech.framework/CoreSpeech`) are **not loose files** — they live only in the dyld shared cache (cryptex). The analyzable on-disk target is the **daemon** `corespeechd`, which statically links the whole pipeline and carries the Obj-C metadata and entitlements. That is what we analyzed.

> **Evidence vs. interpretation.** Facts below are directly from REA output (file digests, entitlements, load commands, class/selector names). Apple's internal **codenames** (Hearst, Remora, Myriad, Gibraltar, Darwin) are labeled *interpretation* — the class *names* are evidence; what the codename *refers to* is informed inference from public RE lore plus the surrounding symbols.

---

## 3. Target identity (evidence)

**Binary:** `/System/Library/PrivateFrameworks/CoreSpeech.framework/corespeechd`

| Field | Value |
|---|---|
| SHA-256 | `ca671b6a871dd90f26624d6850121f9832c77993de79c50aac2939836cfec5a1` |
| Format | Mach-O universal, 3 slices |
| Slices | `x86_64`, `arm64e`, **`arm64e.x1`** (cputype `0x0100000C`, subtype 12) |
| arm64e UUID | `2B994431-FA28-36AA-907E-F6EBF0A88AEE` |
| Built for | macOS 27.2 (min + SDK), linker `LD 27106.0` |
| Code signature | Apple: *macOS Software Signing → Apple Code Signing CA → Apple Root CA* |
| Signing id | `com.apple.corespeechd`; designated req `identifier "com.apple.corespeechd" and anchor apple` |
| Surface | 958 symbols · 67 dylib load-commands · 95 Obj-C classes · 181 protocols · 294 ivars · 1054 dispatch impls |

The **`arm64e.x1`** third slice is the standout. Combined with the entitlement `com.apple.private.exclaves.conclave-host` and the presence of `CoreSpeechExclave` / `SiriRaiseToSpeakExclave` frameworks on the system, this indicates a **secure-world ("exclave"/conclave) audio path** — the microphone-adjacent trigger code runs in an isolated execution environment, not ordinary userspace. *(Interpretation: the `.x1` variant is the exclave-page-table ABI; the entitlement confirms this daemon hosts a conclave.)*

---

## 4. Capabilities from entitlements (146 total)

The entitlement set is the clearest statement of what this daemon is allowed to do. Grouped by what matters for a wake-word design:

**Always-on / low-power audio capture**
- `com.apple.private.audio.hal.aop-audio.user-access` — **Always-On Processor (AOP) audio**: the dedicated low-power coprocessor that listens for the trigger while the main CPU sleeps.
- `com.apple.coreaudio.CanRecordPastData` — read the **rolling pre-trigger audio buffer** (capture the audio from *before* detection completed; this is how "Hey Siri" grabs the phrase it already heard).
- `com.apple.coreaudio.private.SystemWideTap`, `app-tap`, `CanTapTelephony`, `CanRecordWithoutSessionActivation`.
- `com.apple.private.audio.dark-wake-audio`, `notification-wake-audio`, `com.apple.private.iokit.darkwake-control` — process audio during **dark wake** (screen off / sleep).
- `com.apple.private.isolated.audio.coreaudioclient.shareddsp` / `.historicalaudio` — shared **DSP** + historical (pre-roll) audio.
- `com.apple.private.audio.suppress-mic-indicator`, `mediaexperience.microphoneattribution.allow` — the trigger pass runs **without lighting the mic indicator** (first pass is a classifier, not a "recording").

**On-device inference**
- `com.apple.aned.private.ANEAccess.allow` + `com.apple.security.ts.ane-client` — the model runs on the **Apple Neural Engine**.
- `com.apple.coreaudio.i-am-siri` — the privileged "I am Siri" audio-client flag.

**Personalization / learning (on-device)**
- `com.apple.private.DistributedEvaluation.RecordAccess-com.apple.fides.{asr,borealis,phs}` and `…siri.speech-dictation-personalization`.
- `com.apple.private.speech-model-training`, `com.apple.private.security.storage.SiriVocabulary`.

**Attention / arbitration / activation**
- `com.apple.private.attentionawareness` (+ `samplewhileabsent`) — gaze/attention-aware suppression.
- `com.apple.siri.deviceselection.allow` — multi-device "which device answers" arbitration.
- `com.apple.voicetrigger.voicetriggerservice`, `com.apple.siri.activation`, `com.apple.private.siri.invoke`, `com.apple.siri.embeddedspeech`.

*(Full 146-key list is in the evidence JSON; the above is the wake-word-relevant subset.)*

---

## 5. The pipeline, from linked frameworks (67 dylibs)

The dependency graph is effectively a block diagram of the trigger path:

| Framework | Role in the wake-word path |
|---|---|
| **VoiceTrigger** | First-pass always-on "Hey Siri" detector (paired with the `voicetriggerservice` entitlement) |
| **SpeechDetector** | Voice-activity / speech-presence detection |
| **EmbeddedAcousticRecognition** | On-device acoustic recognition — **second-pass** checker |
| **CoreEmbeddedSpeechRecognition**, **LocalSpeechRecognitionBridge** | On-device ASR after trigger |
| **SpeakerRecognition** (`SSR…`) | **Personalized "Hey Siri"** — voice ID of the enrolled user |
| **CoreML** + **Accelerate** (+ ANE entitlements) | Model execution (NN on the Neural Engine) |
| **SoundAnalysis** | Acoustic event classification |
| **SiriCrossDeviceArbitration** | Which nearby device responds |
| **DistributedEvaluation** + **Trial/TrialProto** | On-device evaluation & A/B threshold experiments |
| **MediaExperience**, **CoreAudio**, **AVFAudio**, **AudioToolbox(Core)** | Audio I/O, taps, session handling |
| **Biome(Library/Streams)** | On-device event-stream storage |
| **IntelligencePlatformLibrary** | Apple Intelligence integration |

---

## 6. Component architecture, from Obj-C classes (evidence)

The 95 classes name the design directly. Highlights (codename meanings = interpretation):

**Two-pass detection**
- `CSVoiceTriggerFirstPassHearst`, `CSVoiceTriggerFirstPassRemora` — **first pass** (cheap, always-on). *Hearst ≈ built-in mic path; Remora ≈ accessory/AirPods path (interp.).*
- `CSHearstSecondPassRequest`, `CSRemoraSecondPassRequest`, `CSRemoraSecondPassAudioStreamHoldingContext` — **second pass** (heavier, re-checks the first pass on the main processor; the "holding context" buffers the audio stream across the hand-off).
- `CSVoiceTriggerSecondChanceContext` — a **"second chance"** re-evaluation to recover borderline misses (lower false-reject rate).
- `CSVoiceTriggerAssetHandler` — loads the trigger **model assets** (the `.hmm`/`.bin` files in §6.1).

**Device-type handlers & arbitration**
- `CSDarwinVoiceTriggerHandler` (*this Mac / iOS-class device*) vs `CSGibraltarVoiceTriggerHandler` (*HomePod*).
- `CSMyriadNotifier`, `CSOpportuneSpeakListenerDeviceManager`, `CSTrpTcuMapper` — **"Myriad"** multi-device arbitration (devices that all heard you negotiate who answers).

**False-trigger suppression**
- `CSSelfTriggerDetectorUseAVVCEnabledPolicyASMac` — don't trigger on the device's **own TTS output** ("Siri" spoken by Siri); AVVC = acoustic echo/voice cancellation.
- `CSAttSiriIOSGazeNode`, `CSAttSiriSpeechPresenceASRFeatureExtractor`, `CSAttSiriNLDAClassifierNode`, `CSAttSiriOSDProvider`, `CSAttendingStatesProvidingProxy` — the **attention ("AttSiri")** subsystem: gaze + speech-presence features gate activation.
- `CSShadowMicScoreCreator`, `CSAggressiveECModeHandler` — multi-mic "shadow" scoring and aggressive echo-cancellation modes.

**Speaker ID, endpointing, UX**
- `CSAttSiriInRequestSpkrIdNode`, `SSRVoiceProfileXPCConnection` — in-request **speaker identification** against the enrolled voice profile.
- `CSSpeechEndHostTimeEstimator` — **endpointing** (estimate when the user stopped speaking).
- `CSSmartSiriVolume*` — adapt response volume to ambient noise.

**Testing hooks (useful to us)**
- `CSFileAudioInjectionHearstEngine`, `CSAudioInjectionEngineFactory`, `CSAudioRecorderFactory` — Apple can **inject a WAV file into the first-pass engine** to test detection deterministically. This is exactly the harness pattern NEXUS should copy for wake-word regression tests.

### 6.1 Wake-word model assets (evidence)

**The "Hey Siri" acoustic models** (`…/AssetsV2/com_apple_MobileAsset_UAF_Siri_Understanding/…/*_en_US_heySiri_Siri.hmm`) — three variants (`azki8ryuty`, `gqkcdbvwnb`, `v77km9893p`). Each is an **ASCII HMM topology** for the wake phrase:

```
hs    58   1 1.001  …(40× self-loop)…  2 3.191  3 2.719  …   22 3.665
siri  42   1 1.001  …(30× self-loop)…  11 3.12   12 5.159 …   22 3.665
gsm   1    0 1.001
sil30 30   1 1.001  …(30× self-loop)
```

Read: phonetic units **`hs`** (58 states) + **`siri`** (42 states) + **`gsm`** (garbage/filler) + **`sil30`** (30-state silence), each row = per-state transition index + duration/penalty. This is the classic HMM-over-phrase trigger model.

**On-device speech models** (`CoreSpeech.framework/.../Resources/`):

| File | Size | What it is |
|---|---:|---|
| `sil_pos_gen.nnet` | 1.39 MB | Silence/speech-posterior NN (endpointing front-end) — **Kaldi `nnet1` format** |
| `endpoint_v3.nnet` | 2.1 KB | Endpoint decision net — Kaldi format (`<Nnet><Rescale><AddShift>…`) |
| `RCAccept.nnet` | 981 B | Accept/reject classifier net |
| `hybridendpointer.json` | 2.0 KB | Endpointer front-end config (see below) |
| `adk_model.bin` | 39 KB | Acoustic detection kit blob (+ `.cert`/`.sig`) |
| `inva200.bin` | 320 KB | Acoustic model blob |
| `jbl_begin.bin` | 35 KB | Acoustic model blob |
| `regex.json` | 47 KB | Command/NL regex set |

**Front-end DSP spec** (from `hybridendpointer.json`, model `AM-5.0.0.0`): 16 kHz · **40 mel-filterbank bins** · frame length **25 ms** / shift **10 ms** · Hamming window · HTK-compatible · energy off · splice **left-context 20, right-context 0** → `sil_pos_gen.nnet`.

---

## 7. What this means for NEXUS (actionable)

Mapped against NEXUS's actual wake-word pipeline (`wakeword_oww.rs`: openWakeWord / tract-onnx, `melspectrogram.onnx` → `embedding_model.onnx` → `nexus.onnx`; 1280-sample / 80 ms chunks @ 16 kHz; max-over-12-frame detection + 500 ms RMS≥0.01 confirmation; silence gate RMS<0.0005; AGC ≤50×; ×32768 int16 scaling; 3 s refractory).

1. **NEXUS's "confirmation" is energy-only — Apple's second pass is acoustic. Upgrade it.** Today NEXUS confirms a raw trigger with a 500 ms **RMS** check (`detect_chunk`). Apple's `CSHearstSecondPassRequest` re-runs an *acoustic* model over the **buffered utterance**, not just energy. Recommendation: on a first-pass hit, re-score the retained wake audio through `nexus.onnx` (or a larger second-pass model) at a stricter threshold. This is the highest-leverage change — it directly attacks the Intel-SST false-trigger/phantom-cascade problem documented in `AGENTS.md` without lowering recall.

2. **Add the pre-trigger ring buffer — it unblocks two known gaps at once.** Apple's `CanRecordPastData` + `…SecondPassAudioStreamHoldingContext` keep a rolling buffer and re-read the phrase after detection. NEXUS's **speaker verification is explicitly unwired** for exactly this missing piece (`voice_profile.rs` is `#[allow(dead_code)]` "until an audio ring buffer is added to retain the wake utterance for embedding extraction" — see Known Limitations). One ~1.5–2 s ring buffer of pre/at-trigger audio feeds **both** the acoustic second pass (#1) **and** speaker-embedding verification. Build it once, close two gaps.

3. **"Second chance" re-check for borderline misses.** `CSVoiceTriggerSecondChanceContext` recovers near-misses. NEXUS's max-over-12-frames already catches single good frames; a cheap second-chance = re-score the ring buffer at a lower threshold when the top frame lands in a "maybe" band (e.g. 0.25–0.35). Raises recall on quiet/whispered "NEXUS" (complements the existing AGC) without raising steady-state false accepts.

4. **Self-trigger suppression during TTS.** `CSSelfTriggerDetector…AVVC` stops Siri waking on its own voice. NEXUS plays Kokoro TTS through the same room; gate wake detection (or hard-mute the classifier) while `isRustTtsPlaying()` / Kokoro is speaking, plus simple echo rejection. Prevents NEXUS waking itself mid-response.

5. **Endpointing as a small speech-posterior NN, not a full VAD.** Apple's `sil_pos_gen.nnet` (Kaldi, speech/silence posteriors) + `hybridendpointer.json` is a tiny, cheap endpointer. The linux port **deferred a VAD gate "until it can be recall-tested"** — a small posterior NN (Apple's exact recipe: 16 kHz, 40 mel, 25/10 ms, 20-frame left context) is the recall-testable, low-cost version of that gate, and would also let NEXUS drop the discarded per-chunk VAD pass cleanly.

6. **Front-end cross-check.** Apple validates the 16 kHz · **40 mel-bin** · 25 ms/10 ms · 20-frame-left-context recipe for a shipping detector. NEXUS inherits openWakeWord's front-end (mel → normalize → `slice[4:80]` → embedding). Worth a one-off A/B of NEXUS's mel config against Apple's parameters — cheapest possible accuracy lever if they diverge.

7. **Formalize a file-injection regression harness.** `CSFileAudioInjectionHearstEngine` feeds WAVs straight into the first pass. NEXUS already has `test_wake_model.py` / `test_live_mic.py`; promote these into a standing injection harness wired to the 2,298-file batch benchmark (92.3% TP / 98.9% TN today) so every model/threshold change is regression-gated. Ties into the ASR-poisoning quarantine work.

8. **Adaptive DSP — already aligned; multi-mic is the future lever.** NEXUS's adaptive mic prober (fan resonance 113.3 Hz → auto high-pass 128.3 Hz, pre-gain, impulsive gate) is conceptually Apple's `CSAggressiveECModeHandler` / `CSSmartSiriVolume` territory. Apple's `CSShadowMicScoreCreator` (multi-mic "shadow" scoring) is the next step **if** NEXUS ever targets multi-mic arrays.

9. **Later levers (not now):** attention/gaze gating (`CSAttSiri*`, needs camera) for false-alarm suppression on screen-visible devices; multi-device arbitration (`CSMyriadNotifier`) only if NEXUS ever runs on several co-located devices.

> **Priority order for NEXUS:** (1) acoustic second pass + (2) pre-trigger ring buffer first — they share one buffer, unblock speaker verification, and hit the known false-trigger problem head-on. Then (3) second-chance and (4) TTS self-trigger gating. (5)/(6) endpointing + front-end cross-check are medium-term.

---

## 8. Limitations

- **No decompilation.** No Hopper/Ghidra/IDA installed → no function pseudocode, no recovered *thresholds/scores/timings*, no control-flow. We have the **architecture and interfaces**, not the decision math. Installing Hopper (REA can, with approval) would enable `decompile`/`function`/`trace` to recover the actual first-pass scoring and threshold logic.
- **Framework bodies not extracted.** `VoiceTrigger`, `SpeechDetector`, `EmbeddedAcousticRecognition`, `SpeakerRecognition` live in the dyld shared cache; we read the daemon that links them, not their internals. Extracting slices from the cache is a follow-up.
- **No dynamic analysis.** SIP on; no LLDB/Frida against a system daemon. All findings are static.
- **Model weights not decoded.** `.bin`/`.nnet` blob *formats* identified (Kaldi nnet1, acoustic blobs); weights not parsed.
- **Codenames are interpretation** (Hearst/Remora/Myriad/Gibraltar/Darwin), as flagged in §3.

---

## 9. Second target — `hitodama.in` (corrected)

The web target was first mis-typed as `hitadoma.in` (transposed vowels — that spelling is unregistered). The correct domain is **`hitodama.in`**, which **is live**: *"Hitodama AI — the voice assistant for Windows,"* a **direct NEXUS competitor** (Windows voice assistant, BYO free Gemini key, ₹499). Next.js on Cloudflare.

This is a competitive-intel target, not part of the Hey Siri RE, so it's written up separately:
➡ **`research/2026-10-competitive-audit/10-hitodama-direct-competitor-2026-11-06.md`** (feature/pricing/stack teardown + NEXUS head-to-head and gap list).

---

## 11. Follow-up: Windows REA pass (planned)

A parallel REA pass on **Windows** is requested (assigned to `@ChitkulLakshya`) to mirror this analysis on that platform — e.g. the Windows voice/wake stack and/or the NEXUS Windows build (PE binaries via REA's `inspect-managed-artifact` / `inspect-pe-imports` / `inspect-native-dispatch-metadata`, etc.). Findings to be appended here or in a sibling doc when available. Tracking issue: *(to be linked)*.

---

## 10. Reproduction

```bash
# REA (no install; engine-free static mode). Node ≥ 26.12.1, Xcode CLT present.
BIN=/System/Library/PrivateFrameworks/CoreSpeech.framework/corespeechd

npx -y rea-agents@latest doctor
npx -y rea-agents@latest inspect-macho "$BIN" --format json
npx -y rea-agents@latest inspect-signature "$BIN" --format json          # entitlements
npx -y rea-agents@latest inspect-native-dispatch-metadata "$BIN" --format json  # Obj-C classes

# Wake-word assets (read-only)
find /System/Library/AssetsV2/com_apple_MobileAsset_UAF_Siri_Understanding -name '*heySiri*.hmm'
cat /System/Library/PrivateFrameworks/CoreSpeech.framework/Versions/Current/Resources/hybridendpointer.json

# hitadoma.in (confirm non-existence)
dig @8.8.8.8 +short hitadoma.in A ; whois hitadoma.in | grep -i 'available\|status'
```

**Evidence IDs (REA):** macho `ev_8952d9e74ec8a96070abe428ac95a72e9d6de82654c52156bd9728ffb7ad6818` · signature `ev_dd3be8f215e9990cdd487d1f36ac6c8a0b5826ad207f8b568b59f6d3706aed3d`. Raw JSON retained in the session scratchpad (`rea-out/corespeechd.{macho,sig,objc}.json`); move into `research/wakeword/evidence/` if we want it version-controlled.

---

*All analysis was static and read-only on first-party hardware for interoperability research. No system files were modified; SIP remained enabled.*
