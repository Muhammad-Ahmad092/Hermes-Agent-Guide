# Layer 3 — Surfaces: Where a Turn Comes From, and Where Output Goes

> **Source of truth:** `gateway/platform_registry.py`
> (the adapter contract), `gateway/run.py` (the gateway),
> `plugins/platforms/` (22 plugin adapters),
> `gateway/platforms/` (9 in-tree adapters),
> `tui_gateway/` (the JSON-RPC boundary).

Layers 1 and 2 are about what the agent can do. Layer 3 is about *who is asking*
and *where the answer lands*. It is the layer people underestimate: Hermes is not
a terminal program with some integrations bolted on. The agent loop is
surface-agnostic, and the surface is genuinely pluggable.

At baseline `cf64ca20c`: **22 platform plugins + 9 in-tree adapters**, driven by
one gateway, plus **6 interactive surfaces** and **2 non-interactive entry points**.

---

## 1. The map

```
   INTERACTIVE                          NON-INTERACTIVE
   ┌──────────────────────┐             ┌─────────────────────┐
   │ CLI      (hermes)    │             │ cron scheduler      │
   │ TUI      (full-screen)│            │ webhooks            │
   │ desktop  (Electron)  │             └──────────┬──────────┘
   │ dashboard (web)      │                        │
   │ ACP      (editors)   │             ┌──────────▼──────────┐
   │ API server (OpenAI-  │             │ GATEWAY             │
   │            compatible)│            │ 31 platform adapters│
   └──────────┬───────────┘             └──────────┬──────────┘
              │                                    │
              └───────────────┬────────────────────┘
                              ▼
                     THE SAME AGENT LOOP
```

Every one of those paths converges on one loop. The differences are which toolset
bundle is active (Layer 1, §4), what the transport looks like, and how output is
rendered.

---

## 2. The six interactive surfaces

| Surface | Toolset bundle | Transport | Notes |
|---|---|---|---|
| **CLI** | `hermes-cli` | direct | `hermes` — the reference surface |
| **TUI** | `hermes-cli` | JSON-RPC over stdio | Full-screen terminal UI |
| **Desktop** | `hermes-cli` + `desktop_ui` + `project` | JSON-RPC | Electron. **The only surface with its own transcript.** |
| **Dashboard** | `hermes-cli` | HTTP + PTY bridge | Web UI that embeds the *real* TUI over a pseudo-terminal |
| **ACP** | `hermes-acp` | ACP protocol | VS Code, Zed, JetBrains |
| **API server** | `hermes-api-server` | HTTP | OpenAI-compatible endpoint |

Two of these are worth dwelling on because they are frequently misunderstood.

**The dashboard does not reimplement the TUI.** It bridges a PTY and runs the
actual TUI inside it. That means there is one terminal rendering implementation,
not two, and the dashboard cannot drift from the TUI's behavior.

**The desktop app is a client, not a runtime.** It can point at a local backend, a
remote SSH backend, a URL, or a cloud backend. This is exactly why toolset gating
must resolve from the session's `source` and not from `HERMES_DESKTOP=1` — see
[01-layer1-toolsets.md §3](01-layer1-toolsets.md). The desktop's exceptional
property is that it keeps its own transcript; every other surface reads session
history from the shared store.

### The JSON-RPC boundary

TypeScript owns the screen; Python owns sessions, tools, and model calls. The
method surface lives in `tui_gateway/` split by concern:

`methods_prompt.py` · `methods_session.py` · `methods_tools.py` ·
`methods_config.py` · `methods_profiles.py` · `methods_complete.py` ·
`methods_images.py`

`_load_enabled_toolsets` in `tui_gateway/server.py` is the function that decides
which toolsets a GUI-sourced session gets. Some commands are TUI-side only and
never appear in the Python `COMMAND_REGISTRY` — `/rewind` is the canonical
example, so don't grep the Python registry and conclude it doesn't exist.

---

## 3. The gateway

`gateway/run.py` is the long-running process that turns
Hermes into a bot. It owns adapter lifecycle, session routing, delivery,
readiness, drain, and restart.

Look at the module list to understand what running a messaging bot in production
actually requires:

