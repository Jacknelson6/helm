```
 _          _
| |__   ___| |_ __ ___
| '_ \ / _ \ | '_ ` _ \
| | | |  __/ | | | | | |
|_| |_|\___|_|_| |_| |_|
```

An [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) for Claude Code that turns your session into an **advisor-orchestrated build loop**: the strongest model you have plans, reviews, and decides when the work is done, while cheaper models do the actual implementation.

## Why

Two reasons to run this way:

1. **Cost-shaped quality.** Frontier models are at their best planning, decomposing, and reviewing. Most implementation chunks don't need that horsepower. Helm keeps the expensive tokens on judgment and spends cheap tokens on typing.
2. **Testing new SOTA models as orchestrators.** When a new top model ships, the interesting question is rarely "can it write a React component" but "can it run a team." Put the new model at the helm, hold the implementation tier constant (say, Sonnet), and the loop's state file gives you a per-chunk scorecard: how many attempts each chunk took, how often it had to escalate, whether its completion condition held up. Swap the advisor, re-run, compare.

The loop is **goal-driven, not vibe-driven**: before any work starts, the advisor must derive a verifiable completion condition (a command that passes, an observable behavior, or an enumerable checklist). If it can't, it interviews you until one is pinned. No "looks done to me" exits.

## How it works

```
you: /helm add rate limiting to all public API routes

advisor (session model):
  ?. ask: auto or custom models?        auto = advisor picks tiers; custom = you pick top/mid/low
  0. derive completion condition        e.g. "all 12 routes in app/api/public/*
     (interview you if it can't)         return 429 after N req/min; verify gate green"
  1. plan + split into delegable chunks, write state file
  2. loop per chunk:
       dispatch  -> implementer subagent (default: sonnet)
       review    -> advisor reads the real git diff, not the agent's summary
       iterate   -> follow-ups to the same agent; escalate to opus after 2 failures
       gate      -> project verify/test gate must be green
       record    -> one log line per attempt in the state file
  3. exit only when the completion check passes, or a real blocker needs you
```

Everything runs **inside the current session**. No cron, no scheduled wakeups, no background daemons. State lives in `.claude/helm/<task-slug>.md` so the loop survives context compaction and long sessions.

## Install

**Global** (available in every project):

```bash
git clone https://github.com/Jacknelson6/helm-skill ~/.claude/skills/helm
```

**Per-project** (checked into the repo, shared with your team):

```bash
git clone https://github.com/Jacknelson6/helm-skill .claude/skills/helm
rm -rf .claude/skills/helm/.git
```

Then in Claude Code:

```
/helm <your task>
```

or just describe it: "run the advisor loop on X", "you plan, delegate the build".

## Choosing models

Every run starts with one question:

> **How should helm pick models for this run?**
> 1. **Auto (recommended)** — the advisor picks the best model per chunk based on difficulty and cost
> 2. **Custom** — you pick the top (escalation), mid (implementer), and low (mechanical) tiers yourself

The advisor is always whatever model your session runs on. In Auto mode, implementers default to:

| Tier | Default | Used for |
|---|---|---|
| Advisor | session model | planning, review, escalation decisions, the completion check |
| Implementer | `sonnet` | standard chunks |
| Escalation | `opus` | chunks the default tier failed twice |
| Mechanical | `haiku` | renames, codemods, bulk edits |

You can also name models inline; helm treats that as Custom with your choices prefilled:

```
/helm migrate the settings pages to the new form kit, implement with haiku, escalate to sonnet
```

To evaluate a new SOTA model as the orchestrator, start your session on that model (`/model`), keep the implementation tier fixed, and compare the `## Log` sections of the state files across runs.

## The state file

Each run writes `.claude/helm/<slug>.md`:

```markdown
# Helm: rate-limit public routes
Goal: every public API route enforces per-IP rate limiting
Completion check: `npm test rate-limit` green AND all 12 routes return 429 in the harness
Out of scope: admin routes, auth endpoints
Models: advisor=opus-4-8 implementer=sonnet escalation=opus
## Chunks
- [x] 1. middleware + config — acceptance: unit tests pass
- [x] 2. wire 12 routes — acceptance: grep shows all 12 wrapped
- [ ] 3. e2e harness — acceptance: 429 observed on each route
## Log
- chunk 1: sonnet, 1 attempt, accepted
- chunk 2: sonnet, 3 attempts (escalated to opus), accepted
```

Gitignore `.claude/helm/` if you don't want run logs in your repo; keep them if you're benchmarking.

## Design notes

- **The advisor never implements.** Small one-line unblock fixes only. If the top model writes the feature itself, you've measured nothing and spent everything.
- **Diffs over self-reports.** The advisor reviews `git diff`, not the subagent's summary of what it did.
- **3-strikes escalation.** Default tier fails once → sharper retry. Twice → fresh dispatch on the escalation tier. Escalation tier fails on the same root cause → stop and ask the human. That's a real blocker; type errors and red tests are not.
- **Interview, don't guess.** An unverifiable goal is the loop's only unrecoverable input, so it's the one thing the skill is required to push back on.

## License

MIT
