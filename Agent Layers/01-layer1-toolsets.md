# Layer 1 — Toolsets: What the Model Is Allowed to Call

> **Source of truth:** `toolsets.py` (1,083 lines, repo root).
> **User-facing menu:** `hermes_cli/tools_config.py:96`.
> **Registration:** `tools/registry.py`.

A **tool** is one callable function the model sees in its schema. A **toolset** is
a named bundle of tools. Nothing else in Hermes decides what the model can reach
this turn — if a tool is not in an enabled toolset, the model does not know it
exists.

At baseline `cf64ca20c`: **59 toolsets**, **135 distinct tool names**, backed by
**121 modules** under `tools/`.

---

## 1. The three families of toolset

The 59 keys look like a flat dict but they serve three completely different
purposes. Confusing them is the single most common mistake when configuring
Hermes.

| Family | Count | Purpose | Do you enable these directly? |
|---|---|---|---|
| **Primitives** | 32 | One capability each (`web`, `file`, `memory`) | **Yes** — this is what `hermes tools` toggles |
| **Postures** | 3 | A curated selection for a mode of work (`coding`, `debugging`, `safe`) | No — selected automatically per session |
| **Surface bundles** | 24 | The full set a given surface ships with (`hermes-cli`, `hermes-slack`) | No — chosen by the surface you launch |

```python
# toolsets.py — a toolset is a three-key dict
"web": {
    "description": "Web research and content extraction tools",
    "tools": ["web_search", "web_extract"],
    "includes": []              # other toolsets to fold in
},
```

`includes` gives you composition. `debugging` is the clearest example:

```python
"debugging": {
    "description": "Debugging and troubleshooting toolkit",
    "tools": ["terminal", "process"],
    "includes": ["web", "file"]   # resolves to 8 tools total
},
```

Resolution is recursive and de-duplicating, handled by `get_toolset()` and
`resolve_toolset()` in the same file.

---

## 2. The primitives — the 32 you actually configure

**26** of the 32 appear in the `hermes tools` configurator
(`hermes_cli/tools_config.py:96`). The remaining **6** are enabled by machinery
rather than by you: `search`, `project`, `desktop_ui`, `kanban`, `feishu_doc`,
`feishu_drive`.

The configurator actually shows 27 rows — the 27th is `stt`, which is **not** a
`TOOLSETS` key at all. It appears there purely so you can pick a provider and enter
a credential; it contributes zero tools to the model schema. See §7c.

### 2a. Machine control

| Toolset | Tools exposed | Notes |
|---|---|---|
| `terminal` | `terminal`, `process` | Shell + long-running process management. Runs on whichever Layer 4 environment is configured. |
| `file` | `read_file`, `write_file`, `patch`, `search_files` | `patch` does fuzzy matching (`tools/fuzzy_match.py`); `search_files` covers content **and** filenames. |
| `code_execution` | `execute_code` | Runs Python that calls other tools programmatically. Exists to cut LLM round-trips — one script replaces ten tool calls. |
| `computer_use` | `computer_use` | Screenshots, mouse, keyboard, scroll, drag via `cua-driver` on macOS / Windows / Linux. **Does not steal your cursor or focus** — it drives a background session. |

### 2b. Web and browser

| Toolset | Tools exposed | Notes |
|---|---|---|
| `web` | `web_search`, `web_extract` | 8 pluggable search backends (Layer 4). `extract_char_limit` defaults to 15,000 chars/page. |
| `search` | `web_search` | Search **without** scraping. Use when you want discovery but not page fetch. |
| `browser` | 13 tools + `web_search` | `browser_navigate`, `browser_snapshot`, `browser_click`, `browser_type`, `browser_scroll`, `browser_back`, `browser_press`, `browser_get_images`, `browser_vision`, `browser_console`, `browser_cdp`, `browser_dialog`, `browser_exec` |
| `x_search` | `x_search` | Read-only public X/Twitter discovery via xAI's Responses tool. For *authenticated* X actions use the `xurl` skill instead (Layer 2). |

`browser_exec` is a swap, not an addition: when `browser.backend` is
`"browser-use"`, that single tool **replaces** the granular browser tools. The
default `""` auto-detects — Browser Use if its CLI is available, built-in tools
otherwise, and Camofox setups always keep the built-in tools because they expose
no CDP surface (`hermes_cli/config_defaults.py:459`).

### 2c. Perception

