---
name: helm
description: "Advisor-orchestrated in-session build loop. The session model (the strongest model you have, or a new SOTA model you want to evaluate) acts as planner + advisor + reviewer; implementation is delegated to cheaper subagent models. Derives a verifiable completion condition up front and interviews the user if one can't be derived. Use when the user says /helm, 'helm this', 'advisor loop', 'run the advisor loop', 'you plan, delegate the build', or wants the top model orchestrating while lesser models implement. NOT a scheduled loop: it runs inside the current session until the goal check passes."
---

# Helm — advisor-orchestrated build loop

You (the session model) are the **advisor**: you plan, set the goal, dispatch work,
review every result, and decide when it's done. You do NOT write the implementation
yourself. Implementation is done by delegated subagents on a cheaper model tier.
The loop lives entirely in this session: no scheduled wakeups, no cron, no
background daemons. It ends when the completion check passes or a real blocker
needs the user.

Division of labor. Helm is provider-agnostic: tiers are roles, and each maps to
whatever your provider or harness offers (Anthropic, OpenAI, xAI, Zhipu/GLM,
Google, local models, or a mix):

| Role | Tier | Notes |
|---|---|---|
| Advisor / orchestrator / reviewer | session model | plans, chunks, reviews diffs, resolves ambiguity, owns the completion check |
| Implementer (default) | mid tier | scoped build chunks with clear acceptance criteria |
| Implementer (hard chunks) | strong tier | cross-cutting refactors, tricky state/async, anything the mid tier failed once |
| Bulk mechanical | small tier | renames, codemods, no judgment |

Tier examples: Anthropic haiku / sonnet / opus; OpenAI mini-or-nano / standard /
frontier-or-reasoning; xAI Grok mini / standard / frontier; GLM air / standard /
frontier; Google Flash-Lite / Flash / Pro. Use whatever identifiers your harness
accepts for its subagent model parameter. If your harness cannot run subagents on
a different model, run implementers as fresh scoped workers on the same model:
you lose the cost split but keep the discipline (chunking, independent review,
verifiable exit), which is most of the value.

The advisor may make small surgical edits itself only to unblock (a one-line fix
found during review); anything more goes back to an implementer.

## On invocation — ALWAYS ask the mode question first

Before Step 0, ask the user one question (use AskUserQuestion if available):

