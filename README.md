# The Hermes Agent Guide

![Image](Image.png)

A newcomer's path through this codebase, and a contributor's path to a merged PR.

Everything here was written against `main` @ `cf64ca20c` (2026-08-17) by reading
the tree, not from memory. Where the code disagrees with this guide, **the code
wins** — and please fix the guide.

---

## Who this is for

| You are… | Start at | Then read |
|---|---|---|
| New to the repo, want the mental model | [01 · What Hermes Is](01-what-hermes-is.md) | 02, 03 |
| Setting up to develop | [03 · Dev Setup](03-dev-setup.md) | 01, 06 |
| About to build a feature | [04 · Extending Hermes](04-extending-hermes.md) | 05, 07 |
| Looking for a copy-paste recipe | [05 · Recipes](05-recipes.md) | 06 |
| About to open a PR | [07 · Contributing & PRs](07-contributing-prs.md) | 06 |
| Reviewing someone else's PR | [07 · Contributing & PRs](07-contributing-prs.md) | 04, 06 |

---

## The eight files

| # | File | What it covers |
|---|---|---|
| 01 | [what-hermes-is.md](01-what-hermes-is.md) | The mental model: one loop, six surfaces, two sacred invariants, one turn traced end to end |
| 02 | [architecture.md](02-architecture.md) | The map: agent core, tool layer, state, surfaces, gateway, extension edges — plus a "where do I look for X" index |
| 03 | [dev-setup.md](03-dev-setup.md) | Install, run each surface, `config.yaml` vs `.env`, profiles, logs, debugging |
| 04 | [extending-hermes.md](04-extending-hermes.md) | The Footprint Ladder: how to decide *where* your feature belongs before you write it |
| 05 | [recipes.md](05-recipes.md) | Seven worked end-to-end recipes: slash command, config key, skill, core tool, plugin, platform adapter, model provider |
| 06 | [testing.md](06-testing.md) | `scripts/run_tests.sh`, where tests live, and the four test patterns that are banned outright |
| 07 | [contributing-prs.md](07-contributing-prs.md) | The full path to merge: search → rung → branch → commit → CI → attribution → review, and why PRs get closed |
| — | [progress/](progress/README.md) | **How Hermes got here** — a monthly history from July 2025 to August 2026, with what shipped each month |

Read 01 → 02 → 03 for orientation (about 40 minutes). Read 04 → 05 → 06 → 07
when you are ready to build something. Read [progress/](progress/README.md) when
you want to know *why* something is the way it is — most odd decisions in this
codebase are legible once you know what month they were made in.

---

## The 60-second version

Hermes is a personal AI agent. One synchronous conversation loop — call the
model, run the tools it asked for, repeat — is wrapped in six front ends: a
CLI, a terminal UI, a messaging gateway (28 platform adapters), an Electron
desktop app, a browser dashboard, and an ACP server for editors. None of them
reimplements the loop.

Two rules shape nearly every design decision:

1. **Prompt caching is sacred.** A long conversation reuses a cached prefix
   every turn. Anything that rewrites past context, swaps toolsets, or rebuilds
   the system prompt mid-conversation destroys that cache and multiplies the
   user's bill.
2. **The core is a narrow waist.** Every core tool's schema is sent on *every*
   API call, forever. So capability belongs at the edges — skills, plugins, CLI
   commands, MCP servers — not in the core toolset.

If you internalize only those two, you will make mostly correct decisions here.

---

## Vocabulary

Terms used throughout the codebase and this guide.

