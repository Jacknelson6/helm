# Model intel: harness inventory, Pareto snapshot, refresh

The cached model-selection intelligence helm reads instead of re-researching
the model landscape on every invocation. Normal runs read sections 1-3 and
never fetch anything. Only `/helm refresh` (section 4) touches the network.

**Snapshot date: 2026-08-09.** Staleness policy: when this date is more than
~30 days old, add one clause to the plan confirmation ("model snapshot is N
weeks old; run /helm refresh to update") and proceed with the cached data.
Never block or delay a build run on freshness.

## 1. Harness inventory (what can this session actually dispatch?)

A recommendation for a model the harness can't run is noise. Before proposing
tiers to a user (and always during `/helm refresh`), establish:

1. **Session model** — what is driving this conversation. It is the default
   advisor seat and the price ceiling every dispatch is arbitraged against.
2. **Dispatchable models** — what the harness's subagent mechanism accepts
   as a model parameter. In Claude Code, read the Agent tool's `model` enum
   in its schema (plus the agent-type definitions' model frontmatter);
   elsewhere, read the harness's subagent/worker docs. Only these models can
   fill the builder / escalation / mechanical seats on the subscription lane.
3. **Cross-provider routing** — can a dispatch reach a different provider
   than the session model (via the harness natively, or via a gateway like
   OpenRouter)? If not, the cross-provider column of any recommendation is
   irrelevant here; use the single-provider column.
4. **Latent API lanes** — environment keys that could reach more models:
   OPENROUTER_API_KEY, OPENAI_API_KEY, ANTHROPIC_API_KEY, GOOGLE_API_KEY /
   GEMINI_API_KEY, XAI_API_KEY, DEEPSEEK_API_KEY, AA_API_KEY. Report these
   as *available lanes only*. A key is never consent to spend on it: the
   billing rule in SKILL.md requires an explicit user choice (run
   confirmation or a profile's `billing:` line) before any metered dispatch,
   and one clarifying question when both a subscription and a usable key
   could serve the same model.
5. **Billing mode per lane** — subscription/plan-billed (harness subagents)
   vs metered API (per-token spend). Label every inventoried model with its
   lane so cost talk compares like with like: a subscription dispatch has
   zero marginal dollar cost but consumes plan quota; an API dispatch has a
   real per-token price (section 2).

## 2. Snapshot (Artificial Analysis, fetched 2026-08-09)

Source: artificialanalysis.ai leaderboard. Intelligence = AA Intelligence
Index; price = AA blended $/1M tokens; speed = output tokens/sec. Blanks were
not captured in this fetch; refresh fills them.

| Model | Intelligence | Blended $/1M | Speed t/s |
|---|---|---|---|
| Claude Opus 5 (xhigh) | 63 | 1.80 | 61 |
| Claude Opus 5 (max) | 63 | 2.34 | 58 |
| Claude Fable 5 | 62 | 3.14 | 70 |
| GPT-5.6 Sol (max) | 61 | 1.23 | 69 |
| Kimi K3 (max) | 60 | 0.84 | 43 |
| Qwen3.8 Max | 58 | 1.13 | 85 |
| GPT-5.6 Terra (max) | 57 | 0.51 | 142 |
| Claude Sonnet 5 (max) | - | 1.54 | - |
| DeepSeek V4 Flash | 52 | 0.03 | 128 |
| GPT-5.6 Luna (max) | 52 | 0.05 | 196 |
| Claude Haiku 4.5 | - | 0.77 | - |
| MiMo-V2.5 | 38 | 0.01 | 90 |

**Cost-vs-intelligence Pareto frontier** (no model above-left of it):
Opus 5 xhigh (63 @ 1.80) -> GPT-5.6 Sol (61 @ 1.23) -> Kimi K3 (60 @ 0.84)
-> GPT-5.6 Terra (57 @ 0.51) -> DeepSeek V4 Flash (52 @ 0.03) ->
MiMo-V2.5 (38 @ 0.01).

Off-frontier but still recommended when the harness or lane demands it:
Fable 5 (strongest Claude Code session seat; the 1-point gap to Opus 5 is
within eval noise), Sonnet 5 / Haiku 4.5 (the only mid/small tiers a
single-provider Anthropic harness can dispatch), GPT-5.6 Luna (Luna beats
DeepSeek on speed and is the native OpenAI-harness small tier).

## 3. Picking a stack (Pareto discipline)

Work from the frontier restricted to what section 1 says is dispatchable,
seat by seat:

- **Advisor**: the highest-intelligence point available; price matters least
  here because routing (references/routing.md) already minimizes advisor
  tokens. Session model by default.
- **Escalation**: the next frontier point below the advisor that is still a
  clear step above the builder.
- **Builder**: the knee of the curve, the cheapest point within ~5
  intelligence points of the escalation tier. This seat takes most of the
  tokens, so its rate sets the run's economics (the r in routing.md's
  break-even math).
- **Mechanical**: the cheapest model that reliably follows format
  instructions; intelligence is nearly irrelevant.

Tie-breaks: an intelligence gap of ~2 points or less is noise, take the
cheaper or faster model. Prefer the subscription lane at equal capability.
Speed matters for interactive runs (many small dispatches), much less for
long autonomous ones.

## 4. Refresh (`/helm refresh`)

Run when the user invokes it, and suggest it when the snapshot is stale.
Steps, in order:

1. **Harness inventory** (section 1) — re-derive it for the current session
   and summarize it to the user: session model, dispatchable tiers, lanes.
2. **Fetch fresh data from Artificial Analysis.**
   - Preferred, if AA_API_KEY is set:
     `curl -s https://artificialanalysis.ai/api/v2/data/llms/models -H "x-api-key: $AA_API_KEY"`
     (free tier: median metrics and input/output prices; docs at
     https://artificialanalysis.ai/data-api/docs). AA requires attribution:
     link artificialanalysis.ai wherever the data is shown.
   - Fallback, no key: WebFetch https://artificialanalysis.ai/models and
     /leaderboards/models and extract intelligence, blended price, speed.
   - If both fail (offline), say so and stop; never invent numbers.
3. **Rewrite section 2** with the fresh table, recompute the Pareto
   frontier, and bump the snapshot date at the top of this file.
4. **Update the stack table in SKILL.md** (and README.md if its copy
   drifted) only when a seat's recommendation actually changes; note each
   change and the why in one line to the user.
5. **Audit saved profiles** — for every profile in `profiles.local.md`,
   check each seat against the new frontier. Where a profile's model is now
   dominated (a frontier model is cheaper AND at least as capable, or the
   model was deprecated/renamed), tell the user the suggested swap and the
   evidence. Suggest only; never edit a profile without the user's yes.
6. **Architecture pulse-check** — one short WebSearch pass on current
   multi-model orchestration practice (advisor/worker loops, routing,
   verification patterns). If something material has shifted (e.g. harnesses
   now report subagent cost natively, or a better-verified loop shape is
   published), summarize it in 3 lines or less for the user and, if they
   agree it warrants it, open a follow-up task; refresh itself never
   rewrites the loop.
7. **Commit** — if the skill directory is a git clone, commit the refreshed
   files ("chore: model-intel refresh <date>") and push if a remote is
   configured; skip silently otherwise.

Refresh is read-mostly and cheap; it must never dispatch build subagents or
touch a project repo.