| Toolset | Tools | Notes |
|---|---|---|
| `vision` | `vision_analyze` | Image understanding. |
| `video` | `video_analyze` | **Off by default.** Requires a video-capable model. |
| *(stt)* | — | Speech-to-text is **config-only**: it ships zero model tools. It powers gateway voice messages and voice mode. Listed in `_CONFIG_ONLY_TOOLSETS`. |

### 2d. Generation

| Toolset | Tools | Notes |
|---|---|---|
| `image_gen` | `image_generate` | 7 backends (Layer 4). |
| `video_gen` | `video_generate`, `xai_video_edit`, `xai_video_extend` | **Off by default** — niche, paid, slow. One `video_generate` covers text-to-video, image-to-video, and reference-to-video. |
| `bfl` | 6 tools | Black Forest Labs FLUX 3 via the Nous tool gateway: `bfl_flux3_text_to_video`, `_image_to_video`, `_keyframes_to_video`, `_video_continuation`, `_get_result`, `_prompting_guide`. Generations take minutes, so submit returns a job id and the model polls. |
| `tts` | `text_to_speech` | 11 providers including three fully local ones. |

### 2e. Agent machinery

| Toolset | Tools | Notes |
|---|---|---|
| `memory` | `memory` | One tool, three actions: `add`, `replace`, `remove`. |
| `skills` | `skills_list`, `skill_view`, `skill_manage` | The Layer 2 entry point. |
| `todo` | `todo` | Multi-step planning and tracking. |
| `session_search` | `session_search` | Search past conversations, with summarization. |
| `clarify` | `clarify` | Ask the user a multiple-choice or open question mid-turn. |
| `delegation` | `delegate_task` | Spawn subagents with isolated context. |
| `cronjob` | `cronjob` | Create / list / update / pause / resume / remove / trigger scheduled jobs, with optional attached skills. |
| `context_engine` | *(empty)* | Deliberately empty in the static dict — tools are injected at runtime by the active context engine. |

### 2f. Integrations

| Toolset | Tools | Default |
|---|---|---|
| `homeassistant` | `ha_list_entities`, `ha_get_state`, `ha_list_services`, `ha_call_service` | Off; auto-enables when `HASS_TOKEN` is set |
| `spotify` | `spotify_playback`, `_devices`, `_queue`, `_search`, `_playlists`, `_albums`, `_library` | Off |
| `discord` | `discord` | Off — fetch messages, search members, create threads |
| `discord_admin` | `discord_admin` | Off — list channels/roles, pin, assign roles |
| `yuanbao` | `yb_query_group_info`, `yb_query_group_members`, `yb_send_dm`, `yb_search_sticker`, `yb_send_sticker` | Platform-scoped |
| `feishu_doc` | `feishu_doc_read` | Platform-scoped |
| `feishu_drive` | `feishu_drive_list_comments`, `_list_comment_replies`, `_reply_comment`, `_add_comment` | Platform-scoped |

### 2g. Machinery-gated primitives — you do not set these by hand

These six are real toolsets that never appear in the configurator, because
something other than user preference decides them.

| Toolset | Tools | Who enables it |
|---|---|---|
| `project` | `project_list`, `project_create`, `project_switch` | GUI gateway, for desktop-sourced sessions only |
| `desktop_ui` | `read_terminal`, `close_terminal`, `open_preview`, `read_preview`, `read_window_below`, `focus_pane`, `react_to_message`, `setup_mcp` | GUI gateway, same rule |
| `kanban` | 14 tools (see below) | The kanban dispatcher, via `HERMES_KANBAN_TASK` |
| `search` | `web_search` | A narrower alternative to `web`, selected by config |
| `feishu_doc` | `feishu_doc_read` | Folded in by the `hermes-feishu` bundle |
| `feishu_drive` | 4 comment tools | Folded in by the `hermes-feishu` bundle |

The `kanban` 14: `kanban_show`, `kanban_list`, `kanban_complete`, `kanban_block`,
`kanban_request_review`, `kanban_request_changes`, `kanban_heartbeat`,
`kanban_comment`, `kanban_create`, `kanban_link`, `kanban_unblock`,
`kanban_attach`, `kanban_attach_url`, `kanban_attachments`.

Note `kanban_request_review` is deliberately **not** a block — review is a
first-class state in the workflow, distinct from "stuck, needs a human."

---

## 3. The session-vs-process rule — read this before touching gating

This is the trap that has bitten contributors repeatedly, and the code comments
in `toolsets.py` shout about it in three separate places.

