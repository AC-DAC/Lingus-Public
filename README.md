# Lingus — Talk now, learn as you go. (Public)

Self-hosted Korean ↔ English language tool built for live calls (voice/video) with Korean-speaking family and for structured study between calls. Three tabs: **Listen** (real-time KO→EN transcription), **Speak** (EN→KO composition), and **Study** (spaced curriculum built from real call data).

Production deployment: `lingus.alexchuc.au`

---

## The problem

Every common translation app — Google Translate, Papago, Microsoft Translator — stops transcribing on silence. Natural conversation, especially with an elder speaker, has 3–5 second pauses between thoughts. Every pause kills the transcript and breaks the flow.

A secondary constraint shaped the entire architecture: the target phone runs GrapheneOS, which blocks Google Play Services. The browser-native Web Speech API depends on Google's speech recognition — blocked entirely. That pushed the full STT/translation/TTS stack off-device onto edge compute and ruled out browser-native speech recognition across the whole app.

---

## Architecture

```
┌──────────────────────────────────┐
│  Mac (BlackHole virtual audio)   │
│  captures call audio digitally   │
└────────────┬─────────────────────┘
             │ 1. clean audio, no re-recording
             ▼
┌──────────────────────────────────┐
│  Browser — React SPA             │
│  AudioWorklet · silence-boundary │
│  VAD segments per utterance      │
└────────────┬─────────────────────┘
             │ 2. WAV per utterance
             ▼
┌──────────────────────────────────┐
│  Cloudflare Workers AI           │
│  Whisper large-v3-turbo → STT   │
│  Gemma 4 → translation           │
│  (both directions, 5-turn        │
│  rolling context window)         │
└────────────┬─────────────────────┘
             │ 3. static build + Pi-side persistence
             ▼
┌──────────────────────────────────┐
│  Pi 4B (self-hosted)             │
│  nginx · Cloudflare Tunnel       │
│  studystore service (Python)     │
│  (no inbound ports required)     │
└──────────────────────────────────┘
```

**Architecture non-negotiables:** desktop browser for live calls; phone for study. The Pi serves the static React build and the Study/Speak persistence store only — STT and translation go directly from the browser to Cloudflare Workers AI and Azure. The Pi never touches live call traffic.

**Azure Speech** handles TTS and pronunciation scoring in the Study tab. It is scoped there only — Workers AI handles all translation and STT for live calls.

---

## Listen — KO → EN

Real-time Korean speech to English transcription during a live call (voice/video). Audio is captured digitally via BlackHole (no mic re-recording) and segmented by silence boundaries via AudioWorklet VAD — utterances are not cut mid-sentence regardless of pause length. Each segment is sent to Whisper large-v3-turbo with a forced `ko` language hint, then to Gemma 4 with a 5-turn rolling context window for coherent pronoun resolution across turns.

The live pipeline health rail shows backend connectivity, transcription and translation latency, and confidence score per utterance in real time.

![Listen tab — live Korean transcription with pipeline health rail](assets/screenshots/listen-1.png)

**Key implementation decisions:**

- **AudioWorklet replaced MediaRecorder.** Whisper rejects WebM outright. Fixed-interval chunking cut Korean's SOV sentences mid-utterance — because the verb comes last, cutting at four seconds loses the entire meaning. AudioWorklet with silence-boundary VAD sends one complete utterance per request.
- **Transcript lines ordered by capture time, not STT arrival** — prevents out-of-order display when network latency varies between concurrent requests. Each segment is timestamped at speech onset and inserted by capture time, not arrival order.
- **Gemma 4 keep-warm heartbeat** runs every 45 seconds during live calls to minimise cold-start latency. Gemma cold-starts at 10–40s; warm it runs at ~0.5s. Whisper latency variance is Workers AI queueing — warming does not help and was tried, measured, and reverted.
- **Rolling context window (5 turns)** — stateless translators guess wrong on homophones and idioms without conversational context. Korean has many characters with multiple meanings depending on what preceded them.
- **Gemma 4 over Llama — benchmark-driven, not assumed.** Llama introduced subject-pronoun errors on Korean under concurrent load: a sentence meaning "I feel the cold" was translated as "You feel the cold." Gemma 4 handled 8 parallel requests at 750ms with zero failures in a proper concurrent benchmark. The initial Gemma → Llama → Gemma reversion arc: Gemma queued catastrophically in July 2026 (6–78s under 8 concurrent requests) and was temporarily replaced by Llama on the live path. An August re-benchmark found the issue did not reproduce; Llama's Korean accuracy problems were the deciding factor to revert. Lesson: benchmark concurrency, not just sequential warm calls.
- **Client-side idiom dictionary** — detects known idioms and proverbs and flags their actual meaning regardless of which STT model produced the text, even when the translation is technically correct but culturally wrong. Matches on pattern, not perfect text, so it catches STT errors on known phrases.
- **Deterministic Korean numeral parser** — dates, times, and money amounts are computed by code, not the model. Gemma 4 produced consistent numeral errors (wrong month, wrong denomination) that a parser eliminates entirely. Whisper writes numbers as digits or words depending on context; the parser handles both. A prompt nudge to fix this was tested and failed — it is a model capability limit, not a wording problem. The rule: the LLM does language; code does the math.
- **Live confidence flagging** — Whisper's per-segment `avg_logprob` and `no_speech_prob` are computed into a 0–1 confidence score. Lines below threshold render with an amber tint and a prompt to ask for repetition. Flags mumbled or noisy audio; does not catch fluent hallucinations.
- **Speaker diarization prototyped and removed** — complexity did not survive real-call validation. Pause/Unpause replaced it.
- **Whisper prompt biasing tested and removed** — introduced echo regression.

---

## Speak — EN ⇄ KO

