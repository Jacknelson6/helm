# What success looks like

Worked examples for Step 0. The advisor reads this when deriving a completion
condition, and holds its own condition to this bar before starting work.

## The bar

A completion condition is good when a stranger with repo access could run it and
get the same pass/fail answer you would. It has three properties:

- **Executable or observable** — a command, a rendered state, or a greppable fact.
- **Total** — covers the whole request, not the easy half of it.
- **Bounded** — an explicit out-of-scope list, so "done" can't drift.

## Worked examples by task type

### Feature

> Request: "add rate limiting to all public API routes"

```
Completion check:
  1. `npm test rate-limit` exits 0
  2. every route file under app/api/public/ imports and wraps withRateLimit
     (grep -rL "withRateLimit" app/api/public/ returns empty)
  3. manual probe: 61 requests in 60s to any one route returns 429 on the 61st
Out of scope: admin routes, auth endpoints, per-user (vs per-IP) limits
```

### Bug fix

> Request: "fix the crash when a user has no organization"

```
Completion check:
  1. new regression test reproducing the crash (user with organization_id=null)
     exists and passes
  2. the original failing flow (login -> /dashboard) renders without a 500,
     verified in the running app
  3. full test suite still green
Out of scope: backfilling null org rows in the DB
```

### Refactor / migration

> Request: "migrate all date inputs to the shared DatePicker"

```
Completion check:
  1. `grep -rn 'type="date"' app/ components/` returns empty
  2. every replaced call site renders (route-by-route smoke pass or e2e run)
  3. verify gate green
Out of scope: the DatePicker component's own API, date formats in API payloads
```

### UI

> Request: "make the settings page match the new design"

UI "matches the design" is not verifiable as stated, so decompose it into an
enumerable checklist during Step 0 (interview the user against the mock if needed):

```
Completion check, all of:
  1. page uses only design-system tokens (no hardcoded hex/px, lint rule green)
  2. sections render in mock order: profile, notifications, billing, danger zone
  3. each section uses the shared Card + SectionHeader primitives (greppable)
  4. screenshot at 1280w and 375w shows no overflow or clipped controls
Out of scope: dark/light theme work beyond existing tokens, copy changes
```

### Performance

> Request: "the dashboard is slow, speed it up"

Never accept "faster". Pin a number first, measuring the baseline yourself:

```
Baseline (measured before work): /dashboard TTFB 2.4s, 14 sequential queries
Completion check:
  1. TTFB under 800ms on the same fixture data (measured the same way)
  2. query count for the page <= 4 (logged)
  3. verify gate green, no feature behavior changed (existing e2e green)
Out of scope: CDN/caching infra, other pages
```

## Invalid conditions (never accept these)

| Stated condition | Why it fails | Fix |
|---|---|---|
| "the code is cleaner" | not observable | name the lint/complexity rule that must pass |
| "works on my machine" | not reproducible | name the command and fixture |
| "all the pages look right" | not enumerable | list the pages and the per-page check |
| "no more errors" | unbounded | name the error, the flow that raised it, and the test that proves it's gone |
| "improved performance" | no baseline | measure first, then set a numeric target |

## A gold-standard state file

```markdown
# Helm: rate-limit public routes
Goal: every public API route enforces per-IP rate limiting
Completion check: `npm test rate-limit` green AND grep shows all 12 routes
  wrapped AND manual 61-request probe returns 429
Out of scope: admin routes, auth endpoints, per-user limits
Models: advisor=opus-4-8 implementer=sonnet escalation=opus  (mode: auto)
Route: 3 helm: cross-cutting change to shared auth-adjacent middleware
Status: complete
## Chunks
- [x] 1. middleware + config — acceptance: unit tests pass
- [x] 2. wire 12 routes — acceptance: grep shows all 12 wrapped
- [x] 3. e2e probe harness — acceptance: 429 observed on each route
## Log
- chunk 1: sonnet, 1 attempt, accepted
- chunk 2: sonnet, 3 attempts (escalated to opus: kept missing the two
  streaming routes), accepted
- chunk 3: sonnet, 1 attempt, accepted
- completion check: all three legs pass, marked complete
```

Success for the run = every leg of the completion check passing, the log telling
a true story of how it got there, and nothing outside scope touched.
