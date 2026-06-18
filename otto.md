# Personal Assistant App — Plan, Specs & Design (v0.1 draft)

> **Working name:** (TBD) — referred to as **the App** below.
> **Scope of this document:** architecture, subsystems, connector feasibility, voice/battery strategy, the "no-LLM" approach, cross-platform strategy, phased roadmap, risks, and open decisions. No implementation yet — this is the blueprint to react to.

---

## 1. Product summary

A privacy-first, on-device personal assistant for **Android (first)** and **iOS (later)**. It connects to your apps/accounts, keeps a **local, normalized index** of that data, and answers questions using **deterministic code + on-device NLU (no LLM)**. Two ways to operate it:

- **Launch + tap-to-talk / type** (primary, reliable, battery-cheap)
- **Custom voice wake word** (secondary; Android only for true always-on; iOS via Siri hand-off)

The design centers on three goals you stated: **low battery, fast response, no LLM.** All three point to the same architecture: *pre-sync data in the background, query a local index, recognize a bounded set of intents on-device.*

---

## 2. Guiding design principles

| Principle | Why | Consequence |
|---|---|---|
| **Local-first** | Speed, privacy, offline, battery | A normalized on-device index is the source of truth for queries; connectors sync *into* it |
| **Answer from index, not live fetch** | "Operation won't take too long" | Queries hit local SQLite, not 6 network APIs in series |
| **Bounded intents, on-device** | "No LLM" + battery + determinism | Speech-to-intent grammar; an explicit "I can't answer that yet" path |
| **Android-first** | Deeper system integration is possible | iOS gets a reduced-capability variant, not parity |
| **Connectors are pluggable** | Apps come and go; APIs change | Common adapter interface; add/remove without touching the core |
| **Honest capability surface** | Some apps simply can't be read | Don't promise WhatsApp history; promise what's actually deliverable |

---

## 3. The three hard constraints (reality check)

### 3.1 Voice wake word + low battery

- True always-on hot-word detection ("Hey Google") runs on a **dedicated low-power audio DSP** that the OS reserves for its own assistant. **Third-party apps cannot use it.**
- **Android:** you *can* run a custom wake word via a **foreground service** + an efficient engine. This consumes CPU continuously, so battery management is a real engineering task, not a setting.
- **iOS:** no continuous background mic for third parties. Realistic options: **App Intents / SiriKit** ("Hey Siri, ask *the App*…") or **foreground-only** listening while the app is open.

**Recommended model:**
- **Primary trigger:** open app + tap-to-talk, plus OS-assistant hand-off (Siri/Assistant → opens the App with the query). Reliable, near-zero idle battery.
- **Secondary (Android only):** opt-in always-on custom wake word, with the battery mitigations in §6.
- **Set expectations:** iOS will not have a "from any state, hands-free" custom phrase. Communicate this early.

### 3.2 "No LLM"

Fully workable for a **defined intent set**. Use **on-device speech-to-intent** (audio → intent + slots within a grammar) rather than full transcription + reasoning. Benefits: ~instant, offline, private, deterministic, battery-light. Cost: it only answers what's in the grammar. **Treat the intent catalog as the product roadmap** (§7). Keep an explicit fallback for unrecognized requests.

> Optional later: a *small* on-device classifier (not an LLM) to improve paraphrase tolerance. Still no cloud, still no generative model.

### 3.3 Connectors

Most value comes from **email, calendar, Slack, notes (some), contacts**. Several requested apps are **closed** for personal-data reads. See the feasibility matrix in §8. The Android **NotificationListenerService** is the universal fallback that partially rescues WhatsApp/IG/etc. — but as a *real-time notification stream*, not history.

---

## 4. System architecture