`project` and `desktop_ui` only work when a GUI renderer is on the other end of
the connection. The naive gate is "was this process launched by Electron?" —
i.e. check `HERMES_DESKTOP=1`. **That is wrong.**

```python
# toolsets.py, on the desktop_ui toolset:
# Enabled by the GUI gateway for a session whose SOURCE is the desktop app
# (tui_gateway/server.py::_load_enabled_toolsets), NOT by a process env var.
# The renderer is a CLIENT — it can be driving a local, SSH, URL, or cloud
# backend — so "was this process spawned by Electron?" is the wrong
# question and silently strips these tools from every remote gateway.
```

The desktop app is a *client*. It can point at a local backend, an SSH backend, a
URL, or a cloud backend. A process env var describes where the Python process
happens to be running, which tells you nothing about whether a GUI is listening.

**Rule: surface capability resolves from the session's `source`, never from a
process environment variable.** The resolution lives in
`tui_gateway/server.py::_load_enabled_toolsets`.

The same reasoning explains why the GUI toolsets are excluded from
`_HERMES_CORE_TOOLS` even though they are "core" features of the desktop app —
keeping them out preserves the narrow waist for every CLI, messaging, and cron
schema.

---

## 4. `_HERMES_CORE_TOOLS` — the shared spine

Most surface bundles are not hand-maintained lists. They point at one shared
constant near the top of `toolsets.py`:

```python
_HERMES_CORE_TOOLS = [
    "web_search", "web_extract",
    "terminal", "process",
    "read_file", "write_file", "patch", "search_files",
    "vision_analyze", "image_generate",
    "bfl_flux3_text_to_video", ... (6 BFL tools),
    "skills_list", "skill_view", "skill_manage",
    "browser_navigate", ... (13 browser tools),
    "text_to_speech",
    "todo", "memory",
    "session_search",
    "clarify",
    "execute_code", "delegate_task",
    "cronjob",
    "ha_list_entities", ... (4 Home Assistant tools),
    "kanban_show", ... (14 kanban tools),
    "computer_use",
]
```

Edit this once and every platform bundle updates simultaneously. That is the
whole point — and it is why `hermes-telegram`, `hermes-whatsapp`, `hermes-slack`,
`hermes-signal`, `hermes-email`, `hermes-sms`, `hermes-matrix`,
`hermes-mattermost`, `hermes-dingtalk`, `hermes-weixin`, `hermes-qqbot`,
`hermes-wecom`, `hermes-wecom-callback`, `hermes-bluebubbles`,
`hermes-homeassistant`, `hermes-cli`, and `hermes-cron` all have literally
identical tool lists.

Three bundles extend it:

```python
"hermes-discord":  _HERMES_CORE_TOOLS + ["discord", "discord_admin"]
"hermes-feishu":   _HERMES_CORE_TOOLS + [5 feishu doc/drive tools]
"hermes-yuanbao":  _HERMES_CORE_TOOLS + [5 yuanbao tools]
```

### One deliberate absence

There is **no agent-callable `send_message` tool** in any bundle. The comment is
explicit:

> agents do NOT get an agent-callable send_message tool — outbound platform
> messaging is handled outside the agent loop (cron delivery, the gateway kanban
> notifier, and the `hermes send` CLI), not by the model deciding to send on its
> own.

If you are designing an automation that "messages the user," you are reaching for
cron delivery or `hermes send`, not a tool call. This is a safety boundary, not an
oversight — a model that can autonomously message arbitrary channels is a
different risk profile.

---

## 5. The webhook posture — untrusted input gets a smaller surface

The one bundle that is aggressively narrow, and the reason is worth understanding
because it generalizes:

```python
_HERMES_WEBHOOK_SAFE_TOOLS = [
    "web_search",
    "web_extract",
    "vision_analyze",
    "clarify",
]
```

> Webhook events may originate from untrusted third-party content (for example,
> public PR titles/comments). Keep the default webhook toolset intentionally
> constrained to avoid local file/system execution by prompt injection.

No `terminal`. No `file`. No `execute_code`. A public PR title is attacker-
controlled text that will land in a model's context; if that context has shell
access, the PR title is a remote code execution vector.

**Generalize this:** any automation whose input you do not control should get a
reduced toolset. That principle applies to your own cron jobs, not just webhooks.

---

## 6. Postures — automatic per-session narrowing

Three toolsets represent a *mode of work* rather than a capability.