| Concern | Modules |
|---|---|
| Lifecycle | `readiness.py`, `drain_control.py`, `restart.py`, `restart_loop_guard.py`, `shutdown_flush.py`, `shutdown_watchdog.py`, `shutdown_forensics.py`, `lifecycle_ledger.py` |
| Sessions | `session.py`, `session_state.py`, `session_context.py`, `session_stall.py`, `turn_context.py`, `turn_lease.py` |
| Delivery | `delivery.py`, `delivery_ledger.py`, `dead_targets.py`, `mirror.py`, `rich_sent_store.py` |
| Resource pressure | `agent_cache_pressure.py`, `memory_monitor.py`, `memory_status.py`, `disk_status.py`, `cgroup_cleanup.py`, `scale_to_zero.py` |
| Routing / auth | `platform_registry.py`, `profile_routing.py`, `channel_directory.py`, `authz_mixin.py`, `pairing.py`, `slash_access.py` |
| Streaming | `stream_consumer.py`, `stream_dispatch.py`, `stream_events.py`, `streaming_tts_consumer.py` |

`turn_lease.py` and `session_stall.py` exist because a chat platform can deliver
two messages before the first turn finishes. `dead_targets.py` and
`delivery_ledger.py` exist because delivery can fail *after* the agent
successfully produced output — a distinction the cron code also makes explicitly:

> a job can succeed (agent produced output) but fail delivery (platform down)
> — `cron/jobs.py:2382`

### Windows support

The gateway runs as a daemon on Windows via Task Scheduler
(`hermes_cli/gateway_windows.py`, using
`schtasks`), with `systemd` on Linux and `launchd` on macOS. There is also a
`scripts/check-windows-footguns.py` in
CI, which tells you Windows is a supported target and not an afterthought.

---

## 4. The adapter contract — the most instructive dataclass in the repo

`PlatformEntry` in `gateway/platform_registry.py`
is the complete definition of "a place Hermes can talk." Reading its fields
teaches you more about production chat integration than any prose summary.

### Registration

```python
from gateway.platform_registry import platform_registry, PlatformEntry

platform_registry.register(PlatformEntry(
    name="irc",
    label="IRC",
    adapter_factory=lambda cfg: IRCAdapter(cfg),
    check_fn=check_requirements,
    validate_config=lambda cfg: bool(cfg.extra.get("server")),
    required_env=["IRC_SERVER"],
    install_hint="pip install irc",
))
```

### The fields, grouped by what problem they solve

**Existence and construction**

| Field | Purpose |
|---|---|
| `name`, `label`, `emoji` | Identity and display |
| `adapter_factory` | Builds the adapter from a `PlatformConfig` |
| `source` | `"builtin"` or `"plugin"` |
| `plugin_name` | Lets `hermes gateway setup` auto-enable the owning plugin when you configure its platform |

**Readiness — and the two-field lesson**

| Field | Purpose |
|---|---|
| `check_fn` | **Passive** probe: are deps importable *right now*? Must never pip-install. |
| `ensure_deps_fn` | **Active** installer: make deps available, installing if needed |
| `validate_config` | Is this config properly filled in? |
| `is_connected` | Is it actually up? Used by status displays |

