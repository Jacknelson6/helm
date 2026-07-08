```
 _          _
| |__   ___| |_ __ ___
| '_ \ / _ \ | '_ ` _ \
| | | |  __/ | | | | | |
|_| |_|\___|_|_| |_| |_|
```

A provider-agnostic [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that turns your coding-agent session into an **advisor-orchestrated build loop**: the strongest model you have plans, reviews, and decides when the work is done, while cheaper models do the actual implementation. Works with any model family (Anthropic, OpenAI, xAI/Grok, GLM, Gemini, local) and any harness that can spawn scoped subagents; first-class in Claude Code.

## Why

Two reasons to run this way:

1. **Cost-shaped quality.** Frontier models are at their best planning, decomposing, and reviewing. Most implementation chunks don't need that horsepower. Helm keeps the expensive tokens on judgment and spends cheap tokens on typing.
2. **Testing new SOTA models as orchestrators.** When a new top model ships (a new GPT, Grok, GLM, Gemini, or Claude), the interesting question is rarely "can it write a React component" but "can it run a team." Put the new model at the helm, hold the implementation tier constant, and the loop's state file gives you a per-chunk scorecard: how many attempts each chunk took, how often it had to escalate, whether its completion condition held up. Swap the advisor, re-run, compare.

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
       dispatch  -> implementer subagent (mid-tier model)
       review    -> advisor reads the real git diff, not the agent's summary
       iterate   -> follow-ups to the same agent; escalate to the strong tier after 2 failures
       gate      -> project verify/test gate must be green
       record    -> one log line per attempt in the state file
  3. exit only when the completion check passes, or a real blocker needs you
```

Everything runs **inside the current session**. No cron, no scheduled wakeups, no background daemons. State lives in `.helm/<task-slug>.md` at the repo root so the loop survives context compaction and long sessions.

## Install

**Global** (available in every project):

```bash
git clone https://github.com/Jacknelson6/helm ~/.claude/skills/helm
```

**Per-project** (checked into the repo, shared with your team):

```bash
git clone https://github.com/Jacknelson6/helm .claude/skills/helm
rm -rf .claude/skills/helm/.git
```

Then in Claude Code:

```
/helm <your task>
```

or just describe it: "run the advisor loop on X", "you plan, delegate the build".

**Other agents** (Codex CLI, Gemini CLI, opencode, Cursor, or anything AGENTS.md-driven): the skill is plain markdown with no Claude-only requirements. Either:

- clone it anywhere in the repo (e.g. `docs/helm/`) and add one line to your `AGENTS.md`: "When asked to 'helm' a task or run the advisor loop, follow docs/helm/SKILL.md", or
- register `SKILL.md` as a custom command/prompt in your harness.

The only hard requirement is a harness that can spawn scoped subagents, ideally with a selectable model per subagent. If model selection isn't available, helm still runs: implementers become fresh same-model workers and you keep the chunking, independent review, and verifiable exit.

## Choosing models

Every run starts with one question:

> **How should helm pick models for this run?**
> 1. **Auto (recommended)** — the advisor picks the best model per chunk based on difficulty and cost
> 2. **Custom** — you pick the top (escalation), mid (implementer), and low (mechanical) tiers yourself

The advisor is always whatever model your session runs on. Tiers are roles, not model names; map them to whatever your provider offers (mixing providers is fine if your harness supports it):

| Tier | Used for | Anthropic | OpenAI | xAI | GLM | Gemini |
|---|---|---|---|---|---|---|
| Advisor | planning, review, escalation decisions, the completion check | session model | session model | session model | session model | session model |
| Implementer (mid) | standard chunks | Sonnet | standard | standard | Air | Flash |
| Escalation (strong) | chunks the mid tier failed twice | Opus | frontier / reasoning | frontier | frontier | Pro |
| Mechanical (small) | renames, codemods, bulk edits | Haiku | mini / nano | mini | flash | Flash-Lite |

You can also name models inline; helm treats that as Custom with your choices prefilled:

```
/helm migrate the settings pages to the new form kit, implement with the small tier, escalate to mid
```

To evaluate a new SOTA model as the orchestrator, start your session on that model (in Claude Code: `/model`), keep the implementation tier fixed, and compare the `## Log` sections of the state files across runs.

### Recommended stack (July 2026)

Our current picks if your harness can mix providers; revisit as models ship:

| Tier | Recommendation |
|---|---|
| Orchestrator / advisor | Claude Fable 5 |
| Builder (mid) | GPT 5.6 Terra |
| Escalation (strong) | GPT 5.6 Sol or Claude Opus 4.8 |
| Mechanical (small) | Claude Haiku or GPT 5.6 Luna |

Single-provider? Use your provider's column from the tier table above.

## Auto-update

If you installed by `git clone` (the global install above), helm keeps itself current: on every invocation it quietly runs `git pull --ff-only` on its own folder before starting, and re-reads itself if anything changed. Offline or diverged clones just run the version they have; it never blocks or prompts.

Prefer updating at session start instead of invocation time? Claude Code users can add a SessionStart hook to `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [ { "type": "command", "command": "git -C ~/.claude/skills/helm pull --ff-only --quiet 2>/dev/null || true" } ] }
    ]
  }
}
```

To pin a version instead, remove the `.git` folder (the per-project install above does this already) or check out a tag; helm skips self-update when it isn't a git clone.

## The state file

Each run writes `.helm/<slug>.md`:

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

Gitignore `.helm/` if you don't want run logs in your repo; keep them if you're benchmarking.

## What's in the repo

- `SKILL.md` — the skill itself
- `references/success-criteria.md` — what success looks like: worked completion conditions by task type (feature, bug fix, refactor, UI, performance), the invalid conditions helm must reject, and a gold-standard state file
- `references/dispatch-and-review.md` — the dispatch prompt template, the advisor's per-chunk review checklist, and the log-line format that doubles as the model scorecard

The references load on demand (progressive disclosure), so the skill stays cheap in context until a run actually starts.

## Design notes

- **The advisor never implements.** Small one-line unblock fixes only. If the top model writes the feature itself, you've measured nothing and spent everything.
- **Diffs over self-reports.** The advisor reviews `git diff`, not the subagent's summary of what it did.
- **3-strikes escalation.** Default tier fails once → sharper retry. Twice → fresh dispatch on the escalation tier. Escalation tier fails on the same root cause → stop and ask the human. That's a real blocker; type errors and red tests are not.
- **Interview, don't guess.** An unverifiable goal is the loop's only unrecoverable input, so it's the one thing the skill is required to push back on.

## License

MIT
