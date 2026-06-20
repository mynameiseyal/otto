# Otto — Privacy Policy (draft v0.1)

> ⚠️ **Draft — not yet legal-ratified.** This document restates the privacy posture decided in the Phase-0 gates (`otto.md` §13.5, decisions D1/D3/D4/D5) in user-facing language. It is the artifact counsel/DPO and the Play release-compliance owner review before public release. Treat as the *recommended* policy text; the human sign-offs in D3 (compliance) and D4/D5 (counsel/DPO) remain outstanding.

**Last updated:** 2026-06-20 · **Applies to:** Otto for Android (MVP). iOS noted where it differs.

---

## 1. The one-paragraph version

Otto is a personal assistant that runs **entirely on your device**. It connects to apps and accounts *you* choose (email, calendar, Slack, contacts, notes), keeps a **local, encrypted index** of that data on your phone, and answers your questions from that index using on-device code. **Otto has no backend server** — we (the developer) never receive, store, or see your content or who you talk to. The only time data leaves your device is if **you** turn on the optional **Assistant mode**, which sends the specific context you approve to **your own** AI provider (Claude/OpenAI/Google) using **your own** API key. You can revoke any connection, purge your data, or export it at any time.

---

## 2. Who is responsible for your data

- **Otto runs on your device, under your control.** Because there is **no Otto server** (decision D1 — pure on-device), the developer is **not a controller of your indexed content**: we never receive it. We are responsible only for the app software itself and for the minimal, content-free diagnostics described in §8.
- **Your connected accounts are your own relationships.** When Otto syncs Gmail, Calendar, Slack, etc., it uses *your* OAuth tokens to pull data *you already lawfully hold*, directly from those providers to your device. Those providers' own privacy terms govern that data (decision D4).
- **Your AI provider (Assistant mode only) is your own relationship too.** In Assistant mode, Otto calls the provider with *your* API key; that provider processes the request under *its* terms, which you accept with it directly (decision D5).

> **Legal basis (recommended posture, pending ratification):** your use of Otto to organize data you already hold, on your own device, with no onward sharing, falls under the **GDPR household exemption** (Art. 2(2)(c)) — materially the same as the email or notes app you already use. See `otto.md` §13.5 D4 for the full reasoning.

---

## 3. What Otto accesses, and where it goes

| Source | What Otto reads | Where it goes |
|---|---|---|
| **Gmail / Outlook (M365)** | Messages, metadata, labels (via your OAuth scopes) | Your device only — local index |
| **Google / Apple Calendar** | Events, participants, times | Your device only — local index |
| **Contacts** | Names, identifiers (for matching "who" in your queries) | Your device only — local index |
| **Slack** | Messages/channels you grant scopes to | Your device only — local index |
| **Notion / OneNote** (later) | Notes/pages you grant access to | Your device only — local index |
| **Notification Recap** (opt-in, Android) | Incoming notifications **only from apps you pick** | Your device only — local index |
| **Assistant mode** (opt-in) | The **specific index slice relevant to your request** | **Leaves your device** to *your* chosen AI provider, via *your* key |

**Default state:** nothing leaves your device. Only Assistant mode (§5) causes egress, and only for requests you initiate, with the slice shown to you.

---

## 4. Notification Recap (optional, Android only)

If you enable it (it is **off by default**):

- Otto reads incoming notifications **only from the apps you explicitly select** (default: none), so it can answer "what did I miss?".
- **Processing is entirely on-device** — notification content is **never uploaded or shared** (decision D3, consistent with D1).
- **Forward-only:** Otto cannot and does not read notification *history* — only notifications that arrive after you enable it.
- **Sensitive content:** on **Android 15+**, Otto (a standard third-party listener) **cannot see one-time passcodes or other sensitive notifications** — the system withholds them. On **Android 14 and below the OS does not withhold them**, so if you enable Notification Recap for an app that posts OTPs or other sensitive content, those notifications can be read and indexed **locally on your device**. On older OS versions we recommend not including such apps; we will surface this caveat in the enable flow.
- You can change which apps are included, or turn it off entirely, at any time.

