<div align="center">

```
 _          _
| |__   ___| |_ __ ___
| '_ \ / _ \ | '_ ` _ \
| | | |  __/ | | | | | |
|_| |_|\___|_|_| |_| |_|
```

**The strongest model you have plans, reviews, and decides when it's done.
Cheaper models do the typing.**

[![Agent Skill](https://img.shields.io/badge/agent-skill-blue)](https://docs.claude.com/en/docs/claude-code/skills)
[![Provider agnostic](https://img.shields.io/badge/providers-Anthropic%20%C2%B7%20OpenAI%20%C2%B7%20xAI%20%C2%B7%20GLM%20%C2%B7%20Gemini%20%C2%B7%20local-8A2BE2)](#routing-and-choosing-models)
[![License: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

</div>

A provider-agnostic [Agent Skill](https://docs.claude.com/en/docs/claude-code/skills) that turns your coding-agent session into an **advisor-orchestrated build loop**: the top model derives a verifiable completion condition, splits the work into chunks, dispatches them to cheaper implementer models, reviews every diff, and exits only when the goal check passes. Works with any model family and any harness that can spawn scoped subagents; first-class in Claude Code.

---

**Contents:** [Quick start](#quick-start) · [Commands](#commands) · [How it works](#how-it-works) · [Why](#why) · [Routing and models](#routing-and-choosing-models) · [Profiles](#profiles) · [Model intel and refresh](#model-intel-and-refresh) · [Evaluate](#evaluate) · [The state file](#the-state-file) · [Auto-update](#auto-update) · [Repo map](#whats-in-the-repo) · [Design notes](#design-notes)

## Quick start

```bash
# global (available in every project)
git clone https://github.com/Jacknelson6/helm ~/.claude/skills/helm
```

```
/helm add rate limiting to all public API routes
```

That's it. Helm routes the run, confirms the plan and models in one question, derives a verifiable completion condition, and runs the loop until the check passes.

<details>
<summary><b>Other install options</b> (per-project, Codex CLI, Gemini CLI, opencode, Cursor, ...)</summary>

**Per-project** (checked into the repo, shared with your team):

```bash
git clone https://github.com/Jacknelson6/helm .claude/skills/helm
rm -rf .claude/skills/helm/.git
```

**Other agents** (anything AGENTS.md-driven): the skill is plain markdown with no Claude-only requirements. Either:

- clone it anywhere in the repo (e.g. `docs/helm/`) and add one line to your `AGENTS.md`: "When asked to 'helm' a task or run the advisor loop, follow docs/helm/SKILL.md", or
- register `SKILL.md` as a custom command/prompt in your harness.

The only hard requirement is a harness that can spawn scoped subagents, ideally with a selectable model per subagent. If model selection isn't available, helm still runs: implementers become fresh same-model workers and you keep the chunking, independent review, and verifiable exit.

</details>

## Commands

Everything is `/helm` plus an optional first word. In prose-driven harnesses, "helm this", "run the advisor loop", or "you plan, delegate the build" all work too.

### Build runs

| Command | What it does |
|---|---|
| `/helm <task>` | The full flow: route the run, confirm route + model tiers in one question, then loop until the completion check passes. |
| `/helm auto <task>` | Same, but skips the confirmation: helm announces its derived plan in one line and starts immediately. |
| `/helm custom <task>` | Skips straight to you picking the route and/or tiers (any models your harness can run, mixed providers welcome). |
| `/helm <profile> <task>` | Runs with a saved profile's models and billing lane, no tier question. Example: `/helm all-anthropic migrate the settings pages`. |

You can also name models inline in any of these ("... implement with haiku, escalate to opus"); helm treats that as Custom with your choices prefilled.

### Profile management

| Command | What it does |
|---|---|
| `/helm profile save <name>: <spec>` | Saves a named routing preset. Free-text spec is fine: `fable advising, sonnet building, opus escalation, haiku mechanical, subscription`. |
| `/helm profile list` | All saved profiles, one line each (models + billing lane). |
| `/helm profile show <name>` | The full profile block. |
| `/helm profile delete <name>` | Removes a profile (asks first). |

### Maintenance and reporting

| Command | What it does |
|---|---|
| `/helm refresh` | Re-audits which models your harness can dispatch, pulls fresh intelligence/price/speed data from [Artificial Analysis](https://artificialanalysis.ai), recomputes the Pareto frontier, and flags saved profiles a newer model now dominates. |
| `/helm evaluate` | Reads the repo's `.helm/` run ledgers and reports measured savings, the per-model scorecard, and evidence-grounded tuning suggestions. |

`auto`, `custom`, `profile`, `refresh`, and `evaluate` are reserved words; anything else after `/helm` is treated as your task (or a profile name if it matches one).

## How it works

![The helm loop: the advisor plans and dispatches chunks to implementer workers, reviews each diff, gates on verify, retries or escalates on failure, logs every attempt to the state file, and exits only when all chunks are complete and the completion condition is verified](assets/helm-loop.png)

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

The loop is **goal-driven, not vibe-driven**: before any work starts, the advisor must derive a verifiable completion condition (a command that passes, an observable behavior, or an enumerable checklist). If it can't, it interviews you until one is pinned. No "looks done to me" exits.

## Why

Two reasons to run this way:

1. **Cost-shaped quality.** Frontier models are at their best planning, decomposing, and reviewing. Most implementation chunks don't need that horsepower. Helm keeps the expensive tokens on judgment and spends cheap tokens on typing.
2. **Testing new SOTA models as orchestrators.** When a new top model ships, the interesting question is rarely "can it write a React component" but "can it run a team." Put the new model at the helm, hold the implementation tier constant, and the state file gives you a per-chunk scorecard: attempts, escalations, whether its completion condition held up. Swap the advisor, re-run, compare.

### The data

Anthropic's [@ClaudeDevs](https://x.com/ClaudeDevs) published SWE-bench Pro results (curated subset, 482 problems, July 2026) for exactly this pattern via their [advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool): Sonnet 5 executors steered by a Fable 5 advisor called about once per task:

| Setup | Accuracy (Wilson 95% CI) | Cost per task (notional) |
|---|---|---|
| Sonnet 5 solo | ~0.75 | ~$0.75 |
| **Sonnet 5 + Fable 5 advisor** | **~0.84** | **~$1.45** |
| Fable 5 solo | ~0.91 | ~$2.25 |

That's **~92% of Fable 5's score at ~63% of the price**, and roughly 9 points over Sonnet solo for about double its cost. Helm leans into the same structure and goes further on the advisor side: it doesn't just steer once per task, it reviews every chunk's diff and owns a verifiable exit condition, spending a bit more advisor time to close part of that remaining 8% gap.

<details>
<summary><b>Relation to Anthropic's advisor tool</b></summary>

Anthropic ships this pattern natively at the API layer: the [advisor tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/advisor-tool) (`advisor_20260301`, beta) lets an executor model consult a stronger advisor mid-generation, server-side, inside a single request. It's the same bet as helm with the topology inverted:

|  | Advisor tool (API) | Helm (session) |
|---|---|---|
| Who owns the task | the cheap **executor**; it decides when to ask for advice | the strong **advisor**; it decides what gets delegated |
| Advice granularity | a plan or course-correction, ~1-3 calls per task | full plan up front, plus a diff review of every chunk |
| Verification | none built in; executor self-assesses | verifiable completion condition + gate, owned by the advisor |
| Where it runs | any Claude API integration (add one tool to `tools`) | any agent harness that can spawn subagents |

They compose rather than compete: helm's advisor holds the goal, the chunking, and the exit condition, things a mid-generation advice call can't own, while an implementer that supports the advisor tool can get its own mid-chunk steering. If your harness passes API tools through to subagents, giving mid-tier implementers an advisor is a cheap way to cut failed attempts before helm's escalation ladder kicks in.

The loop's discipline is the part the API can't give you: goal-driven exits, independent review of diffs, and the per-chunk scorecard.

</details>

## Routing and choosing models

Every run starts by ROUTING: the session model picks the cheapest loop shape that holds quality (solo / dispatch / helm-lite / helm, see `references/routing.md`). On the helm-lite route a strong-tier subagent runs the whole advisor loop and the session model only sets the completion condition and verifies the exit, so the priciest model in the loop is spent only where frontier judgment is needed.

Tiers are roles, not model names; map them to whatever your provider offers (mixing providers is fine if your harness supports it):

| Tier | Used for | Anthropic | OpenAI | xAI | GLM | Gemini |
|---|---|---|---|---|---|---|
| Advisor | planning, review, escalation decisions, the completion check | session model | session model | session model | session model | session model |
| Implementer (mid) | standard chunks | Sonnet | standard | standard | Air | Flash |
| Escalation (strong) | chunks the mid tier failed twice | Opus | frontier / reasoning | frontier | frontier | Pro |
| Mechanical (small) | renames, codemods, bulk edits | Haiku | mini / nano | mini | flash | Flash-Lite |

To evaluate a new SOTA model as the orchestrator, start your session on that model (in Claude Code: `/model`), keep the implementation tier fixed, and compare the `## Log` sections of the state files across runs.

