# Hermes Agent — Progress Timeline

How Hermes got from a 200-line research harness to a 1.7M-line product, month by
month.

Read this when you want to know **why** something is the way it is. Most odd
decisions in this codebase are legible once you know what month they were made
in and what else was happening that month.

**Baseline:** `main` @ `cf64ca20c`, read 2026-08-18. All counts come from git
history and the filesystem.

---

## The files

| File | Period | The month in one line |
|---|---|---|
| [00 · Origins](00-origins.md) | Jul 2025 – Jan 2026 | A research harness for generating tool-calling trajectories |
| [01 · February 2026](01-february-2026.md) | Feb | The pivot: it becomes a personal agent with memory, skills, and a gateway |
| [02 · March 2026](02-march-2026.md) | Mar | Public, versioned, and everywhere — MCP, profiles, plugins, six platforms |
| [03 · April 2026](03-april-2026.md) | Apr | Peak feature month: the TUI, the dashboard, and Kanban all arrive |
| [04 · May 2026](04-may-2026.md) | May | Hardening: security, supply chain, checkpoints — and the desktop app starts |
| [05 · June 2026](05-june-2026.md) | Jun | The desktop app takes over; billing, voice groundwork, and the relay begin |
| [06 · July 2026](06-july-2026.md) | Jul | Busiest month ever: voice, relay parity, artifacts, theme SDK |
| [07 · August 2026](07-august-2026.md) | Aug 1–17 | The plugin platform grows up: manifest v2, capabilities, consent, packs |

---

## The arc in five sentences

1. **Jul 2025 – Jan 2026** — Nous Research builds an internal harness to generate
   agentic tool-calling trajectories for model training. 92 commits in six
   months. A terminal tool, a batch runner, a trajectory compressor.
2. **Feb 2026** — On the 19th it pivots hard: a SQLite session store, persistent
   memory, agent-created skills, subagent delegation, and a messaging gateway
   land in a single fortnight. It stops being a data-generation tool and becomes
   a personal agent.
3. **Mar–Apr 2026** — Reach explodes. MCP, profiles, plugins, ~20 chat
   platforms, an OpenAI-compatible API server, a React TUI, a web dashboard, a
   multi-agent Kanban board. Twelve releases in eight weeks.
4. **May–Jun 2026** — The surface stops widening and starts hardening: promptware
   defense, supply-chain auditing, secret redaction on by default, checkpoints
   v2. Simultaneously an Electron desktop app appears and immediately becomes the
   largest single area of work.
5. **Jul–Aug 2026** — The extension surfaces become products in their own right:
   a cross-surface theme SDK, a TUI widget SDK, a desktop plugin SDK, plugin
   manifest v2 with capability declarations and consent. Hermes becomes a
   platform other people build on.

---

## Growth, month by month

Measured on the last commit of each month (by **committer** date — author dates
are unreliable here because of heavy rebasing and PR salvage).

| Month end | `.py` files | `.ts`/`.tsx` | `SKILL.md` | `apps/desktop` | `tests/` | `tools/*.py` |
|---|---|---|---|---|---|---|
| Feb 2026 | 208 | 0 | 46 | — | 63 | 34 |
| Mar 2026 | 631 | 2 | 117 | — | 385 | 47 |
| Apr 2026 | 1,358 | 363 | 150 | — | 915 | 69 |
| May 2026 | 2,018 | 686 | 177 | 323 | 1,360 | 82 |
| Jun 2026 | 2,714 | 1,165 | 174 | 801 | 1,869 | 89 |
| Jul 2026 | 3,705 | 1,934 | 182 | 1,446 | 2,621 | 106 |
| Aug 17 | 4,328 | 2,268 | 200 | 1,777 | 3,142 | 121 |

Two things to notice.

**Tests grew faster than code.** From 63 test files to 3,142 — outpacing the
Python source count in relative terms and reaching roughly 0.73 test files per
source file.

