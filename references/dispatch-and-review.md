# Dispatch prompts and review standards

Templates for Step 2. The advisor reads this before its first dispatch of a run.

## Dispatch prompt template

Every implementer prompt is self-contained; the subagent has no access to this
conversation. Fill every slot:

```
GOAL (context only, not your task): <one sentence, the run's goal>

YOUR CHUNK: <exactly what to build, and nothing else>

FILES: <exact paths to create/modify; name the closest existing sibling to
  copy patterns from>

CONVENTIONS THAT BIND YOU (quoted, do not go looking for more):
- <rule 1, quoted from the project's CLAUDE.md / AGENTS.md / design docs>
- <rule 2>

ACCEPTANCE CHECK (your definition of done): <the per-chunk check from the
  state file, exactly>

BEFORE RETURNING:
1. run <the project's verify/test gate command>
2. return: list of changed files, the gate's output verbatim, and any
   decision you made that wasn't specified above

DO NOT: touch files outside FILES, refactor adjacent code, add dependencies,
or commit.
```

A dispatch is well-formed when the implementer never needs to guess scope,
style, or "done". If you can't fill the ACCEPTANCE CHECK slot, the chunk isn't
ready to dispatch; split or re-plan it.

## Review checklist (advisor, per chunk)

Run against the actual `git diff`, in order. Reject on the first failure; don't
keep reading to be polite.

1. **Acceptance** — does the diff satisfy the chunk's acceptance check, verified
   by you (re-run the check, re-run the grep), not by the agent's word?
2. **Scope** — every changed hunk traces to the chunk. Drive-by refactors,
   formatting sweeps, or "improvements" = reject with the exact hunks to revert.
3. **Correctness** — read the diff as a hostile senior reviewer: edge cases
   (null/empty/unauthorized), error paths, async races, off-by-ones.
4. **Convention fit** — matches the quoted rules and the surrounding code's
   style; no new patterns where an existing one exists.
5. **Gate** — the verify output the agent returned is green AND you re-run it
   yourself after accepting (trust but verify).

## Feedback that works

When rejecting, send the implementer specifics, not sentiment:

- Bad: "this has some issues, please fix"
- Good: "REJECT. (1) app/api/x/route.ts:41 returns 200 on missing auth; must
  be 401 before any data access. (2) revert the rename in lib/utils.ts, out of
  scope. Keep everything else. Re-run the gate and return the new diff summary."

One rejection message per attempt; bundle every finding. Drip-feeding findings
across attempts burns the escalation ladder on your own review latency.

Slow is not failed. Before dispatching a presumed-retry for an agent that
hasn't returned, check its task status/output, or SendMessage the running
agent; never double-dispatch on latency alone. Measured: one redundant
verify dispatch spent 128k confirming a diff the original agent had already
finished.

## Log lines (the scorecard)

One line per DISPATCHED AGENT, appended to the state file. Chunks:

```
- chunk <n>: <model>, <k> attempt(s)[ (escalated to <model>: <one-clause why>)], <accepted|blocked>, tokens=<n>
```

Scouts (exploration, planning research) get their own line, same shape:

```
- scout: <purpose>, <model>, tokens=<n>
```

Rules that keep the ledger machine-auditable (a script parses these lines;
free-form prose here breaks the weekly audit):

- `tokens=` is the harness-reported total for that agent's whole
  conversation (Claude Code surfaces it in the task-completion
  notification). **One agent = one line.** If you iterate with the same
  agent or send it follow-up fixes, UPDATE that agent's existing line to the
  new total; never append a second line with a running "cumulative" count.
  Duplicate lines per agent double-count the run.
- The same update-in-place rule holds at run level: a REOPENED run updates
  the original `Totals:` line to the new totals (audit parsers read only the
  first `Totals:` line); never append a second "Totals (reopened leg):" line.
- If the harness reports nothing, write `tokens=unreported` rather than
  omitting the field, so gaps are visible instead of silent. That is a
  per-line state only: the exit `Totals:` line stays numeric, so a
  non-reporting worker tier (e.g. a codex harness) gets an explicit estimate
  with its basis stated on the line below (the ledger's cross-run median per
  dispatched agent works), never impl-tokens=0; audits read zero as a free
  run.
- A scout line whose model is the session model is a leak: append `LEAKED`
  to that line and count it in `leaked-scouts` at exit.
- Follow-up defects found at the gate or in e2e route back to the agent that
  owns that surface (SendMessage keeps its context) rather than a fresh
  dispatch; its line's token count absorbs the fix.

The "why" clause matters: when the run doubles as a model evaluation, the
escalation reasons are the finding. "Kept missing the streaming routes" tells
you something about a model; "3 attempts" alone doesn't.

At exit, append ONE machine-parseable totals line: single numbers only, no
ranges, no commas, no tildes. Prose commentary goes on the lines after it,
never inside it.

```
Totals: impl-tokens=<n> dispatches=<n> scout-tokens=<n> mech-tokens=<n> advisor-est=<n> leaked-scouts=<n> economics=<savings|quality> route=<dispatch|helm-lite|helm>
```

- `impl-tokens`: sum of mid and strong tier build dispatches; `dispatches`
  counts them.
- `scout-tokens`: sum of scout dispatches. `mech-tokens`: sum of small-tier
  mechanical dispatches. Zero when none.
- `advisor-est`: single-number estimate of the session model's own tokens
  (harnesses rarely report the main loop; pick the midpoint of your honest
  range). Rough guide: 30-60k for planning plus 15-40k per chunk reviewed,
  more if the advisor ran e2e itself.
- `economics`: `savings` when the implementer tier is materially cheaper
  than the advisor; `quality` when the user deliberately chose a strong-tier
  implementer. Declaring it lets the audit judge each run against its own
  purpose: savings runs on the split, quality runs on review yield (rejects
  caught, escalations avoided).
- `route`: the loop shape the run used (references/routing.md). On a
  helm-lite run, `advisor-est` is the DELEGATED advisor agent's reported
  token count (a real number, billed at strong-tier rates) plus the session
  model's small triage-and-exit overhead; note the split in prose when they
  differ materially.

This is what makes the cost claim auditable run over run: a run that cannot
show its split cannot prove helm saved anything. Healthy split for a savings
run: `advisor-est` at or below roughly 30% of the run's total tokens, and
falling as runs get bigger.
