# Evaluate: what did helm actually save?

`/helm evaluate` turns the run ledgers into a measured answer: how much the
advisor/worker split saved, how each model tier performed, and what to change.
Read-only: it never dispatches build subagents, never edits profiles, and
never touches project code.

## 1. Collect

Parse every state file in the current repo's `.helm/` directory (the ledger
format is defined in references/dispatch-and-review.md). If the directory is
empty or missing, say so and stop; there is nothing to evaluate. If the user
names a different scope ("evaluate the last run", "across my repos"), honor
it, but default to this repo, all runs.

Per run, extract:

- the `Totals:` line: `impl-tokens`, `dispatches`, `scout-tokens`,
  `mech-tokens`, `advisor-est`, `leaked-scouts`, `economics`, `route`
- the `Models:` line (tiers, and `profile=` / `billing=` when present)
- per-agent log lines: model, attempts, escalations with their why-clauses,
  accepted/blocked, tokens
- hygiene gaps: missing Totals line, `tokens=unreported`, duplicate
  per-agent lines

## 2. Compute

Prices come from the model-intel snapshot (references/model-intel.md
section 2, blended $/1M); note the snapshot date in the output. For a model
not in the snapshot, say so rather than guessing its price.

Per savings run (`economics=savings`):

- worker tokens W = impl + scout + mech; total T = W + advisor-est;
  advisor share a = advisor-est / T
- actual cost in session-model-equivalent tokens:
  `advisor-est + sum(tokens_per_agent x price_agent_model / price_advisor_model)`
- solo baseline: T / m, with m = 1.4 (the measured 1.3-1.5x subagent
  overhead midpoint; state the assumption)
- saving = baseline minus actual, reported in session-equivalent tokens and
  as a percentage; convert to dollars at the advisor model's blended price.
  On a subscription lane, present dollars as "at API rates" context for the
  plan quota saved, not as cash; on an `api:` lane it is an actual spend
  estimate.

Per quality run (`economics=quality`): never claim savings. Report review
yield instead: rejections caught, escalations avoided or triggered, and the
extra cost over solo (the same formula, which will come out negative).

Aggregate across runs:

- per-model scorecard: dispatches, first-attempt acceptance rate,
  escalation count with the recurring why-clauses quoted, blocked chunks
- advisor share trend (should be flat or falling run over run)
- leaked scouts and hygiene gaps (these cap how much the numbers can be
  trusted; say so when they are nonzero)
- per-profile rollup when `profile=` appears: same stats grouped by profile

## 3. Report

Skim-first: a compact table (one row per run: date/slug, route, economics,
total tokens, advisor share, saving) then a 3-6 line verdict:

- headline: total measured saving across savings runs, in tokens and
  dollars, with the m and price assumptions named once
- best and worst performing tier, with the evidence (acceptance rates,
  escalation why-clauses)
- concrete suggestions, each one line with its evidence: a tier swap where
  a model repeatedly failed a seat or a frontier model now dominates it, a
  profile update (suggest only; the user edits via /helm profile), or a
  discipline fix (leaked scouts, missing token reports)

If every run is a quality run, lead with review yield and say plainly that
no cost-saving claim is possible from this data.
