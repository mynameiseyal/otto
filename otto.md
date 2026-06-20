# Personal Assistant App — Plan, Specs & Design (v0.1 draft)

> **Working name:** (TBD) — referred to as **the App** below.
> **Scope of this document:** architecture, subsystems, connector feasibility, voice/battery strategy, the "no-LLM" approach, cross-platform strategy, phased roadmap, risks, and open decisions. No implementation yet — this is the blueprint to react to.

---

## 1. Product summary

A privacy-first, on-device personal assistant for **Android (first)** and **iOS (later)**. It connects to your apps/accounts, keeps a **local, normalized index** of that data, and answers questions using a **deterministic, on-device, no-LLM core** for its bounded intents — with an **optional, opt-in "Assistant mode"** that reasons over the index using **your own** cloud LLM key (Claude/GPT/Gemini) for open-ended requests (see §13.5 D5). Two ways to operate it:

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

### 3.2 "No LLM" — for the core (see §13.5 D5 for the open-ended layer)

Fully workable for a **defined intent set**. Use **on-device speech-to-intent** (audio → intent + slots within a grammar) rather than full transcription + reasoning. Benefits: ~instant, offline, private, deterministic, battery-light. Cost: it only answers what's in the grammar. **Treat the intent catalog as the product roadmap** (§7). Keep an explicit fallback for unrecognized requests.

> Optional later: a *small* on-device classifier (not an LLM) to improve paraphrase tolerance. Still no cloud, still no generative model.

> **Scope of "no LLM" (per D5, §13.5):** "no LLM" is the rule for the **bounded core** — and that core is the default, always-on, zero-egress surface. Open-ended requests ("compile a report," "find everything about X across my apps") that the grammar can't serve are handled by an **opt-in Assistant mode** using the **user's own** cloud LLM key. The LLM is a reasoning layer **on top of** the local index, never on the always-on voice path, and bring-your-own-key only — so no Otto backend appears and D1/D4 hold. The bounded core remains fully functional with Assistant mode off.

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

**Fallback contract:** if no intent matches confidently → say so and offer the closest supported action ("I can't search WhatsApp history, but I can recap recent WhatsApp notifications — want that?"). This keeps trust intact and is your roadmap signal. **With Assistant mode on (D5, §13.5),** the fallback gains a branch: "…or I can reason over your indexed data to answer that — it'll send the relevant slice to your {provider} via your own key. Do that?" — explicit, opt-in, with an egress badge. Assistant mode off → the original fallback is unchanged.

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

#### D3 — Decision record (2026-06-18): **opt-in, separately-consented, promoted feature — app fully usable without it**

**Decision:** Ship the notification-recap connector as an **opt-in feature gated behind its own prominent disclosure + affirmative consent**, with the app **fully functional without it**. It *is* promoted in the listing (so the access maps to a real, disclosed feature), but it is **not a precondition** to use Otto. Bounded connectors (Gmail/Calendar/Slack/Contacts) carry the MVP; notification recap is the differentiated add-on.

**Two corrections to this gate's original framing:**
1. **There is no Play "Permissions Declaration Form" for notification access.** That form covers **SMS/Call Log** (which we already drop, §10). `BIND_NOTIFICATION_LISTENER_SERVICE` is a signature-level permission the *user* grants via system Settings → Notification access. Compliance instead hinges on Google Play's **sensitive-data policy**: access must be (a) **necessary for a feature promoted in the store listing**, (b) preceded by **prominent in-app disclosure + affirmative consent**, (c) accurately reflected in the **Data safety** section, and (d) covered by a public, non-geofenced **privacy policy**.
2. **D1 (zero-cloud) is the single biggest de-risker.** Google Play "collection" means data **sent off the device**. Because Otto processes notifications **on-device only with no backend**, the Data safety section can declare **no collection/sharing off-device** for notification content — dramatically lowering review risk. We still must **disclose the on-device access**.