```
            ┌─────────────────────────────────────────────┐
            │                 VOICE LAYER                  │
            │  Wake word → Speech-to-Intent (on-device)    │
            │  (Android: always-on opt-in; iOS: Siri/fg)   │
            └───────────────┬─────────────────────────────┘
                            │ intent + slots
            ┌───────────────▼─────────────────────────────┐
            │            QUERY PLANNER / ROUTER            │
            │  intent → data sources, entity resolution    │
            │  (resolve "Avital" → contact; "tomorrow"→date)│
            └───────────────┬─────────────────────────────┘
                            │ structured query
            ┌───────────────▼─────────────────────────────┐
            │        LOCAL INDEX (source of truth)         │
            │  SQLite/Room • normalized entities • FTS      │
            │  messages, events, notes, contacts, notifs    │
            └───────────────▲─────────────────────────────┘
                            │ background sync (incremental)
            ┌───────────────┴─────────────────────────────┐
            │            CONNECTOR FRAMEWORK               │
            │  pluggable adapters: auth • sync • normalize  │
            │  Gmail | Graph | Calendar | Slack | Notes |…  │
            │  + Android NotificationListener (universal)   │
            └───────────────┬─────────────────────────────┘
                            │ answer payload
            ┌───────────────▼─────────────────────────────┐
            │              RESPONSE LAYER                  │
            │  TTS (on-device) + on-screen card/result     │
            └─────────────────────────────────────────────┘
```

**Layer responsibilities**

1. **Voice layer** — wake-word detection, then on-device speech-to-intent. No audio leaves the device.
2. **Query planner/router** — maps `(intent, slots)` to the data sources needed, resolves entities (people, dates, channels), composes the query, ranks/formats the answer.
3. **Local index** — normalized, full-text-searchable store. *This is what queries actually read.* Latency is local-disk, not network.
4. **Connector framework** — adapters that authenticate, sync incrementally into the index, and normalize each app's schema to common entity types.
5. **Sync engine** — background workers + push where available; respects battery/network constraints.
6. **Response layer** — TTS + a visual result card (always show the answer on screen too).

---

## 5. Data & sync layer (the key to "fast" + "low battery")

- **Normalized entity model** across sources: `Message`, `Event`, `Note`, `Contact`, `Notification`, `Task`. Each carries `source`, `timestamp`, `participants`, `body`, `links`.
- **Full-text search** (SQLite FTS5 / equivalent) for "did anyone email me about X".
- **Sync strategy (battery-aware):**
  - **Incremental/delta sync** only (history IDs, change tokens, `updatedTime` cursors).
  - **Push where possible** (e.g., Gmail push via Pub/Sub, Graph subscriptions) so you don't poll.
  - **Batch + opportunistic** scheduling: prefer Wi-Fi + charging windows for heavy backfills; light deltas on a throttled cadence.
  - **Android:** `WorkManager` with constraints. **iOS:** `BGAppRefreshTask` / `BGProcessingTask` (OS-budgeted — expect coarser freshness).
- **Freshness vs latency vs battery tradeoff:**
  - Default: **answer from index** (fast, cheap).
  - For time-critical intents (e.g., "any new mail in the last 5 min from my boss?") allow a **targeted live refresh** of *one* source before answering.
- **Retention/limits:** cap stored history (e.g., 90 days configurable), per-source quotas, user-purgeable.

---

## 6. Voice & wake-word subsystem

**Pipeline:** `Mic → VAD → Wake word → { bounded intent: Speech-to-Intent | proper-noun intent: STT → phonetic contact match } → router`

> **Split by intent type (see §13.5 D2):** bounded, no-proper-noun commands go through **Rhino speech-to-intent** (instant, battery-light). Queries with a **contact name** route to **on-device STT + phonetic matching** against the contact list, because a fixed grammar can't hold an open, per-user, mutable vocabulary. The always-on path stays Rhino-only; STT only runs for tap-to-talk / post-wake-word name queries.

**Battery mitigations (Android always-on):**
- **Cascade detection:** cheap **VAD** (is there *any* speech?) gates the wake-word model so it isn't scoring silence.
- Small wake-word model, **16 kHz mono**, short frames, native/optimized engine.
- Foreground service with a persistent (honest) notification; user opt-in toggle.
- Optional conditions: only always-listen while charging / on a chosen Wi-Fi / certain hours.
- Measure power draw as a first-class PoC metric (§13).

**Engine options (no LLM, on-device):**

| Function | Candidate | Notes |
|---|---|---|
| Wake word | **Picovoice Porcupine** / openWakeWord | Efficient, custom phrase, on-device |
| **Speech → intent** (bounded) | **Picovoice Rhino** | Audio → intent+slots in a defined grammar; *skips transcription* — ideal for "no LLM" + battery. **Fixed context compiled ahead of time; cannot hold live contact names** (see §13.5 D2) |
| NLU on text | **Snips NLU** (on-device) | Intent + slot filling on transcribed text, no cloud, no LLM |
| **STT (proper-noun queries)** | Android `SpeechRecognizer` (offline) / iOS `Speech` / Vosk | **Required for contact-name slots** — transcribe, then phonetically match to contacts. Also any free-text capture |
| Phonetic name match | Double Metaphone / Jaro–Winkler (+ phoneme model for non-Latin) | Match spoken name → on-device contact list; disambiguate close candidates |
| TTS | Android `TextToSpeech` / iOS `AVSpeechSynthesizer` | On-device output |