The comment explaining why these are two separate fields is the best bug story in
the codebase (issue #79812):

> when the ACTIVE installer was registered as `check_fn`, every status display
> pip-installed SDKs as a side effect (desktop boot-loop at 94%); when the PASSIVE
> probe was registered instead, `create_adapter()` returned None before
> `connect()` could lazy-install, so the deps never installed at all (Teams
> deadlock). Splitting the two roles makes both call sites correct by
> construction.

One field serving two callers with opposite needs produced two opposite bugs.
"Correct by construction" is the fix worth stealing.

**Authorization**

| Field | Purpose |
|---|---|
| `allowed_users_env` | e.g. `IRC_ALLOWED_USERS` — comma-separated allowlist |
| `allow_all_env` | e.g. `IRC_ALLOW_ALL_USERS` — truthy opens it up |

A bot reachable by anyone who knows its handle, holding `terminal` access to your
machine, is the obvious catastrophe. Every adapter declares its allowlist env var.

**Platform reality**

| Field | Purpose |
|---|---|
| `max_message_length` | Drives smart-chunking. `0` = no limit |
| `pii_safe` | If true, session descriptions redact phone numbers and other PII |
| `platform_hint` | Injected into the system prompt, e.g. *"You are on IRC. Do not use markdown."* |
| `allow_update_command` | Whether `/update` is permitted from this platform |

`platform_hint` is a small, elegant idea: the model is told about the medium's
constraints rather than the renderer trying to strip markdown after the fact.

**Configuration bridging**

| Field | Purpose |
|---|---|
| `env_enablement_fn` | Read env vars, seed `PlatformConfig.extra` before the adapter is built — so `gateway status` reflects env-only config without instantiating anything |
| `apply_yaml_config_fn` | Translate this platform's `config.yaml` keys into env vars / `extra`, so core `gateway/config.py` doesn't have to know every platform's schema |

`apply_yaml_config_fn` is the narrow waist in action: the platform owns its own
config translation. Env still beats YAML — plugin authors are told to guard with
`not os.getenv(...)`.

**Delivery and targeting**

| Field | Purpose |
|---|---|
| `cron_deliver_env_var` | e.g. `IRC_HOME_CHANNEL`. **When set, this platform becomes a valid cron `deliver=` target.** |
| `parse_target_ref_fn` | Declare native target syntax (e.g. `fmsg:@alice@example.com`) without hard-casing it in core |
| `validate_target_ref_fn` | Post-parse validation, returning `True`/`False`/diagnostic string |
| `send_message_handler` | Whole-request custom delivery |
| `standalone_sender_fn` | **Async send with no live gateway adapter** |

`standalone_sender_fn` solves a real deployment problem. When cron runs in a
separate process from the gateway, the in-process adapter weakref is `None`.
Without this hook a plugin platform **cannot serve as a cron `deliver=` target**
in that topology. If you write a platform plugin and want scheduled jobs to
deliver to it, implement this.

---

## 5. The 22 platform plugins

Each is a directory under `plugins/platforms/` with a
`plugin.yaml` manifest.

### Consumer / personal

| Plugin | Label |
|---|---|
| `telegram` | Telegram |
| `whatsapp` | WhatsApp |
| `discord` | Discord |
| `sms` | SMS (Twilio) |
| `line` | LINE |
| `simplex` | SimpleX Chat |
| `photon` | iMessage via Photon |
| `email` | Email |

### Team / workplace

| Plugin | Label |
|---|---|
| `slack` | Slack |
| `teams` | Microsoft Teams |
| `google_chat` | Google Chat |
| `mattermost` | Mattermost |
| `matrix` | Matrix |
| `irc` | IRC |

### China-region enterprise

| Plugin | Label |
|---|---|
| `feishu` | Feishu / Lark |
| `dingtalk` | DingTalk |
| `wecom` | WeCom (Enterprise WeChat) |

### Non-chat channels

| Plugin | Label | Notes |
|---|---|---|
| `ntfy` | ntfy | Push notifications |
| `homeassistant` | Home Assistant | Smart-home events as a message source |
| `a2a` | A2A | Agent-to-agent protocol. Off by default. |
| `buzz` | Buzz | |
| `raft` | Raft | |

## 6. The 9 in-tree adapters

Not everything is a plugin. `gateway/platforms/` holds
adapters that are still built in:

| File | Platform |
|---|---|
| `signal.py` (+ `signal_format.py`, `signal_rate_limit.py`) | Signal |
| `bluebubbles.py` | iMessage via a local BlueBubbles server |
| `whatsapp_cloud.py` (+ `whatsapp_common.py`) | WhatsApp Cloud API |
| `weixin.py` | Weixin / personal WeChat via iLink |
| `yuanbao.py` (+ `_media`, `_proto`, `_sticker`) | Yuanbao |
| `qqbot/` | QQ via Official Bot API v2 |
| `webhook.py` (+ `webhook_filters.py`) | Generic webhooks |
| `msgraph_webhook.py` | Microsoft Graph subscriptions |
| `api_server.py` | OpenAI-compatible HTTP surface |

> **⚠ Doc drift.** `gateway/platforms/ADDING_A_PLATFORM.md`
> still references `telegram.py`, `discord.py`, and `whatsapp.py` as if they lived
> here. They were migrated to `plugins/platforms/`. Read the doc for the *contract*,
> read `plugins/platforms/irc/` for the current *shape*.

The direction of travel is clear: adapters move out of the tree into plugins over
time. `platform_registry.py` says so:

> Built-in adapters continue to use the existing if/elif in `_create_adapter()`
> for now. Plugin adapters register here … and are looked up first — if nothing is
> found the gateway falls through to the legacy code path.

Registry first, legacy fallback. When you add a platform, you add a plugin.

---

## 7. Deferred registration

Importing 22 platform SDKs at startup would be slow and would fail loudly for
every platform you don't use. So discovery registers a cheap **deferred loader**
per platform:

```python
platform_registry.register_deferred(...)   # :296
```

The real `register()` runs only when the platform is actually looked up. A
concretely-registered built-in takes precedence over a deferred loader
(`gateway/platform_registry.py:309`),
and `is_registered()` counts a deferred, not-yet-imported platform as registered —
because from the user's point of view, it is.

---

## 8. The gateway's safety posture

Three things constrain what a message from the outside world can do.

**1. Webhook input gets a reduced toolset.** `_HERMES_WEBHOOK_SAFE_TOOLS` is
`web_search`, `web_extract`, `vision_analyze`, `clarify` — no shell, no files, no
code execution. Public PR titles are attacker-controlled text.

**2. Authorization is per-platform and explicit.** `allowed_users_env` /
`allow_all_env` on every adapter.

**3. The agent cannot send unsolicited messages.** There is no agent-callable
`send_message` tool in any bundle. Outbound goes through cron delivery, the
gateway kanban notifier, or `hermes send` — never a model decision. See
[01-layer1-toolsets.md §4](01-layer1-toolsets.md).

Terminal access over chat is guarded by dangerous-command approval rather than
being removed — the toolset descriptions say "full access (terminal has safety
checks)". The relevant machinery is `tools/approval.py`
and the `approvals` / `command_allowlist` config sections.

