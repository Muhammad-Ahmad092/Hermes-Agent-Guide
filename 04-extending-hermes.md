# 04 · Extending Hermes

**For:** anyone about to add a capability.
**Prerequisites:** [01 · What Hermes Is](01-what-hermes-is.md)
**After this file you will know:** how to decide *where* your feature belongs
before you write a line of it — and which shapes are refused outright.
**Next:** [05 · Recipes](05-recipes.md) for the code

> This is the highest-leverage file in the guide. Most rejected PRs here are not
> badly written; they are built on the **wrong rung**. Spend ten minutes here to
> save a week.

---

## 1. The Footprint Ladder

Each rung adds more permanent surface than the one above it. **Choose the
highest rung that correctly solves the problem.**

| Rung | Mechanism | Permanent core footprint | When it's right |
|---|---|---|---|
| **1** | Extend existing code | none | Your capability is a variation of something that already exists |
| **2** | CLI command + skill | none | Manages config, state, or infra expressible as shell commands |
| **3** | Service-gated tool (`check_fn`) | none unless configured | Needs structured params/returns **and** a prerequisite |
| **4** | Plugin | none | Third-party, niche, or user-specific; doesn't ship in core |
| **5** | MCP server in the catalog | none | Genuinely needs to be a tool, but isn't core-fundamental |
| **6** | New core tool | **permanent, paid on every API call** | Fundamental, near-universal, unreachable via terminal + file |

Rung 2 is the **default choice** for subscriptions, scheduled tasks, and service
setup. Existing examples: `hermes webhook`, `hermes cron`, `hermes tools`.

Correct rung-6 tools, for calibration: `terminal`, `read_file`, `web_search`,
`browser_navigate`. That is the bar — things nearly every user needs on nearly
every turn.

### Why rung 6 is expensive

A core tool's JSON schema is serialized into **every** API request for **every**
user of **every** surface, forever. A 300-token description on a 50-turn
conversation is 15,000 tokens of pure overhead — and worse, it dilutes the
model's attention across tools that matter. Tool-schema bloat degrades tool
*selection accuracy*, not just cost.

---

## 2. The decision procedure

Answer in order. Stop at the first "yes".

```
1. Does something in the tree already do 80% of this?
   → YES: extend it. RUNG 1. Read the original commit's intent first
     (git log -p -S "<symbol>") so you extend rather than fight the design.

2. Can the agent accomplish it by running shell commands, guided by prose?
   → YES: RUNG 2 — a CLI subcommand plus a skill that teaches its use.
     Zero schema cost. This covers far more than people expect.

3. Does it need structured arguments and a structured return, AND only make
   sense when a prerequisite is configured (a key, a daemon, a device)?
   → YES: RUNG 3 — a tool with check_fn. Invisible until configured.

4. Is it someone else's product, a niche integration, or user-specific?
   → YES: RUNG 4 — a plugin. Note the in-tree policies in §4 below.

5. Does it need to be an invocable tool but isn't fundamental to Hermes?
   → YES: RUNG 5 — build an MCP server, add it to the catalog. Reusable by
     any MCP host, zero core schema.

6. Is it fundamental, needed by nearly every user, and impossible via
   terminal + read_file/write_file or an MCP server?
   → Only now: RUNG 6. Expect to justify it in review.
```

### Worked examples

| Capability | Rung | Why |
|---|---|---|
| Manage webhook subscriptions | 2 | State management expressible as commands → `hermes webhook` + skill |
| Control Home Assistant devices | 3 | Structured I/O, gated on `HASS_TOKEN` → invisible without it |
| Support a new chat platform | 4 | Platform plugin, zero core changes needed |
| Add an inference provider | 4 | `plugins/model-providers/<name>/` |
| Read a proprietary internal API | 5 | MCP server; not core-fundamental |
| Query a Postgres database | 5 | MCP server exists; don't add a core tool |
| Generate a chart from data | 2 | Skill + a script in `scripts/`, run via `terminal` |
| Search the web | 6 | Already core — this is the calibration example |
| A tool your team alone needs | 4 | `~/.hermes/plugins/` — never a core patch |

---

## 3. Rung-by-rung mechanics

### Rung 1 — Extend existing code

Before extending, **read the original intent**:

```bash
git log -p -S "<symbol>" -- path/to/file.py
```

A limitation that looks like an oversight is often deliberate. Real example: a PR
adding live config inheritance from the default profile was closed because
profile isolation *is* the design — the copy-at-creation `--clone` path already
covers the legitimate "start from my default" case.

### Rung 2 — CLI command + skill

Two artifacts:

1. A CLI subcommand. Either add it under `hermes_cli/` and wire the argparse
   tree, or — better for anything optional — register it from a plugin via
   `ctx.register_cli_command(...)`, which needs no change to `main.py`.
2. A skill (`SKILL.md`) that teaches the agent when and how to run it.