**Tool count nearly stopped growing.** 34 → 121 tool modules over seven months,
while skills went 46 → 200 and TypeScript went 0 → 2,268. That is the "narrow
waist" policy visible in the file counts: capability moved to the edges, not into
the tool schema.

---

## Velocity, month by month

| Month | Commits | `feat` | `fix` | fix : feat |
|---|---|---|---|---|
| Feb 2026 | 481 | 91 | 75 | 1 : 0.8 |
| Mar 2026 | 2,463 | 413 | 1,184 | 1 : 2.9 |
| Apr 2026 | 3,865 | 600 | 2,119 | 1 : 3.5 |
| May 2026 | 3,249 | 412 | 1,698 | 1 : 4.1 |
| Jun 2026 | 3,700 | 498 | 1,931 | 1 : 3.9 |
| Jul 2026 | 6,060 | 653 | 3,223 | 1 : 4.9 |
| Aug 1–17 | 3,495 | 379 | 1,919 | 1 : 5.1 |

The **fix : feat ratio is the maturity curve**. In February the project shipped
more features than fixes. By August it ships five fixes per feature. Nothing
about the feature rate declined — `feat` commits went *up* — the hardening load
simply grew faster than the feature load, which is what happens when 25 chat
platforms, 8 execution backends, 3 desktop OSes, and 35 providers all have to
keep working at once.

---

## Releases

24 tagged releases from v0.2.0 to v0.20.2.

| Month | Releases |
|---|---|
| Mar 2026 | v0.2.0 · v0.3.0 · v0.4.0 · v0.5.0 · v0.6.0 |
| Apr 2026 | v0.7.0 · v0.8.0 · v0.9.0 · v0.10.0 · v0.11.0 · v0.12.0 |
| May 2026 | v0.13.0 · v0.14.0 · v0.15.0 · v0.15.1 |
| Jun 2026 | v0.16.0 · v0.17.0 |
| Jul 2026 | v0.18.0 · v0.18.1 · v0.18.2 · v0.19.0 · v0.19.1 |
| Aug 2026 | v0.20.0 · v0.20.1 · v0.20.2 |

Release cadence slowed as the product grew — six releases in April, two in June —
while commit volume rose. Bigger releases, more testing between them.

---

## When each subsystem was born

The month a directory or file first appeared in the tree:

| Subsystem | Born |
|---|---|
| `run_agent.py`, `model_tools.py` | Jul 2025 |
| `toolsets.py` | Sep 2025 |
| `tools/`, `tests/`, `batch_runner.py` | Oct 2025 |
| `skills/`, `docs/`, `cli.py`, `trajectory_compressor.py` | Jan 2026 |
| `agent/`, `gateway/`, `hermes_cli/`, `hermes_state.py`, `cron/`, `evals/` | Feb 2026 |
| `acp_adapter/`, `website/`, `optional-skills/`, `mcp_serve.py` | Mar 2026 |
| `ui-tui/`, `tui_gateway/`, `web/`, `plugins/` | Apr 2026 |
| `apps/desktop/`, `providers/` | May 2026 |
| `native/` (CJK FTS5) | Jul 2026 |

---

## How to read a monthly file

Each one has the same shape:

- **At a glance** — commits, features, fixes, releases
- **Where things stood** at the start of the month
- **The month's defining shift** — the one thing that mattered most
- **What shipped**, grouped by area, with real feature names
- **New in the tree** — subsystems born that month
- **Where things stood** at the end

Feature names are taken verbatim from commit subjects, so you can find any of
them:

```bash
git log --oneline --grep="<feature name>"
```

---

## Caveat on dates

This repo has 2,765 commit authors and a strong culture of *salvaging* stale PRs
— rebuilding an abandoned contribution on current `main`. That means:

- **Author dates** say when work was *written*.
- **Committer dates** say when it *landed on `main`*.

They can differ by months. The growth and velocity tables above use committer
dates (what shipped when). Feature attribution in the monthly files follows the
same convention, so a feature credited to July may have been authored in May by
someone whose PR was salvaged later.