---

## 9. Adding a platform

Rung 4 of the Footprint Ladder — a plugin.

1. `mkdir plugins/platforms/mything/` with a `plugin.yaml`
2. Implement an adapter against the base class in `gateway/platforms/base.py`
3. `platform_registry.register(PlatformEntry(...))` in your plugin's `register()`
4. Fill in the fields that apply — at minimum `check_fn`, `required_env`,
   `install_hint`, `max_message_length`, `allowed_users_env`
5. Add `cron_deliver_env_var` **and** `standalone_sender_fn` if scheduled jobs
   should be able to deliver there
6. Add a `platform_hint` if the medium has formatting constraints
7. Add a toolset bundle in `toolsets.py` if you need
   platform-specific tools — otherwise reuse `_HERMES_CORE_TOOLS`

Read `plugins/platforms/irc/` as the reference
implementation; it is small enough to hold in your head and exercises most of the
contract.

> **Closed list (June 2026):** no new third-party-*product* plugins. A *protocol*
> or platform adapter is still in scope; a plugin that wraps one vendor's SaaS
> product generally is not — that's a skill. Check
> `AGENTS.md` → "The Footprint Ladder" before you start.

---

## 10. Self-check

<details>
<summary>1. Why does the dashboard embed a PTY instead of reimplementing the TUI?</summary>

So there is exactly one terminal rendering implementation. A reimplementation
would drift from the TUI's behavior on every change. The dashboard runs the real
TUI inside a pseudo-terminal.
</details>

<details>
<summary>2. Why are <code>check_fn</code> and <code>ensure_deps_fn</code> separate fields?</summary>

They serve callers with opposite needs. Status displays must probe without side
effects; `create_adapter()` must be able to install. One field for both produced a
desktop boot-loop when it installed, and a Teams deadlock when it didn't
(#79812). Splitting them makes both call sites correct by construction.
</details>

<details>
<summary>3. Your cron job delivers to your custom platform in dev but not in production. Why?</summary>

In production the gateway and cron are separate processes, so the in-process
adapter weakref is `None`. You need `standalone_sender_fn` — an async sender that
opens its own ephemeral connection. Also confirm `cron_deliver_env_var` is set,
which is what makes the platform a valid `deliver=` target at all.
</details>

<details>
<summary>4. Why does a webhook-triggered turn get four tools when a Telegram turn gets ~50?</summary>

Trust. A Telegram message comes from an authorized user in your allowlist. A
webhook payload may embed arbitrary third-party text — a public PR title. If that
text reaches a context with shell access, it is a remote code execution vector.
</details>

<details>
<summary>5. Where is <code>/rewind</code> implemented?</summary>

TUI-side, via `tui_gateway/methods_prompt.py`.
It is not in the Python `COMMAND_REGISTRY`, so grepping there and concluding it
doesn't exist is a mistake — TypeScript owns the screen and some commands are
purely a screen concern.
</details>

---

**Next:** [04-layer4-backends.md](04-layer4-backends.md) — the 35 model providers,
8 memory stores, 8 execution environments, and every other swappable engine.
