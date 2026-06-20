# Otto — Intent Catalog (spec v0.1 draft)

> **Purpose:** expand [`otto.md`](./otto.md) §7's starter set into the working intent catalog — the bounded, deterministic things Otto understands and does **without an LLM**. Per §7 the catalog *is* the product roadmap: shipping an intent = shipping a capability.
>
> **Two halves now:** since the connector spec added writes (Q5, [`CONNECTORS.md`](./CONNECTORS.md) §3.1), the catalog covers **query intents** ("when's my next meeting?") *and* **action intents** ("reply to Dana", "schedule lunch Thursday"). Actions flow through the confirm-gated write contract.
>
> **Boundary with Assistant mode:** the catalog is the **no-LLM core**. Open-ended requests it can't express ("compile a report on X", "find everything about that subject and summarize it") fall through to **Assistant mode** (D5) — see the fallback ladder (§6). The catalog stays deterministic; the LLM is the escape hatch, not the default.

---

## 1. How an intent is defined

Each intent is a small, declarative record the NLU + planner share:

```
intent        snake_case id (also the analytics + roadmap key)
type          QUERY | ACTION | DEVICE
utterances    sample phrasings (training/grammar seed)
slots         name : type (required | optional)
sources       connector(s) + Capability needed (CONNECTORS.md §4)
voice         RHINO (bounded grammar) | STT_PHONETIC (proper-noun) | TYPED_ONLY
phase         1 | 2 | 3   (roadmap tag, not a promise of order)
handler       one line: what it reads/does and what it answers
```

**Slot types** (resolved by the entity layer, §5): `person`, `timeframe`, `datetime`, `duration`, `keyword`, `text`, `channel`, `label`, `field` (a contact field, e.g. phone/email), `event_ref` (a calendar event), `thread` (a message/thread), `note_ref` (an existing note), `rsvp` (`Rsvp` enum — CONNECTORS.md §3.1).

---

## 2. Voice routing (recap of D2)

How an utterance reaches an intent depends on whether it carries a **proper noun**:

- **`RHINO`** — bounded, no open vocabulary ("what's on today?", "what did I miss?"). On-device speech-to-intent: instant, offline, battery-light. Runs on the always-on path.
- **`STT_PHONETIC`** — carries a **contact name / free text** ("when do I meet *Avital*?"). On-device STT → intent parse → phonetic match against contacts (D2). Tap-to-talk / post-wake-word only, never always-on.
- **`TYPED_ONLY`** — available by keyboard regardless of voice accuracy; every proper-noun intent also works typed (the D2 guaranteed baseline).

> Every `STT_PHONETIC` intent is also reachable `TYPED_ONLY`. Voice is an input mode, not a separate capability.

---

## 3. Query intents (read)

### 3.1 Calendar & time

| intent | utterances | slots | sources | voice | phase |
|---|---|---|---|---|---|
| `next_meeting` | "When's my next meeting?" | — | Calendar `READ_EVENTS` | RHINO | **1** |
| `today_agenda` | "What's on today?", "What's my day look like?" | timeframe? | Calendar `READ_EVENTS` | RHINO | **1** |
| `meeting_with_person` | "When do I meet Avital?", "Next call with Dana?" | person | Calendar `READ_EVENTS` + Contacts `READ_CONTACTS` | STT_PHONETIC | **1** |
| `free_slot` | "Am I free at 3?", "When am I free tomorrow?" | datetime/timeframe | Calendar `READ_EVENTS` | RHINO | 2 |

### 3.2 Email

| intent | utterances | slots | sources | voice | phase |
|---|---|---|---|---|---|
| `unread_from` | "Any unread mail from my boss?" | person/label, timeframe? | Email `READ_MESSAGES` | STT_PHONETIC | **1** |
| `last_message_from` | "What did Dominik last send me?" | person | Email `READ_MESSAGES` (+ Slack `READ_MESSAGES`) | STT_PHONETIC | **1** |
| `search_mail` | "Did I get the invoice from Acme?" | keyword, timeframe? | Email `READ_MESSAGES` | STT_PHONETIC | 2 |
| `find_old_mail` | "Find that email from years ago about the lease" | keyword, person? | Email `SEARCH_HISTORY` | TYPED_ONLY | 2 |

### 3.3 Messaging / Slack & notifications

| intent | utterances | slots | sources | voice | phase |
|---|---|---|---|---|---|
| `search_messages` | "Did anyone message me about the release?" | keyword, timeframe? | Email / Slack / Notifs `READ_MESSAGES` | STT_PHONETIC | **1** |
| `notifications_recap` | "What did I miss?" | timeframe? | Notifications `READ_MESSAGES` | RHINO | **1** |
| `unread_in_channel` | "Anything new in the release channel?" | channel | Slack `READ_MESSAGES` | STT_PHONETIC | 2 |

### 3.4 Notes & contacts

| intent | utterances | slots | sources | voice | phase |
|---|---|---|---|---|---|
| `notes_about` | "Find my notes about the workplan" | keyword | Notes `READ_NOTES` | STT_PHONETIC | **1** |
| `contact_info` | "What's Dana's number?" | person, field? | Contacts `READ_CONTACTS` | STT_PHONETIC | 2 |

### 3.5 Cross-source

