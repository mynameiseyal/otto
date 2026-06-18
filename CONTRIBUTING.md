# Working on Otto

Solo project, branch-based workflow. **No Jira** — work is tracked by branches and PRs against `main`. (GitHub issues are optional; use them if you like.)

Conventions are enforced by the always-on rule `.cursor/rules/git-conventions.mdc` — that file is the source of truth; this doc just applies it to Otto.

## Branch naming

```
<type>/<issue#>-<short-slug>   # with a GitHub issue
<type>/<short-slug>            # no issue (the solo default)
```

Lowercase, hyphenated slug, no spaces. Allowed `<type>`: `feat, fix, refactor, perf, chore, docs, test, ci, build`.

Phase-0 validation gates are investigative, so they use **`chore/`** (decision write-ups that land in `otto.md` are committed as `docs:`).

Examples: `chore/d1-cloud-vs-zero-cloud`, `feat/gmail-connector`, `docs/intent-catalog`.

## Flow

1. Branch off `main`: `git switch -c <type>/<slug>`.
2. Commit in small, focused steps. **Conventional Commit titles**: `<type>: <imperative description>` (no issue number in the title), body explains *why*.
3. Push: `git push -u origin <branch>`.
4. Open a PR into `main`. If it resolves a GitHub issue, put `Closes #<issue>` in the body. Self-review, merge (squash). Delete the branch after merge.

## Phase-0 gates (must close before Phase 1)

Tracked as `chore/` branches, each ending in a short decision note appended to `otto.md` `§13.5`:

| Gate | Branch | Exit = |
|---|---|---|
| D1 Cloud vs zero-cloud | `chore/d1-cloud-vs-zero-cloud` | A / B / C posture committed |
| D2 Proper-noun voice | `chore/d2-proper-noun-voice` | Ship vs de-scope person intents |
| D3 Notification connector | `chore/d3-notification-connector` | Core / opt-in / cut |
| D4 Third-party data legal | `chore/d4-third-party-data-legal` | Go / constrain / no-go |

Resolve **D1 first** — it cascades into the other three.