### Recommended stack (August 2026)

Our current picks, from the cached [Artificial Analysis](https://artificialanalysis.ai) snapshot in `references/model-intel.md`:

| Tier | Single-provider (e.g. Claude Code) | Cross-provider harness |
|---|---|---|
| Orchestrator / advisor | session model (Claude Fable 5 or Opus 5) | Claude Fable 5 or Opus 5 |
| Builder (mid) | Claude Sonnet 5 | GPT-5.6 Terra |
| Escalation (strong) | Claude Opus 5 | GPT-5.6 Sol or Claude Opus 5 |
| Mechanical (small) | Claude Haiku 4.5 | GPT-5.6 Luna or DeepSeek V4 Flash |

Run [`/helm refresh`](#model-intel-and-refresh) to keep this current; build runs read the cached snapshot only, so nothing is fetched at build time.

## Profiles

Save a named routing preset once, reuse it forever:

```
/helm profile save all-anthropic: fable advising, sonnet building, opus escalation, haiku mechanical, subscription
/helm all-anthropic migrate the settings pages to the new form kit
```

Profiles live in a gitignored `profiles.local.md` and record models plus the billing lane:

- `subscription`: dispatches ride the harness's plan-billed subagents (the default).
- `api:<provider>`: you have explicitly authorized metered spend through that lane (e.g. `api:openrouter`).
- `ask`: helm confirms the lane on each run.

**Helm never routes spend through an API key just because one exists in the environment.** A key is availability, not consent: leaving the subscription lane always takes an explicit choice, in the run confirmation or on the profile. When both lanes could serve the model you picked, helm asks once and offers to remember the answer.

Profiles pin models, never routes: the solo/dispatch/helm-lite/helm decision is still made fresh per task. Full spec: `references/profiles.md`.

## Model intel and refresh

`/helm refresh` keeps helm's model picks honest. It:

1. re-audits which models your harness can actually dispatch (and on which billing lane),
2. pulls fresh intelligence/price/speed data from [Artificial Analysis](https://artificialanalysis.ai),
3. recomputes the cost-vs-intelligence Pareto frontier and rewrites the cached snapshot in `references/model-intel.md`,
4. flags any saved profile whose models a newer frontier point now dominates (suggests only, never edits), and
5. does a short pulse-check on orchestration best practice.

### Setup: Artificial Analysis API key (optional, per user)

Refresh works without any setup by reading the public leaderboard pages. For structured data (the full model list with evaluations, pricing, and speed), give it your own free key:

1. Create an account at [artificialanalysis.ai](https://artificialanalysis.ai) and generate a key in your API settings ([data API docs](https://artificialanalysis.ai/data-api/docs)).
2. In the skill directory: `cp .env.local.example .env.local`, then paste your key into `.env.local`. Alternatively, export `AA_API_KEY` in your shell profile.
3. Done; the next `/helm refresh` picks it up automatically.

`.env.local` is gitignored, and so are your saved profiles (`profiles.local.md`): both are per-machine files that must never be committed. Only the `.env.local.example` template ships with the repo. If a key does leak into a commit, rotate it at Artificial Analysis first, then scrub the history.

## Evaluate

`/helm evaluate` reads the `.helm/` run ledgers back and reports what helm actually bought you:

- **Measured savings** per run in tokens and dollars, with the overhead and price assumptions named (savings runs only; quality runs report review yield and never a savings claim).
- **Advisor-share trend**: the fraction of tokens spent on the expensive seat, which should be flat or falling.
- **Per-model scorecard**: first-attempt acceptance rate, escalations and why, blocked chunks.
- **Tuning suggestions** grounded in the evidence: tier swaps, profile updates, discipline fixes (leaked scouts, missing token reports).

Read-only, always. Spec: `references/evaluate.md`.

## The state file

Each run writes `.helm/<slug>.md`:

```markdown
# Helm: rate-limit public routes
Goal: every public API route enforces per-IP rate limiting
Completion check: `npm test rate-limit` green AND all 12 routes return 429 in the harness
Out of scope: admin routes, auth endpoints
Models: advisor=fable-5 implementer=sonnet escalation=opus profile=all-anthropic billing=subscription
## Chunks
- [x] 1. middleware + config, acceptance: unit tests pass
- [x] 2. wire 12 routes, acceptance: grep shows all 12 wrapped
- [ ] 3. e2e harness, acceptance: 429 observed on each route
## Log
- chunk 1: sonnet, 1 attempt, accepted
- chunk 2: sonnet, 3 attempts (escalated to opus), accepted
```

Gitignore `.helm/` if you don't want run logs in your repo; keep them if you're benchmarking (they're what `/helm evaluate` reads).

## Auto-update

If you installed by `git clone` (the global install above), helm keeps itself current: on the first invocation of a session it quietly runs `git pull --ff-only` on its own folder before starting, and re-reads itself if anything changed (later invocations in the same session skip the check). Offline or diverged clones just run the version they have; it never blocks or prompts.

<details>
<summary><b>Update at session start instead, or pin a version</b></summary>

Claude Code users can add a SessionStart hook to `~/.claude/settings.json`:

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

</details>

## What's in the repo

| File | What it is |
|---|---|
| `SKILL.md` | The skill itself: the loop, invocation modes, token-economics rules. |
| `references/routing.md` | The four loop shapes (solo / dispatch / helm-lite / helm), the triage rubric, and the break-even math. |
| `references/success-criteria.md` | Worked completion conditions by task type, the invalid conditions helm must reject, and a gold-standard state file. |
| `references/dispatch-and-review.md` | The dispatch prompt template, the advisor's per-chunk review checklist, and the ledger format. |
| `references/model-intel.md` | Harness model inventory, the cached Artificial Analysis Pareto snapshot, and the `/helm refresh` procedure. |
| `references/profiles.md` | Saved model-routing profiles: format, commands, billing-lane rules. |
| `references/evaluate.md` | The `/helm evaluate` ledger report: savings math, scorecard, suggestions. |
| `.env.local.example` | Template for your Artificial Analysis API key (`.env.local` itself is gitignored). |
| `profiles.local.md` | Your saved profiles (gitignored, created on first save). |

The references load on demand (progressive disclosure), so the skill stays cheap in context until a run actually starts.

## Design notes

- **The advisor never implements.** Small one-line unblock fixes only. If the top model writes the feature itself, you've measured nothing and spent everything.
- **Diffs over self-reports.** The advisor reviews `git diff`, not the subagent's summary of what it did.
- **3-strikes escalation.** Default tier fails once: sharper retry. Twice: fresh dispatch on the escalation tier. Escalation tier fails on the same root cause: stop and ask the human. That's a real blocker; type errors and red tests are not.
- **Interview, don't guess.** An unverifiable goal is the loop's only unrecoverable input, so it's the one thing the skill is required to push back on.
- **Keys are availability, not consent.** Billing lanes change only on an explicit user choice, never because a key was found in the environment.

## License

[MIT](LICENSE)