| intent | utterances | slots | sources | voice | phase |
|---|---|---|---|---|---|
| `about_person` | "Catch me up on Dana", "What's going on with Avital?" | person | Email / Slack / Notifs `READ_MESSAGES` + Calendar `READ_EVENTS` | STT_PHONETIC | 2 |
| `about_topic` | "What do I have on Project Falcon?" | keyword | all `READ_*` sources | TYPED_ONLY | 2 |

> `about_*` are the boundary intents: a deterministic **gather-and-list** ("here are 6 recent items about Dana"). Turning that gathered set into a written **summary/report** is Assistant mode (§6), not a bounded intent.

---

## 4. Action intents (write — confirm-gated, CONNECTORS.md §3.1)

**Every action is confirmed before execution and undoable where the provider allows.** Actions are `STT_PHONETIC`/`TYPED_ONLY` (they carry names/text) and never run on the always-on path.

| intent | utterances | slots | sources | confirm → undo | phase |
|---|---|---|---|---|---|
| `create_event` | "Schedule lunch with Dana Thursday at 1" | text, person?, datetime, duration? | Calendar `WRITE_EVENTS` | "Create event …?" → delete | 2 |
| `respond_event` | "Accept the 3pm invite", "Decline standup" | event_ref, rsvp | Calendar `WRITE_EVENTS` | "RSVP {accept} …?" → re-RSVP | 2 |
| `reply_message` | "Reply to Dana: on my way" | thread \| person, text | Email/Slack `SEND_MESSAGE` | "Send to Dana …?" → undo-send window | 2 |
| `send_message` | "Tell the team I'll be late" | person (recipients), text | Email/Slack `SEND_MESSAGE` | "Send …?" → undo-send / delete | 2 |
| `create_note` | "Note: pick up the dry cleaning" | text, note_ref? | Notes `WRITE_NOTES` | "Add note …?" → delete | 2 |

### 4.1 Device actions (no connector — on-device OS)

These need no account; they're the cheapest "assistant" wins and the heart of the original "hey otto, set an alarm" ask.

| intent | utterances | slots | mechanism | voice | phase |
|---|---|---|---|---|---|
| `set_alarm` | "Set an alarm for 7am" | datetime | OS AlarmClock intent | RHINO | **1** |
| `set_timer` | "Timer for 10 minutes" | duration | OS timer | RHINO | **1** |
| `set_reminder` | "Remind me to call Dana at 5" | text, datetime | local reminder store | STT_PHONETIC | 2 |

> Device actions are low-risk and reversible, so they confirm implicitly (just do it + show a card) rather than via the §3.1 send-style prompt.

---

## 5. Entity resolution (shared by all intents)

- **`person`** → Contacts via phonetic + edit-distance match (D2); ambiguity → quick disambiguation ("which Dana — Cohen or Levi?"). Spoken names arrive as STT text; typed names match directly.
- **`timeframe` / `datetime` / `duration`** → on-device parser ("tomorrow", "last week", "after 3pm", "10 minutes").
- **`channel` / `label`** → mapped to Slack channel / mail label IDs via synced metadata.
- **`keyword`** → full-text query against the index (FTS), or `SEARCH_HISTORY` for the cold tier.

---

## 6. The fallback ladder (no confident match)

Order matters — degrade honestly, never guess:

1. **Disambiguate** if the intent is clear but a slot is ambiguous ("which Dana?").
2. **Closest supported** if near a known intent ("I can't search WhatsApp history, but I can recap recent WhatsApp notifications — want that?").
3. **Assistant mode (D5)**, *if enabled*: "I can't answer that with a built-in command — want me to reason over your indexed data via your own {provider} key?" (egress badge, §3.1 confirm style). This is where **"compile a report"** and open-ended topic synthesis live.
4. **Honest no** if Assistant mode is off and nothing fits: say so, and log it — the unmatched utterance is the roadmap signal (a candidate next intent).

---

## 7. Phase-1 MVP cut (the "8 intents" in otto.md §1/§14)

Committed read core + the cheapest device wins, all on the Phase-1 connectors (Gmail, Calendar, Slack, Contacts):

`next_meeting` · `today_agenda` · `meeting_with_person` · `unread_from` · `last_message_from` · `search_messages` · `notes_about` · `notifications_recap`
— plus device `set_alarm` · `set_timer` (no connector, trivial, high "assistant" value).

Everything tagged phase 2/3 above (writes, cross-source `about_*`, history search, channels) is breadth that lands after the loop is proven.

---

## 8. Open questions

1. **Is Phase 1 still read-only, or does one write ship in the MVP?** Q5 put writes in scope; the cheapest first write would be `create_event` or `set_reminder`. Recommend: keep MVP read-only + device alarms, add writes in Phase 2 — but it's your call.
2. **Confirmation modality for actions** — voice-confirm ("say yes") vs. always require an on-screen tap for writes? (Leaning: on-screen tap for anything irreversible.)
3. **`about_*` gather size** — how many items before it's noise? Default top-N per source?
4. **Wake-word vs. command verbs** — do device actions ("set alarm") need the wake word every time, or a press-and-speak affordance?
5. **Reminder storage** — local-only reminders vs. writing them into the user's calendar/tasks (connector) so they sync.

---

*Spec v0.1 — expands `otto.md` §7 across query + action intents, wired to `CONNECTORS.md` capabilities and the D2 voice routing / D5 Assistant-mode boundary. Next: lock the Phase-1 cut, then the Phase-0 spike plan or the engine scaffold.*
