# Routing: pick the cheapest loop that holds quality

Read at invocation, before the mode question. The advisor seat is a cost,
not a default: the session model's first job is to decide, in one cheap
pass, how much of itself the task actually needs, then give the rest away.

## The four routes

| Route | Shape | Session-model spend | When |
|---|---|---|---|
| 0 solo | just do it (or one small-tier dispatch); no state file, no loop | the edit itself | trivial: one file, one obvious change, orchestration would cost more than the work |
| 1 dispatch | one implementer dispatch + one diff review; no loop | ~10-30k | 1-2 chunks, bounded files, an existing sibling to copy, mechanical acceptance |
| 2 helm-lite | a strong-tier subagent runs the whole advisor loop; session model does triage, the completion condition, and the exit check only | ~15-40k | 2+ chunks of pattern-following work (routes, UI, tests, migrations that copy existing shapes) with acceptance checks that are commands and greps |
| 3 helm | the session model advises; the full SKILL.md loop | 30-60k plus 15-40k per chunk | any route-3 signal below |

Route-3 signals (ANY one keeps the session model in the advisor seat):

- novel architecture: no existing pattern in the repo to copy
- cross-cutting change to shared state, auth, money, or data-loss paths
- the completion condition needs judgment or a user interview mid-run
- prior attempts (any tier, any session) already failed on this task
- the run doubles as a model evaluation (the scorecard is the point)
- the user asked for the session model by name

Torn between 2 and 3: pick 2. A mis-routed run self-corrects through the
escalation below; a session-advised run never gets cheaper on its own.

## Triage discipline

Route from the REQUEST, not the code: no file reads, no scouts, before the
route is chosen. Reading the codebase to decide how to read the codebase is
the leak this file exists to close. If the request is too vague to route,
that vagueness IS the answer: route 3, because deriving the completion
condition will need the session model anyway.

Record the decision in the state file, under the Models line:

```
Route: <1|2|3> <dispatch|helm-lite|helm>: <one-clause why>
```

and `route=<dispatch|helm-lite|helm>` in the exit Totals line. Route 0
never appears in a state file (it skips the state file entirely); its only
trace is the one-line announcement to the user that SKILL.md requires.

## Route 2 mechanics: the delegated advisor

The session model:

1. Runs Step 0 itself (the completion condition; it may need the user).
2. Writes the state file skeleton: goal, completion check, out of scope,
   Models, Route. Chunking belongs to the delegated advisor.
3. Dispatches ONE strong-tier agent as the advisor using the template
   below. Everything in SKILL.md Step 1-2 (planning, chunking, dispatch,
   hostile diff review, the escalation ladder, the ledger) is that agent's
   job, at strong-tier rates instead of session rates.
4. On return: re-runs the completion check itself, literally and fully.
   Commands and greps only; never re-read the diffs (that review already
   happened one tier down). Green = recap and exit. Red = send the gap
   back to the SAME advisor agent once; a second red escalates per the
   section below.

Advisor-agent dispatch template (fill every slot):

```
You are the ADVISOR for a helm run: planner, dispatcher, reviewer. You do
not write the implementation yourself.

First read <skill-dir>/SKILL.md and
<skill-dir>/references/dispatch-and-review.md. Follow ONLY their Step 1
(plan), Step 2 (the loop), and the ledger rules; Steps -1, 0, and 3 and
the invocation question are the session model's job and are already done.
You never message the user directly. Bindings:
- state file: <repo>/.helm/<slug>.md. Goal, completion check, out of
  scope, Models, and Route are already written; you own chunking, the
  loop, the log lines, and the Totals line (with route=helm-lite).
- your implementer tier: <model>; mechanical tier: <model>.
- verify gate: <command>
- conventions that bind every chunk (quote them into each dispatch):
  <quoted from the project's CLAUDE.md / AGENTS.md>

You may dispatch subagents. If this harness refuses nested dispatch,
return immediately with the single line NESTED-DISPATCH-UNAVAILABLE; do
not fall back to building everything yourself.

Blockers (as SKILL.md Step 3 defines them) are NOT yours to resolve:
write the blocker to the state file and return BLOCKED plus the
narrowest possible question. When done: state file complete, Totals line
written, then return at most 20 lines: the recap and your own total
token count for the advisor-est field. Nothing else.
```

If the advisor agent returns NESTED-DISPATCH-UNAVAILABLE, the harness
cannot nest subagents: fall back to route 3 but keep route-2 spending
habits (review diffs only, no whole-file re-reads, scouts on cheap tiers).

## Escalation across routes

One-way, evidence-carrying, logged:

- route 0 that grows past a single bounded edit re-enters the skill at
  routing, now with evidence instead of a guess
- route 1 that sprouts extra chunks writes the state file and hands the
  remainder to a delegated advisor (route 2) by default, or stays
  session-advised (route 3) if a route-3 signal has appeared
- route 2 that returns BLOCKED twice on the same root cause, or red-gates
  twice at the session model's exit check, becomes route 3; the session
  model inherits the state file and log and continues, never restarts
- log every change (`- route change: <from> -> <to>: <one-clause why>`)
  and update the state file's Route: line to the final route so it never
  disagrees with the Totals line

The weekly ledger audit reads route= against the outcome: a helm-lite run
that escalated is fine (that is the mechanism working); a session-advised
run whose log shows only pattern-following chunks and zero judgment calls
was mis-routed, and the rubric above should be tightened.

## Why this beats "just ask the session model to build it"

Helm's overhead is m = 1.3-1.5x raw tokens (subagent system prompts,
re-reads). With an implementer r times cheaper than the session model and
advisor share a of total run tokens, a helm run costs m x (a + (1-a)/r)
of the solo cost in session-model-equivalent tokens. Helm wins while that
product stays under 1:

- r=5, m=1.4: wins while a < 0.64 (every measured savings run to date
  sits at 0.15-0.33)
- r=2, m=1.4: wins while a < 0.43
- r=1 (implementer = session tier): never wins on cost; that is a quality
  run, declare economics=quality and buy the independent review on purpose

Routing exists to push a from the measured ~0.25 session-advised average
toward ~0.03-0.05 on runs that never needed frontier judgment: route 2
swaps the session model's per-chunk review spend for strong-tier review
spend, which is the same rate arbitrage helm already applies to
implementation, applied one level up. Review quality holds because the
strong tier runs the identical hostile-review checklist, and the session
model still owns the two judgment-heavy ends: the completion condition
and the exit check.
