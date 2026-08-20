# Composing the Four Layers into Real Automations

> **Source of truth:** `cron/jobs.py` (the automation API),
> `cron/blueprint_catalog.py` (16 blueprints),
> `cron/suggestion_catalog.py` (4 starters),
> `cron/scheduler.py` (execution),
> `tools/delegate_tool.py` (fan-out).

The four layers are useful separately, but the point is the composition. A single
cron job selects a **toolset** (Layer 1), attaches **skills** (Layer 2), delivers
to a **platform** (Layer 3), and pins a **model and execution environment**
(Layer 4). One function call, all four layers.

This file covers the five composition patterns and the anti-patterns that waste
the most time.

---

## 1. `create_job` — the whole automation API in one signature

Everything scheduled in Hermes goes through this
(`cron/jobs.py:1780`):

```python
def create_job(
    prompt,                  # must be SELF-CONTAINED
    schedule,                # "30m" | "every 2h" | "0 9 * * *" | ISO timestamp
    name=None,
    repeat=None,             # None = forever, 1 = once
    deliver=None,            # "origin" | "local" | "telegram" | …
    origin=None,
    skill=None,              # legacy single skill
    skills=None,             # ordered list of skills to load    ← LAYER 2
    model=None,              # per-job model override            ← LAYER 4
    provider=None,           # per-job provider override         ← LAYER 4
    base_url=None,
    script=None,             # stdout feeds the job
    context_from=None,       # chain from another job's output
    enabled_toolsets=None,   # restrict to these toolsets        ← LAYER 1
    workdir=None,            # AGENTS.md injection + tool cwd
    no_agent=False,          # skip the LLM entirely
    attach_to_session=None,
    monitor_script=None,     # change-detection source
    monitor_url=None,        # change-detection source (URL)
)
```

Read that as a capability list, not a parameter list. Six of those arguments are
worth their own section.

### `schedule` — four accepted forms

| Form | Meaning |
|---|---|
| `"30m"`, `"2h"` | **Once**, in 30 minutes / 2 hours |
| `"every 30m"`, `"every 2h"` | **Recurring** interval |
| `"0 9 * * *"` | Standard cron expression (5 or 6 fields) |
| `"2026-02-03T14:00"` | Once, at a timestamp |

Parsed by `parse_schedule()` into `{kind: "once" | "interval" | "cron", …}`.

### `enabled_toolsets` — narrow the surface per job

```python
enabled_toolsets=["web", "clarify"]
```

Two reasons to always set this:

1. **Token cost.** The docstring says it directly — *"only tools from these
   toolsets are loaded, reducing token overhead."* An unrestricted job pays for
   ~50 tool schemas on every tick, forever.
2. **Blast radius.** A job that reads a web page does not need `terminal`. Apply
   the webhook lesson from [01-layer1-toolsets.md §5](01-layer1-toolsets.md) to
   your own jobs: reduce the toolset to what the task needs.

### `skills` — attach procedural knowledge

```python
skills=["competitor-news-monitor", "grounded-citations"]
```

Loaded, in order, before the prompt runs. This is how a 200-word prompt gets the
behavior of a 2,000-word one without paying for it in the prompt.

### `no_agent=True` — automation with no LLM at all

```python
create_job(prompt=None, schedule="every 15m",
           script="check_disk.sh", no_agent=True)
```

> skip the agent entirely — run `script` on schedule and deliver its stdout
> directly. Empty stdout = silent (no delivery). Requires `script` to be set.
> **Ideal for classic watchdogs and periodic alerts that don't need LLM
> reasoning.**

This is an unusually honest feature for an AI agent framework: a first-class path
for "this task does not need a model." If your automation is a threshold check, use
it. Zero tokens, zero latency, zero nondeterminism.

Scripts resolve under `~/.hermes/scripts/`; `.sh`/`.bash` run via bash, anything
else via Python.

### `monitor_script` / `monitor_url` — change detection before spending tokens

```python
create_job(prompt="Summarize what changed on the pricing page.",
           schedule="every 1h",
           monitor_url="https://example.com/pricing")
```

The mechanism:

1. Each tick, the monitor source runs/fetches **first**
2. Its output is hashed as exact bytes
3. **Unchanged → the agent run is suppressed entirely** (recorded as a silent
   `no_change` tick)
4. **Changed → a `MONITOR CHANGE DETECTED` block** — unified diff plus new output —
   is injected into the prompt, then a normal agent run happens

An hourly page monitor that changes twice a week costs you two model calls a week,
not 168. The caveat is in the docstring: *"Scripts should emit stable output (no
timestamps)"* — a timestamp in your monitor output makes every tick look like a
change and defeats the whole mechanism.

Mutually exclusive with each other; incompatible with `no_agent=True`.

### `context_from` — chain jobs

```python
create_job(..., context_from="job_abc123")
create_job(..., context_from=["job_a", "job_b"])
```

> Optional job ID (or list of job IDs) whose most recent output is injected into
> the prompt as context before each run. Useful for chaining cron jobs: job A finds
> data, job B processes it.

This is your pipeline primitive. Job A does expensive collection on a slow
schedule; job B does cheap analysis on a fast one and reads A's last output.

### `workdir` — give a job a project

```python
create_job(..., workdir="/home/me/projects/api")
```

`AGENTS.md`, `CLAUDE.md`, and `.cursorrules` from that directory are injected into
the system prompt, and `terminal`/`file`/`code_execution` use it as their working
directory via `TERMINAL_CWD`. This is what turns a cron job into "work on this
repo" rather than "run a command somewhere."

---

## 2. The `[SILENT]` convention — why monitors are livable

```python
SILENT_MARKER = "[SILENT]"     # cron/scheduler.py:513
```

A job whose output is `[SILENT]` produces **no notification at all**. Combined
with an LLM's judgment, this is the difference between a monitor you keep and one
you mute in a week.

Four of the 16 blueprints use it. The important-mail monitor's prompt:

> Pipe candidates through the urgency classifier … and deliver ONLY what it
> returns. If nothing clears the bar, respond with `[SILENT]` so the user is not
> pinged.

Design your own jobs this way. "Check X hourly and tell me if it matters" is a good
automation. "Check X hourly and tell me" is a notification you will disable.

### The classifier that makes it work

`cron/scripts/classify_items.py` — poll a
source, LLM-score each item 0–10 against your criteria, return only above-threshold
items:

```bash
python3 -m cron.scripts.classify_items --threshold 7 --criteria "..."
```

The header of `cron/suggestion_catalog.py`
records an architectural decision worth reading:

> The "important-mail monitor" entry is where the old proactive-monitor engine
> lives now: its `classify_items.py` (poll a source → LLM-score urgency → surface
> only above-threshold) is **ONE catalog automation, not a standalone feature.**

A whole subsystem was collapsed into one catalog entry plus a script. That is the
narrow waist being enforced retroactively — and route its scoring through
`auxiliary.monitor` on a cheap model (see
[04-layer4-backends.md §3](04-layer4-backends.md)).

---

## 3. The 16 blueprints — guided automation builders

`cron/blueprint_catalog.py` holds wizards. Each
declares typed slots (`time`, `enum`, `text`, `weekdays`) and builds the job from
your answers.

| Blueprint | The pattern it teaches |
|---|---|
| Morning briefing | Aggregate several sources at a fixed time |
| Important-mail monitor | Poll → LLM-score → threshold → `[SILENT]` |
| Price & availability watch | Poll a page, alert on a condition described in English |
| Competitor news watch | Named entities + event categories that matter |
| Topic news digest | N bullets on a topic, chosen weekdays |
| Weekly review | Recurring reflection on a chosen day |
| Workday start reminder | Time-of-day trigger |
| Evening wind-down | Time-of-day trigger, other end |
| Custom reminder | Free-text + weekday recurrence |
| Bills & renewals reminder | Deadline tracking |
| Habit check-in | Recurring prompt with continuity |
| Hydration & movement nudge | Interval within a daily window (`start_hour`/`end_hour`) |
| Weekly meal plan | Multi-slot preference collection (diet, meals, effort) |
| Daily learning drip | Recurring generative content |
| Gratitude & reflection prompt | Recurring prompt |
| On-this-day discovery | Recurring generative content |