---

## 5. Assistant mode (optional, bring-your-own-key)

Otto's core answers come from on-device code with **no AI model and no egress**. For open-ended requests the core can't handle ("compile a report," "find everything across my apps about X"), you may enable **Assistant mode** (off by default):

- **You bring your own API key** for your chosen provider (Claude/OpenAI/Google). Otto does **not** host or proxy a model and has **no shared key** — the call goes **directly from your device to your provider** (decision D5).
- **Only the index slice relevant to your request is sent** — never your whole index. Otto shows you what is being sent, and a clear **"this leaves your device to {provider}"** badge distinguishes these answers from on-device ones.
- **The provider is your processor, under its terms.** How your provider handles API requests (including any training/retention policy) is governed by the agreement *you* have with them. The **developer/API terms** that BYO-key use falls under **typically exclude your data from model training** — unlike the consumer web apps (ChatGPT, the Claude web app) — though this varies by provider and your account settings. Otto links you to those terms; it does not warrant them, so check your provider's current policy.
- Your API key is stored in the device secure store (Android Keystore / iOS Keychain) and is never transmitted to the developer.

---

## 6. How your data is stored

- **Local, encrypted index** on your device (SQLCipher / OS file protection). It is the only copy Otto maintains.
- **No OS cloud backup of your data.** Otto **disables OS-level automatic backups** of its index and tokens (Android Auto Backup excluded via backup rules / `allowBackup=false`; iOS files excluded from iCloud/iTunes backup). This keeps the zero-cloud promise intact — your indexed data is **never** copied to Google Backup or iCloud. The only off-device copy is the **encrypted export you create yourself** (§7).
- **Tokens and keys** (OAuth tokens, your AI API key) live in the OS secure store (Keystore/Keychain), never in the developer's hands.
- **Retention caps:** stored history is capped (configurable, e.g. 90 days) and purgeable per source.

---

## 7. Your controls

- **Revoke any connector** — disconnect a source; its indexed data is removed.
- **Purge** — delete Otto's entire local index from your device.
- **Export** — produce an encrypted export to your own storage (Otto offers no managed cloud backup — there is no Otto server to hold it).
- **Toggle the optional features** — Notification Recap and Assistant mode are independently on/off.

---

## 8. Diagnostics (what, if anything, we collect)

- Otto is designed for **content-free, PII-redacted, local-capable telemetry only**. We do not collect your messages, contacts, calendar, notifications, or queries.
- For Google Play **Data safety**, this means notification/message content and app-activity content are declared **not collected and not shared** (they are processed on-device only) — see the mapping in `otto.md` §13.5 D3.

---

## 9. Special-category and children's data

- **Special-category data** (Art. 9 — health, beliefs, etc.) may incidentally appear in your messages. Otto indexes and retrieves it **as-is for your explicit queries only** and performs **no special-category profiling or inference** (hard design rule, decision D4). The no-LLM core builds no sensitive profiles; Assistant mode only sees the slice you approve.
- **Children:** Otto is **not directed at children**. Terms of Service set an **age floor of 16** (aligned to the GDPR digital-consent age). Incidental third-party data in your messages is covered by the same household-exemption logic.

---

## 10. Changes that would re-open this policy

This posture rests on Otto staying **zero-cloud**. We will revise this policy and seek fresh review **before** shipping any of:

1. An **Otto-hosted or proxied** AI model, a shared/managed key, or any backend that receives your content or metadata (would make the developer a processor/controller — re-opens D1, D4, D5).
2. Sending content to any provider **without** your explicit per-request, approved-context consent.
3. **Proactive** sending of your data (Assistant mode acting without you initiating).

---

## 11. Contact

Questions or requests about this policy: **{privacy contact — TODO before publication}**.

---

*Draft v0.1 — restates `otto.md` §13.5 (D1/D3/D4/D5) for users. Open TODOs before publication: privacy contact (§11), final retention default (§6), and the human sign-offs (compliance for D3; counsel/DPO for D4/D5).*
