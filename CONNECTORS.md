# Otto — Connector Adapter Interface (spec v0.1 draft)

> **Purpose:** flesh out the connector framework sketched in [`otto.md`](./otto.md) §8 into a concrete, implementable contract — the interface every data source implements, the normalized entities it emits, the sync-engine semantics around it, and a conformance bar. This is the bridge from "all decisions made" (D1–D5) to "ready to write code."
>
> **Audience:** whoever builds the shared core and the first connectors (Gmail, Calendar, Slack, Contacts — Phase 1).
>
> **Status:** draft, built to be argued with. Kotlin is illustrative (the shared core is **KMP**, §11); signatures are the contract, not final code.

---

## 1. Where this fits

```
 Query planner ──reads──▶ Local index ◀──writes── Sync engine ──drives──▶ Connector(s) ──▶ provider API
                          (source of truth)        (scheduling, retry,     (this spec)
                                                     backoff, isolation)
```

The connector is **only** an adapter: it authenticates, pulls deltas, and maps provider records to Otto's normalized entities. It owns **no scheduling, no retry policy, no storage** — those belong to the sync engine and the index. This keeps connectors small, testable, and swappable (§8 "connectors are pluggable").

**Hard constraints inherited from the Phase-0 decisions:**
- **D1 (zero-cloud):** a connector talks **directly** from the device to the provider with the user's own OAuth token. No Otto backend is ever in the path. Freshness comes from `syncDelta` (background poll) + `refreshOne` (on-demand), never a server push — see the `FreshnessSource` seam (§7).
- **D4 (non-controller):** the connector pulls data the user already holds, under the user's credentials. It must never transmit content anywhere but the local index.
- **§10 secure-store rule:** tokens live in Keystore/Keychain; the index DB is excluded from OS auto-backup. Connectors never persist tokens themselves — they receive/refresh them via the `TokenStore` abstraction.

---

## 2. Normalized entity model

`normalize()` maps each provider record to one or more of these. Every entity shares a common envelope so the index and planner are source-agnostic (§5).

```kotlin
/** Common envelope across all sources. */
interface Entity {
    val sourceEntityId: SourceEntityId  // stable, per-source; the dedupe/upsert key (§6)
    val source: ConnectorId             // "gmail", "gcal", "slack", ...
    val timestamp: Instant              // canonical event/sort time (sent, starts-at, created)
    val participants: List<PersonRef>   // resolved later to contacts (entity resolution, §7 of otto.md)
    val body: String?                   // full-text-indexed content, if any
    val deepLink: Uri?                  // open this item back in its source app
    val raw: Map<String, String>        // small, source-specific extras the planner may use
}

sealed interface NormalizedEntity : Entity
data class Message(...)      : NormalizedEntity  // email, Slack msg, notification text
data class Event(...)        : NormalizedEntity  // calendar event
data class Note(...)         : NormalizedEntity  // Notion/OneNote page
data class Contact(...)      : NormalizedEntity  // person; critical for resolution
data class Notification(...) : NormalizedEntity  // Android notification feed item
data class Task(...)         : NormalizedEntity  // todo / action item

data class PersonRef(val displayName: String?, val handle: String?, val emailOrId: String?)
```

> **Tombstones:** deletions are first-class. `normalize()` may emit `Tombstone(sourceEntityId)` so the index can remove items the provider reports as gone. Without this, a delta sync can never shrink.

```kotlin
sealed interface SyncRecord
data class Upsert(val entity: NormalizedEntity) : SyncRecord
data class Tombstone(val sourceEntityId: SourceEntityId) : SyncRecord
```

---

## 3. The `Connector` interface