The agent then calls `terminal` with `hermes <subcommand> …`. Zero schema cost,
and the human gets a usable command too.

### Rung 3 — Service-gated tool

A normal tool registration plus a `check_fn`:

```python
def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

registry.register(
    name="example_tool",
    toolset="example",
    schema={...},
    handler=...,
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

Two things to get right:

- **`check_fn` answers reachability or user opt-in — never "which surface am I
  on".** "Is the daemon up?", "did the user enable reactions?" → fine. "Was I
  spawned by Electron?" → wrong; that is a session property (see §5).
- **`check_fn` results are TTL-cached process-wide.** One process serves many
  sessions, so a per-session answer must not live there at all.

### Rung 4 — Plugin

A directory with `plugin.yaml` and `__init__.py` exposing `register(ctx)`. It can
register tools, lifecycle hooks, CLI subcommands, platform adapters, memory
providers, model providers, image-gen backends, and dashboard pages.

Discovery sources: `~/.hermes/plugins/`, `./.hermes/plugins/` (opt-in via
`HERMES_ENABLE_PROJECT_PLUGINS`), pip entry points, and the bundled
`plugins/` directory.

**The hard rule:** a plugin must never modify core files (`run_agent.py`,
`cli.py`, `gateway/run.py`, `hermes_cli/main.py`). If your plugin needs something
the framework doesn't expose, widen the **generic** plugin surface — add a hook
or a `ctx` method that any plugin could use. Never special-case your plugin in
core.

### Rung 5 — MCP server

Hermes ships a native MCP client with HTTP/SSE transports, OAuth 2.1 PKCE, mTLS,
dynamic tool discovery via `notifications/tools/list_changed`, sampling support,
and selective tool loading. MCP servers appear as **standalone toolsets**, so
they cost nothing until enabled.

There is a curated Nous-approved catalog with an interactive picker. Adding your
server there is the cheapest way to give the agent a genuinely new invocable
capability.

### Rung 6 — Core tool

Two files, both required:

1. `tools/your_tool.py` with a top-level `registry.register()` call.
2. An entry in `toolsets.py` — either `_HERMES_CORE_TOOLS` or a new toolset.

Step 2 is **not optional**: auto-discovery imports and registers the schema, but
the tool is only *exposed to an agent* if a toolset names it. Full recipe in
[05 § Recipe 4](05-recipes.md#recipe-4--add-a-core-tool).

---

## 4. Closed lists — read before you write

These are settled policies. PRs against them are closed with a pointer, not a
review. None of them is a quality judgment; all of them are about maintenance
coupling.

### No new in-tree memory providers (May 2026)

The set under `plugins/memory/` is **closed**. New memory backends ship as
standalone plugin repos installed into `~/.hermes/plugins/` or via pip entry
points. They implement the same `MemoryProvider` ABC, register through the same
discovery path, and integrate via `hermes memory setup` / `post_setup()`.
Existing in-tree providers stay; bug fixes to them are welcome.

### No new third-party-product plugins in-tree (June 2026)

Same rule beyond memory: observability/metrics backends, vendor SaaS connectors,
analytics dashboards, paid-service tie-ins — all ship as standalone plugin repos.
The reason is explicit: every product absorbed into the tree becomes the
maintainers' burden to keep working against a fast-moving core, for a backend
they don't own.

The `observability/`, `kanban/`, and `disk-cleanup/` directories already in tree
are **existing precedent, not an invitation** to add more alongside them.

Promote standalone plugins in the Nous Research Discord
(`#plugins-skills-and-skins`).

### No new `HERMES_*` env vars for non-secret config

`.env` is credentials only. Behavior goes in `config.yaml`. Bridge to an internal
env var if a mechanism needs one, but user-facing docs point at the config key.

### No speculative extension points

A hook, callback, or extension point with **no concrete consumer** is rejected.
Adding one is easy; removing one after plugins depend on it is not.

A hook is *not* speculative if a contributor has a real, stated use case — even
if the consumer ships separately. Say so in the PR.

### No lazy-reading escape hatches on instructional tools

No `offset`/`limit` pagination on tools that load content the agent must read
fully — skills, prompts, playbooks. Models read page 1 and skip the rest.

### No outbound telemetry without an opt-in gate

No new analytics, third-party identifier tagging, or attribution tags until a
generic user-facing opt-in exists (config gate + setup prompt + `hermes tools`
toggle). Park behind a label; do not merge.

### No new core tool when terminal + file already do the job

And if the only barrier is file visibility on a remote backend, **fix the mount,
not the toolset.**

---

## 5. The session-vs-process trap

If your capability only works because of *who is on the other end of the
connection* — desktop panes, the in-app browser, message reactions, Projects —
it is a **session** property, never a process-environment property.