> Verify licensing/pricing of any third-party engine (Picovoice has per-app tiers) during validation.

**iOS specifics:** ship **App Intents / SiriKit** phrases so users invoke via Siri; in-app foreground tap-to-talk otherwise. No custom background hot-word.

---

## 7. NLU / intent engine ("no-LLM" done properly)

**Approach:** a curated **intent catalog**, each intent = grammar + required/optional slots + a handler that queries the index. Entity resolution turns slot text into concrete IDs.

**Example intents (starter set):**

| Intent | Sample utterance | Slots | Sources |
|---|---|---|---|
| `next_meeting` | "When's my next meeting?" | (person?, day?) | Calendar |
| `meeting_with_person` | "When do I meet Avital?" | person | Calendar + Contacts |
| `unread_from` | "Any unread mail from my boss?" | person/label, timeframe | Email |
| `search_messages` | "Did anyone message me about the release?" | keyword, timeframe | Email, Slack, Notifications |
| `last_message_from` | "What did Dominik last send me?" | person | Slack, Email, Notifications |
| `notes_about` | "Find my notes about the workplan" | keyword | Notes |
| `today_agenda` | "What's on today?" | — | Calendar + Tasks |
| `notifications_recap` | "What did I miss?" | timeframe | NotificationListener feed |

**Entity resolution:**
- **People:** resolve names → **Contacts** (on-device) via **phonetic + edit-distance matching** (the contact list is the candidate set), so spoken/typed "Avital", "Dawid", "Dominik" map to the nearest-sounding contact; handle ambiguity ("which Dawid?") with a quick disambiguation prompt. For voice, the name arrives as **STT text** (not a Rhino slot) — see §13.5 D2. Typed queries skip STT and match text directly, so person-name intents work even if voice STT is weak.
- **Time:** on-device date/time parser ("tomorrow", "last week", "after 3pm").
- **Channels/labels:** map "the release channel" → Slack channel ID via synced metadata.

**Fallback contract:** if no intent matches confidently → say so and offer the closest supported action ("I can't search WhatsApp history, but I can recap recent WhatsApp notifications — want that?"). This keeps trust intact and is your roadmap signal.

---

## 8. Connector framework & feasibility matrix

**Adapter interface (every connector implements):** `authenticate()` · `syncDelta()` · `normalize()` · `capabilities()` · `revoke()`.

**Failure, limit & lifecycle semantics (every connector must declare and handle):**

| Concern | Contract |
|---|---|
| **Rate limits** | Each adapter exposes its known limit class and honors server backoff (`Retry-After`, 429/503). Sync engine applies **per-connector token-bucket + exponential backoff with jitter**; never global-stall other connectors. |
| **Partial / degraded sync** | `syncDelta()` returns a **resumable cursor** + status (`complete` / `partial` / `rate_limited` / `failed`). Partial syncs persist progress and resume; the index records per-source `lastSyncedAt` + `freshnessState`. |
| **Token expiry / revocation** | Distinguish *recoverable* (refresh token) from *terminal* (user revoked, scope changed, app uninstalled) failures. Terminal → surface a **re-auth prompt**, mark source `stale`, never silently drop data. |
| **Auth scope drift** | `capabilities()` re-evaluated after each auth; if a scope is lost, the planner degrades affected intents and tells the user rather than returning wrong answers. |
| **Idempotency / dedupe** | `normalize()` emits a stable `sourceEntityId`; re-sync is idempotent (upsert, no duplicates). |
| **Live-refresh path** | Optional `refreshOne(query)` with a hard **timeout budget (≤1.5s default)**; on timeout/error it falls back to indexed data and **labels the answer as possibly stale** — it must never block or fail the response. |
| **Quota exhaustion** | On sustained limit exhaustion, downgrade to a slower cadence and emit a user-visible freshness badge, not an error. |
| **Backoff isolation** | One connector failing/limited must not delay queries answered from the index or other healthy sources. |

