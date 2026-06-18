# Otto

A privacy-first, on-device personal assistant for **Android (first)** and **iOS (later)**. It connects to your apps/accounts, keeps a **local, normalized index** of that data, and answers questions using **deterministic code + on-device NLU (no LLM)**.

## Status

Early planning. The full blueprint — architecture, connector feasibility, voice/battery strategy, the "no-LLM" approach, cross-platform strategy, roadmap, and open decisions — lives in [`otto.md`](./otto.md).

Phase 1 is gated on the four Phase-0 decisions and their spike exit criteria (see `§13.5` in [`otto.md`](./otto.md)):

1. **Cloud vs zero-cloud** — push requires a server; resolve first.
2. **Proper-noun voice strategy** — dynamic contact-name recognition feasibility.
3. **Notification connector status** — core vs opt-in vs cut (Play policy).
4. **Legal posture on third-party data** — go / constrain / no-go.