| Posture | Contents | Selected by |
|---|---|---|
| `coding` | 30 tools: file, terminal, search, web docs, skills, todo, memory, session_search, clarify, delegate, vision, full browser | `agent/coding_context.py` — auto-detected in a code workspace |
| `debugging` | `terminal`, `process` + includes `web`, `file` | Config / manual |
| `safe` | includes `web`, `vision`, `image_gen` — **no terminal, no file** | Config / manual |

`coding` deliberately **drops** messaging, tts, image_gen, spotify,
home-assistant, cron, and computer-use. It is marked with a flag:

```python
"posture": True,
```

That flag matters: posture toolsets are selected per-session and must never be
auto-recovered into per-platform tool config. There is an explicit
non-configurable-toolset recovery loop in
`hermes_cli/tools_config.py` that skips them.

Note that `coding` does not list the GUI pane tools even though you would want
them while pairing on code in the desktop app. That is the narrow-waist rule
again: pane access belongs to the *client surface*, not the *posture*, so the GUI
gateway folds `desktop_ui` in alongside whatever posture is active.

---

## 7. Gating — four mechanisms, applied in order

A toolset being defined does not mean its tools reach the model. Four independent
filters sit in between.

### 7a. `check_fn` — runtime capability probes

Every registered tool may carry a `check_fn`
(`tools/registry.py:213`). It answers "can this
tool actually work right now?" If it returns false, **the schema is never shown to
the model** — the model does not see a tool it would only fail to use.

Results are TTL-cached with a `_check_fn_last_good` timestamp per callable
(`tools/registry.py:275`) so a probe that shells
out isn't re-run on every turn.

Live examples:

- `homeassistant` — gated on `HASS_TOKEN`
- `computer_use` — gated on `cua-driver` being installed
- `kanban` — gated on `HERMES_KANBAN_TASK` or explicit profile opt-in
- `x_search` — gated on xAI OAuth tokens or `XAI_API_KEY`

The `x_search` comment captures why this beats a static config flag:

> The tool's `check_fn` means the schema still won't appear to the model if the
> credential later goes missing or expires.

### 7b. `_DEFAULT_OFF_TOOLSETS` — not pre-selected for new installs

```python
_DEFAULT_OFF_TOOLSETS = {"homeassistant", "spotify", "discord",
                         "discord_admin", "video", "video_gen",
                         "x_search", "a2a"}
```

Still available at runtime if you enable them; the first-run setup checklist just
doesn't tick them. Some auto-enable on credential detection (`homeassistant` on
`HASS_TOKEN`, `x_search` on xAI creds).

### 7c. `_CONFIG_ONLY_TOOLSETS` — appear in the menu, ship no tools

```python
_CONFIG_ONLY_TOOLSETS = {"stt"}
```

`stt` shows up in `hermes tools` so you can pick a provider and enter an API key,
but it contributes zero entries to the model schema. It feeds voice transcription
in the gateway and voice mode.

### 7d. Per-platform filtering

Each surface's config narrows its bundle. `hermes-cron` mirrors `hermes-cli`
exactly, then `_get_platform_tools()` filters per your `hermes tools` settings.

---

## 8. The two-step exposure trap

This is the number one reason a new tool "doesn't work."

Registering a tool and exposing a tool are **separate steps**:

1. **Discovery** — import-time `registry.register()` makes the tool *exist*.
2. **Membership** — the tool must also be a member of an *enabled toolset*.

Step 1 without step 2 produces a tool that is fully implemented, imports
cleanly, passes its tests, and is completely invisible to the model. There is no
error message. Nothing warns you.

If you add a tool and the model never calls it, check membership before you debug
anything else.

