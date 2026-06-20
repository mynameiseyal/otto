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

    /** Revoke access; optionally purge this source's rows from the index. */
    suspend fun revoke(options: RevokeOptions)
}
```

Design notes:
- **Generic `RAW`** keeps `syncDelta` and `normalize` honest: the connector owns its wire type; the engine only ever sees `SyncRecord`. `normalize` is **pure** so it is trivially unit-testable against captured fixtures.
- **`syncDelta` is one page.** The engine loops it while `page.hasMore`, persisting `page.cursor` between calls. This gives resumable backfill and incremental delta with the same method. Failures don't ride in the page — they're **thrown** (§4.1), so a `SyncPage` always means success.
- **No method blocks on network in the query path** except `refreshOne`, which is explicitly bounded.

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
    READ_MESSAGES, READ_THREADS, READ_EVENTS, WRITE_EVENTS, READ_NOTES,
    READ_CONTACTS, LIVE_REFRESH, PUSH_HINTS /* reserved for future relay, D1 Option B */
}

// --- Auth ---
sealed interface AuthResult {
    data class Ok(val grantedScopes: Set<String>) : AuthResult
    data class NeedsUserConsent(val authUrl: Uri) : AuthResult
    data class Failed(val error: ConnectorError) : AuthResult
}

// --- Live refresh ---
data class RefreshQuery(val keywords: List<String>, val since: Instant?, val budget: Duration = 1.5.seconds)
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

---

## 6. Security & storage obligations

- Connectors **never** persist tokens or index rows directly. They receive tokens from `TokenStore` (backed by Keystore/Keychain) and emit `SyncRecord`s to the engine, which writes the encrypted index.
- `TokenStore` honors the §10 rule: Keystore keys are hardware-bound/non-backed-up; Keychain items use a `ThisDeviceOnly` accessibility attribute; the index DB is excluded from OS auto-backup.
- `normalize` must **redact nothing and infer nothing sensitive** (D4 special-category guardrail) — it maps as-is for explicit user queries; no profiling.
- A connector must declare, via `capabilities()`, exactly what it can read, so the Data-safety surface (§10) and the planner stay truthful.

---

## 7. Conformance suite (every connector must pass)

A shared test harness runs each connector against captured fixtures + a mock provider:

1. **Pure-normalize golden tests** — fixture `RAW` → expected `SyncRecord`s, including a tombstone case and a unicode/non-Latin name case (feeds D2 phonetic matching).
2. **Resumability** — kill after page _n_; resume from persisted cursor; assert no dupes, no gaps (idempotency).
3. **Error mapping** — provider 401/403/429/5xx/404 are thrown as `ConnectorException` with the correct `ConnectorError` variant (recoverable vs terminal).
4. **Backoff isolation** — a connector stuck in `RATE_LIMITED` does not delay another connector's sync or an index-answered query.
5. **Live-refresh budget** — `refreshOne` over budget returns `FellBackToIndex` within the timeout; never throws.
6. **Revoke+purge** — after `revoke(purgeIndexedData=true)`, no rows for that source remain.

---

## 8. Worked example — Gmail (Phase 1)

```kotlin
class GmailConnector(private val tokens: TokenStore) : Connector<GmailMessage> {
    override val id = ConnectorId("gmail")
    override val source = SourceType.EMAIL

    override fun capabilities() = CapabilitySet(
        setOf(Capability.READ_MESSAGES, Capability.READ_THREADS, Capability.LIVE_REFRESH)
    )

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
    override suspend fun revoke(options: RevokeOptions) { /* token revoke + optional purge */ }
}
```

Push for Gmail (`users.watch` → Pub/Sub) is deliberately **not** here: it needs a server, which D1 rules out for MVP. It would arrive later as a `PushFreshness` source, not a connector change.

---

## 9. Open questions (to settle before/while building)

1. **Backfill depth default** — initial sync window (the §5 90-day cap) per source, and whether the user can extend it.
2. **`refreshOne` budget** — is 1.5s right on cellular? Per-source override?
3. **Contacts as connector vs. ambient** — Contacts feeds entity resolution for *all* connectors; does it implement `Connector` or sit beside the engine as a resolver input? (Leaning: a `Connector` that the resolver subscribes to.)
4. **Notification connector shape** — it's event-pushed by the OS, not pulled; does it implement `syncDelta` (drains a buffer) or a separate `StreamingConnector` sub-interface? (Leaning: a buffer-draining `syncDelta` so the engine stays uniform.)
5. **Write capabilities** — `WRITE_EVENTS` (create/respond to calendar events) is listed but Phase 1 is read-only; confirm defer.

---

*Spec v0.1 — implements `otto.md` §8 against decisions D1/D4 and the §10 secure-store rule. Argue with the interface, then we lock it and scaffold the engine + Gmail connector.*