**Feasibility (verify at build time — these APIs change):**

| App / source | Read personal data? | Mechanism | Status |
|---|---|---|---|
| **Gmail** | ✅ | Gmail API / IMAP (OAuth) + push | 🟢 Build |
| **Outlook / M365 mail** | ✅ | Microsoft Graph | 🟢 Build |
| **Google Calendar** | ✅ | Calendar Provider (on-device) / API | 🟢 Build |
| **Apple Calendar** (iOS) | ✅ | EventKit (on-device) | 🟢 Build (iOS) |
| **Contacts** | ✅ | Contacts Provider / Contacts framework | 🟢 Build — **critical for entity resolution** |
| **Slack** | ✅ | Slack Web API (OAuth scopes) | 🟢 Build |
| **Notion** | ✅ | Notion API | 🟢 Build |
| **OneNote** | ✅ | Microsoft Graph | 🟢 Build |
| **Evernote** | ⚠️ | Legacy API | 🟡 Optional |
| **Google Keep** | ❌ official | Unofficial only | 🔴 Avoid |
| **Apple Notes** | ❌ | No public API | 🔴 Avoid |
| **WhatsApp** (personal) | ❌ history | Android NotificationListener = incoming notifs only | 🟡 Notifications-only (Android); 🔴 iOS |
| **Facebook** | ❌ feed/messages | Graph locked down | 🔴 Drop |
| **Instagram** (personal) | ❌ | Basic Display deprecated; Graph is business-only | 🔴 Drop (🟡 notifs on Android) |
| **TikTok** | ⚠️ own content only | Login Kit / Display API; no DMs | 🟡 Low value |
| **SMS / Call log** (Android) | ✅ but restricted | Providers + permissions | 🟡 **Heavy Play policy risk** |
| **Android Notifications (any app)** | ✅ incoming | `NotificationListenerService` | 🟢 **Universal fallback channel** |

**Key architectural insight:** instead of chasing per-app APIs that don't exist (WhatsApp/IG/etc.), treat the **Android notification stream as one connector** that ingests *everything* in real time and indexes it. It gives "what did I miss across all my apps" with a single integration — but only **going forward**, not historical, and **Android-only**.

---

## 9. Response layer

- **Always render on screen** (result card) in addition to TTS — voice can be ambiguous, and it serves the launch-mode users.
- **TTS** on-device; short, structured spoken answers ("Your next meeting is at 2pm with Avital").
- **Deep links:** results link back into the source app (open the Slack message, the calendar event).

---

## 10. Privacy, security & store-policy

- **Local-first** is the headline privacy story — audio and indexed data stay on device by default.
- **Token storage:** Android **Keystore**, iOS **Keychain**; encrypted DB at rest (SQLCipher / file protection).
- **Scoped OAuth** per connector; clear consent screens; per-source revoke + purge.
- **Compliance:** GDPR-aligned (data minimization, export, delete). Privacy policy + data-handling doc required.
- **Store-policy landmines (plan for review friction):**
  - **NotificationListener** access is policy-sensitive on Google Play — justify the assistant use case explicitly.
  - **SMS/Call Log** permissions are tightly restricted — likely **defer or drop**.
  - **Accessibility API** for scraping closed apps → **don't**; it gets apps removed and is a privacy hazard.
  - **ToS risk:** programmatically reading WhatsApp/Meta data violates their terms — keep to the notification channel and **get legal sign-off**.

---

## 11. Cross-platform strategy

The heavy parts (background audio, notification listener, content providers, OS scheduling) **must be native**. So:

- **Recommended: Kotlin Multiplatform (KMP)** for the **shared core** — connector framework, local index, query planner, NLU orchestration, sync logic — with **native platform layers** for voice, background services, and system access.
- Alternative: fully native ×2 (more code, max control).
- **Not recommended:** Flutter/React Native as the base — you'd still write all the system-integration natively, losing most of the cross-platform benefit while adding a bridge.

---

## 12. Suggested tech stack (starting point)

| Area | Android | iOS | Shared (KMP) |
|---|---|---|---|
| Language/UI | Kotlin + Compose | Swift + SwiftUI | Kotlin core |
| Local index | Room/SQLite + FTS5 | SQLite/GRDB + FTS | SQLDelight |
| Background | WorkManager, Foreground Service | BGTaskScheduler | scheduling abstraction |
| Voice | Porcupine + Rhino | App Intents/Siri + Rhino (fg) | intent contracts |
| Auth | AppAuth (OAuth) | AppAuth | token model |
| Secure storage | Keystore | Keychain | interface |

