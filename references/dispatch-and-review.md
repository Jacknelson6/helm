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

## Log lines (the scorecard)

One line per chunk, appended to the state file at accept/escalate time:

```
- chunk <n>: <model>, <k> attempt(s)[ (escalated to <model>: <one-clause why>)], <accepted|blocked>
```

The "why" clause matters: when the run doubles as a model evaluation, the
escalation reasons are the finding. "Kept missing the streaming routes" tells
you something about a model; "3 attempts" alone doesn't.