```kotlin
interface Connector<RAW> {
    val id: ConnectorId
    val source: SourceType                 // EMAIL | CALENDAR | CHAT | NOTES | CONTACTS | NOTIFICATIONS

    /** Static + post-auth dynamic capabilities. Re-evaluated after every authenticate(). */
    fun capabilities(): CapabilitySet

    /** Acquire/refresh authorization. Idempotent; safe to call to repair a recoverable token. */
    suspend fun authenticate(ctx: AuthContext): AuthResult

    /**
     * Pull one page of changes since `cursor` (null = initial backfill).
     * MUST be resumable: persist nothing itself; return the cursor to resume from.
     * Returns ONLY successful pages; throws ConnectorException(error) on failure (§4.1).
     */
    suspend fun syncDelta(cursor: SyncCursor?): SyncPage<RAW>

    /** Pure mapping of a raw record to normalized records. No I/O. Deterministic. */
    fun normalize(raw: RAW): List<SyncRecord>

    /**
     * Optional targeted live refresh of a single source for a time-critical query.
     * Bounded by a hard timeout (default ≤1.5s). On timeout/error it MUST signal
     * fallback-to-index, never throw into the response path. (§8 live-refresh)
     */
    suspend fun refreshOne(query: RefreshQuery): RefreshOutcome<RAW> = RefreshOutcome.Unsupported

    /**
     * Full-history search on the PROVIDER side, for queries the 90-day hot index misses
     * (§5.2 two-tier memory). Bounded + paginated; raw records flow through normalize().
     * Throws ConnectorException on failure. Only offered if SEARCH_HISTORY is granted.
     */
    suspend fun searchHistory(query: HistoryQuery, cursor: SyncCursor?): SyncPage<RAW> =
        throw UnsupportedOperationException()

    /**
     * Perform a write/action on the user's behalf (Phase 1: calendar, email, Slack, notes).
     * MUST be preceded by explicit user confirmation (§3.1). Returns an undo handle where
     * the provider allows it. Only offered for the WRITE_* capability the connector grants.
     */
    suspend fun write(action: WriteAction): WriteResult = throw UnsupportedOperationException()

    /**
     * Reverse a previous write via the token from its UndoHandle (§3.1), while the handle
     * is still valid. Throws ConnectorException on failure (e.g. window elapsed).
     */
    suspend fun undo(token: String) = throw UnsupportedOperationException()

    /** Revoke access; optionally purge this source's rows from the index. */
    suspend fun revoke(options: RevokeOptions)
}
```

Design notes:
- **Generic `RAW`** keeps `syncDelta` and `normalize` honest: the connector owns its wire type; the engine only ever sees `SyncRecord`. `normalize` is **pure** so it is trivially unit-testable against captured fixtures.
- **`syncDelta` is one page.** The engine loops it while `page.hasMore`, persisting `page.cursor` between calls. This gives resumable backfill and incremental delta with the same method. Failures don't ride in the page — they're **thrown** (§4.1), so a `SyncPage` always means success.
- **No method blocks on network in the query path** except `refreshOne`, which is explicitly bounded.

### 3.1 Write actions (Phase 1 includes writes — decision Q5)

