# Otto

A privacy-first, on-device personal assistant for **Android (first)** and **iOS (later)**. It connects to your apps/accounts, keeps a **local, normalized index** of that data, and answers its bounded intents with a **deterministic, on-device, no-LLM core** — plus an **optional, opt-in "Assistant mode"** that reasons over the index using **your own** cloud LLM key (Claude/GPT/Gemini) for open-ended requests.

## Status

Early planning. The full blueprint — architecture, connector feasibility, voice/battery strategy, the "no-LLM" approach, cross-platform strategy, roadmap, and open decisions — lives in [`otto.md`](./otto.md).

Phase 1 is gated on the Phase-0 decisions and their spike exit criteria (see `§13.5` in [`otto.md`](./otto.md)) — **all resolved**:

1. **Cloud vs zero-cloud** — push requires a server; resolve first. → **A, pure on-device.**
2. **Proper-noun voice strategy** — dynamic contact-name recognition feasibility. → **Ship, via STT + phonetic match.**
3. **Notification connector status** — core vs opt-in vs cut (Play policy). → **Opt-in, promoted, on-device.**
4. **Legal posture on third-party data** — go / constrain / no-go. → **GO with constraints.**
5. **LLM posture** — bounded no-LLM vs open-ended. → **Hybrid: no-LLM core + opt-in bring-your-own-key Assistant mode.**