**Why opt-in/non-core (not core):** if notification access *is* the app's promoted core purpose, reviewers scrutinize it as a "read everything" app and rejection risk spikes. Framing it as one **optional, separately-consented capability** within a broader assistant (a) keeps the app reviewable on its bounded connectors alone, (b) still satisfies "tied to promoted functionality" because the recap feature is itself promoted, and (c) gives the user an explicit on/off decision.

**Implementation constraints (make the policy posture true in code):**
- **Per-app ingestion allowlist** — user explicitly picks which apps to recap; default none.
- **Global kill switch** + easy revoke; honor `onListenerDisconnected`.
- **Forward-only** — no historical backfill (the API can't anyway); retention cap per §5.
- **On-device only** — never transmit notification content (consistent with D1).
- **Sensitive redaction is automatic:** on Android 15+, a third-party listener without `RECEIVE_SENSITIVE_NOTIFICATIONS` (which we won't hold — it's signature/role) **cannot see OTP/sensitive notification content**. We treat that as a *feature*, not a gap, and document the limit.
- **Honest foreground-service notification** for the always-on voice path is separate (needs `POST_NOTIFICATIONS` on Android 13+).

**Capability limits to communicate:** Android-only; forward-only (not history); OTP/sensitive content redacted by the OS; quality depends on what each app puts in its notifications.

**Draft compliance artifacts (for compliance-owner + legal review):**

*Prominent-disclosure consent copy (shown before enabling, in-app, before the system settings deep-link):*
> **Turn on Notification Recap?** Otto will read incoming notifications **only from the apps you choose** so it can answer "what did I miss?". This happens **entirely on your device** — notification content is never uploaded or shared. Otto can't see one-time passcodes or other sensitive notifications. You can turn this off or change which apps anytime. [Choose apps] [Not now]

*Store-listing justification snippet (ties access to promoted feature):*
> Otto's optional "Notification Recap" reads notifications from apps you select to summarize what you missed across messaging and social apps. All processing is on-device.

*Data safety mapping (given D1 = on-device only):*
| Data type | Collected (sent off device)? | Shared? | Notes |
|---|---|---|---|
| Messages / in-app content (from notifications) | **No** | **No** | Processed on-device only; never transmitted |
| App activity (notification metadata) | **No** | **No** | On-device index only |

**Residual sign-off needed:** release-compliance owner approves listing copy + the disclosure flow; legal/DPO approves the privacy-policy text covering notification access. These are approvals, not blockers to building the PoC.

**Net:** notification recap **ships** as an opt-in, on-device, per-app, forward-only feature with explicit consent — the zero-cloud decision makes the Data-safety story clean, and opt-in framing minimizes review risk. The "Permissions Declaration Form" worry was a misclassification.

### Decision 4 — Legal posture on third-party data (no build without this)

**Why it's a gate:** the index is built largely from **other people's** personal data (senders, participants, contacts, notification contents) and will inevitably include **special-category data**. The "GDPR-aligned" bullet (§10) doesn't establish a lawful basis for third-party data subjects.

**Spike / artifact (legal-led, 5 days):** Produce (a) a **data-flow inventory** per connector (what's read, where it goes, retention), (b) a **household-exemption analysis** vs DPIA trigger, (c) handling rules for special-category and **children's** data, (d) third-party-data stance, and (e) **transfer map** if Decision 1 = B/C.
**Exit criteria (go / constrain / no-go):**
- Documented lawful basis (or defensible household-exemption rationale) for processing third-party data on the user's behalf.
- Special-category handling rule defined (e.g., no special-category-derived inferences; standard storage controls only).
- Decision 1's posture reflected in the transfer map; SMS/Call-log confirmed **dropped** (§10) in writing.
- Sign-off recorded before Phase 1 starts; otherwise scope is constrained until met.
**Owner:** Legal counsel + DPO.

#### D4 — Decision record (2026-06-18): **GO, with constraints — pending lawyer ratification**

> ⚠️ **Not legal advice.** This is a structured posture to hand to counsel/DPO for ratification. The gate's human sign-off remains outstanding; everything below is the *recommended* position and reasoning.

**Decision (recommended):** **GO** for Phase 1, on the posture that **D1 (zero-cloud) does most of the legal work**:
- **The developer is not a controller of indexed content.** Because there is **no backend** (D1 = A), the developer never receives, stores, or processes users' or third parties' content. Processing happens entirely on the user's device, under the user's control. The developer is only a controller for whatever it *actually* collects — which, with content-free telemetry, is approximately nothing.
- **The user's processing falls under the GDPR household exemption** (Art. 2(2)(c)): a natural person indexing data they **already lawfully hold** (their own inbox, messages, notifications, contacts) for **purely personal** organization, with **no onward disclosure**. This is materially the same as the email client, notes app, or calendar app the user already runs. (Guardrail from *Ryneš*/*Lindqvist*: the exemption is lost if data is published or shared to an indefinite audience — Otto does neither; it's strictly local and single-user.)

**Per-item resolution of the gate's checklist:**
1. **Lawful basis / third-party data:** household exemption for the user; developer = non-controller of content. No separate lawful basis needed *from* the third parties, because Otto (the developer) isn't the one processing their data and the user's use is personal/household.
2. **Special-category data (Art. 9):** will inevitably appear in messages, but (a) it's covered by the household exemption for the user, and (b) **design constraint: no special-category inference/profiling** — Otto indexes and retrieves as-is for explicit user queries; the "no-LLM, bounded-intent" design already avoids building sensitive profiles. Document this as a hard rule.
3. **Children's data:** Otto is **not directed at children**; set an age floor in ToS (≥16, aligned to GDPR digital-consent age) and don't market to children. Incidental third-party children's data in the user's messages is covered by the same household-exemption logic. Set store age ratings/Data-safety accordingly.
4. **Transfer map:** **empty on Otto's side** — no backend means no international transfers by Otto. Connector data flows are the **user's own pre-existing relationships** with Google/Microsoft/Slack (their controllership, their transfer terms), not Otto's.
5. **SMS/Call-log:** **dropped** — confirmed in writing here and in §10.

**DPIA:** a formal Art. 35 DPIA is likely **not triggered** (no large-scale processing *by the developer*), but a **documented DPIA-lite / privacy risk assessment is recommended** as good practice — mainly to record this non-controller + household-exemption reasoning and the special-category guardrail.

**Hard dependency / re-open trigger:** this entire posture **rests on D1 = zero-cloud**. If Otto ever adopts a backend (Option B/C), the developer likely becomes a **controller/processor** of content or metadata → full GDPR obligations, lawful basis, DPIA, transfer map, and a fresh DPO review. **Re-open D4 if D1 changes.**

**Constraints that must hold (make the posture true):**
- Stay zero-cloud (D1); content-free, PII-redacted, local-only-capable telemetry.
- No special-category profiling/inference; no onward sharing of indexed data.
- Retention caps + user purge/export (§5); per-source revoke.
- Public privacy policy stating what's indexed, on-device-only processing, retention/deletion, and contact point.

**Residual sign-off (the actual gate close):** counsel/DPO ratifies the household-exemption + non-controller analysis and approves the privacy policy + ToS age floor. Recorded as **provisionally GO**; treat as ratified for solo planning, formalize with a lawyer before public release.

**Net:** **GO** — the zero-cloud architecture (D1) is what makes Otto legally clean: developer-as-non-controller + user-under-household-exemption, with a special-category guardrail and an explicit re-open trigger if the cloud posture ever changes.

### Decision 5 — LLM posture (bounded-deterministic core vs LLM-orchestrated open-ended)

**Why it's a gate:** the product summary (§1) and §3.2 commit to **"no LLM"** as an absolute. That choice is what makes the bounded-intent NLU (§7), the battery story (§6), and the legal posture (D4) clean. But it also caps capability at the **fixed intent catalog**: open-ended requests — *"compile a report on X," "find everything across my apps about that talk," "whatever I ask"* — are exactly the requests a fixed grammar must answer with §7's "I can't answer that yet." So there is a real fork between **what Otto promised** (deterministic, on-device, bounded) and **what an open-ended assistant implies** (a reasoning model over the index). Resolving it is upstream of the intent catalog scope, the privacy doc, and a possible re-open of D1/D4 — because any *cloud* model means content leaves the device. Pick a posture:

| Option | What it means | Cost |
|---|---|---|
| **A. Hold no-LLM** | Bounded intents only. Open-ended requests are unsupported (§7 fallback). | Cleanest privacy/legal/battery; weakest capability — fails the "whatever I ask / compile a report" ask |
| **B. On-device small LLM** | A local quantized model (Gemini Nano / Apple on-device / llama.cpp / MLC) reasons + summarizes over the local index. | Stays **zero-cloud** → D1/D4 untouched. Cost: battery, RAM, device-class floor, app size, weaker quality |
| **C. Bring-your-own cloud LLM** | User supplies **their own** API key (Claude/GPT/Gemini). Open-ended requests send **user-approved slices of the local index** directly to that provider. | Best quality; matches "use whatever LLM I'm using." Content leaves device → re-opens D1/D4 unless framed as the **user's own** provider relationship |

#### D5 — Decision record (2026-06-20): **hybrid — deterministic no-LLM core (A) is always-on; bring-your-own-LLM "Assistant mode" (C) is opt-in; on-device LLM (B) deferred behind a seam**

**Decision:** Keep the **no-LLM bounded core as the default and always-available** surface — it carries every flagship intent (§7) at zero battery/privacy/legal cost. Add an **opt-in "Assistant mode" that uses the user's *own* cloud LLM (Option C)** for open-ended requests the catalog can't serve. Defer an **on-device model (Option B)** behind an `LlmBackend` seam, to be revisited when on-device models are good/cheap enough to serve open-ended reasoning without the cloud. *This converts the absolute "no LLM" into "**no-LLM core + optional, user-keyed LLM assistant**."*

**The key realization — the LLM is the user's own relationship, exactly like a connector.** D4's legal cleanliness rests on the developer being a **non-controller**: connector data flows are *"the user's own pre-existing relationships with Google/Microsoft/Slack — their controllership, their terms, not Otto's."* **The same logic extends to the model provider.** In Assistant mode the user brings **their own API key** and **their own account** with Anthropic/OpenAI/Google; the device sends context the user **explicitly approved** **directly** to that provider. Otto operates **no backend and no shared key** in this path, so the developer still never receives or processes the content → **still a non-controller**, and the user is still acting under the household exemption (choosing to process their own data through a tool they already use). D5 therefore does **not** break D4 — *as long as the call is device→provider with the user's key.*

**Why C and not "Otto proxies the model":** the moment Otto routes LLM calls through its **own** server or its **own** key, the developer becomes a **processor/controller** of the content sent → full GDPR obligations, transfer map, DPIA, fresh DPO review (precisely D4's re-open trigger). So MVP is **bring-your-own-key only**; no Otto-hosted or Otto-proxied model. This mirrors D1's "stay zero-cloud" constraint: *the moment a backend appears, the posture changes.*

**Why a hybrid and not pure-C:** routing every request through a cloud LLM would throw away the things that make Otto Otto — instant offline answers, zero battery for common asks, no data leaving the device for "when's my next meeting?". So the **router decides** (§4): bounded intent recognized → deterministic handler, answer from index, nothing leaves the device. No confident intent **and** Assistant mode enabled → LLM path over the index slice. Assistant mode **off** → §7 fallback unchanged. The bounded core remains fully functional with Assistant mode disabled, exactly as D3's notification connector keeps the app usable when off.

**Constraints that must hold (make the posture true):**
- **Bring-your-own-key only** in MVP — no Otto-hosted or Otto-proxied model. Keys in Keystore/Keychain (§10). *(No Otto backend appears → D1 = A holds, D4 non-controller holds.)*
- **Opt-in, off by default;** the no-LLM core works without it. Assistant mode is a separately-consented capability (cf. D3).
- **Context minimization + explicit egress consent:** only the **index slices relevant to the query** are sent, never the whole index; the user can see what is sent; a clear **"this leaves your device to {provider}"** badge distinguishes LLM answers from on-device ones. Reuse the special-category guardrail (D4): no special-category-derived profiling; the user controls what context is eligible.
- **Provider terms are the user's to accept:** surface, don't bury, that API-tier handling is governed by the user's chosen provider (Otto states the egress, links the provider's data terms; does not warrant them).
- **No always-on LLM:** the LLM never runs on the always-on voice path (that stays Rhino-only, §6); Assistant mode is a tap/explicit-request surface, so the battery story holds.
- **Architectural seam:** put it behind an `LlmBackend` abstraction with `CloudByoKey` (C, MVP) and a future `OnDeviceModel` (B) implementation, so adding on-device inference later is additive — same pattern as D1's `FreshnessSource`.

**Re-open D5/D4 triggers (keep it falsifiable):** (1) Otto adds an **Otto-hosted or proxied** model, or a shared/managed key → developer becomes processor; (2) content is sent to a provider **without** explicit per-query/eligible-context consent; (3) Assistant mode starts **proactively** sending context (not user-initiated). Any of these → re-run the D4 controller analysis before shipping.

**What this does to the rest of the doc:** §1 and §3.2 "no LLM" become **"no-LLM core + optional bring-your-own LLM assistant"**; §7's fallback gains a branch ("…or, with Assistant mode on, I can reason over your indexed data via your own {provider} key"); §10 adds key storage + egress-consent UX; §14 roadmap adds Assistant mode as a Phase-2+ capability (the bounded core ships first, in Phase 1, unchanged).

**Residual sign-off:** DPO confirms the non-controller analysis **survives** for the BYO-key egress path (it should — no backend, user's key, user-approved context, no onward sharing by Otto); privacy policy gains a clause naming the on-device-vs-egress split and that the model provider is the user's chosen processor. Recorded as **GO for a hybrid**, with the bounded no-LLM core unaffected and shippable first.

**Net:** the "no LLM" line is **relaxed, not abandoned** — the deterministic, on-device, zero-egress core stays the default and ships first; open-ended "whatever I ask / compile a report" capability arrives as an **opt-in assistant on the user's own model key**, which keeps D1 (no Otto backend) and D4 (developer non-controller) intact by construction. Otto is therefore **not** "just another LLM client": the model is an optional reasoning layer **on top of** the connectors + local index, which remain the moat.

**Gate summary**

| # | Decision | Exit = | Owner | Status (2026-06-18) |
|---|---|---|---|---|
| 1 | Cloud vs zero-cloud | A / B / C posture committed | Backend (+DPO, Security) | ✅ **A — pure on-device** |
| 2 | Proper-noun voice | Ship vs de-scope person intents | Voice/ML | ✅ **Ship — STT + phonetic match** |
| 3 | Notification connector | Core / opt-in / cut | Compliance (+Security, DPO) | ✅ **Opt-in, promoted, on-device** |
| 4 | Third-party data legal | Go / constrain / no-go | Legal + DPO | ✅ **GO w/ constraints** (pending lawyer ratification) |
| 5 | LLM posture (bounded vs open-ended) | Hold no-LLM / on-device / bring-your-own | Product (+DPO) | ✅ **Hybrid — no-LLM core + opt-in BYO-key assistant** |

> **All four original Phase-0 gates are resolved (2026-06-18); D5 added (2026-06-20).** Recurring theme: **D1 (zero-cloud) cascades** — it underpins the battery story (D1), the notification Data-safety posture (D3), the legal posture (D4), **and now the LLM posture (D5)**: bring-your-own-key keeps Otto backendless, so the developer stays a non-controller even with a cloud model in the loop. The no-LLM bounded core (Phase 1) is unaffected by D5 and ships first; the optional LLM assistant is Phase 2+. The remaining empirical work (battery PoC §13.1, offline-STT name accuracy per D2, notification-feed PoC §13.2) and the human sign-offs (compliance for D3, counsel/DPO for D4/D5) are tracked but do not block Phase 1 planning.

---

## 14. Phased roadmap

| Phase | Goal | Contents |
|---|---|---|
| **0 — Validate** | De-risk | The 6 spikes (§13) **+ the 4 decision gates (§13.5)** — Phase 1 is blocked until D1–D4 are decided or their spikes pass/fail against exit criteria |
| **1 — Android MVP** | Prove the loop | Launch + tap-to-talk; Gmail + Calendar + Slack + Contacts; 8 intents; local index; on-screen + TTS |
| **2 — Voice + breadth** | Hands-free + coverage | Opt-in always-on wake word; NotificationListener connector; +Notion/OneNote; bigger intent catalog; **opt-in Assistant mode (BYO-key LLM over the index, D5)** for open-ended requests |
| **3 — iOS** | Second platform | KMP core reuse; Siri/App Intents; EventKit; foreground voice |
| **4 — Polish/scale** | Robustness | More intents, disambiguation UX, optional small on-device NLU model, sync tuning |

---

## 15. Open decisions (need your input)

1. **Always-on voice:** ship it (Android, opt-in, battery cost) or start launch-only and add later?
2. **Closed apps:** are you OK telling users **WhatsApp/IG = recent notifications only**, and **Facebook/TikTok dropped**? Or is one of them a must-have we should re-scope around?
3. **iOS expectation:** acceptable that iOS has no custom hands-free wake word (Siri hand-off instead)?
4. **Cloud or zero-cloud:** **DECIDED (2026-06-18) → Option A, pure on-device** for MVP. Posture: *Otto runs entirely on-device; connectors pull directly from providers with the user's own OAuth tokens; freshness comes from on-demand targeted live refresh plus a battery-aware background poll; there is no Otto backend, so no Otto server ever sees user content or metadata. A thin content-free push relay (Option B) is deferred behind a `FreshnessSource` seam and only revisited if proactive alerts ship, poll-freshness proves too stale in the field, or iOS background budgets force it.* See `§13.5` D1 decision record. Posture restated for users in [`PRIVACY.md`](./PRIVACY.md) (draft).
5. **Build model:** KMP shared core vs fully native ×2?
6. **First connector set:** confirm Gmail + Calendar + Slack + Contacts as Phase 1.
7. **LLM posture:** **DECIDED (2026-06-20) → hybrid** (§13.5 D5). No-LLM bounded core is the default and ships first; an **opt-in Assistant mode using the user's own cloud LLM key** handles open-ended requests over the local index. Bring-your-own-key only (no Otto backend/proxy), so D1 (zero-cloud) and D4 (non-controller) hold. On-device model (Option B) deferred behind an `LlmBackend` seam. On-device-vs-egress split and "provider is the user's chosen processor" stated for users in [`PRIVACY.md`](./PRIVACY.md) §5 (draft).

> **Gated decisions:** items above intersect the Phase-0 gates in §13.5 — **#4 → D1 (cloud)**, **#7 → D5 (LLM posture)**. Also tracked there: **D2 proper-noun voice** (feasibility of person-name voice intents), **D3 notification-connector status** (core vs opt-in vs cut), and **D4 legal posture on third-party data** (go/constrain/no-go). All five gates are now resolved (§13.5 gate summary).

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