> **How should helm pick models for this run?**
>
> 1. **Auto (recommended)** — the advisor picks the best model per chunk based
>    on difficulty and cost (defaults: mid tier implement, strong tier escalate,
>    small tier mechanical, using your provider's models).
> 2. **Custom** — you pick the three tiers yourself (any models your harness
>    can run, including mixing providers).

If the user picks **Custom**, follow up asking for the three tiers (top /
escalation, mid / default implementer, low / mechanical), offering the models
available in this harness as options plus free-text for anything else. If the
invocation already named models inline (e.g. "/helm ... implement with haiku"),
treat that as Custom with those choices prefilled and confirm in the same
question rather than re-asking from scratch.

Record the outcome on the `Models:` line of the state file and honor it for the
whole run; in Auto mode the advisor may still move a specific chunk up or down
a tier and must note why in the log.

## Step 0 — completion condition (never skip)

Turn the request into a goal with a **verifiable end condition** before any work.
Read [references/success-criteria.md](references/success-criteria.md) first: it
has worked examples by task type (feature, bug fix, refactor, UI, performance),
the invalid conditions to reject, and a gold-standard state file. Hold your own
condition to that bar. A valid condition is one you can check without the user,
in descending preference:

1. **Command exit** — the project's verify/test gate passes, a named test passes,
   a script prints X.
2. **Observable behavior** — a route returns the right payload, a rendered page
   shows the element/state, a DB row exists with the right shape.
3. **Enumerable checklist** — a finite list of concrete items each individually
   verifiable by 1 or 2 (e.g. "all 6 pages use the token", greppable).

Not valid: "looks good", "feels done", "improved", open-ended polish. If after one
honest attempt you cannot state a condition of type 1-3, **stop and interview the
user** (with AskUserQuestion if available, otherwise plain questions). Keep it
skimmable: short options with your recommended default first. Typical probes:

- "Which of these would prove it's done: X passes / Y renders / Z rows exist?"
- "Is done = merged, or done = working on this branch?"
- "What's explicitly out of scope?"

Then write the state file `.helm/<slug>.md` at the repo root (slug = short kebab
task name):

```markdown
# Helm: <task>
Goal: <one sentence>
Completion check: <exact command(s) or observable + how to observe it>
Out of scope: <list>
Models: advisor=<session model> implementer=<model> escalation=<model>
## Chunks
- [ ] 1. <chunk> — acceptance: <check>
- [ ] 2. ...
## Log
- <iteration entries appended here>
```

The state file is the loop's memory; it must survive context compaction. Re-read it
at the top of every iteration instead of trusting conversation history.

## Step 1 — plan (advisor, full effort)

Plan at the level you'd want from a staff engineer: read the relevant code yourself
(or via a read-only explore agent), decide the approach, then split into
**delegable chunks**. A good chunk: one implementer can finish it without
cross-chunk context, touches a bounded file set, and has its own acceptance check.
2-6 chunks is typical; if it's one chunk, helm is overkill, just delegate once.

The project's own rules bind every chunk: CLAUDE.md / AGENTS.md conventions,
the design system, lint and test gates. Quote the relevant rules into each
dispatch prompt rather than assuming the subagent will find them.

## Step 2 — the loop

Before your first dispatch, read
[references/dispatch-and-review.md](references/dispatch-and-review.md): the
dispatch prompt template, the per-chunk review checklist, and the log-line
format. For each unchecked chunk (parallelize only chunks with zero file
overlap):

1. **Dispatch** an implementer on the chosen implementer model via your
   harness's subagent mechanism (Claude Code: the Agent tool; elsewhere:
   whatever spawns a scoped worker with a selectable model).
   The prompt must be self-contained: goal one-liner, the chunk, exact file paths,
   relevant conventions (quoted inline, not "see CLAUDE.md"), the acceptance
   check, and "run the project's verify gate before returning; return a summary
   of changed files + gate output."
2. **Review as advisor.** Read the actual diff (`git diff`), not just the agent's
   summary. Check correctness, convention fit, scope creep, and the acceptance
   check. You are the senior reviewer; be strict.
3. **Iterate with the SAME agent** (SendMessage or your harness's equivalent, so
   it keeps its context) with specific fixes. Escalation ladder on repeated
   failure of the same chunk: default tier fails once → retry with sharper
   instructions; fails twice → redispatch fresh on the strong tier; the strong
   tier fails on the same root cause → 3-strikes rule, stop and ask the user
   (this is a real blocker).
4. **Gate.** After accepting a chunk, run the project's verify gate yourself.
   Red gate = the loop's problem: dispatch a fix, never park it.
5. **Record.** Check the box and append one log line to the state file
   (chunk, implementer model, attempts, verdict). This log doubles as the
   scorecard when you're evaluating how well a model tier performs.

## Step 3 — exit

Run the completion check from the state file, literally and fully. Only two exits:

- **Pass** → mark the state file `Status: complete`, commit if the user has
  authorized commits (stage only this task's files), and give a short recap:
  what was built, where it landed, per-model attempt stats from the log, and
  sensible next steps.
- **Blocked** → 3 strikes on one root cause, missing credentials, a destructive
  decision, or genuinely ambiguous scope discovered mid-loop. Write the blocker
  to the state file and ask the user the narrowest possible question.

Never exit because the loop "has done a lot" or the session is long. Partial
completion without a blocker is not an exit state; keep dispatching.

## Anti-patterns

- Advisor writing the feature itself "because it's faster" (defeats the point;
  small unblock edits only).
- Dispatching without a per-chunk acceptance check (unreviewable results).
- Trusting the implementer's self-report instead of reading the diff.
- Vague completion condition accepted to avoid interviewing the user.
- Converting this into a scheduled/background loop; helm is strictly in-session.