---

## 13. Validation spikes (do these *before* committing)

1. **Battery PoC:** Android always-on wake word for N hours; measure mAh/hour with and without VAD gating. Decide if always-on is viable or opt-in-only.
2. **Notification feed PoC:** NotificationListener ingest of WhatsApp/Slack/IG notifications → index → "what did I miss" query.
3. **Speech-to-intent PoC:** Rhino grammar for 8 starter intents; measure accuracy + latency on-device.
4. **Connector spike:** Gmail + Calendar + Slack delta sync into the index; end-to-end "any unread from X" answered from local data.
5. **iOS reality check:** App Intents/Siri hand-off invoking the App with a query.
6. **API re-verification:** confirm current availability/limits of every 🟢/🟡 connector (these shift).

---

## 13.5 Phase-0 critical decisions & spike gates (must close before Phase 1)

> These four decisions are **upstream of everything else**: each one changes architecture, legal posture, store submission, and battery numbers. Treat them as **gates** — Phase 1 does not start until each is either *decided* or its spike has *passed/failed* against the exit criteria below. Each gate has a single accountable owner.

### Decision 1 — Cloud or zero-cloud (resolve first; it cascades)

**Why it's a gate:** §5's battery strategy assumes **push** (Gmail→Pub/Sub, Graph→webhook), but push **requires a server**. Multi-device, restore/backup, and heavy backfill also pull toward a backend. So "strictly on-device" (§15.4) and "push-driven, battery-cheap" are currently contradictory. Pick one of three postures:

| Option | What it means | Cost |
|---|---|---|
| **A. Pure on-device** | No backend. **No push** → poll on delta cursors only. Single-device. No managed restore. | Worse freshness/battery; simplest privacy + legal story |
| **B. Thin relay backend** | Stateless push relay (Pub/Sub topic + Graph webhook → APNs/FCM ping only, **no content stored**). Device still pulls content directly from providers. | Best battery/freshness; adds a transfer + attack surface to legal/security scope |
| **C. Backend + encrypted sync** | Relay **plus** encrypted multi-device index sync / restore. | Most capable; largest legal, security, and ops burden |

**Spike (3 days):** Stand up a throwaway relay for **one** Gmail account: provider → Pub/Sub → FCM data-only ping → device performs targeted delta pull. Measure wake-ups/hour and end-to-end notify→indexed latency. Compare against a **poll-only** baseline (no relay).
**Exit criteria (decide A/B/C):**
- Push relay yields **≥50% fewer background wake-ups** *and* **<60s** notify→indexed latency vs poll baseline — else push isn't worth the backend; choose A.
- Relay can be operated with **zero message content at rest** (ping-only) — required to keep B viable for legal/security.
- A written one-paragraph posture statement is committed to §15.4 and the privacy doc.
**Owner:** Backend/data engineer (decision), with DPO + Security sign-off.

#### D1 — Decision record (2026-06-18): **Option A (pure on-device) for MVP**

**Decision:** Ship **A — pure on-device, poll + on-demand live refresh** for Android MVP. Defer **B** (thin relay) behind a feature seam; **C** (encrypted multi-device sync) is out of scope until iOS/multi-device is real. *Gate closed by decision* — the relay spike is **not** required to decide, for the reasons below.

**Why push isn't core (the key realization):** the product is **ask-based**, not proactive-alert-based. Every flagship intent is pulled by the user ("when's my next meeting?", "any unread from my boss?"). Freshness only has to be correct *at the moment of asking*, which `§5`'s **targeted live refresh** of the single relevant source already covers. A modest background **poll** keeps the index "warm enough" for instant answers between uses. Proactive push ("you just got an important email") is not a stated v1 feature — so the main thing a backend buys us isn't something we're selling yet.

