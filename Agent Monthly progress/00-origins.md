# 00 · Origins — July 2025 to January 2026

**The month in one line:** Hermes wasn't a personal agent. It was a research
harness for generating tool-calling training data.

**Next:** [01 · February 2026](01-february-2026.md) — the pivot

---

## At a glance

| | |
|---|---|
| Period | 2025-07-22 → 2026-01-31 |
| Commits | **92** (in six months) |
| Merged PRs | 11 (#1 through #14) |
| Releases | none — no versioning existed yet |
| At the end | 208 `.py` files · 34 tool modules · 46 skills · 63 test files |

Ninety-two commits in six months. For comparison, the project would later ship
6,060 commits in July 2026 alone. This was a side project inside Nous Research,
not a product.

---

## What it was for

Nous Research trains models. Training a model to *use tools* requires trajectory
data: transcripts of an agent calling tools, getting results, and continuing. So
they built a harness to generate that data at scale.

Read the early commit log and the purpose is unmistakable:

```
2025-07-22  initital commit
2025-07-25  terminal tool
2025-07-31  implement first pass of scrape/crawl content compression
2025-08-04  add vision model tool, cli updates for exclusive and inclusive toolsets
2025-08-09  add mixture of agents tool
2025-08-09  add image generation tool
2025-08-31  Fix Web Tools, Upgrade MoA to GPT5, Add Trajectory Saving
2025-09-10  Update to use toolsets and make them easy to create and configure
2025-10-06  Add batch processing capabilities with checkpointing and statistics
2026-01-23  Add mini-swe-agent runner and trajectory compressor
```

**Trajectory saving. Batch processing with checkpointing. A mini-SWE-agent
runner. A trajectory compressor.** These are data-generation concerns, not
assistant concerns. Nobody builds a batch runner with statistics tracking for a
personal chat agent.

---

## What existed by the end of January 2026

### The agent core

`run_agent.py` and `model_tools.py` date from the very first commits (July 2025)
and are still the load-bearing files today. The loop that drives a turn — call
model, dispatch tools, repeat — was there from week one and has never been
replaced, only extracted and hardened.

### Toolsets

`toolsets.py` arrived in September 2025 with the commit *"Update to use toolsets
and make them easy to create and configure."* The idea that tools come in
named, selectable bundles predates almost everything else — and the `all` / `*`
alias landed in October.

This matters for understanding the "narrow waist" policy: toolsets were not
invented later to control schema bloat. They were the original organizing idea,
and the policy grew out of them.

### The tools

By the end of January the harness had: terminal, web scrape/crawl with content
compression, vision, image generation, mixture-of-agents, browser automation
(added 2026-01-29), and skills tools (2026-01-30). 34 tool modules.

### Batch and research infrastructure

- `batch_runner.py` (Oct 2025) — parallel batch processing with checkpointing and
  statistics.
- `mini_swe_runner.py` (Jan 2026) — a mini-SWE-agent runner.
- `trajectory_compressor.py` (Jan 2026) — compress trajectories for training,
  with sampling options.
- Ephemeral system prompt support in batch and agent runners, explicitly *"not
  saved to trajectories"* — a detail that only makes sense if trajectories are the
  product.

All four still exist in the tree today. `batch_runner.py` is 57 KB and
`trajectory_compressor.py` is 70 KB — the research lineage never got deleted, it
just stopped being the point.

### The CLI — added in the last 48 hours of the era

```
2026-01-31  Add a claude code-like CLI
```

This is the hinge. On January 31, 2026, someone added an interactive CLI. Two and
a half weeks later the project became a personal agent.

---

## What did *not* exist yet

Worth listing, because it shows how much of Hermes is younger than it looks:

- **No memory.** No persistent state at all — no `hermes_state.py`, no SQLite.
- **No sessions.** Nothing to resume.
- **No gateway.** No Telegram, no Discord, no messaging of any kind.
- **No `agent/` directory.** All agent logic lived in `run_agent.py`.
- **No `hermes_cli/` package.** No subcommands, no setup wizard.
- **No profiles.** One instance, one config.
- **No plugins.** No extension mechanism whatsoever.
- **No cron.** No scheduling.
- **No TypeScript.** No TUI, no dashboard, no desktop app.
- **No provider plugins.** Provider handling was inline.
- **No tests to speak of.** 63 test files, mostly validation scripts.

---

## Themes that survived

Three ideas from this era shaped everything after it.

**Toolsets as the unit of capability.** Tools are grouped, and groups are
selectable per context. This became `_HERMES_CORE_TOOLS` and the 34-key `TOOLSETS`
dict, and it is the mechanism that makes the narrow-waist policy enforceable.

**Content compression as a first-class concern.** The very first substantive
feature after the terminal tool was *"scrape/crawl content compression"* (July
31, 2025). Managing how much text reaches the model has been a preoccupation from
the start — and it is why context compaction is the most heavily engineered
subsystem in the tree today.

**Trajectories matter.** Because the project was born to produce training data,
it has always recorded what happened in detail. That instinct shows up now as
session persistence, `sessions export --format trace`, MoA trace persistence, and
the compaction eval harness.

---

## Also visible: the problems that never went away

The early log is full of leakage bugs:

```
2025-07-26  fix history leakage
2025-11-03  fix leakage
2025-11-04  prevent leakage of morph instances between tasks
```

State bleeding between tasks was a problem in month one and is still a problem
class today — it is why every profile gets its own `HERMES_HOME`, why subagents
get isolated contexts, why kanban workers get a pinned board env var, and why
`_get_scoped_secret()` must fail closed. The isolation obsession in this codebase
is scar tissue from 2025.

---

## The state of the repo on 2026-01-31

```
run_agent.py            the agent loop
model_tools.py          tool orchestration
toolsets.py             named tool bundles
cli.py                  a claude-code-like CLI (one day old)
batch_runner.py         parallel trajectory generation
mini_swe_runner.py      SWE-agent harness
trajectory_compressor.py  training-data compression
tools/                  34 modules
skills/                 46 SKILL.md files
docs/                   architecture notes
tests/                  63 files
```

208 Python files. No memory, no sessions, no messaging, no UI beyond the
one-day-old CLI.

Nineteen days later it had all of them.

---

**Next:** [01 · February 2026 →](01-february-2026.md)