Plugins can register into an existing toolset, and `get_toolset()` merges those in
by default. Note the `include_registry=False` mode exists for a specific reason
documented at `toolsets.py` (issue #49622): platform
reverse-mapping needs the *static* authored view, so that a tool a plugin
registered into a toolset doesn't cause the whole toolset to drop out of platform
inference.

---

## 9. Configuring toolsets

### The config key

```yaml
# ~/.hermes/config.yaml
toolsets: ["hermes-cli"]     # default
```

You can list several, or narrow to primitives:

```yaml
toolsets: ["web", "browser", "file"]
```

The comment at `hermes_cli/config_defaults.py:1836`
notes that narrowing this way expresses "I want *these*" — an allowlist, not a
patch on the default.

### The interactive route

```bash
hermes tools          # toggle toolsets, configure providers, enter credentials
```

This is the recommended path because several toolsets have multi-step setup
(video generation walks you through provider **and** model selection; X search
walks you through credentials).

### Subagents get their own

```python
DEFAULT_TOOLSETS = ["terminal", "file", "web"]   # tools/delegate_tool.py:1016
```

A subagent is not a copy of its parent. It gets a deliberately small default set,
overridable per delegation. One toolset is blocked outright from composite
delegation:

```python
_COMPOSITE_BLOCKED_TOOLSETS = frozenset({"delegation"})   # :1176
```

Children cannot spawn grandchildren through a composite. That is the recursion
guard.

---

## 10. Complete toolset index — all 59

Primitives (32):
`web` · `search` · `x_search` · `vision` · `video` · `image_gen` · `video_gen` ·
`bfl` · `computer_use` · `terminal` · `skills` · `browser` · `cronjob` · `file` ·
`tts` · `todo` · `memory` · `context_engine` · `session_search` · `project` ·
`desktop_ui` · `clarify` · `code_execution` · `delegation` · `homeassistant` ·
`kanban` · `discord` · `discord_admin` · `yuanbao` · `feishu_doc` ·
`feishu_drive` · `spotify`

Postures (3):
`debugging` · `safe` · `coding`

Surface bundles (24):
`hermes-acp` · `hermes-api-server` · `hermes-cli` · `hermes-cron` ·
`hermes-telegram` · `hermes-discord` · `hermes-whatsapp` · `hermes-slack` ·
`hermes-signal` · `hermes-bluebubbles` · `hermes-homeassistant` ·
`hermes-email` · `hermes-mattermost` · `hermes-matrix` · `hermes-dingtalk` ·
`hermes-feishu` · `hermes-weixin` · `hermes-qqbot` · `hermes-wecom` ·
`hermes-wecom-callback` · `hermes-yuanbao` · `hermes-sms` · `hermes-webhook` ·
`hermes-gateway`

`hermes-gateway` is pure composition — it includes all 19 messaging bundles and
defines no tools of its own.

---

## 11. Adding a tool — where this sits on the ladder

A new core tool is **the top rung of the Footprint Ladder**, meaning it is the
last thing you should reach for, not the first. In ladder order:

1. Extend existing code
2. CLI command + skill
3. Service-gated tool (`check_fn`)
4. Plugin
5. MCP server
6. New core tool

Every tool added to `_HERMES_CORE_TOOLS` costs schema tokens in **every** session
on **every** surface, forever. That is why the bar is high and why the reviewer's
first question is "why isn't this a skill, a plugin, or an MCP server?"

Full procedure: `AGENTS.md` → "The Footprint Ladder".
Worked code: `CONTRIBUTING.md`.

---

## 12. Self-check

<details>
<summary>1. You wrote a tool, registered it, tests pass — the model never calls it. Why?</summary>

Almost certainly step 2 of the two-step exposure: it isn't a member of an enabled
toolset. Registration makes it exist; membership makes it visible. Check
`_HERMES_CORE_TOOLS` or the relevant primitive, then check whether that toolset is
enabled for your surface. Second candidate: a `check_fn` returning false.
</details>

<details>
<summary>2. Why doesn't the agent get a <code>send_message</code> tool?</summary>

Deliberate. Outbound messaging is handled outside the loop — cron delivery, the
gateway kanban notifier, and `hermes send`. A model that can autonomously send to
arbitrary channels is a materially different risk profile from one that can only
reply where it was addressed.
</details>

<details>
<summary>3. Your webhook automation needs to read a local file. What do you do?</summary>

Stop and reconsider. `_HERMES_WEBHOOK_SAFE_TOOLS` excludes file access because
webhook payloads are attacker-controlled text. If you genuinely need it, the
correct shape is a *narrow, purpose-built tool* with validated inputs — not
enabling the full `file` toolset on untrusted input.
</details>

<details>
<summary>4. Why is gating on <code>HERMES_DESKTOP=1</code> wrong for <code>desktop_ui</code>?</summary>

Because the desktop app is a client that may be driving a local, SSH, URL, or
cloud backend. The env var describes the Python process's launch context, not
whether a GUI renderer is listening. Gate on the session's `source` via
`tui_gateway/server.py::_load_enabled_toolsets`.
</details>

<details>
<summary>5. What's the difference between <code>web</code> and <code>search</code>?</summary>

`web` gives `web_search` **and** `web_extract` (fetch and scrape page content).
`search` gives only `web_search`. Use `search` when you want the agent to find
URLs without pulling page bodies into context.
</details>

---

**Next:** [02-layer2-skills.md](02-layer2-skills.md) — the 82 skills, and why
procedural knowledge is a different layer from capability.