**Why push can't be zero-cloud (so B/C mean a real backend):**
- **Gmail:** `users.watch` publishes to a **Google Cloud Pub/Sub** topic; the notification carries only a `historyId` (no content), and the watch **expires ~7 days** (needs renewal). Delivering that to the device needs either on-device Pub/Sub credentials + a persistent connection (kills battery, leaks creds) or a **server** webhook → FCM. → server.
- **Microsoft Graph:** mail subscriptions require a **public HTTPS webhook** with a validation handshake and periodic renewal. → server.
- In both, content is still pulled **directly by the device** with the user's OAuth token, so a relay *can* avoid storing content — **but it still sees metadata**: which mailbox changed, and the timing/volume of changes. That is personal data with a transfer surface, DPIA weight, and an always-on service to secure/operate.

**Cost/benefit that drives A:**

| Factor | A (on-device) | B (thin relay) |
|---|---|---|
| Freshness *when asking* | Good (live refresh) | Good |
| Freshness *between uses* | ~15 min Android (WorkManager floor); coarse on iOS (BG budget) | Near-real-time |
| Battery | Fine if polls are delta-only, batched, constrained | Best |
| Privacy/legal | **Cleanest** — no transfer, no metadata leak, no server | Relay sees mailbox + change metadata; transfer map + DPIA + ops |
| Backup/restore | User-driven **encrypted export** to their own storage (no managed service needed) | Same |

The freshness gap A leaves (proactive/between-use staleness) is exactly the part the product doesn't currently sell, so we don't pay the backend's legal+ops+security cost to close it yet.

**Architectural seam to keep B cheap later:** put background freshness behind a `FreshnessSource` abstraction with two implementations — `PollingFreshness` (A, default) and a future `PushFreshness` (B). The connector `syncDelta()`/`refreshOne()` contracts in `§8` are identical for both, so adding a relay later is additive, not a rewrite.

**Revisit-B triggers (write these down so the decision is falsifiable):** (1) we add a **proactive-alert** feature users actually ask for; (2) field data shows the 15-min poll floor makes "what did I miss" feel stale; (3) iOS background budget proves too coarse for acceptable freshness. If any fires, build the relay spike then — content-free, ping-only, with a transfer map.

**Sign-off needed to ratify:** DPO + Security confirm A as the committed posture (trivial, since A removes their hardest surface). SMS/Call-log remain dropped (`§10`).

### Decision 2 — Proper-noun voice strategy (biggest feasibility risk)

**Why it's a gate:** the flagship intents (`meeting_with_person`, `last_message_from`) need arbitrary **contact names** to survive the *audio→intent* stage, but fixed-grammar speech-to-intent (§6) treats proper nouns as out-of-vocabulary. If this fails, person-name voice intents don't ship.

**Spike (4 days):** Inject ~200 real contact names (incl. non-English, e.g. Hebrew given names) into the speech-to-intent slot via **dynamic vocabulary/phonetic matching**. Test utterances per name across 2–3 speakers in quiet + noisy conditions. Include an explicit **disambiguation + "did you mean"** fallback path.
**Exit criteria (decide ship / de-scope):**
- **Top-1 name slot accuracy ≥85%** (quiet) and **≥70%** (noisy) on the test set, **or** top-3 + disambiguation **≥90%**.
- Dynamic vocab refresh (contacts change) runs **on-device in <2s** and needs no rebuild/cloud.
- If thresholds miss → **de-scope person-name voice intents from MVP**; keep them in type-to-query, and record the limitation in §7's fallback contract.
**Owner:** Voice/on-device ML engineer.

#### D2 — Decision record (2026-06-18): **person-name voice intents IN, via STT + phonetic match (not Rhino slots)**

**The spike's premise was wrong.** It assumed we could "inject ~200 contact names into the speech-to-intent slot" with an on-device refresh in <2s. But **Picovoice Rhino compiles its grammar + slot values into a fixed `.rhn` context *ahead of time* (in Picovoice Console)** — there is **no on-device context compiler**. You cannot stream a user's live, changing contact list into a Rhino slot at runtime, let alone in <2s without a cloud/rebuild. So the exit criterion as written is **unachievable by construction** with Rhino. Speech-to-intent is the wrong tool for an **open, per-user, mutable vocabulary** like contact names.

**Decision:** Keep person-name voice intents **in scope for MVP**, but deliver them with a different mechanism, and split the voice pipeline by intent type:

| Query type | Mechanism | Why |
|---|---|---|
| **Bounded, no proper noun** (`next_meeting`, `today_agenda`, `notifications_recap`, `unread_from` with a label like "boss") | **Rhino speech-to-intent** | Instant, offline, battery-light, deterministic — Rhino's sweet spot |
| **Proper-noun slot** (`meeting_with_person`, `last_message_from`, "…from Avital") | **On-device STT → intent parse → phonetic contact match** | Open vocabulary needs transcription + fuzzy matching, not a fixed grammar |

**Name resolution is a matching problem, not a recognition problem.** STT only has to get the name *phonetically close*; we then match the spoken token against the **on-device contact list** using phonetic + edit-distance scoring (Double Metaphone / Jaro–Winkler, plus a phoneme model for non-Latin scripts). The contact list is the candidate set, so "Avital", "Dawid", "Dominik" resolve by nearest-sounding match, with a **disambiguation prompt** when top candidates are close. This refreshes trivially when contacts change (it's just a lookup table), satisfying the *intent* of the old <2s criterion without compiling anything.

**This is still "no LLM."** STT (Android `SpeechRecognizer` offline / iOS `Speech` / Vosk) + grammar/fuzzy NLU + phonetic match — no generative model, on-device, private. Heavier STT only runs for name-bearing queries, which are **tap-to-talk / post-wake-word** moments (not always-on scoring), so the battery story in §6 holds: the always-on path stays Rhino-only.

**Guaranteed baseline + fallbacks (so this can't hard-fail):**
1. **Typed queries** always do reliable text → phonetic contact match (no STT in the loop) — person-name intents work on day one via keyboard regardless of voice accuracy.
2. **OS-assistant hand-off** (Siri / Google Assistant) for voice name capture as a fallback — those recognizers are already **biased to the user's contacts** and handle proper nouns far better than offline STT.
3. If offline STT proves too weak on non-English names, name-by-voice degrades to (1)/(2); bounded voice intents are unaffected.

**What the spike should *actually* measure (reframed, ~3 days):** offline-STT + phonetic-match top-1/top-3 accuracy on ~200 real contacts incl. Hebrew given names, quiet + noisy, 2–3 speakers — to decide whether **offline STT** is good enough or whether name-by-voice should lean on **OS hand-off**. The architecture decision (above) does **not** depend on this number; only the *voice-capture engine choice* does. Keep the **≥85% quiet / ≥70% noisy, or top-3+disambiguation ≥90%** bar as the offline-STT go/no-go.

**Net:** the original "Rhino can't do proper nouns → person intents don't ship" blocker is **dissolved** — we don't ask Rhino to do them. Updates applied to §6 (pipeline + engine roles) and §7 (entity resolution). 

### Decision 3 — Notification connector status (core vs optional)

**Why it's a gate:** `NotificationListenerService` is both the most differentiated capability ("what did I miss across all apps") **and** the biggest Play-policy and privacy exposure. Its status changes the store submission and the Data-safety form.

**Spike (3 days):** Build the §13 notification-feed PoC **and** draft the Play **Permissions Declaration + limited-use attestation** and the matching **Data safety** entries. Dry-run the disclosure UX (prominent in-app consent, per-app ingestion toggles, kill switch).
**Exit criteria (decide core / optional / cut):**
- A purpose string + limited-use justification exists that we believe is submittable, reviewed by the release-compliance owner.
- Runtime behavior **exactly matches** the Data-safety declaration (content-free telemetry confirmed).
- Per-app ingestion toggle + global kill switch + "no historical backfill" are implemented in the PoC.
- Default decision: ship as **opt-in, non-core** (app fully functional without it) unless compliance owner clears it as core.
**Owner:** App-store/release-compliance owner, with Security + DPO.

### Decision 4 — Legal posture on third-party data (no build without this)

**Why it's a gate:** the index is built largely from **other people's** personal data (senders, participants, contacts, notification contents) and will inevitably include **special-category data**. The "GDPR-aligned" bullet (§10) doesn't establish a lawful basis for third-party data subjects.

**Spike / artifact (legal-led, 5 days):** Produce (a) a **data-flow inventory** per connector (what's read, where it goes, retention), (b) a **household-exemption analysis** vs DPIA trigger, (c) handling rules for special-category and **children's** data, (d) third-party-data stance, and (e) **transfer map** if Decision 1 = B/C.
**Exit criteria (go / constrain / no-go):**
- Documented lawful basis (or defensible household-exemption rationale) for processing third-party data on the user's behalf.
- Special-category handling rule defined (e.g., no special-category-derived inferences; standard storage controls only).
- Decision 1's posture reflected in the transfer map; SMS/Call-log confirmed **dropped** (§10) in writing.
- Sign-off recorded before Phase 1 starts; otherwise scope is constrained until met.
**Owner:** Legal counsel + DPO.

**Gate summary**

| # | Decision | Spike length | Exit = | Owner |
|---|---|---|---|---|
| 1 | Cloud vs zero-cloud | 3d | A / B / C posture committed | Backend (+DPO, Security) |
| 2 | Proper-noun voice | 4d | Ship vs de-scope person intents | Voice/ML |
| 3 | Notification connector | 3d | Core / opt-in / cut | Compliance (+Security, DPO) |
| 4 | Third-party data legal | 5d | Go / constrain / no-go | Legal + DPO |

---

## 14. Phased roadmap

| Phase | Goal | Contents |
|---|---|---|
| **0 — Validate** | De-risk | The 6 spikes (§13) **+ the 4 decision gates (§13.5)** — Phase 1 is blocked until D1–D4 are decided or their spikes pass/fail against exit criteria |
| **1 — Android MVP** | Prove the loop | Launch + tap-to-talk; Gmail + Calendar + Slack + Contacts; 8 intents; local index; on-screen + TTS |
| **2 — Voice + breadth** | Hands-free + coverage | Opt-in always-on wake word; NotificationListener connector; +Notion/OneNote; bigger intent catalog |
| **3 — iOS** | Second platform | KMP core reuse; Siri/App Intents; EventKit; foreground voice |
| **4 — Polish/scale** | Robustness | More intents, disambiguation UX, optional small on-device NLU model, sync tuning |

---

## 15. Open decisions (need your input)

1. **Always-on voice:** ship it (Android, opt-in, battery cost) or start launch-only and add later?
2. **Closed apps:** are you OK telling users **WhatsApp/IG = recent notifications only**, and **Facebook/TikTok dropped**? Or is one of them a must-have we should re-scope around?
3. **iOS expectation:** acceptable that iOS has no custom hands-free wake word (Siri hand-off instead)?
4. **Cloud or zero-cloud:** **DECIDED (2026-06-18) → Option A, pure on-device** for MVP. Posture: *Otto runs entirely on-device; connectors pull directly from providers with the user's own OAuth tokens; freshness comes from on-demand targeted live refresh plus a battery-aware background poll; there is no Otto backend, so no Otto server ever sees user content or metadata. A thin content-free push relay (Option B) is deferred behind a `FreshnessSource` seam and only revisited if proactive alerts ship, poll-freshness proves too stale in the field, or iOS background budgets force it.* See `§13.5` D1 decision record. (Privacy doc: TODO — restate this posture there.)
5. **Build model:** KMP shared core vs fully native ×2?
6. **First connector set:** confirm Gmail + Calendar + Slack + Contacts as Phase 1.

> **Gated decisions:** items above intersect the Phase-0 gates in §13.5 — **#4 → D1 (cloud)**. Also tracked there but not yet in this list: **D2 proper-noun voice** (feasibility of person-name voice intents), **D3 notification-connector status** (core vs opt-in vs cut), and **D4 legal posture on third-party data** (go/constrain/no-go). Close §13.5 before locking these.

---

## 16. Risk register

| Risk | Impact | Mitigation |
|---|---|---|
| Always-on wake word drains battery | High | VAD gating, opt-in, conditional listening; PoC gate in Phase 0 |
| iOS can't do hands-free custom phrase | Med | Reframe iOS UX around Siri/App Intents; set expectations |
| WhatsApp/Meta/TikTok have no read API | High (vs expectations) | Notification-channel-only; drop dead ends; legal sign-off |
| Play Store policy (NotificationListener/SMS/Accessibility) | High | Strong justification; avoid Accessibility scraping; likely drop SMS |
| "No LLM" limits flexibility | Med | Intent catalog as roadmap; clear fallback; optional small model later |
| Entity ambiguity (which "Dawid"?) | Med | Disambiguation prompts; contact-backed resolution |
| Connector API churn | Med | Pluggable adapters; re-verify each release |

---

*Draft v0.1 — built to be argued with. Tell me which of the §15 decisions you want to lock, and I'll deepen that section (e.g., full intent catalog, the connector adapter interface in detail, or a Phase-0 spike plan with metrics).*