Reads are safe; **writes act on the world** (a sent email can't be unsent). Phase 1 supports writes for **calendar, email, Slack, and notes**, under a strict contract:

- **Confirm before every write.** The planner/UI MUST show the exact action and get explicit user approval before `write()` is called ("Send this email to Dana? — [Send] [Cancel]"). No silent or batched writes.
- **Undo where the provider allows it.** `WriteResult` carries an `UndoHandle` when reversal is possible (delete a created event/note, Gmail "undo send" window, delete a Slack message). When it's genuinely irreversible (email past the undo window), say so in the confirmation step rather than implying it can be undone.
- **Writes are off the always-on path.** They only originate from an explicit user request (typed, or post-wake-word), never from background sync or proactive logic.
- **Per-connector + per-capability.** A connector only exposes `write()` for the `WRITE_*` capability it actually grants; the planner degrades gracefully if a write scope is missing (§5 scope-drift).

```kotlin
sealed interface WriteAction {
    data class CreateEvent(val title: String, val start: Instant, val end: Instant, val invitees: List<PersonRef>) : WriteAction
    data class RespondToEvent(val sourceEntityId: SourceEntityId, val response: Rsvp) : WriteAction
    data class SendMessage(val to: List<PersonRef>, val body: String, val replyTo: SourceEntityId? = null) : WriteAction // email or chat, per connector
    data class CreateNote(val title: String?, val body: String, val appendTo: SourceEntityId? = null) : WriteAction
}
enum class Rsvp { ACCEPTED, DECLINED, TENTATIVE }
data class WriteResult(val created: SourceEntityId?, val undo: UndoHandle?)
class UndoHandle(val expiresAt: Instant?, val token: String)  // pass token to undo(); null expiry = undoable until further notice
```

---

## 4. Supporting types

```kotlin
// --- Sync ---
// syncDelta returns ONLY successful pages; failures are thrown as ConnectorException
// (§4.1), so rich error detail (retryAfter, missing scopes) isn't flattened into an enum.
data class SyncPage<RAW>(
    val records: List<RAW>,
    val cursor: SyncCursor?,       // opaque resume token / historyId / updatedTime watermark; null if no progress
    val hasMore: Boolean,          // engine loops syncDelta while true
)
@JvmInline value class SyncCursor(val token: String)

// --- Capabilities ---
data class CapabilitySet(val granted: Set<Capability>)
enum class Capability {
    // Read
    READ_MESSAGES, READ_THREADS, READ_EVENTS, READ_NOTES, READ_CONTACTS,
    // Retrieval helpers
    LIVE_REFRESH,        // refreshOne(): bounded on-demand refresh of the hot index
    SEARCH_HISTORY,      // searchHistory(): provider-side full-history search (§5.2 cold tier)
    // Write (Phase 1 — decision Q5; every write is confirm-gated, §3.1)
    WRITE_EVENTS,        // create / RSVP calendar events
    SEND_MESSAGE,        // send / reply email or Slack message (per connector)
    WRITE_NOTES,         // create / append notes
    // Reserved
    PUSH_HINTS,          // future relay, D1 Option B
}

// --- Auth ---
sealed interface AuthResult {
    data class Ok(val grantedScopes: Set<String>) : AuthResult
    data class NeedsUserConsent(val authUrl: Uri) : AuthResult
    data class Failed(val error: ConnectorError) : AuthResult
}

// --- Live refresh & history search ---
// budget: engine picks by network — 1.5s on Wi-Fi, 2.5s on cellular (decision Q2, fine-tune later)
data class RefreshQuery(val keywords: List<String>, val since: Instant?, val budget: Duration = 1.5.seconds)
// History search has a looser budget than refreshOne — it's an explicit, user-confirmed cold-tier lookup (§5.2)
data class HistoryQuery(val keywords: List<String>, val before: Instant? = null, val budget: Duration = 8.seconds)
sealed interface RefreshOutcome<out RAW> {
    data class Fresh<RAW>(val records: List<RAW>) : RefreshOutcome<RAW>  // engine runs these through normalize()
    data object FellBackToIndex : RefreshOutcome<Nothing>   // timed out / errored — answer from index, label stale
    data object Unsupported : RefreshOutcome<Nothing>
}

data class RevokeOptions(val purgeIndexedData: Boolean = true)
```

### 4.1 Error taxonomy — recoverable vs terminal (§8 token-expiry row)

The single most important distinction in the framework: the engine must know whether to **retry/refresh** or to **stop and ask the user**.

```kotlin
sealed interface ConnectorError {
    // Recoverable — engine retries with backoff or refreshes the token silently.
    data class RateLimited(val retryAfter: Duration?) : ConnectorError
    data class Transient(val cause: String) : ConnectorError          // 5xx, network blips
    data object TokenExpiredRefreshable : ConnectorError              // have a refresh token

    // Terminal — engine stops, marks the source stale, surfaces re-auth/disable UX.
    data object RevokedByUser : ConnectorError
    data class ScopeLost(val missing: Set<Capability>) : ConnectorError
    data object AccountGone : ConnectorError
    data class Permanent(val cause: String) : ConnectorError
}

/** syncDelta throws this on failure; the engine catches and branches recoverable vs terminal. */
class ConnectorException(val error: ConnectorError) : Exception()
```

`syncDelta` **throws** `ConnectorException(error)` on any failure (so a returned `SyncPage` always means success); `authenticate` surfaces the same set via `AuthResult.Failed(error)`. Either way the engine branches on recoverable (retry/refresh) vs terminal (re-auth/disable).

---

## 5. Sync-engine contract (what wraps the connector)

The engine — **not** the connector — implements every row of §8's failure/limit table. The connector just reports status and errors honestly; the engine reacts.

| Concern | Engine behavior |
|---|---|
| **Rate limits** | Per-connector **token-bucket**; honor `RateLimited.retryAfter`; **exponential backoff with jitter**. |
| **Backoff isolation** | Each connector has its own scheduler lane. One connector being limited/failed **never** delays queries answered from the index or other connectors' syncs. |
| **Partial sync** | Loop `syncDelta` while `page.hasMore`; persist `cursor` after every page so a kill resumes mid-backfill. Record per-source `lastSyncedAt` + `freshnessState`. |
| **Token expiry** | `TokenExpiredRefreshable` → silent refresh via `TokenStore`, then retry. Terminal errors → mark `stale`, never silently drop indexed data, raise re-auth. |
| **Scope drift** | After each `authenticate`, diff `capabilities()`; on `ScopeLost`, the planner **degrades** affected intents and tells the user, rather than returning wrong answers. |
| **Idempotency** | Upserts keyed by `sourceEntityId`; re-syncing a page is a no-op. Tombstones delete. |
| **Live refresh** | `refreshOne` runs only for time-critical intents, bounded; `FellBackToIndex`/`Unsupported` → answer from index, label "possibly stale". Must never block the response. |
| **Quota exhaustion** | Sustained limits → downgrade cadence + emit a user-visible freshness badge, not an error. |
| **Scheduling** | Android `WorkManager` (delta-only, batched, Wi-Fi+charging for heavy backfill); iOS `BGTaskScheduler` (coarser). Connectors are platform-agnostic; the **scheduler** is the platform layer. |

### 5.1 The `FreshnessSource` seam (D1)

Background freshness is abstracted so Option B (thin push relay) can be added later without touching connectors:

```kotlin
interface FreshnessSource { fun scheduleSync(id: ConnectorId, policy: SyncPolicy) }
class PollingFreshness(...) : FreshnessSource   // A — default, ships in MVP
// class PushFreshness(...) : FreshnessSource   // B — future; needs PUSH_HINTS + a relay (re-opens D1/D4)
```

`syncDelta`/`refreshOne` are identical under both, so adding push is additive (per D1's decision record).

### 5.2 Two-tier memory: hot index + cold history search (decision Q1)

The 90-day index cap (§5) keeps Otto fast, small, and battery-cheap — but the answer you need might be **ten years old**. Otto solves this with two tiers instead of indexing everything:

- **Hot tier — the local index (~90 days, configurable).** Always synced, instant, offline. Answers the overwhelming majority of queries.
- **Cold tier — provider-side full history.** Gmail, Microsoft Graph, and Slack all expose **server-side search over the user's entire history**. Otto doesn't store that decade locally; it reaches it **on demand** via `searchHistory()`.

**Planner flow on an index miss:**

> Index miss → *"I didn't find anything in the last 90 days. Want me to search your full Gmail history?"* → on **yes**, run `searchHistory()` over all history → return hits, and optionally **pin** them into the index so the follow-up is instant.

This keeps a tiny hot index while still reaching arbitrarily far back. Trade-offs to communicate: the cold path needs a **network round-trip**, only works while that source stays **connected**, and depends on the **provider's own search quality**. Connectors that can't search history (e.g. the notification feed) simply don't grant `SEARCH_HISTORY`, and the planner won't offer it.

---

## 6. Security & storage obligations

- Connectors **never** persist tokens or index rows directly. They receive tokens from `TokenStore` (backed by Keystore/Keychain) and emit `SyncRecord`s to the engine, which writes the encrypted index.
- `TokenStore` honors the §10 rule: Keystore keys are hardware-bound/non-backed-up; Keychain items use a `ThisDeviceOnly` accessibility attribute; the index DB is excluded from OS auto-backup.
- `normalize` must **redact nothing and infer nothing sensitive** (D4 special-category guardrail) — it maps as-is for explicit user queries; no profiling.
- A connector must declare, via `capabilities()`, exactly what it can read **and write**, so the Data-safety surface (§10) and the planner stay truthful. Write scopes (`WRITE_*`) are requested separately and gated by the §3.1 confirm-before-write contract.

---

## 7. Conformance suite (every connector must pass)

A shared test harness runs each connector against captured fixtures + a mock provider:

1. **Pure-normalize golden tests** — fixture `RAW` → expected `SyncRecord`s, including a tombstone case and a unicode/non-Latin name case (feeds D2 phonetic matching).
2. **Resumability** — kill after page _n_; resume from persisted cursor; assert no dupes, no gaps (idempotency).
3. **Error mapping** — provider 401/403/429/5xx/404 are thrown as `ConnectorException` with the correct `ConnectorError` variant (recoverable vs terminal).
4. **Backoff isolation** — a connector stuck in `RATE_LIMITED` does not delay another connector's sync or an index-answered query.
5. **Live-refresh budget** — `refreshOne` over budget returns `FellBackToIndex` within the timeout; never throws.
6. **History search** (if `SEARCH_HISTORY`) — `searchHistory` returns matches older than the hot-index window; results flow through `normalize` identically to sync; respects its budget.
7. **Write confirm + undo** (if any `WRITE_*`) — `write()` is reachable only after a confirm step in the harness; where reversal is supported, the returned `UndoHandle.token` drives `undo()` and the action is gone; a connector with no write scope rejects `write()`/`undo()` cleanly.
8. **Revoke+purge** — after `revoke(purgeIndexedData=true)`, no rows for that source remain.

---

## 8. Worked example — Gmail (Phase 1)

```kotlin
class GmailConnector(private val tokens: TokenStore) : Connector<GmailMessage> {
    override val id = ConnectorId("gmail")
    override val source = SourceType.EMAIL

    override fun capabilities() = CapabilitySet(setOf(
        Capability.READ_MESSAGES, Capability.READ_THREADS,
        Capability.LIVE_REFRESH, Capability.SEARCH_HISTORY,   // gmail search covers full history
        Capability.SEND_MESSAGE,                              // send/reply (confirm-gated, §3.1)
    ))

    override suspend fun authenticate(ctx: AuthContext): AuthResult { /* AppAuth OAuth, scoped */ }

    // historyId is the cursor; null = initial backfill via list+get, then deltas via history.list
    override suspend fun syncDelta(cursor: SyncCursor?): SyncPage<GmailMessage> { /* ... */ }

    override fun normalize(raw: GmailMessage): List<SyncRecord> = listOf(
        Upsert(Message(
            sourceEntityId = SourceEntityId(raw.id),
            source = id,
            timestamp = raw.internalDate,
            participants = raw.from + raw.to,
            body = raw.snippetOrBody,
            deepLink = Uri("https://mail.google.com/mail/u/0/#inbox/${raw.threadId}"),
            raw = mapOf("threadId" to raw.threadId, "labels" to raw.labelIds.joinToString(",")),
        ))
    )

    override suspend fun refreshOne(query: RefreshQuery): RefreshOutcome<GmailMessage> { /* one bounded history.list */ }

    // cold tier (§5.2): provider-side search over ALL history when the 90-day index misses
    override suspend fun searchHistory(query: HistoryQuery, cursor: SyncCursor?): SyncPage<GmailMessage> { /* messages.list?q=... */ }

    // write (§3.1): confirm-gated send/reply; undo = Gmail's send-undo window
    override suspend fun write(action: WriteAction): WriteResult { /* messages.send; return UndoHandle(expiresAt=+30s) */ }
    override suspend fun undo(token: String) { /* cancel within send-undo window / messages.delete */ }

    override suspend fun revoke(options: RevokeOptions) { /* token revoke + optional purge */ }
}
```

Push for Gmail (`users.watch` → Pub/Sub) is deliberately **not** here: it needs a server, which D1 rules out for MVP. It would arrive later as a `PushFreshness` source, not a connector change.

---

## 9. Resolved decisions (2026-06-20)

| # | Question | Decision |
|---|---|---|
| **Q1** | Backfill depth / data older than 90 days | **Two-tier memory** (§5.2): 90-day hot index (user-extendable) + on-demand provider-side `searchHistory()` for the cold tail, offered on an index miss. |
| **Q2** | `refreshOne` budget | **1.5s on Wi-Fi, 2.5s on cellular** (engine picks by network); fine-tune later. `HistoryQuery` gets a looser ~8s budget since it's explicit + user-confirmed. |
| **Q3** | Contacts: connector or ambient | **A normal `Connector`** that the entity resolver subscribes to — keeps the engine uniform, no special case. |
| **Q4** | Notification feed shape | **Buffer-draining `syncDelta`** — the OS callback fills a local queue the connector drains on the normal cadence (force-drained before a "what did I miss?" answer). One engine path; real-time indexing (a streaming sub-interface) isn't worth the second code path for an ask-based product. |
| **Q5** | Write scope | **Read *and* write** for **calendar, email, Slack, notes** — every write **confirm-gated** with undo where the provider allows it (§3.1). Writes never run on the always-on path. |

### Remaining to settle during build

- Per-source backfill **window override** UX (Q1) and whether pinned cold-tier hits count against the 90-day cap.
- Whether `searchHistory` results are **ephemeral** or auto-pinned into the index (default: pin, with retention cap).
- Undo-window specifics per provider (Gmail send-undo vs. Slack delete vs. irreversible-after-window email).

---

*Spec v0.1 — implements `otto.md` §8 against decisions D1/D4, the §10 secure-store rule, and the Q1–Q5 resolutions above. Next: lock the interface and scaffold the engine + Gmail connector.*
