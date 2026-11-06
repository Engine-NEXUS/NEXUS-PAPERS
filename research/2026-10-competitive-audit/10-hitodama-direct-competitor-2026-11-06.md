# Hitodama — Direct Competitor Teardown (hitodama.in)

**Date:** 2026-11-06
**Target:** https://hitodama.in/ — "Hitodama AI: the voice assistant for Windows"
**Why this matters:** This is the **closest direct competitor to NEXUS seen so far** — a voice-first Windows assistant that acts on your real computer, local-first-ish, BYO free Gemini key, sold to the same India-first, cost-sensitive audience at ₹499. Same lane, same wedge.
**Method:** Live page review + HTTP/stack fingerprint (read-only, public site). Deep REA bundle/app analysis not yet run — see [§8](#8-recommended-next-recon).

---

## 1. One-line

> **Hitodama** ("人魂", Japanese *soul-fire*) — *"Give your computer a soul. The voice AI that runs your computer for you."* Windows-only. By **Mainak Deb Das / "Otism"** (`support@hitodama.in`, © 2026). Ships now; ₹499 one-time Founders Edition running on the user's **own free Google Gemini API key**.

Tagline in `<meta>`: *"A voice-first Windows companion that researches, remembers, and acts across your real computer."* — this is almost verbatim NEXUS's own pitch.

---

## 2. Product surface (what they advertise)

| Feature | Hotkey | What it does | NEXUS equivalent |
|---|---|---|---|
| **Circle to Ask** | `Ctrl+Shift` | Circle anything on screen, ask about it aloud (screen understanding / visual Q&A) | **Gap** — NEXUS has no screen-region "circle to ask"; closest is the Architect/screen-agent stack |
| **Dictation** | `Ctrl+Win` | Hold-to-talk voice typing anywhere in Windows; cleans disfluencies ("uh so nine… no, ten?" → "Let's meet at 10"); clean text drops at cursor | **Live Mode / Ghostwriter dictation** (`live/`, clipboard-paste pattern) — parity, arguably NEXUS is ahead here |
| **To-do list** | — | Voice-managed task list | Partial (no first-class to-do) |
| **Scheduled search** | — | "Search Amazon for gift wraps on 10 Nov at 11am" → opens amazon.in at that time | **Routines/Command Center** can compose, but time-scheduled triggers are a **gap** |
| **Routine** | — | "Every morning at 9:00, open Unity and Spotify" | Command Center does compound ("X then Y"); **recurring/time-based = gap** |
| **Search** | — | "Search how to cook spaghetti on YouTube" | Parity (deterministic + 9Router) |
| **Control Centre ("island")** | scroll on island | Dynamic-Island-style hub: music, settings, apps in one place | NEXUS has the **orb/aura + sidebar**; no "island/Control Centre" metaphor |
| **5-slot "Loadout / Soul"** | — | Customize five slots to personalize the assistant ("make your soul yours") | **Gap** — NEXUS has no user-facing modular "loadout" concept |
| **Typing-speed demo** | — | Interactive "race it": saying 220 wpm vs typing 35 wpm → **6.3× faster** | Marketing device NEXUS lacks |

**Design/brand:** heavy "soul" theme (人魂/魂, "give your computer a soul"), Japanese kanji accents, Dynamic-Island UI language, polished interactive landing page. Strong, coherent brand identity — stronger *marketing* polish than NEXUS currently presents.

---

## 3. Business model & positioning

- **Founders Edition — ₹499 one-time.** No monthly fee; every Founders update free; **runs on the user's own free Gemini API key**; "you keep it even after it stops selling." Classic **scarcity wedge** (sold only until the monthly Pass launches).
- **Hitodama Pass — monthly, "coming later."** Pitched as *stronger* because it won't use free-tier keys → faster/more efficient. So the roadmap is: hook with cheap one-time BYO-key edition, upsell to a hosted monthly later.
- **Cost story:** "$0.00 this month / $0.00 next month — FREE." They lean *hard* on "no subscription, no usage bill" by keeping everyday use inside Gemini's free tier. Billing only if the user opts in with Google.
- **Price point ₹499** = deliberately India-first, impulse-buy pricing. Same audience NEXUS targets (Hindi/Telugu work, amazon.in/Swiggy/WhatsApp intents).

---

## 4. Technical signals

**Marketing site:** Next.js (App Router) + React, static export on **Cloudflare** (CF nameservers `elma/isaac.ns.cloudflare.com`, IPs `104.21.75.51` / `172.67.214.108`, `server: cloudflare`). `_next/static/chunks/app/{layout,page,not-found}` present. HTTP/2, `x-content-type-options: nosniff`, `permissions-policy: camera=(), geolocation=(), microphone=()` on the site itself.

**The app (from their own copy):**
- **"Gemini supplies the words. The Windows tools, memory, routines and safety checks that let it act on your PC are Hitodama's own."** They pre-empt the "just a Gemini wrapper?" question — so the moat they *claim* is the Windows action layer + memory + routines + **safety checks**, not the model.
- **BYO key**, "✓ Saved on this PC only" → key stored locally; app talks to Google directly. Implies a **local-first client** (like NEXUS) with the LLM in the cloud (Gemini free tier) — **no cloud backend of their own in the free path** (good for their margins, same as NEXUS's 9Router/Worker-optional stance).
- **Single LLM dependency (Gemini).** No mention of local STT, local wake-word, or on-device NLU. This is the **key architectural difference** (see §5).

---

## 5. Hitodama vs NEXUS — honest head-to-head

| Dimension | Hitodama | NEXUS |
|---|---|---|
| Platform | **Windows only** | Windows + **macOS + Linux** (port done) — NEXUS is broader |
| Brain | **Gemini only** (cloud, user's free key) | **9Router cascade** (Cerebras→Groq→Gemini→Worker) + local BERT-Mini NLU + optional Qwen local brain — NEXUS is more resilient/local |
| STT | not stated (likely cloud/Gemini) | **Local Moonshine** + Groq cloud, privacy mode — NEXUS clearly ahead on local STT |
| Wake word | not advertised (hotkey-driven) | **On-device openWakeWord** ("NEXUS") — NEXUS has a true wake word; Hitodama appears **hotkey-first** |
| Screen "circle to ask" | **Yes, flagship** | **Gap** |
| Time-scheduled routines | **Yes, flagship** | **Gap** (Command Center is compound, not scheduled) |
| Dictation | Yes | Yes (Live Mode/Ghostwriter) — parity |
| Commerce/MCP | Amazon via scheduled search | **MCP sub-center** (Swiggy/Amazon/WhatsApp, OAuth) — NEXUS deeper |
| Pricing | ₹499 one-time + future monthly | (NEXUS model TBD) — **Hitodama has a shipped, concrete monetization** |
| Brand/marketing | **Very polished** ("soul", island, interactive demos) | Functional; **NEXUS marketing is behind** |

**Where NEXUS wins:** cross-platform, local STT + on-device wake word, model resilience (not single-vendor), deeper MCP/commerce + confirmation-gated safety, on-device NLU.
**Where Hitodama wins / NEXUS gaps:** (1) **Circle-to-ask screen understanding**, (2) **time-scheduled routines**, (3) a **shipped pricing model + brand**, (4) a crisp "island/Control Centre" UX metaphor, (5) a **5-slot personalization** hook.

---

## 6. Strategic takeaways for NEXUS

1. **Close the two feature gaps that are their headline:** *circle-to-ask* (screen-region capture → VLM Q&A) and *time-scheduled routines* ("do X at 9am / on 10 Nov"). These are their two most differentiated demos and both are achievable on NEXUS's existing Command Center + screen-agent stack.
2. **Lean into what they can't easily match:** cross-platform (they're Windows-only), **true on-device wake word + local STT** (privacy + offline), and **multi-model resilience** (they're 100% Gemini — a single point of failure/policy risk). Make "works offline / not locked to one AI vendor" a headline.
3. **Their BYO-free-key model validates NEXUS's cost thesis.** ₹499 one-time + "runs on your own free key, $0 bill" is exactly NEXUS's 9Router free-tier logic. NEXUS can match the "$0 running cost" story *and* add "even works when the cloud is down."
4. **Marketing/brand is a real gap.** Hitodama's "give your computer a soul," kanji identity, interactive typing-race and island demos are doing heavy lifting. NEXUS needs a comparable landing page + a memorable identity (the orb/aura is a strong visual asset already — use it).
5. **Watch the "safety checks" claim.** They explicitly market safety checks as part of the moat. NEXUS already has confirmation-gated destructive actions (WhatsApp send, GitHub merge, MCP writes) — make that visible as a selling point too.

---

## 7. Facts worth tracking

- **People:** Mainak Deb Das, brand "Otism." Contact `support@hitodama.in`, a YouTube presence (setup video "4:17"), grievances/refunds/licence pages.
- **Stage:** shipping, Founders Edition on sale now; monthly Pass unannounced-date. The "sold until the Pass launches" scarcity is live.
- **Trademark note on their page:** disclaims affiliation with Google/Gemini/Windows/Amazon/Netflix/Spotify/YouTube — i.e. they integrate/drive those apps, same surface as NEXUS.

---

## 8. Recommended next recon

1. **REA bundle analysis of the live site** (`list-browser-targets` → `analyze-web-bundle` via the installed Brave/Chromium): recover their Next.js routes, any pricing/checkout/analytics endpoints, Gemini-key handling hints, and the real feature list behind the marketing.
2. **RE the Windows app itself** (the planned `@ChitkulLakshya` Windows REA pass): what Windows automation APIs it uses (UIA/Win32), how it stores the key, how "memory/routines/safety checks" actually work, and whether there's any backend call-home beyond Google. This is the highest-value competitive intel and is exactly what a Windows REA run is for.
3. Pull the setup YouTube video + licence/privacy/refund pages for the full monetization and data-handling picture.

> **Note:** an earlier pass mis-typed the domain as `hitadoma.in` (transposed vowels), which is unregistered. The correct target is **`hitodama.in`**.
