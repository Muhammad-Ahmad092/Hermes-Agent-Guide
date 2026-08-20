# 01 · February 2026

**The month in one line:** the pivot — in a single fortnight the research harness
grew memory, skills, subagents, and a messaging gateway, and became a personal
agent.

**Previous:** [00 · Origins](00-origins.md) · **Next:** [02 · March 2026](02-march-2026.md)

---

## At a glance

| | |
|---|---|
| Commits | **481** (5× the previous six months combined) |
| `feat` commits | 91 |
| `fix` commits | 75 |
| fix : feat | **1 : 0.8** — the only month features outnumbered fixes |
| Releases | none yet — versioning arrived in March |
| Feature window | **Feb 19 – Feb 28** — zero `feat` commits in the first 18 days |
| Month end | 208 `.py` · 46 skills · 34 tools · 63 test files |

---

## The defining shift

February splits cleanly in two, and the split is visible in the commit types.

**Feb 1–18: 131 commits, not one of them a feature.** Steady work — 5 to 19
commits a day — all fixes, chores, and restructuring on the harness that January
left behind. Groundwork.

**Feb 19–28: 350 commits, and the entire product.** On February 19 alone, these
all landed:

```
feat: add persistent memory system + SQLite session store
feat: introduce skill management tool for agent-created skills and skills migration to ~/.hermes
feat: add messaging gateway startup functionality
feat: implement cross-channel messaging functionality
feat: implement code execution sandbox for programmatic tool calling
feat: introduce clarifying questions tool for interactive user engagement
```

Memory, skills, a gateway, cross-channel messaging, a code sandbox, and a
clarify tool — in one day, after eighteen days of no features at all. The next day
brought subagent delegation and multi-provider model selection. Nine days later the
month closed with a hooks system, Home Assistant integration, and dynamic skill
slash commands.

**This is the month Hermes stopped being infrastructure for training models and
became something a person talks to.** Every one of those six features exists to
serve a human user, not a batch job.

---

## What shipped

### Memory and state — the foundation

`hermes_state.py` is born this month. Everything that makes Hermes feel
continuous starts here.

- **Persistent memory system + SQLite session store** — the single most
  consequential commit of the month
- Database schema and message persistence, then a schema update days later
- Timestamp formatting for session metadata
- Memory management features in `AIAgent` and the CLI
- Session search with parent-session resolution and **parallel summarization**
- Session reset policy for messaging platforms
- Ephemeral prefill messages and system-prompt loading

### Skills — the agent starts writing its own

- **Skill management tool for agent-created skills**, plus migration of skills to
  `~/.hermes/`
- Skills management surfaced in both `AIAgent` and the CLI
- **Dynamic skill slash commands for CLI and gateway** — the mechanism that still
  powers `/<skill-name>` today
- New skills: PPTX editing and creation, OCR and document extraction, arXiv
  search, ascii-art, Solana blockchain (converted from a tool — an early example
  of the tool→skill demotion the rubric now prescribes), Superpowers software
  development skills, Notion block-type references

### The messaging gateway

The `gateway/` directory is born.

- Gateway startup functionality
- **Cross-channel messaging** and a channel directory with message mirroring
- User authorization checks in `GatewayRunner`
- Pairing store and an event hook system
- Telegram: document processing for PDF, text, and Office files; animated GIF
  support
- Slack and WhatsApp setup prompts in the setup wizard
- Model command resolution from environment and config

### Delegation — subagents arrive

- **Subagent delegation for task management**
- Task delegation with spinner updates and progress display
- Tool progress callbacks threaded through `delegate_tool`

### Providers and models

- **Multi-provider authentication and inference provider selection**
- Interactive model selection with saving
- Provider deactivation
- Dynamic max-tokens handling per provider
- Reasoning-effort configuration

### Approvals and safety

- **Interactive prompts for sudo password and command approval** in the CLI
- Password masking and placeholder text in CLI input
- Improved password prompt handling in the terminal tool
- API-key requirement checks for toolsets

### The CLI becomes usable

Nine separate commits on CLI presentation in one month — borders, ANSI handling,
input-area height, horizontal rules replacing framed input, spinner output
handling, shell noise filtering, using the user's login shell for command
execution. The `/verbose` slash command lands, and toolset selection becomes
platform-specific with a proper checklist UI.

### Memory providers — Honcho, the first one

- Honcho AI-native memory integration
- Honcho + `USER.md` memory system
- Honcho for cross-session user modeling

This is the origin of the pluggable-memory idea, though the `MemoryProvider` ABC
that generalizes it doesn't arrive until April.

### Scheduling, hooks, and integrations

- `cron/` is born — job delivery mechanism in the scheduler
- **Event hooks system for lifecycle management**, and a `session:end` event
- **Home Assistant integration** — REST tools plus a WebSocket gateway
- Persistent error logging for tool failures

### Infrastructure

- `hermes_cli/` and `agent/` directories are born
- `evals/` is born
- Node.js installation support in the setup script
- `docker_volumes` config for custom volume mounts
- Docker `--storage-opt` support check
- `config.yaml` values integrated into the environment
- A landing page, and a branding/visuals pass across the project

---

## New in the tree this month

| Path | What it became |
|---|---|
| `hermes_state.py` | The SQLite session store — now 13,086 lines |
| `agent/` | Agent internals — now 140 modules |
| `gateway/` | The messaging gateway |
| `hermes_cli/` | CLI subcommands — now 204 modules |
| `cron/` | The scheduler |
| `evals/` | Evaluation harnesses |

Six directories that are still structural pillars, all created in ten days.

---

## Where things stood at the end of February

**Working:** an interactive CLI with memory that survives restarts, skills the
agent can write and load, subagent delegation, a messaging gateway with
cross-channel mirroring, Telegram document handling, multi-provider model
selection, command approval, session search, cron delivery, Honcho memory, and
Home Assistant control.

**Not there yet:** MCP (March), profiles (March), plugins (March–April), the TUI
(April), the web dashboard (April), Kanban (April), the desktop app (May),
provider plugins (May), billing (June), voice (July). No versioning, no releases,
no `CONTRIBUTING.md` discipline yet — and only 63 test files against 208 source
files.

The bill for that last one comes due immediately: March ships 1,184 fix commits.

---

## Reading this month in today's code

Several February decisions are still visible and still load-bearing:

- **Skill slash commands are injected as user messages.** Established here, and
  now the canonical example of protecting the prompt cache.
- **The clarify tool** with configurable timeout and countdown display — still
  the mechanism behind `clarify.respond` on every surface.
- **Cross-channel mirroring and the channel directory** — the reason a
  conversation can move between Telegram and the CLI.
- **`config.yaml` values integrated into the environment** — the ancestor of
  today's strict rule that behavior lives in `config.yaml` and `.env` holds only
  secrets. The bridging pattern (`terminal.cwd → TERMINAL_CWD`) starts here.

---

**Previous:** [← 00 · Origins](00-origins.md) · **Next:** [02 · March 2026 →](02-march-2026.md)
