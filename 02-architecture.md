# 02 · Architecture

**For:** anyone who needs to find the right file before changing something.
**Prerequisites:** [01 · What Hermes Is](01-what-hermes-is.md)
**After this file you will know:** what each layer owns, how the layers talk,
and where to look for any given behavior.
**Next:** [03 · Dev Setup](03-dev-setup.md)

> File counts shift constantly. The canonical source is the filesystem —
> everything below points at load-bearing entry points, not an exhaustive tree.

---

## 1. The stack, top to bottom

```
SURFACES        CLI · TUI · Desktop · Dashboard · Gateway · ACP
                        │
TRANSPORT       tui_gateway — JSON-RPC methods + events
                        │
AGENT CORE      AIAgent.run_conversation
                  model call → tool dispatch → compress → post-turn hooks
                        │
                ┌───────┴───────┐
TOOLS           tools/registry   STATE  hermes_state.SessionDB
                121 modules             SQLite + FTS5
                8 exec backends
                        │
EDGES           plugins/ (providers, platforms, memory, image_gen)
                skills/ + optional-skills/
```

Two boundaries matter more than the rest:

1. **Surfaces never touch tools or the session store directly.** They speak
   JSON-RPC to `tui_gateway`. This is why a feature added to the Ink TUI appears
   in the dashboard automatically — the dashboard *embeds* the TUI.
2. **Tools are registered at import time but only exposed if a toolset names
   them.** Discovery and exposure are separate steps, deliberately.

---

## 2. Agent core

**Owns:** the turn loop, provider adapters, prompt assembly, context
compression, credential pooling, budgets, retries and failover.

| File | Role |
|---|---|
| `run_agent.py` (9,051 lines) | The `AIAgent` class. `__init__` takes ~60 parameters. `run_conversation()` is now a thin forwarder. |
| `agent/conversation_loop.py` | **The actual loop.** The ~3,900-line `run_conversation` body, extracted. Start here when debugging turn behavior. |
| `agent/agent_init.py` | Construction and wiring of the ~60-parameter surface. |
| `agent/prompt_builder.py`, `agent/system_prompt.py` | Prefix assembly. Cache-sensitive. |
| `agent/context_compressor.py`, `agent/conversation_compression.py` | Compaction: what to summarize, when to trigger, how to rotate. |
| `agent/prompt_cache_boundary.py`, `agent/prompt_cache_scope.py` | Where the cached prefix ends. |
| `agent/error_classifier.py` | Classifies provider failures into retry / failover / surface. |
| `agent/credential_pool.py` | Same-provider key pools with rotation. |
| `agent/auxiliary_client.py` | The side-LLM path. `_resolve_auto` decides which model does non-conversational work. |
| `agent/transports/`, `agent/*_adapter.py` | Per-provider API shapes: Anthropic native, Codex Responses, Bedrock, Vertex, Gemini native, Azure identity. |
| `agent/tool_executor.py` | Concurrent dispatch of a response's tool calls. |
| `agent/display.py` | `KawaiiSpinner` — the animated faces and the `┊` activity feed. |

**The loop, in essence:**

```python
while (api_call_count < self.max_iterations
        and self.iteration_budget.remaining > 0) or self._budget_grace_call:
    if self._interrupt_requested:
        break
    response = client.chat.completions.create(
        model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

Messages are OpenAI-shaped (`{"role": "system|user|assistant|tool", ...}`).
Reasoning content lives in `assistant_msg["reasoning"]`.

**Entry points you will actually call:**

```python
agent.chat(message) -> str                      # simple: final response text
agent.run_conversation(user_message, ...) -> dict  # full: final_response + messages
```

---

## 3. Tool layer

**Owns:** everything the model can *do*.

### The dependency chain

```
tools/registry.py       (no deps — imported by every tool file)
       ↑