Bidirectional composition panel for practising Korean in a live or in-person conversation. Left column: type English, Gemma 4 translates to Korean with formal/casual tone toggle (반말/존댓말), with per-entry sentence breakdown. Right column: the other speaker's Korean side, transcribed and translated to English in real time.

Conversation history is persisted server-side on the Pi. Browser localStorage was evicted by Brave's aggressive memory management under load — server-side persistence was the correct fix.

![Speak tab — bidirectional EN/KO composition](assets/screenshots/speak-1.png)

---

## Study — structured curriculum from real call data

A structured Korean curriculum across five stages, each introducing new grammar or vocabulary before gating the next. Study data is persisted server-side on the Pi via a small Python stdlib service (`studystore`) for cross-device access — a design decision forced by the same localStorage eviction issue discovered in Speak. Real conversation phrases can also be imported from KakaoTalk chat exports into the phrasebook.

### Your Path

Staged curriculum across five stages, each gating on completion before the next opens:

| Stage | Topic |
|---|---|
| 0 | Staying in Korean |
| 1 | Making Your Own Sentences |
| 2 | Past & Future Tense |
| 3 | Negatives & Casual Speech |
| 4 | Baby Care Vocabulary |
| 5 | More to come ... |

Curriculum content for 반말/존댓말 (casual/formal speech registers) verified by a native speaker.

![Study — Your Path with staged curriculum](assets/screenshots/study-1.png)

### Phrases

Browsable phrasebook with formal/casual display toggle, per-phrase meaning and grammar breakdown, TTS playback at 1.0× and 0.6× slow, male/female voice selection. New phrases added via a Gemma 4-powered wizard: search a missed phrase, translate and confirm, hear it, edit the Korean if needed, save.

![Study — Phrases with meaning & grammar breakdown](assets/screenshots/study-2.png)
![Study — New Phrase wizard (Gemma 4 EN→KO translation)](assets/screenshots/study-3.png)

### Write it

Tile-arrangement practice. Given an English sentence, arrange Korean word tiles into correct SOV order. Targets the structural difference between English and Korean that causes the most spoken errors. Single-word phrases are tiled by syllable block so every phrase in the curriculum is reachable.

![Study — Write it tile-arrangement practice](assets/screenshots/study-4.png)

### Say it

Pronunciation practice via Azure Speech. Hear the phrase at native speed or 0.6× slow (female or male voice), record, and submit. Azure scores accuracy, fluency, completeness, and effort against the native model, with side-by-side waveform comparison.

![Study — Say it pre-attempt](assets/screenshots/study-5.png)
![Study — Say it scored result (98% accuracy)](assets/screenshots/study-6.png)

### Checkpoint

Interleaved recall test across all completed stages. 12 mixed questions — recall, word order, and spoken — no hints, no timer, no pressure. Quit any time and the attempt is discarded with nothing written to mastery scores. Only finished stages are sampled so anything actively being drilled stays out of it. No gamification, no streaks, no daily cadence pressure — the app is built around real conversations that happen when they happen, not a habit loop.

![Study — Checkpoint pre-start screen](assets/screenshots/study-7.png)

---

## Infrastructure

| Component | Implementation |
|---|---|
| Hosting | Raspberry Pi 4B, Debian Trixie ARM64 |
| Web server | nginx — serves the static React build |
| Study/Speak persistence | `studystore` — Python stdlib service on the Pi, persists study progress and speak history as JSON over a local HTTP API |
| External access | Cloudflare Tunnel — no inbound ports; ISP blocks inbound 80 and 443 on the residential plan |
| TLS | Let's Encrypt via Certbot DNS-01 challenge (Cloudflare plugin) |
| Access control | Cloudflare Access — email-allowlisted PIN gate on the subdomain |
| API key handling | Workers AI and Azure keys live only in the Worker's environment — never in the browser |
| nginx rate limiting | Backstop on `/transcribe` and `/translate` endpoints |
| nginx caching | `index.html` served `no-cache` (SPA entry, always revalidated on deploy); `/assets/` served `immutable` with a 1-year TTL (content-hashed) |

**Rejected alternatives:**

- **Port forwarding** — ISP blocked inbound ports on the residential plan. Cloudflare Tunnel eliminated the dependency entirely.
- **HTTP-01 certificate challenge** — the admin subdomain is intentionally internet-unreachable. DNS-01 was the only viable option.
- **Browser localStorage for Speak and Study** — Brave evicts aggressively under memory pressure. Moved to Pi-side persistence.
- **Cloudflare KV for persistence** — viable, but the Pi already hosts the app. Keeping storage local keeps the architecture simple and avoids a second external dependency.
- **Express/Google Translate proxy** — removed in favour of direct Workers AI calls from the browser. Eliminated a maintenance surface with no benefit.
- **Speaker diarization** — prototyped, rejected after real-call validation.
- **Naver CLOVA** — Naver Cloud Platform does not support account registration from Australia.
- **SenseVoice (Alibaba)** — self-hosted GPU model, no native streaming, Korean accuracy unproven.
- **AWS Lambda for STT** — no GPU; Whisper on CPU with cold starts would be slower, not faster. Hardware is not the latency lever; streaming STT is.

---

## Security

- Cloudflare Access PIN gate — email allowlist enforced; no password stored
- API keys in Worker environment only — never exposed to the browser or bundled into the static build
- nginx rate limiting on transcription and translation endpoints
- Cloudflare Tunnel — no open inbound ports on the Pi or router
- robots.txt + noindex meta — app is not publicly indexed

---

## Stack

`React` `Cloudflare Workers AI` `Whisper large-v3-turbo` `Gemma 4` `Azure Speech` `Python` `nginx` `Cloudflare Tunnel` `Cloudflare Access` `Let's Encrypt` `systemd` `Raspberry Pi`