Even if you never use the wizard, **read this file before writing your first job.**
It is 16 worked examples of self-contained prompts written by the people who built
the scheduler.

Note the interval-within-a-window pattern from the hydration blueprint — an
interval schedule plus `start_hour`/`end_hour` slots so it doesn't ping you at 3am.
That pattern is not obvious and is easy to get wrong.

### The 4 starter suggestions

`cron/suggestion_catalog.py` — offered via
`/suggestions`, accepted explicitly. **Nothing here auto-schedules.**

`Daily briefing` · `Important-mail monitor` · `Weekly review` ·
`Workday start reminder`

Each has a stable `key` that is never re-offered once dismissed.

---

## 4. Pattern: delegation / fan-out

```python
delegate_task(...)     # from the `delegation` toolset
```

| Property | Value | Source |
|---|---|---|
| Max concurrent children | **10** | `hermes_cli/config_defaults.py:1865` |
| Default child toolsets | `["terminal", "file", "web"]` | `tools/delegate_tool.py:1016` |
| Blocked from composites | `{"delegation"}` | `tools/delegate_tool.py:1176` |
| Isolation | optional git worktree | `tools/subagent_worktree.py` |

> **⚠ Doc drift.** `AGENTS.md` documents `delegation.max_concurrent_children` as
> 3. The code says **10** — raised in August 2026 (PR #86745).

Children can run on a different, cheaper model than the parent:

```yaml
delegation:
  model: ""        # empty = inherit parent
  provider: ""     # empty = inherit parent + credentials
```

Children cannot spawn grandchildren through a composite — `delegation` is in
`_COMPOSITE_BLOCKED_TOOLSETS`. That is the recursion guard, and it is why
"delegate a task that delegates" does not silently fork-bomb.

**Use it for:** auditing N files in parallel, migrating N call sites, researching N
sources, reviewing a diff along N independent dimensions.

**Use worktree isolation when** children write to the same repo — otherwise
parallel edits collide.

---

## 5. Pattern: kanban orchestration

The `kanban` toolset (14 tools) is only live when the agent was spawned by the
kanban dispatcher — gated on `HERMES_KANBAN_TASK` via `check_fn`. The dispatcher
runs inside the gateway by default (`kanban.dispatch_in_gateway`).

What the toolset description tells you about the model:

> Lets workers mark tasks done with structured handoffs, enter first-class review
> (`request_review` — **not a block**), return review changes, block for human
> input, heartbeat during long ops, comment on threads, attach files, and (for
> orchestrators) list, unblock, and fan out tasks.

Three distinctions that make this more than a task list:

- **Review is a state, not a block.** `kanban_request_review` ≠ `kanban_block`. A
  task awaiting review is progressing; a blocked task needs a human.
- **Heartbeats exist** (`kanban_heartbeat`) because a long-running worker is
  otherwise indistinguishable from a dead one.
- **Handoffs are structured**, so the next worker gets machine-readable state
  rather than prose.

Delegation is fan-out inside one turn. Kanban is a durable multi-agent workflow
that survives restarts. Reach for kanban when the work outlives a session.

---

## 6. Pattern: MCP — the escape hatch

`tools/mcp_tool.py`, with OAuth in
`tools/mcp_oauth_manager.py` and schema caching
in `tools/mcp_schema_cache.py`.

Point Hermes at any MCP server and those tools join the loop. Relevant config:

```yaml
mcp_discovery_timeout: ...
mcp_single_query_discovery_timeout: ...
mcp: {...}
```

There is a `setup_mcp` tool, but note it lives in the `desktop_ui` toolset — it is
a GUI affordance, not a general tool.

**This is rung 5 of the Footprint Ladder, and it is the right answer more often
than people expect.** If an integration already has an MCP server, wrapping it as a
Hermes plugin duplicates work that someone else maintains.

---

## 7. Worked examples — the four layers in one job each

### A. Repo watchdog, zero tokens

```
Layer 1: none (no_agent)
Layer 2: none
Layer 3: deliver=telegram
Layer 4: none
```
```python
create_job(prompt=None, schedule="every 10m", script="ci_status.sh",
           no_agent=True, deliver="telegram", name="CI watchdog")
```
Empty stdout = silence. No model involved at all.

### B. Competitor monitor that stays quiet

```
Layer 1: enabled_toolsets=["web"]
Layer 2: skills=["competitor-news-monitor", "grounded-citations"]
Layer 3: deliver="origin"
Layer 4: auxiliary.monitor on a cheap model
```
```python
create_job(
    prompt="Check the named competitors for material news since the last run. "
           "Deliver a cited digest of material events only. If there is nothing "
           "material, respond with [SILENT].",
    schedule="0 8 * * 1-5",
    skills=["competitor-news-monitor", "grounded-citations"],
    enabled_toolsets=["web"],
    deliver="origin",
    name="Competitor watch",
)
```

### C. Pricing page diff, agent only when something changed

```
Layer 1: enabled_toolsets=["web"]
Layer 2: none
Layer 3: deliver="slack"
Layer 4: monitor_url hashing suppresses most ticks
```
```python
create_job(
    prompt="The pricing page changed. Summarize exactly what changed and whether "
           "it affects our plan. Be specific about numbers.",
    schedule="every 1h",
    monitor_url="https://competitor.example.com/pricing",
    enabled_toolsets=["web"],
    deliver="slack",
)
```

### D. Nightly repo hygiene, sandboxed

```
Layer 1: enabled_toolsets=["file","terminal","delegation"]
Layer 2: skills=["simplify-code","requesting-code-review"]
Layer 3: deliver="origin"
Layer 4: terminal.backend = docker
```
```python
create_job(
    prompt="Review commits from the last 24h in this repo for obvious quality "
           "issues. Open no PRs. Write findings to reports/nightly.md. "
           "If there are no findings, respond with [SILENT].",
    schedule="0 2 * * *",
    workdir="/home/me/projects/api",
    skills=["simplify-code", "requesting-code-review"],
    enabled_toolsets=["file", "terminal", "delegation"],
)
```
`workdir` pulls in that repo's `AGENTS.md`. Set `terminal.backend: docker` before
running anything like this unattended.

### E. Two-stage pipeline

```python
collector = create_job(
    prompt="Fetch and normalize yesterday's support tickets to JSON. Output JSON only.",
    schedule="0 6 * * *", enabled_toolsets=["web"], deliver="local")

create_job(
    prompt="From the ticket JSON in context, identify the top 3 recurring themes "
           "with counts and one representative quote each.",
    schedule="0 7 * * *", context_from=collector["id"],
    enabled_toolsets=[], deliver="origin")
```
The second job needs **no tools at all** — its input arrives via `context_from`.

---

## 8. Anti-patterns

### Assuming chat context

Every blueprint prompt is self-contained, and the file header says why:

> Keep prompts self-contained (cron jobs run with no chat context).

`"Continue what we discussed"` does nothing. `"Follow up on the migration"` does
nothing. Name the repo, the file, the URL, the criteria — every time.

### Leaving `enabled_toolsets` unset

You pay for ~50 tool schemas on every tick forever, and you hand a scheduled job a
capability surface it doesn't need.

### Running autonomous shell against `local`

The single riskiest configuration in Hermes. Use `docker` or a cloud sandbox
([04-layer4-backends.md §4](04-layer4-backends.md)).

### Using an LLM for a threshold check

`no_agent=True` exists for this. If the logic is `if disk > 90: alert`, no model
should be involved.

### Timestamps in monitor output

Defeats hash suppression — every tick looks like a change and you get the cost of
an unmonitored job plus the complexity of a monitored one.

### Expecting the agent to message you

There is no agent-callable `send_message`. Delivery is `deliver=`, the kanban
notifier, or `hermes send` — never a model decision.

### Forgetting `standalone_sender_fn` on a custom platform

Your job delivers in dev (gateway co-resident) and silently fails in production
(separate processes). See [03-layer3-surfaces.md §4](03-layer3-surfaces.md).

---

## 9. Operational notes

**Inactivity timeout: 600 seconds**, overridable via `HERMES_CRON_TIMEOUT`
(`0` = unlimited). `_DEFAULT_CRON_INACTIVITY_TIMEOUT = 600.0` at
`cron/jobs.py:209`.

> **⚠ Doc drift.** `AGENTS.md` describes a "3-minute hard interrupt" on cron. It is
> a **600s inactivity** timeout — a different mechanism with a different number.

**Success and delivery are separate outcomes.**

> a job can succeed (agent produced output) but fail delivery (platform down)
> — `cron/jobs.py:2382`

Check the execution ledger, not just whether you got a message.

**Delivery defaults to `origin`** when an origin is known, else `local`
(`cron/jobs.py:1873`) — a job created from Telegram
replies to Telegram by default.

**Jobs are profile-scoped.** Job storage anchors per-profile, deliberately not at
the shared root (`cron/jobs.py:74`). Jobs created under
one profile do not appear under another.

**A platform is only a valid `deliver=` target if it declares
`cron_deliver_env_var`.**

---

## 10. Where to start

1. Read `cron/blueprint_catalog.py` — 16 worked
   prompts written by the scheduler's authors
2. Run `/suggestions` and accept **Daily briefing** — see the whole loop end to end
3. Build one `no_agent=True` watchdog — automation with no LLM variables
4. Build one `[SILENT]` monitor — the pattern that makes alerts sustainable
5. Only then reach for delegation or kanban

And before you write any code for a new capability, walk the Footprint Ladder in
`AGENTS.md` → "The Footprint Ladder". Most of what feels like a
missing feature is a skill, a cron job, or an MCP server.

---

## 11. Self-check

<details>
<summary>1. Your hourly monitor costs 168 model calls a week for 2 real changes. Fix?</summary>

`monitor_url` or `monitor_script`. The source is hashed each tick; unchanged output
suppresses the agent run entirely. You pay for 2 calls, not 168. Make sure the
source emits stable bytes — no timestamps.
</details>

<details>
<summary>2. A job says "check the deploy we set up yesterday" and does nothing useful. Why?</summary>

Cron jobs run with **no chat context**. There is no "yesterday" for the job to
recall. Every prompt must name the repo, environment, URL, and criteria explicitly.
</details>

<details>
<summary>3. When should an automation have <code>no_agent=True</code>?</summary>

Whenever the logic is deterministic — threshold checks, classic watchdogs,
periodic alerts. The script's stdout is delivered verbatim; empty stdout is
silence. Zero tokens, zero nondeterminism.
</details>

<details>
<summary>4. Why can't a subagent spawn its own subagents through a composite?</summary>

`_COMPOSITE_BLOCKED_TOOLSETS = frozenset({"delegation"})`. It's the recursion
guard — without it, one delegation could fan out exponentially.
</details>

<details>
<summary>5. Job B needs Job A's output. How?</summary>

`context_from="<job A id>"` — A's most recent output is injected into B's prompt
before each run. Accepts a list for several sources. That's the pipeline primitive.
</details>

<details>
<summary>6. Your job reported success but you got no message. Where do you look?</summary>

The execution ledger. Success (agent produced output) and delivery (platform
accepted it) are tracked separately by design. Also check `dead_targets.py` and
whether the platform declares `cron_deliver_env_var`.
</details>

---

## Where you are now

You've covered all four layers:

| | Layer | File |
|---|---|---|
| 1 | Toolsets — 59 bundles, 135 tools | [01-layer1-toolsets.md](01-layer1-toolsets.md) |
| 2 | Skills — 82 packs | [02-layer2-skills.md](02-layer2-skills.md) |
| 3 | Surfaces — 31 adapters, 6 UIs | [03-layer3-surfaces.md](03-layer3-surfaces.md) |
| 4 | Backends — 35 providers, 8 environments | [04-layer4-backends.md](04-layer4-backends.md) |
| — | Composition | this file |

Next, if you want to contribute rather than configure:
`AGENTS.md` → "The Footprint Ladder" (the ladder) →
`CONTRIBUTING.md` (working code) →
`CONTRIBUTING.md` (getting it merged).