tools/*.py              (each calls registry.register() at import time)
       ↑
model_tools.py          (imports the registry + triggers discovery)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### Two-step exposure

**Step 1 — discovery is automatic.** Any `tools/*.py` with a top-level
`registry.register()` call is imported automatically. There is no manual import
list.

**Step 2 — exposure is manual.** The tool reaches an agent only if its name
appears in a toolset in `toolsets.py`. This is deliberate, and
`_HERMES_CORE_TOOLS` is not dead code — it is the default bundle nearly every
platform's base toolset inherits.

Forgetting step 2 is the single most common "my tool doesn't work" bug.

### Toolsets

Defined as one `TOOLSETS` dict in `toolsets.py`. Current keys:

```
bfl · browser · clarify · code_execution · coding · computer_use · context_engine
cronjob · debugging · delegation · desktop_ui · discord · discord_admin
feishu_doc · feishu_drive · file · homeassistant · image_gen · kanban · memory
project · safe · search · session_search · skills · spotify · terminal · todo
tts · video · video_gen · vision · web · x_search · yuanbao
```

Users toggle them with `hermes tools` (a curses UI) or the
`tools.<platform>.enabled` / `.disabled` lists in `config.yaml`.

Three gating mechanisms, for three different questions:

| Mechanism | Answers | Example |
|---|---|---|
| Toolset membership | "Should this surface pay for this tool?" | `desktop_ui` folded in only for GUI sessions |
| `check_fn` | "Is the prerequisite reachable / did the user opt in?" | Home Assistant tools gated on `HASS_TOKEN` |
| `requires_env` | "Which credentials does this need?" | Feeds the setup wizard |

`check_fn` results are **TTL-cached process-wide** (`tools/registry.py`), so a
per-session answer must never live there — one process serves many sessions.

### Terminal execution backends

`tools/environments/` — where the `terminal` tool actually runs commands:

`local` · `docker` · `ssh` · `singularity` · `modal` · `managed_modal` ·
`daytona` · `vercel_sandbox`, plus `file_sync.py` for pushing files into remote
backends.

Daytona and Modal offer serverless persistence — the environment hibernates when
idle and wakes on demand.

### Notable tool families

| Area | Files |
|---|---|
| Files | `file_tools.py`, `file_operations.py` — `read_file`, `write_file`, `patch`, `search_files`, with delta lint and LSP diagnostics on write |
| Browser | `browser_tool.py`, `browser_cdp_tool.py`, `browser_camofox.py`, `browser_use_cli.py`, `browser_supervisor.py` |
| Delegation | `delegate_tool.py`, `async_delegation.py`, `delegation_live_log.py`, `subagent_worktree.py` |
| Approvals | `approval.py` — dangerous-command detection, user deny rules, hardline blocklist |
| Kanban | `kanban_tools.py` |
| MCP | `mcp_tool.py` — the client; servers become standalone toolsets |
| Skills | `skills_hub.py`, `skill_manager_tool.py`, `skill_linter.py`, `skill_usage.py` |

**Every handler must return a JSON string.** The registry handles schema
collection, dispatch, availability, and error wrapping.

---

## 4. State

**Owns:** sessions, messages, reasoning, search, rewind.

| File | Role |
|---|---|
| `hermes_state.py` (13,086 lines) | `SessionDB` — the SQLite session store |
| `hermes_state_schema.py` | Schema and migrations |
| `hermes_state_search.py` | FTS5 search, including the trigram path |
| `hermes_state_portability.py` | Export/import across machines |
| `hermes_state_common.py` | Shared helpers |
| `native/fts5_cjk/` | Native trigram index so CJK queries actually match |

One database per profile. `messages.active` plus rewind primitives are what make
`/undo`, `/branch`, and the TUI's `/rewind` possible without destroying history.
(`/undo` and `/branch` are in the central `COMMAND_REGISTRY`; `/rewind` is handled
TUI-side through `tui_gateway/methods_prompt.py` and dispatched via
`command.dispatch` — a reminder that not every slash command lives in
`hermes_cli/commands.py`.)

---

## 5. Surfaces

### 5.1 CLI — `cli.py` (20,298 lines) + `hermes_cli/` (204 modules)

- **Rich** for banners and panels; **prompt_toolkit** for input with
  autocomplete.
- `load_cli_config()` merges hardcoded CLI defaults with the user's YAML.
- `process_command()` on `HermesCLI` dispatches on the canonical command name
  resolved through `resolve_command()`.
- The **skin engine** (`hermes_cli/skin_engine.py`) themes the CLI from data —
  banner colors, spinner faces, tool prefix, response box. Skins are pure YAML;
  adding one needs no code.

### 5.2 Slash-command registry — `hermes_cli/commands.py`

One central `COMMAND_REGISTRY` list of `CommandDef` objects. **Every** consumer
derives from it automatically:

| Consumer | Derived artifact |
|---|---|
| CLI | `process_command()` dispatch + alias resolution |
| Gateway | `GATEWAY_KNOWN_COMMANDS` frozenset |
| Gateway help | `gateway_help_lines()` |
| Telegram | `telegram_bot_commands()` → the BotCommand menu |
| Slack | `slack_subcommand_map()` → `/hermes` subcommand routing |
| Autocomplete | `COMMANDS` flat dict → `SlashCommandCompleter` |
| CLI help | `COMMANDS_BY_CATEGORY` → `show_help()` |

Consequence worth remembering: **adding an alias requires only editing the
`aliases` tuple.** Dispatch, help, the Telegram menu, Slack mapping, and
autocomplete all update themselves.

### 5.3 TUI — `ui-tui/` (Ink/React) + `tui_gateway/` (Python)

```
hermes --tui
  └─ Node (Ink)  ──stdio JSON-RPC──  Python (tui_gateway)
       │                                  └─ AIAgent + tools + sessions
       └─ renders transcript, composer, prompts, activity
```

TypeScript owns the screen. Python owns sessions, tools, model calls, and slash
logic. Transport is newline-delimited JSON-RPC; `tui_gateway/server.py` holds the
full method/event catalog.

| Surface | Ink component | Gateway method |
|---|---|---|
| Chat streaming | `app.tsx`, `messageLine.tsx` | `prompt.submit` → `message.delta/complete` |
| Tool activity | `thinking.tsx` | `tool.start/progress/complete` |
| Approvals | `prompts.tsx` | `approval.respond` ← `approval.request` |
| Clarify / sudo / secret | `prompts.tsx`, `maskedPrompt.tsx` | `clarify/sudo/secret.respond` |
| Session picker | `sessionPicker.tsx` | `session.list/resume` |
| Slash commands | local handler + fallthrough | `slash.exec` → `_SlashWorker`, `command.dispatch` |
| Completions | `useCompletion` | `complete.slash`, `complete.path` |
| Theming | `theme.ts`, `branding.tsx` | `gateway.ready` (carries skin data) |

Slash flow: built-in client commands (`/help`, `/quit`, `/clear`, `/resume`,
`/copy`, `/paste`) are handled locally in `app.tsx`; everything else goes to
`slash.exec` in a persistent `_SlashWorker` subprocess, falling back to
`command.dispatch`.

### 5.4 Dashboard — `web/` + `hermes_cli/web_server.py`

The dashboard **embeds the real `hermes --tui`** — it is not a rewrite. See
`hermes_cli/pty_bridge.py` and the `@app.websocket("/api/pty")` endpoint.

- `web/src/pages/ChatPage.tsx` mounts xterm.js (WebGL renderer, `addon-fit`,
  `addon-unicode11`).
- `/api/pty?token=…` upgrades to a WebSocket; auth uses the same ephemeral
  `_SESSION_TOKEN` as REST, passed as a query param (browsers can't set
  `Authorization` on a WS upgrade).
- Frames are raw PTY bytes each way. Resize is `\x1b[RESIZE:<cols>;<rows>]`,
  intercepted server-side and applied with `TIOCSWINSZ`.
- POSIX uses `ptyprocess`; native Windows goes through ConPTY
  (`win_pty_bridge`).

**Rule:** do not re-implement the primary chat experience in React. The
transcript, composer, and terminal belong to the embedded TUI. Structured React
*around* it — sidebars, inspectors, status panels, model pickers — is fine and
encouraged, as long as it isn't a second chat surface.

### 5.5 Desktop — `apps/desktop/` (1,434 files)

The one surface with its own transcript and composer. Electron + React +
nanostores + `@assistant-ui/react`, talking JSON-RPC over WebSocket
(`requestGateway(method, params)`).

- Transport lives in `apps/shared` (`@hermes/shared` — `JsonRpcGatewayClient`),
  which `web/` also consumes. **Desktop has no dependency on the dashboard
  frontend.**
- It spawns a headless `hermes serve` backend: the same server `dashboard` runs,
  minus the browser UI entirely (`headless_backend=True` skips `_build_web_ui`
  and exports `HERMES_SERVE_HEADLESS=1` so `mount_spa()` refuses even a stray
  `web_dist/`).
- `dashboard` and `serve` share `cmd_dashboard`/`start_server` but neither
  launches the other. One back-compat path exists: `backendSupportsServe()` in
  `apps/desktop/electron/main.ts` detects an older runtime that doesn't register `serve` and
  rewrites argv to the legacy `dashboard --no-open`, so a new app against an
  un-upgraded runtime doesn't brick.
- Slash commands are **curated client-side** in
  `apps/desktop/src/lib/desktop-slash-commands.ts`, then dispatched to the backend. Three
  gates: `isDesktopSlashCommand` (execution), `isDesktopSlashSuggestion`
  (discovery), `isDesktopSlashExtensionCommand` (skills and quick commands must
  flow through both discovery paths). The curation exists to hide
  terminal-only noise, **not** to hide user-activated extensions.

Read `apps/desktop/AGENTS.md` for scoped desktop rules before editing it.

### 5.6 Gateway — `gateway/`

The long-lived process that connects chat platforms.

| File | Role |
|---|---|
| `gateway/run.py` | The runner. Command interception, message routing. |
| `gateway/session.py` | Session lifecycle |
| `gateway/platforms/` | In-tree adapters: `api_server`, `webhook`, `signal`, `whatsapp_cloud`, `bluebubbles`, `weixin`, `yuanbao`, `qqbot`, … |
| `plugins/platforms/` | Plugin adapters: telegram, discord, slack, matrix, mattermost, teams, feishu, wecom, dingtalk, line, irc, sms, email, ntfy, google_chat, homeassistant, a2a, photon, raft, simplex, buzz, whatsapp |
| `gateway/authz_mixin.py` | Authorization. `_auth_env()` / `_platform_gate_env()` — read these before touching allowlists. |
| `gateway/delivery.py`, `delivery_ledger.py` | Durable delivery obligations for final responses |
| `gateway/platform_registry.py` | Adapter registration |
| `gateway/builtin_hooks/` | Always-registered hooks (none shipped) |

**Known trap — there are TWO message guards, and both must bypass control
commands.** When an agent is running: (1) the base adapter
(`gateway/platforms/base.py`) queues messages in `_pending_messages` while
`session_key in self._active_sessions`, and (2) the runner (`gateway/run.py`)
intercepts `/stop`, `/new`, `/queue`, `/status`, `/approve`, `/deny` before
`running_agent.interrupt()`. A new command that must reach the runner while the
agent is blocked has to bypass **both** and dispatch inline — not through
`_process_message_background()`, which races session lifecycle.

### 5.7 ACP — `acp_adapter/`

The Agent Client Protocol server for VS Code, Zed, and JetBrains. Its own
session model, edit-approval flow, permissions, and provenance metadata.

---

## 6. Extension edges

### 6.1 General plugins

`hermes_cli/plugins.py` + `plugins/<name>/`. `PluginManager` discovers from
`~/.hermes/plugins/`, `./.hermes/plugins/`, and pip entry points. Each plugin
exposes `register(ctx)` and can:

- register lifecycle hooks: `pre_tool_call`, `post_tool_call`, `pre_llm_call`,
  `post_llm_call`, `on_session_start`, `on_session_end`
- register tools via `ctx.register_tool(...)`
- register CLI subcommands via `ctx.register_cli_command(...)` — wired into
  `hermes` at startup, so `hermes <plugin> <subcmd>` works with no change to
  `main.py`

Hooks fire from `model_tools.py` (tool hooks) and `run_agent.py` (lifecycle).

**Discovery-timing pitfall:** `discover_plugins()` runs only as a side effect of
importing `model_tools.py`. Any code path that reads plugin state without
importing `model_tools.py` first must call `discover_plugins()` explicitly — it
is idempotent.

**Hard rule:** plugins must not modify core files (`run_agent.py`, `cli.py`,
`gateway/run.py`, `hermes_cli/main.py`). If a plugin needs something the
framework doesn't expose, widen the *generic* plugin surface — never special-case
a plugin in core. PR #5295 removed 95 lines of hardcoded honcho argparse from
`main.py` for exactly this reason.

### 6.2 Model-provider plugins

`plugins/model-providers/<name>/` — 36 of them. Each `__init__.py` calls
`providers.register_provider(ProviderProfile(...))` at module load.

`providers/__init__.py._discover_providers()` is a **separate, lazy** discovery
system — scanned on the first `get_provider_profile()` or `list_providers()`
call, *not* by `PluginManager`. Scan order:

1. Bundled: `<repo>/plugins/model-providers/<name>/`
2. User: `$HERMES_HOME/plugins/model-providers/<name>/`
3. Legacy: `<repo>/providers/<name>.py`

User plugins override bundled ones (`register_provider()` is last-writer-wins),
so a third party can swap out any built-in profile without patching the repo.

### 6.3 Memory-provider plugins

`plugins/memory/<name>/` — honcho, mem0, supermemory, byterover, hindsight,
holographic, openviking, retaindb. Each implements the `MemoryProvider` ABC
(`agent/memory_provider.py`), orchestrated by `agent/memory_manager.py`. Hooks:
`sync_turn(turn_messages)`, `prefetch(query)`, `shutdown()`, optional
`post_setup(hermes_home, config)`.

Discovery covers the same four sources as `PluginManager` but with
**bundled-first** precedence — the reverse of the general system — because a
provider is activated *by name*, so a dropped-in directory must not shadow a
shipped one. Discovery enumerates without importing; nothing runs until
`memory.provider` names it.

**Policy (May 2026): this set is closed.** New memory backends ship as
standalone plugin repos. PRs adding a directory under `plugins/memory/` are
closed with a pointer to publish separately. Existing providers stay and bug
fixes are welcome.

### 6.4 Skills

- `skills/` — bundled, loadable by default, organized by category (82 SKILL.md
  files).
- `optional-skills/` — heavier or niche, shipped but inactive; installed with
  `hermes skills install official/<category>/<skill>` (117 files). Adapter:
  `tools/skills_hub.py` (`OptionalSkillSource`).

Skill slash-commands are scanned by `agent/skill_commands.py` and injected as
**user messages** to protect the cache.

Authoring standards are hardline — see [05 · Recipes](05-recipes.md#recipe-3--add-a-skill).

---

## 7. Scheduling and queues

### Cron — `cron/`

`cron/jobs.py` (store) + `cron/scheduler.py` (tick loop). Agents schedule via
the `cronjob` tool; users via `hermes cron <verb>` or `/cron`.

Schedule formats: duration (`"30m"`), "every" phrase (`"every monday 9am"`),
5-field cron (`"0 9 * * *"`), ISO timestamp (one-shot).

Per-job fields include `skills`, `model`/`provider` overrides, `script`
(pre-run data collection whose stdout is injected into the prompt; `no_agent=True`
makes the script the whole job), `context_from` (chain job A's output into job
B), `workdir`, and multi-platform delivery.

Hardening invariants:

- **Inactivity hard-interrupt** on cron sessions — a runaway loop cannot
  monopolize the scheduler. Default **600s** of inactivity
  (`_DEFAULT_CRON_INACTIVITY_TIMEOUT` in `cron/jobs.py`), overridable with
  `HERMES_CRON_TIMEOUT` (`0` = unlimited). *(`AGENTS.md` still calls this a
  "3-minute hard interrupt" — that value is stale; the code says 600s.)*
- Catchup window: half the job's period, clamped 120s–2h.
- Grace window: 120s for a missed one-shot.
- File lock at `~/.hermes/cron/.tick.lock` prevents duplicate ticks across
  processes.
- Cron sessions pass `skip_memory=True`; memory providers intentionally do not
  run during cron.
- Cron deliveries are **not** mirrored into the target gateway session — they
  land in their own cron session with header/footer framing, so the main
  conversation's message-role alternation stays intact.

### Kanban — `cron`-adjacent but separate

A durable SQLite board for multi-profile collaboration. `hermes_cli/kanban.py`
(CLI), `tools/kanban_tools.py` (worker toolset), `plugins/kanban/` (dashboard +
systemd unit).

The dispatcher (default every 60s, running **inside the gateway** via
`kanban.dispatch_in_gateway: true`) reclaims stale claims, promotes ready tasks,
atomically claims, and spawns the assigned profile.

Isolation model:

- **Board is the hard boundary** — workers get `HERMES_KANBAN_BOARD` pinned in
  their env, so they cannot see other boards.
- **Tenant is a soft namespace within a board** — one specialist fleet can serve
  multiple businesses via workspace-path + memory-key isolation.
- After `kanban.failure_limit` consecutive non-success attempts (default 2), the
  dispatcher auto-blocks the task to stop spin loops.

### Delegation — `tools/delegate_tool.py`

Spawns a subagent with isolated context and terminal. Parent waits for the
child's summary by default; `background=true` returns a delegation id
immediately and the result re-enters later through the async-delegation queue.

Two shapes: single (`goal`) or batch (`tasks: [...]`, concurrent).

Two roles: `leaf` (default; cannot call `delegate_task`, `clarify`, `memory`,
`send_message`, `cronjob`, but keeps `execute_code`) and `orchestrator` (keeps
`delegate_task`, bounded by `delegation.max_spawn_depth`).

**Durability rule:** background delegation is detached from the turn but still
*process-local*. Work that must survive a restart belongs in `cronjob` or
`terminal(background=True, notify_on_complete=True)`.

---

## 8. Profiles

Multiple fully isolated instances, each with its own `HERMES_HOME`.

The mechanism: `_apply_profile_override()` in `hermes_cli/main.py` sets
`HERMES_HOME` **before any module imports**. Every `get_hermes_home()` call then
scopes automatically.

Rules for profile-safe code:

1. **Always `get_hermes_home()`** for state paths. Never `Path.home() / ".hermes"`.
2. **Always `display_hermes_home()`** in user-facing messages — it renders
   `~/.hermes` or `~/.hermes/profiles/<name>` correctly.
3. Module-level constants are fine; they cache `get_hermes_home()` at import
   time, which is *after* the profile override.
4. Tests that mock `Path.home()` must **also** set `HERMES_HOME`.
5. Gateway adapters connecting with a unique credential should take a token lock
   (`acquire_scoped_lock()` / `release_scoped_lock()` from `gateway.status`) so
   two profiles can't use the same bot token. Canonical pattern:
   `plugins/platforms/irc/adapter.py`.
6. Profile *operations* are HOME-anchored, not HERMES_HOME-anchored —
   `_get_profiles_root()` returns `Path.home() / ".hermes" / "profiles"` on
   purpose, so `hermes -p coder profile list` sees every profile.
7. **Multiplex profile-scoped env reads must fail closed.** Under
   `gateway.multiplex_profiles`, `os.environ` holds the *default* profile's
   values; a secondary profile's `.env` lives only in its secret scope. A scoped
   miss returns the default — it must **never** fall through to `os.environ`,
   which leaks another profile's value and silently breaks routing and
   admission. Use `_get_scoped_secret()` in adapters and `_auth_env()` /
   `_platform_gate_env()` in gateway authz. This wrapper is copy-pasted across
   ~15 adapters; do not reintroduce the
   `except _UnscopedSecretError: val = os.getenv(...)` shape.

Hardcoded `~/.hermes` paths were the source of five bugs fixed in one PR
(#3575). Take rule 1 seriously.

---

## 9. Where do I look for X?

| I want to change… | Look at |
|---|---|
| How a turn is driven | `agent/conversation_loop.py` |
| What the model sees in its prompt | `agent/prompt_builder.py`, `agent/system_prompt.py` |
| Context running out | `agent/context_compressor.py`, `agent/conversation_compression.py` |
| Provider-specific API shape | `agent/transports/`, `agent/*_adapter.py` |
| Retry / failover behavior | `agent/error_classifier.py` |
| A tool's behavior | `tools/<name>.py` |
| Whether a tool is visible | `toolsets.py` (+ its `check_fn`) |
| Where commands run | `tools/environments/<backend>.py` |
| Approval / dangerous commands | `tools/approval.py` |
| A slash command | `hermes_cli/commands.py` → `cli.py::process_command` → `gateway/run.py` |
| CLI look and feel | `hermes_cli/skin_engine.py`, `agent/display.py` |
| TUI rendering | `ui-tui/src/` |
| TUI ↔ Python protocol | `tui_gateway/server.py` |
| Desktop app | `apps/desktop/` (read its `AGENTS.md` first) |
| Dashboard pages | `web/src/pages/` |
| A chat platform | `plugins/platforms/<name>/` or `gateway/platforms/<name>.py` |
| Platform authorization | `gateway/authz_mixin.py` |
| Sessions and search | `hermes_state.py`, `hermes_state_search.py` |
| Default config values | `hermes_cli/config_defaults.py` (`DEFAULT_CONFIG`) |
| Secret metadata / setup prompts | `hermes_cli/config_defaults.py` (`OPTIONAL_ENV_VARS`) |
| Model catalog | `plugins/model-providers/<name>/`, `scripts/build_model_catalog.py` |
| Pricing and token accounting | `agent/usage_pricing.py` |
| Memory backends | `plugins/memory/<name>/`, `agent/memory_manager.py` |
| Skill lifecycle / archiving | `agent/curator.py`, `tools/skill_usage.py` |
| Scheduled jobs | `cron/jobs.py`, `cron/scheduler.py` |
| The multi-agent board | `hermes_cli/kanban.py`, `tools/kanban_tools.py` |
| Subagents | `tools/delegate_tool.py`, `tools/async_delegation.py` |
| Logging | `hermes_logging.py` |
| Paths and profiles | `hermes_constants.py` |

---

## 10. The size problem (context for reviewers)

Four files carry disproportionate weight:

| File | Lines |
|---|---|
| `cli.py` | 20,298 |
| `hermes_state.py` | 13,086 |
| `run_agent.py` | 9,051 (after the loop was extracted) |
| `hermes_state_search.py` | 2,493 |

`agent/conversation_loop.py` shows the sanctioned way out: move the body to a
sibling module, take the parent instance as the first argument, and keep a thin
forwarder so existing monkeypatches still resolve. If you are about to add 400
lines to `cli.py`, consider extracting instead — and expect reviewers to ask.