**Why the obvious thing is wrong.** The desktop app can be driving a backend
that Electron spawned locally, one over SSH, one behind a plain URL + token, or
Hermes Cloud. Only the first two carry `HERMES_DESKTOP=1`. So
`if os.getenv("HERMES_DESKTOP")` is a silent no-op on the other half of the
topologies — and the failure is invisible, because the tool is stripped from the
schema before the model ever sees it, on the same backend whose platform hint is
telling the model it is "chatting inside the Hermes desktop app."

**The pattern that works:**

- **The toolset is the surface gate.** Keep the tools off `_HERMES_CORE_TOOLS`
  (nobody else should pay their schema) and put them in a named toolset —
  `desktop_ui`, `project`. The GUI gateway's `_load_enabled_toolsets(platform)`
  folds that toolset in when the *session's platform* says GUI. One resolver,
  every topology.
- **`check_fn` answers reachability or opt-in, not surface.**
- **`HERMES_DESKTOP=1` does have a legitimate meaning:** "this backend process
  was spawned by the app." It correctly gates the cron ticker and web-dist
  handling. It does **not** mean "a GUI is watching" — the embedded terminal pane
  (`hermes --tui` against that same backend) is the standing counterexample.

**The test that proves you got it right:** assert that a GUI session gets the
tool **with the env var absent**. That is the assertion the broken gate could
never pass.

---

## 6. When three PRs want the same category

If 3+ open PRs try to integrate the same *category* of thing — memory backends,
providers, notifiers — **don't merge them one at a time.** Design an ABC plus an
orchestrator, wrap the existing built-in as the first provider, and turn the
competing PRs into plugins against that interface.

This is how `plugins/memory/`, `plugins/model-providers/`, and
`plugins/image_gen/` all came to exist. If you notice this pattern forming,
saying so in an issue is a genuinely valuable contribution.

---

## 7. Prove the premise before you build

The most common reason a well-written PR gets closed is not code quality — it is
a **wrong premise**, or treating an **intentional design as a gap**. Four
patterns, all distilled from real closes:

**"Intentional design, not a gap."** A limitation that looks like an oversight is
often deliberate. Ask whether the isolation *is* the design.

**"The premise doesn't hold against how X actually works."** PR rationales
frequently rest on a wrong mental model. Two real closes: a rate-limit
"re-probe during cooldown" PR (the breaker only trips on a *confirmed-empty*
account bucket, so re-probing just hammers a bucket already proven empty); and a
usage-accumulation fix whose new branch **never executes at runtime** because an
earlier guard already popped the state it depended on.

> If you cannot point to the exact line where the bug manifests **and** show
> that your fix changes that line's behavior, you have not verified the premise.

**"The omission was deliberate."** Adding the obvious-looking missing piece can
break what the omission protected. Real example: restoring "missing"
`__init__.py` files made a test tree importable as a dotted package that shadowed
the real plugin, deleting its `register()` at import time. The absence was
load-bearing.

**"Overreached."** Scope creep past an agreed base, or reviving a direction the
maintainers deliberately closed, gets rejected even when the code works. Keep to
the narrow piece that was agreed; offer the rest as a focused follow-up.

When in doubt about intent, **ask** — it is cheaper than shipping a fix that
fights the design.

---

## 8. What the maintainers actively want

So the picture isn't all restriction. From the rubric, in priority order:

1. **Bug fixes** — crashes, incorrect behavior, data loss. Always top priority.
   A good fix reproduces the symptom on current `main`, points at the exact line
   where it manifests, and fixes the whole bug *class* — sibling call paths
   included — not just the one site the reporter hit.
2. **Cross-platform compatibility** — macOS, Linux distros, native Windows, WSL2.
3. **Security hardening** — shell injection, prompt injection, path traversal,
   privilege escalation.
4. **Performance and robustness** — retries, error handling, graceful degradation.
5. **Expanding reach at the edges** — new platform adapters, channels, providers,
   models, and desktop/TUI/dashboard features are welcome and land routinely,
   **including large ones**.
6. **New skills** — broadly useful ones.
7. **New core tools** — rarely needed.
8. **Documentation** — fixes, clarifications, examples.

The balance to read right: the product surface expands aggressively and on
purpose. The restraint is aimed squarely at the **core agent and the model tool
schema** — the one place where every addition is paid for on every API call.

---

## 9. Pre-flight checklist

Before writing code:

- [ ] I searched open **and merged** PRs and issues for this
      (`gh search prs --repo NousResearch/hermes-agent --state all "<terms>"`)
- [ ] I searched the **source** — the issue tracker lags the code, and many
      requested features already exist in-tree
- [ ] I identified the rung and can defend it in one sentence
- [ ] If rung 6: I can name why terminal + file + MCP cannot do this
- [ ] If it's a hook: I can name the concrete consumer
- [ ] If it's config: it's going in `config.yaml`, not `.env`
- [ ] If it's surface-dependent: I'm gating on the session, not the process env
- [ ] I read the original intent of anything I'm about to "fix"
      (`git log -p -S`)
- [ ] It isn't on a closed list (§4)

Then go to [05 · Recipes](05-recipes.md).
