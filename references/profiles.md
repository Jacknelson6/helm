# Profiles: saved model routing

A profile is a named, reusable answer to helm's tier-confirmation question:
which model sits in each seat, and which billing lane pays for it. Profiles
pin MODELS, never routes: the route (solo / dispatch / helm-lite / helm) is
still chosen fresh per task via references/routing.md.

## Storage

Profiles live in `profiles.local.md` in the skill directory, one file per
machine, gitignored (they encode this user's spend preferences and local
lanes, so they never ship with the skill). If the file doesn't exist, there
are no profiles; create it on the first save. Format, one block per profile:

```markdown
## profile: <name>
advisor: <model>
builder: <model>
escalation: <model>
mechanical: <model>
billing: <subscription | api:<provider> | ask>
notes: <optional free text>
```

- `name`: lowercase, hyphens allowed, must not be a reserved word (`auto`,
  `custom`, `profile`, `refresh`) and must not collide with an existing
  profile (saving over an existing name asks first, then overwrites).
- `advisor`: the seat the session model normally fills. If a profile names
  an advisor that differs from the current session model, say so at run
  start and follow SKILL.md routing anyway (the session seat is set by the
  harness, not the profile); the value is aspirational documentation plus
  the pick for delegated-advisor (helm-lite) dispatches.
- `billing`: `subscription` = harness subagents only. `api:<provider>` =
  the user has explicitly authorized metered dispatch through that lane
  (e.g. `api:openrouter`). `ask` = confirm the lane on each run. Per the
  SKILL.md billing rule, only an explicit user statement may set
  `api:<provider>`; helm never writes it just because a key exists.

## Commands

- **`/helm profile save <name>: <spec>`** — parse models from the spec
  (free text is fine: "fable advising, sonnet building, opus escalation,
  haiku mechanical, subscription"). Before writing: validate every model
  against the harness inventory (references/model-intel.md section 1), warn
  on anything not currently dispatchable (save it anyway if the user
  confirms; profiles may describe another harness), and apply the billing
  rule: if the spec names models reachable only via API, or both lanes could
  serve it and the user didn't say which, ask once and record the answer.
  Echo the saved block back.
- **`/helm profile list`** — names + one-line summaries (models + billing).
- **`/helm profile show <name>`** — the full block.
- **`/helm profile delete <name>`** — confirm, then remove the block.
- **`/helm <name> <task>`** — run with that profile (see SKILL.md
  invocation modes). Announce in one line: "Profile <name>: <builder>
  building, <escalation> on escalation, <mechanical> mechanical,
  <billing> lane." Then route and run; skip the tier confirmation. Inline
  model mentions in the task text override the profile for that seat only
  (announce the override).

If the user says `/helm <name>` and no such profile exists, don't guess:
list the saved profiles and ask, or treat it as task text if it plainly
reads as one.

## During a run

- Record `profile=<name>` and `billing=<lane>` on the state file's `Models:`
  line so the ledger shows which preset produced which economics.
- The advisor may still move a specific chunk up or down a tier mid-run
  (SKILL.md Auto behavior), logging why; that never edits the profile.
- If a profiled model repeatedly fails a seat (3-strikes data across runs),
  surface that at exit as a suggestion to update the profile; never edit it
  silently. `/helm refresh` step 5 does the systematic version of this
  against fresh Artificial Analysis data.

## Example

```markdown
## profile: all-anthropic
advisor: claude-fable-5
builder: claude-sonnet-5
escalation: claude-opus-5
mechanical: claude-haiku-4-5
billing: subscription
notes: single-provider stack for Claude Code plan billing
```

Invoked as: `/helm all-anthropic migrate the settings pages to the new form kit`
