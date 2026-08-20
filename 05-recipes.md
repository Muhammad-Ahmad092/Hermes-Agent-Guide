# 05 · Recipes

**For:** you have decided the rung; now you need the code.
**Prerequisites:** [04 · Extending Hermes](04-extending-hermes.md) — decide the
rung first, or you will write the wrong thing well.
**Next:** [06 · Testing](06-testing.md)

Seven worked recipes, ordered from cheapest footprint to most expensive. Every
pattern is taken from real code in the tree — the file references are real, go
read them.

| Recipe | Rung | Files touched |
|---|---|---|
| [1 · Add a slash command](#recipe-1--add-a-slash-command) | 1 | 2–3 |
| [2 · Add a config option](#recipe-2--add-a-config-option) | 1 | 1–2 |
| [3 · Add a skill](#recipe-3--add-a-skill) | 2 | 1 dir + 1 test |
| [4 · Add a core tool](#recipe-4--add-a-core-tool) | 3 or 6 | 2 + tests |
| [5 · Write a plugin](#recipe-5--write-a-plugin) | 4 | 0 core files |
| [6 · Add a messaging platform](#recipe-6--add-a-messaging-platform) | 4 | 0 core files |
| [7 · Add a model provider](#recipe-7--add-a-model-provider) | 4 | 1 dir |

---

## Recipe 1 — Add a slash command

**Rung 1.** Everything derives from one central registry, so this is far cheaper
than it looks.

### Step 1 — Register it

`hermes_cli/commands.py`, in the `COMMAND_REGISTRY` list:

```python
CommandDef("mycommand", "Description of what it does", "Session",
           aliases=("mc",), args_hint="[arg]"),
```

`CommandDef` fields:

| Field | Meaning |
|---|---|
| `name` | Canonical name, no slash (`"background"`) |
| `description` | Human-readable, shown in `/help` |
| `category` | One of `"Session"`, `"Configuration"`, `"Tools & Skills"`, `"Info"`, `"Exit"` |
| `aliases` | Tuple of alternates (`("bg",)`) |
| `args_hint` | Placeholder shown in help (`"<prompt>"`, `"[name]"`) |
| `cli_only` | Only in the interactive CLI |
| `gateway_only` | Only on messaging platforms |
| `gateway_config_gate` | A config dotpath; when set on a `cli_only` command, it becomes available in the gateway if that value is truthy |

### Step 2 — Handle it in the CLI

`cli.py`, in `HermesCLI.process_command()`:

```python
elif canonical == "mycommand":
    self._handle_mycommand(cmd_original)
```

### Step 3 — Handle it in the gateway (only if it applies there)

`gateway/run.py`:

```python
if canonical == "mycommand":
    return await self._handle_mycommand(event)
```

### That's it

Registering once gives you, automatically: CLI dispatch and alias resolution,
`GATEWAY_KNOWN_COMMANDS` membership, `/help` output, the Telegram BotCommand
menu, Slack `/hermes` subcommand routing, and autocomplete.

**Adding an alias to an existing command needs only the `aliases` tuple.** No
other file changes.

For persistent settings, use `save_config_value()` in `cli.py`.

### Gotchas

- **Destructive commands need a confirm prompt** — there is precedent
  (`/clear`, `/exit --delete`).
- **If the command must reach the gateway runner while an agent is running**,
  it has to bypass *both* message guards and dispatch inline. See
  [02 § 5.6](02-architecture.md#56-gateway--gateway).
- **All CLI menu-pickers must use curses** (`hermes_cli/curses_ui.py`). See
  `hermes_cli/tools_config.py` for the pattern.

---

## Recipe 2 — Add a config option

**Rung 1.** Remember the law: **behavior goes in `config.yaml`; `.env` is
secrets only.**

### For a `config.yaml` key

1. Add it to `DEFAULT_CONFIG` in `hermes_cli/config_defaults.py`. (`AGENTS.md`
   says `config.py`; that file only re-exports it — line 945.)
2. **Do not bump `_config_version`** for a new key — deep-merge handles that
   automatically. Bump it *only* when you need to actively migrate existing user
   config (renaming keys, changing structure).
3. Update `cli-config.yaml.example` — the PR template has a checkbox for this.

Then read it through the loader that your code path actually uses:

| Loader | Used by |
|---|---|
| `load_cli_config()` | CLI mode (`cli.py`) |
| `load_config()` | `hermes tools`, `hermes setup`, most subcommands |
| Direct YAML | Gateway runtime (`gateway/run.py`, `gateway/config.py`) |

If the CLI sees your key but the gateway doesn't, you are on the wrong loader —
check `DEFAULT_CONFIG` coverage.

### For a secret (and only a secret)

Add it to `OPTIONAL_ENV_VARS` in `hermes_cli/config_defaults.py`:

```python
"NEW_API_KEY": {
    "description": "What it's for",
    "prompt": "Display name",
    "url": "https://...",
    "password": True,
    "category": "tool",   # provider | tool | messaging | setting
},
```

That metadata is what makes the setup wizard able to prompt for it properly.

### Gotchas

- Non-secret settings in `.env` get the PR rejected.
- If internal code needs an env mirror for back-compat, bridge it *from*
  `config.yaml` in code (see `gateway_timeout`, `terminal.cwd → TERMINAL_CWD`)
  and document the config key, not the env var.
- Don't write a test asserting `DEFAULT_CONFIG["_config_version"] == 21`. That's
  a change-detector — see [06 § 4](06-testing.md#4-dont-write-change-detector-tests).

---

## Recipe 3 — Add a skill

**Rung 2.** The cheapest way to add capability: zero schema cost, and the
standards are enforced hard at review.

### Where it goes

| Directory | For |
|---|---|
| `skills/<category>/<name>/` | Broadly useful, loadable by default |
| `optional-skills/<category>/<name>/` | Heavy dependencies or niche; installed explicitly |

Reviewers check which directory you targeted. Heavy-dep or niche → optional.

### Structure

```
skills/<category>/<name>/
├── SKILL.md            # the playbook
├── scripts/            # helper scripts — ship real logic here
├── references/         # reference material the model can read on demand
└── templates/          # file templates
```

### SKILL.md

A real example from the tree (`skills/github/codebase-inspection/SKILL.md`):

```markdown
---
name: codebase-inspection
description: "Inspect codebases w/ pygount: LOC, languages, ratios."
version: 1.0.0
author: Your Name (@yourhandle), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [LOC, Code Analysis, pygount, Codebase, Metrics]
    related_skills: [github-repo-management]
prerequisites:
  commands: [pygount]
---

# Codebase Inspection with pygount

Analyze repositories for lines of code, language breakdown, file counts, and
code-vs-comment ratios using `pygount`.

## When to Use
## Prerequisites
## How to Run
## Quick Reference
## Procedure
## Pitfalls
## Verification
```

### The hardline standards

Reviewers reject PRs that violate these. All eight are non-negotiable.

**1. `description` ≤ 60 characters, one sentence, ends with a period.** Long
descriptions bloat listings and dilute model attention when many skills load.
State the capability, not the implementation. No marketing words ("powerful",
"comprehensive", "seamless", "advanced"). Don't repeat the skill name. Verify:

```python
import re, pathlib
m = re.search(r'^description: (.*)$',
              pathlib.Path('skills/<cat>/<name>/SKILL.md').read_text(),
              re.MULTILINE)
assert len(m.group(1)) <= 60, len(m.group(1))
```

**2. Reference native Hermes tools, not shell utilities the agent already has
wrapped.** In prose, name the real tool in backticks:

| Don't write | Write |
|---|---|
| `grep` | `search_files` |
| `cat` / `head` / `tail` | `read_file` |
| `sed` / `awk` | `patch` |
| `find` / `ls` | `search_files target='files'` |

If the skill depends on an MCP server, name it and document setup under
`## Prerequisites`. Third-party CLIs and shell pipelines are fine *inside script
files* — they just shouldn't be the headline interaction surface in the prose.

**3. `platforms:` gating audited against actual script imports.** If your
scripts use POSIX-only primitives (`fcntl`, `termios`, `os.setsid`, `/proc`,
hardcoded `/tmp`, `signal.SIGKILL`, bash heredocs, `osascript`, `apt`,
`systemctl`), either fix it cross-platform first — `tempfile.gettempdir()`,
`pathlib.Path`, `psutil.pid_exists()`, Python-level filtering — or declare the
narrower platform list.

**4. `author` credits the human first.** Your real name + GitHub handle first,
"Hermes Agent" second. If you drafted the skill *using* Hermes and the commit
shows "Hermes Agent" as author, replace it with your name. Credit the human, not
the tool.

**5. Modern section order**, as shown above. Target ~200 lines for a complex
skill, ~100 for a simple one. Cut intro fluff and re-explanations of env vars
already covered in `## Prerequisites`.

**6. Ship helper scripts.** Don't expect the model to inline-write parsers, XML
walkers, or non-trivial logic on every call. Put it in `scripts/` and reference
it by path relative to the skill directory.

**7. Tests at `tests/skills/test_<skill>_skill.py`** using only stdlib + pytest +
`unittest.mock`. No live network.

```bash
scripts/run_tests.sh tests/skills/test_<skill>_skill.py -q
```

**8. `.env.example` additions go in a clearly delimited block.** Don't touch the
surrounding file.

### Test it end to end

```bash
hermes --toolsets skills -q "Use the X skill to do Y"
```

### Gotchas

- There is a linter: `tools/skill_linter.py`. Run against it before opening a PR.
- Skill slash-commands are injected as **user messages** to protect the cache —
  don't try to make yours modify the system prompt.
- `AGENTS.md` points at a salvage/modernization checklist in a `hermes-agent-dev`
  skill at `references/new-skill-pr-salvage.md`. **Neither exists in the tree** —
  the real bundled skill is `skills/autonomous-ai-agents/hermes-agent/`, whose
  `references/` holds 18 useful pages (`contributor-guide.md`,
  `slash-commands.md`, `configuration.md`, `desktop-plugins.md`, …).

---

## Recipe 4 — Add a core tool

**Rung 3 (service-gated) or 6 (core).** Read
[04 § 1](04-extending-hermes.md#1-the-footprint-ladder) first — most capabilities
should not be here.

> For custom or local-only tools, **do not edit Hermes core.** Use
> [Recipe 5](#recipe-5--write-a-plugin).

### Step 1 — `tools/your_tool.py`

```python
import json, os
from tools.registry import registry


def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))


def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})


registry.register(
    name="example_tool",
    toolset="example",
    schema={
        "name": "example_tool",
        "description": "...",
        "parameters": {...},
    },
    handler=lambda args, **kw: example_tool(
        param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

### Step 2 — `toolsets.py` (REQUIRED)

Add the tool's name to either `_HERMES_CORE_TOOLS` (all platforms) or a new
toolset entry in the `TOOLSETS` dict.

**This step is not optional.** Auto-discovery imports your file and registers the
schema, but the tool is only *exposed to an agent* if a toolset names it. Missing
this is the #1 "my tool doesn't work" bug.

### Rules that will come up in review

**Every handler must return a JSON string.** The registry handles schema
collection, dispatch, availability checking, and error wrapping.

**Path references in schemas must be profile-aware.** Use
`display_hermes_home()`. The schema is generated at import time, which is *after*
`_apply_profile_override()` sets `HERMES_HOME`.

**State files use `get_hermes_home()`** — never `Path.home() / ".hermes"`. Each
profile gets its own state.

**No cross-tool references hardcoded in schema descriptions.** A description
saying "prefer `web_search`" breaks when `web_search` is unavailable (missing key,
disabled toolset) — the model hallucinates calls to tools that don't exist. If
you need a cross-reference, add it dynamically in `get_tool_definitions()` in
`model_tools.py`; see the `browser_navigate` / `execute_code` post-processing
blocks.

**No `offset`/`limit` on instructional tools.** Anything the agent must read
fully — skills, prompts, playbooks — gets no pagination. Models read page 1 and
stop.

**`check_fn` results are TTL-cached process-wide.** One process serves many
sessions, so a per-session answer must not live in `check_fn` at all.

**Agent-level tools** (`todo`, `memory`) are intercepted by `run_agent.py`
*before* `handle_function_call()`. See `tools/todo_tool.py` if you need that
pattern.

### If it's surface-dependent

Keep it out of `_HERMES_CORE_TOOLS`, put it in a named toolset (`desktop_ui`,
`project`), and let the GUI gateway's `_load_enabled_toolsets(platform)` fold it
in based on the **session's** platform. Then write the test that asserts a GUI
session gets the tool **with `HERMES_DESKTOP` absent**. See
[04 § 5](04-extending-hermes.md#5-the-session-vs-process-trap).

---

## Recipe 5 — Write a plugin

**Rung 4. Zero core files touched** — this is the right answer far more often
than people expect.

### Structure

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml
└── __init__.py        # must define register(ctx)
```

For a bundled plugin the directory is `plugins/my-plugin/` — but read the closed
lists in [04 § 4](04-extending-hermes.md#4-closed-lists--read-before-you-write)
first. Third-party product integrations and new memory backends do **not** land
in this tree.

### `plugin.yaml`

From the real `plugins/disk-cleanup/plugin.yaml`:

```yaml
name: disk-cleanup
version: 2.0.0
description: "Auto-track and clean up ephemeral files created during Hermes sessions."
author: "@handle (original), NousResearch (plugin port)"
hooks:
  - post_tool_call
  - on_session_end
```

Manifest v2 also supports a schema version, `api_version`, inter-plugin
dependencies, a pip-dependency declaration seam, a config schema, and capability
declarations with an install/update consent flow.

### `__init__.py`

The real `register()` from `plugins/disk-cleanup/__init__.py`:

```python
def register(ctx) -> None:
    ctx.register_hook("post_tool_call", _on_post_tool_call)
    ctx.register_hook("on_session_end", _on_session_end)
    ctx.register_command(
        "disk-cleanup",
        handler=_handle_slash,
        description="Track and clean up ephemeral Hermes session files.",
    )
```

### What `ctx` gives you

The Python `PluginContext` (`hermes_cli/plugins.py`) exposes 23 `register_*`
methods as of this writing. Enumerate the current set yourself — it grows:

```bash
grep -oE "def register_[a-z_]+" hermes_cli/plugins.py | sort -u
```

The ones you are most likely to want:

| Method | Purpose |
|---|---|
| `ctx.register_tool(...)` | Add a tool. Its toolset is discovered automatically and can be enabled/disabled without touching `tools/` or `toolsets.py`. |
| `ctx.register_hook(name, fn)` | A lifecycle hook — see the list below |
| `ctx.register_command(...)` | A slash command |
| `ctx.register_cli_command(...)` | An argparse subtree — `hermes <plugin> <subcmd>` works with no change to `main.py` |
| `ctx.register_platform(...)` | A messaging adapter (Recipe 6) |
| `ctx.register_memory_provider(...)` | A memory backend against the `MemoryProvider` ABC |
| `ctx.register_skill(...)` | A skill from your plugin |
| `ctx.register_system_prompt_section(...)` | Contribute to the system prompt |
| `ctx.register_context_engine(...)` / `register_context_reference(...)` | Context assembly |
| `ctx.register_secret_source(...)` | A credential backend (Bitwarden-style) |
| `ctx.register_redaction_patterns(...)` | Extra secret-redaction patterns |
| `ctx.register_approval_transport(...)` | Route approvals somewhere new |
| `ctx.register_auxiliary_task(...)` | A new side-LLM task type |
| `ctx.register_tts_provider(...)` / `register_transcription_provider(...)` | Voice |
| `ctx.register_image_gen_provider(...)` / `register_video_gen_provider(...)` | Generation |
| `ctx.register_web_search_provider(...)` / `register_browser_provider(...)` | Web |
| `ctx.register_dashboard_auth_provider(...)` | Dashboard auth |
| `ctx.register_middleware(...)` | Request middleware |
| `ctx.register_slack_action_handler(...)` | Slack interactive actions |

Plus non-registration helpers:

| Method | Purpose |
|---|---|
| `ctx.dispatch_tool(...)` | Call another tool |
| `ctx.llm(...)` | Run any LLM call from inside the plugin |
| `ctx.call_mcp(...)` | Capability-gated MCP call |
| `ctx.platform_actions` | Capability-gated platform facade |
| `ctx.profile_name` | Session-agnostic profile access |

### Hook names

```bash
grep -oE '"(pre|post|on)_[a-z_]+"' hermes_cli/plugins.py | sort -u
```

Currently: `pre_tool_call` · `post_tool_call` · `pre_llm_call` · `post_llm_call` ·
`pre_api_request` · `post_api_request` · `pre_command` · `pre_gateway_dispatch` ·
`pre_approval_request` · `post_approval_response` · `pre_transcription` ·
`pre_verify` · `on_session_start` · `on_session_end` · `on_session_reset` ·
`on_session_finalize` · `on_stream_start` · `on_stream_delta` · `on_stream_end` ·
`on_interim_message` · `on_skill_lifecycle` · `on_unload` · plus the kanban set
(`on_kanban_dispatch_tick`, `on_kanban_task_updated`, `on_kanban_worker_spawned`,
`on_kanban_worker_exited`, `on_kanban_worker_stale_claim`).

> **Two different plugin surfaces — don't confuse them.** The table above is the
> **Python** `PluginContext`. The **desktop app** has its own TypeScript plugin
> SDK with a different `ctx`, including `ctx.os` (native notifications,
> `openExternal`, `revealPath`, `writeClipboard` — all resolve `false` rather
> than throwing when unavailable) and `ctx.i18n.register({ en, ja, … })` for
> plugin-scoped locale bundles. Those do **not** exist on the Python context.
> Reference: `skills/autonomous-ai-agents/hermes-agent/references/desktop-plugins.md`.

### Rules

**Never modify core files** — `run_agent.py`, `cli.py`, `gateway/run.py`,
`hermes_cli/main.py`. If you need something the framework doesn't expose, widen
the *generic* plugin surface so any plugin could use it. PR #5295 deleted 95 lines
of hardcoded honcho argparse from `main.py` for exactly this reason.

**Keep documented surfaces additive.** Hook payload data arrives as keyword
fields, and callbacks are signature-inspected — an old narrow signature receives
only the fields it declares, while a `**kwargs` callback gets the full payload.
Don't remove or rename `PluginContext` methods; make new parameters optional and
keyword-only; ignore unknown manifest fields.

**Discovery-timing pitfall.** `discover_plugins()` runs only as a side effect of
importing `model_tools.py`. Code that reads plugin state without importing it
first must call `discover_plugins()` explicitly — it's idempotent.

**Debugging:** `HERMES_PLUGINS_DEBUG=1` surfaces discovery logs.

Reference plugins live in the companion repo
[`hermes-example-plugins`](https://github.com/NousResearch/hermes-example-plugins),
not in this tree. The canonical compatibility contract is
`website/docs/developer-guide/plugins/index.md#native-plugin-compatibility-contract`.

---

## Recipe 6 — Add a messaging platform

**Rung 4.** The plugin path requires **zero changes to core Hermes code**. Read
`gateway/platforms/ADDING_A_PLATFORM.md` for the full contract.

### Structure

```
plugins/platforms/<name>/          # or ~/.hermes/plugins/<name>/
├── plugin.yaml
└── adapter.py                     # inherits BasePlatformAdapter
```

`register(ctx)` calls `ctx.register_platform(...)`.

### What you get for free

Adapter creation, config parsing, user authorization, cron delivery,
`send_message` routing, system-prompt hints, status display, and gateway setup.

### `plugin.yaml`

From the real `plugins/platforms/ntfy/plugin.yaml`:

```yaml
name: ntfy-platform
label: ntfy
kind: platform
version: 1.0.0
description: >
  ntfy push-notification gateway adapter. Subscribes to a topic via HTTP
  streaming and publishes replies via HTTP POST.
author: sprmn24
requires_env:
  - name: NTFY_TOPIC
    description: "Topic name to subscribe to (e.g. hermes-in)"
    prompt: "ntfy subscribe topic"
    password: false
optional_env:
  - name: NTFY_TOKEN
    description: "Bearer token or 'user:pass' for Basic auth (optional)"
    prompt: "ntfy auth token (or empty)"
    password: true
```

Those rich-dict `requires_env` / `optional_env` entries auto-populate
`OPTIONAL_ENV_VARS` in `hermes_cli/config.py`, so the setup wizard surfaces
proper descriptions, prompts, password flags, and URLs.

### The optional hooks that cover real edges

| Hook | Why you need it |
|---|---|
| `env_enablement_fn: () -> Optional[dict]` | Seeds `PlatformConfig.extra` (and optional `home_channel`) from env vars **before** the adapter is constructed. Without it, env-only setups don't appear in `hermes gateway status` until the SDK instantiates. |
| `apply_yaml_config_fn: (yaml_cfg, platform_cfg) -> Optional[dict]` | Lets the plugin own its `config.yaml` schema instead of growing core `gateway/config.py`. Mutating `os.environ` is allowed — guard with `not os.getenv(...)` to preserve env > YAML precedence. |
| `cron_deliver_env_var: str` | Names your `*_HOME_CHANNEL` env var so `deliver=<name>` cron jobs route without editing `cron/scheduler.py`'s hardcoded sets. |
| `standalone_sender_fn` | Out-of-process delivery for cron jobs running separately from the gateway. Without it, a `deliver=<name>` job fires but the send returns `No live adapter for platform '<name>'`. Pair with `cron_deliver_env_var`. |

### Patterns worth copying

**Time-window constraints.** When a platform has a hard reply deadline (LINE's
60s single-use reply token, WhatsApp's 24h session window), override
`_keep_typing` to layer a mid-flight bubble at a threshold. Always
`await super()._keep_typing(...)` so the heartbeat keeps running, and tear down
your side task in `finally`. Full pattern: `plugins/platforms/line/`.

**Two transport modes for one platform.** Two adapters sharing a behavior mixin.
WhatsApp does this: the Baileys-bridge adapter (`plugins/platforms/whatsapp/adapter.py`)
and the Meta Cloud API adapter (`gateway/platforms/whatsapp_cloud.py`) both inherit
`WhatsAppBehaviorMixin` from `gateway/platforms/whatsapp_common.py`.

> **Note:** `gateway/platforms/ADDING_A_PLATFORM.md` still describes the bridge
> adapter as `gateway/platforms/whatsapp.py`. That file no longer exists — the
> adapter moved into `plugins/platforms/whatsapp/`. The same doc also points at
> `gateway/platforms/telegram.py` and `discord.py`, which likewise now live under
> `plugins/platforms/`. Read the doc for the *contract*, and the filesystem for
> the *locations*.

**Profile safety.** If your adapter connects with a unique credential, take a
token lock — `acquire_scoped_lock()` in `connect()`/`start()`,
`release_scoped_lock()` in `disconnect()`/`stop()` — so two profiles can't use
the same bot token. Canonical: `plugins/platforms/irc/adapter.py`.

**Profile-scoped secrets must fail closed.** Under `gateway.multiplex_profiles`,
read credentials *and* authorization config (`{PLATFORM}_ALLOWED_USERS`,
`ALLOW_ALL_USERS`, `group_policy`, `allow_bots`) through `_get_scoped_secret()`.
A scoped miss returns the **default** — never fall through to `os.environ`, which
leaks another profile's value and silently breaks admission. Canonical
fail-closed copy: `plugins/platforms/feishu/adapter.py`.

---

## Recipe 7 — Add a model provider

**Rung 4.** One directory. Providers use their own lazy discovery system, not
`PluginManager`.

### Structure

```
plugins/model-providers/<name>/
├── plugin.yaml
└── __init__.py
```

### `__init__.py`

The real `plugins/model-providers/gmi/__init__.py`:

```python
"""GMI Cloud provider profile."""

from hermes_cli import __version__ as _HERMES_VERSION
from providers import register_provider
from providers.base import ProviderProfile

gmi = ProviderProfile(
    name="gmi",
    aliases=("gmi-cloud", "gmicloud"),
    display_name="GMI Cloud",
    description="GMI Cloud — multi-model direct API (slash-form model IDs)",
    signup_url="https://www.gmicloud.ai/",
    env_vars=("GMI_API_KEY", "GMI_BASE_URL"),
    base_url="https://api.gmi-serving.com/v1",
    auth_type="api_key",
    default_headers={"User-Agent": f"HermesAgent/{_HERMES_VERSION}"},
    default_aux_model="google/gemini-3.1-flash-lite-preview",
    fallback_models=(
        "zai-org/GLM-5.1-FP8",
        "deepseek-ai/DeepSeek-V3.2",
        "anthropic/claude-sonnet-5",
    ),
)

register_provider(gmi)
```

### How discovery works

`providers/__init__.py._discover_providers()` is **lazy and separate** — scanned
on the first `get_provider_profile()` or `list_providers()` call, *not* by
`PluginManager`. Scan order:

1. Bundled: `<repo>/plugins/model-providers/<name>/`
2. User: `$HERMES_HOME/plugins/model-providers/<name>/`
3. Legacy: `<repo>/providers/<name>.py` (back-compat)

`register_provider()` is **last-writer-wins**, so a user plugin of the same name
overrides a bundled one. That is how third parties swap out a built-in profile
without patching the repo.

`PluginManager` records `kind: model-provider` manifests but does **not** import
them — importing would double-instantiate `ProviderProfile`. A plugin without an
explicit `kind:` gets auto-coerced by a source-text heuristic
(`register_provider` + `ProviderProfile` in `__init__.py`).

### Gotchas

- **Don't write a change-detector test** asserting your model names are in the
  catalog — model lists change constantly. Assert the *relationship* instead
  ("every model in the catalog has a context-length entry"). See
  [06 § 4](06-testing.md#4-dont-write-change-detector-tests).
- Attribution headers via `default_headers` are picked up by the generic
  `profile.default_headers` fallback in `run_agent.py` and
  `agent/auxiliary_client.py` at client construction time.
- Full authoring guide:
  `website/docs/developer-guide/model-provider-plugin.md`.

---

## Cross-cutting reminders

Whatever recipe you followed:

- **Profile-safe paths.** `get_hermes_home()` for code, `display_hermes_home()`
  for messages. Never `~/.hermes` hardcoded.
- **No ANSI erase-to-EOL** (`\033[K`) in spinner/display code — it leaks as
  literal `?[K` under `prompt_toolkit`'s `patch_stdout`. Space-pad instead:
  `f"\r{line}{' ' * pad}"`.
- **Dependencies need upper bounds.** See
  [03 § 8](03-dev-setup.md#8-dependency-policy-you-will-hit-this).
- **Cross-platform.** Linux, macOS, and native Windows all ship.
- **Write tests, and write the right kind.** [06 · Testing](06-testing.md).