| Term | Meaning |
|---|---|
| **Surface** | A front end a user talks through: CLI, TUI, desktop, dashboard, gateway, ACP. |
| **Agent core** | `AIAgent` in `run_agent.py` plus `agent/` — the loop, provider adapters, compression, prompt assembly. |
| **Turn** | One user message and everything the agent does before it replies: N model calls and tool batches. |
| **Tool** | A function the model can call. Defined in `tools/`, registered at import time, exposed only if a toolset names it. |
| **Toolset** | A named bundle of tool names in `toolsets.py`. Platforms pick a base toolset; `_HERMES_CORE_TOOLS` is the default. |
| **Skill** | A Markdown playbook (`SKILL.md`) that teaches the agent a procedure. Costs no schema space. |
| **Plugin** | A directory with `plugin.yaml` + `register(ctx)` that adds tools, hooks, CLI commands, platforms, or providers without touching core. |
| **Profile** | A fully isolated Hermes instance with its own `HERMES_HOME` (config, keys, sessions, skills). |
| **`HERMES_HOME`** | The state directory. `~/.hermes` by default, `~/.hermes/profiles/<name>` under a profile. Always reach it via `get_hermes_home()`. |
| **Gateway** | The long-lived process that connects chat platforms to the agent (`gateway/run.py`). |
| **Delegation** | Spawning a subagent with its own context and terminal (`delegate_task`). |
| **Curator** | The background job that tracks skill usage and archives stale agent-written skills. |
| **Kanban** | A durable SQLite board where a dispatcher claims tasks and spawns whole profiles as workers. |
| **Compaction / compression** | Summarizing old context to fit the window. The one sanctioned exception to cache stability. |
| **Auxiliary model** | A side LLM used for non-conversational work (titles, vision, curator review, search summarization). |
| **Salvage** | Rebuilding a stale or abandoned PR on current `main` instead of closing it. A first-class practice here. |
| **Footprint** | How much permanent surface a change adds. Lower is better. See [04](04-extending-hermes.md). |

---

## The three documents that outrank this guide

This guide explains and connects; those documents are **normative**.

| Document | Size | What it is |
|---|---|---|
| `AGENTS.md` | ~1,570 lines | The development bible: rubric, footprint ladder, per-subsystem rules, known pitfalls, testing law. |
| `CONTRIBUTING.md` | ~990 lines | Contributor-facing setup, skill/tool decision, cross-platform rules, PR process. |
| `.github/PULL_REQUEST_TEMPLATE.md` | 75 lines | The checklist your PR is graded against. |

### Where those documents have drifted from the code

`AGENTS.md` is normative for **intent and policy** — the rubric, the footprint
ladder, the review bar. But it is prose maintained by hand against a tree that
ships ~50 features a week, and some of its concrete values have gone stale. Every
divergence below was found by checking the code while writing this guide:

| `AGENTS.md` says | The code says | Where |
|---|---|---|
| `max_iterations: int = 500` | **90** | `run_agent.py:446`, `agent/agent_init.py:504` |
| "3-minute hard interrupt" on cron | **600s** inactivity timeout, `HERMES_CRON_TIMEOUT` overrides | `cron/jobs.py` (`_DEFAULT_CRON_INACTIVITY_TIMEOUT`) |
| `delegation.max_concurrent_children` default 3 | **10** (raised in Aug 2026, PR #86745) | `hermes_cli/config_defaults.py` |
| Add config to `DEFAULT_CONFIG` in `hermes_cli/config.py` | Defined in **`config_defaults.py`**; `config.py:945` re-exports it | `hermes_cli/config_defaults.py:7` |

`gateway/platforms/ADDING_A_PLATFORM.md` has drifted too — it points at
`gateway/platforms/whatsapp.py`, `telegram.py`, and `discord.py`, all of which
moved to `plugins/platforms/`. And `AGENTS.md` cites a skill-PR salvage checklist
in a `hermes-agent-dev` skill that is not in the tree at all; the real bundled
skill is `skills/autonomous-ai-agents/hermes-agent/`.

**Rule of thumb:** trust those documents for *why* and *what is allowed*; verify
any specific number, path, or default against the tree. This guide states the
code's value and flags the divergence.

Also worth knowing about:

- `gateway/platforms/ADDING_A_PLATFORM.md` — the platform-adapter contract.
- `website/docs/developer-guide/` — the published docs, including the plugin
  compatibility contract and the model-provider authoring guide.
- `docs/ADR.md`, `docs/rfcs/` — decisions already made, with reasons.

---

## How to keep this guide honest

Every factual claim here should be checkable with one command. When you change
something this guide describes, update it in the same PR — the PR template has
a line for exactly that ("I've updated `CONTRIBUTING.md` or `AGENTS.md` if I
changed architecture or workflows").

Scale reference, so you know what you are walking into:

```
23,405 commits            1,055 merged feature PRs      12,149 fix commits
1.72M lines of Python     516K lines of TS/TSX          3,130 test files
121 tool modules          199 skills (82+117)           35 model providers
22 platform plugins       8 memory backends             8 terminal backends
```
