# Working on Otto

Solo project, branch-based workflow. **No Jira / issue tracker** — work is tracked by branches and PRs against `main`.

## Branch naming

```
<type>/<short-kebab-summary>
```

Types:

- `spike/` — Phase-0 validation / decision gates (see `§13.5` in `otto.md`)
- `feat/` — new capability
- `fix/` — bug fix
- `docs/` — docs / plan changes
- `chore/` — tooling, deps, scaffolding

Examples: `spike/d1-cloud-vs-zero-cloud`, `feat/gmail-connector`, `docs/intent-catalog`.

## Flow

1. Branch off `main`: `git switch -c spike/<name>`.
2. Commit in small, focused steps; imperative subject line (≤72 chars), body explains *why*.
3. Push: `git push -u origin <branch>`.
4. Open a PR into `main`, self-review, merge (squash). Delete the branch after merge.

## Phase-0 gates (must close before Phase 1)

Tracked as `spike/` branches, each ending in a short decision note appended to `otto.md` `§13.5`:

| Gate | Branch | Exit = |
|---|---|---|
| D1 Cloud vs zero-cloud | `spike/d1-cloud-vs-zero-cloud` | A / B / C posture committed |
| D2 Proper-noun voice | `spike/d2-proper-noun-voice` | Ship vs de-scope person intents |
| D3 Notification connector | `spike/d3-notification-connector` | Core / opt-in / cut |
| D4 Third-party data legal | `spike/d4-third-party-data-legal` | Go / constrain / no-go |

Resolve **D1 first** — it cascades into the other three.
