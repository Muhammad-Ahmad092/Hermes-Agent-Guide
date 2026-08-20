# 01 · What Hermes Is

**For:** anyone who just cloned this repo and wants the mental model before
touching code.
**After this file you will know:** what the agent actually does, the two rules
that govern every design decision, and what happens between "user types a
message" and "agent replies".
**Next:** [02 · Architecture](02-architecture.md)

---

## 1. The one-sentence version

Hermes is a personal AI agent: a loop that calls a language model, runs whatever
tools the model asked for, and repeats until the model answers — wrapped in six
different front ends, and able to learn across sessions.

That is it. Everything else in 1.7M lines of Python is (a) making that loop
survive contact with reality, (b) giving it capability, and (c) letting people
reach it from wherever they are.

## 2. What makes it unusual

Most agent frameworks are libraries you build an app on top of. Hermes is a
finished product that happens to be extensible. Three consequences:

**It is not tied to your laptop.** The same agent runs on a $5 VPS, a GPU
cluster, or serverless infrastructure, and you talk to it from Telegram while it
works on a cloud VM. That is why there is a gateway process, seven remote
terminal backends, and a relay layer.

**It closes a learning loop.** It writes skills for itself after complex tasks,
improves them during use, curates and archives them, searches its own past
conversations, and builds a model of the user across sessions. Most agents
forget everything between runs.

**It is model-agnostic on purpose.** 35 provider plugins, a model catalog
refreshed hourly, OAuth flows for a dozen vendors, and `hermes model` to switch
with no code change. No provider SDK is load-bearing.

---

## 3. The two invariants

These are quoted from `AGENTS.md` and they are the lens for reviewing any
change. Learn them before you write code.

### 3.1 Per-conversation prompt caching is sacred

A long-lived conversation reuses a cached prefix on every turn. Providers charge
a fraction for cached tokens — often 10% of the fresh input rate. So a
50-turn conversation is cheap *only if* turns 1–49 stay byte-identical.

Anything that mutates past context, swaps toolsets, or rebuilds the system
prompt mid-conversation invalidates that cache and multiplies the user's cost.

Concretely, this is why:

- **Skill slash-commands are injected as a *user* message**, not appended to the
  system prompt (`agent/skill_commands.py`). A system-prompt edit would
  invalidate everything before it.
- **Toolsets are resolved once per session**, not per turn. A tool appearing or
  disappearing mid-conversation rewrites the schema block that sits in the
  cached prefix.
- **Context compression is called out as the *single* sanctioned exception.**
  When you must break the prefix, you do it deliberately, once, and you rotate
  the session.

There is a dedicated module for the boundary (`agent/prompt_cache_boundary.py`)
and a scope helper (`agent/prompt_cache_scope.py`). If your change touches
message assembly, you are in cache territory — assume you must prove prefix
stability.

### 3.2 The core is a narrow waist; capability lives at the edges

Every core tool's JSON schema is sent on **every** API call for **every** user,
forever. A 400-token tool description on a 50-turn conversation is 20,000 tokens
of pure overhead — and it dilutes the model's attention across the tools that
matter.

So the bar for a new *core* tool is high, and almost all new capability should
arrive as something cheaper: a skill, a CLI command, a plugin, an MCP server, or
a tool that only appears when its prerequisite is configured.

Read this correctly, though: **the product is allowed to grow aggressively.**
New platforms, providers, models, and desktop/TUI/dashboard features land
routinely, including large ones. "Smallest footprint" governs *how a capability
is wired into the core*, not whether the product may expand. Expansive at the
edges, conservative at the waist.

[04 · Extending Hermes](04-extending-hermes.md) turns this into a decision
procedure.

### 3.3 The corollary: surface capability is a property of the session

A tool that only works because of *who is on the other end of the connection* —
the desktop app's panes, message reactions, Projects — must resolve its
availability from **the session's own source**, never from an environment
variable on the backend process.

Why: the client and the backend are separate machines. The desktop app might be
driving a backend spawned locally, one over SSH, one behind a URL + token, or
Hermes Cloud. Only the locally-spawned ones carry `HERMES_DESKTOP=1`. An
env-keyed GUI gate is therefore a silent no-op on half the topologies — and the
failure is invisible, because the tool is stripped from the schema before the
model ever sees it.

The pattern that works: keep the tools out of `_HERMES_CORE_TOOLS`, put them in
a named toolset (`desktop_ui`, `project`), and let the GUI gateway fold that
toolset in when the *session's platform* says GUI.

---

## 4. One turn, traced end to end

Here is what actually happens when a user sends a message. Follow along in
`agent/conversation_loop.py` — the ~3,900-line body extracted out of
`run_agent.py`.

### Step 0 — A surface takes the input

The CLI reads it through `prompt_toolkit`. The TUI sends a `prompt.submit`
JSON-RPC request. The gateway receives a Telegram update. The desktop app posts
from its own composer. All of them end up calling the same
`AIAgent.run_conversation()`.

### Step 1 — Assemble the request

The prefix is rebuilt from cached parts: system prompt, context files
(`AGENTS.md`/`CLAUDE.md`), loaded skills, tool schemas, prior messages. The new
user turn is appended — `@`-references resolved, images attached, platform hints
added.

**This is the step where cache discipline is enforced.** Everything before the
new turn must be identical to last turn.

### Step 2 — Call the provider

Through an adapter, because providers disagree about everything: `chat_completions`,
Anthropic's native API, Codex Responses, Bedrock, Vertex, Gemini native. See
`agent/transports/` and the `*_adapter.py` files in `agent/`.

### Step 3 — Did it ask for tools?

**No** → that text is the reply. Turn over, jump to step 6.

**Yes** → dispatch them. Tool calls in one response can run concurrently
(a `ThreadPoolExecutor`). Each one may hit an approval gate first
(`tools/approval.py`): dangerous-command patterns, user deny rules, the hardline
blocklist for unrecoverable commands. Results are appended as `tool` role
messages, and the loop goes back to step 2.

The loop is bounded three ways: `max_iterations` (default **90** API calls),
`iteration_budget`, and an interrupt flag the user can set from any surface.
There is a one-turn "grace call" so a budget exhaustion can still produce a
final answer instead of dying silently.

### Step 4 — Side path: running out of context

Before a call, if the assembled request would exceed the model's window,
compaction runs: summarize the older part of the conversation, keep the recent
tail, and continue. This is the sanctioned cache break — it typically rotates
the session id (with an in-place mode that keeps one id).

Compaction is the most-tuned subsystem in the tree: temporal anchoring in
summaries, per-model trigger thresholds, lean tail mode, and a recall eval
harness under `evals/compaction/`.

### Step 5 — Side path: the provider failed

`agent/error_classifier.py` decides what kind of failure it was, and that
decides the response: retry with backoff, fail over to the next provider in the
fallback chain, drop to the fallback model, or surface an actionable error.
Status noise is buffered and only shown on terminal failure, so a transient
blip doesn't spam the user.

### Step 6 — Post-turn work

The turn is persisted to SQLite (`hermes_state.py`). Then, out of band:

- **Memory providers** get the completed turn (`agent/memory_manager.py`).
- **Background review** may propose memory writes and skill improvements using
  an auxiliary model (`agent/background_review.py`) — behind an approve/deny gate.
- **The curator** eventually reviews skill usage and archives stale ones.
- **File-mutation verification** may run if enabled for this surface.
- **Plugin lifecycle hooks** fire (`on_session_end`, `post_llm_call`, …).

That out-of-band work is the learning loop. It is deliberately *not* inline in
the prompt — inline nudges were replaced by background review precisely because
they broke the cache.

---

## 5. What runs where

Six surfaces, three process shapes.

**In-process.** The CLI (`cli.py`) and one-shot mode (`hermes -z`) instantiate
`AIAgent` directly in the same process.

**Two processes over JSON-RPC.** The TUI is Node (Ink/React) for the screen and
Python (`tui_gateway/`) for everything real, talking newline-delimited JSON-RPC
over stdio. The desktop app is Electron + React talking JSON-RPC over WebSocket
to a headless `hermes serve` backend. The dashboard serves a Vite SPA and
embeds the *real* TUI through a PTY bridge.

**Long-lived service.** The gateway (`gateway/run.py`) holds connections to
chat platforms, caches an `AIAgent` per session (for prompt caching), and runs
the cron ticker and the kanban dispatcher by default.

The rule that keeps this from forking into six agents: **TypeScript owns the
screen; Python owns sessions, tools, model calls, and slash-command logic.**

---

## 6. What "self-improving" actually means here

Four independent mechanisms, each with its own store and its own governor. None
of them is magic; all of them are auditable.

| Mechanism | Where | Governor |
|---|---|---|
| **Memory** — durable facts about the user and the work | 8 pluggable providers behind `MemoryProvider` ABC | Per-profile isolation; approve/deny gate on writes |
| **Skills** — reusable procedures, some agent-written | `~/.hermes/skills/`, `skills/`, `optional-skills/` | The curator: usage telemetry, auto-archive, never deletes, exempts pinned |
| **Session recall** — search its own history | SQLite FTS5 + CJK trigram index | Single tool, no LLM in the path |
| **Background review** — propose improvements after hard turns | `agent/background_review.py` | Auxiliary model, off the critical path, gated writes |

The curator's invariants are worth internalizing as an example of the taste in
this codebase: it only touches skills with `created_by: agent` provenance,
bundled and hub-installed skills are off-limits, it never deletes (archive is
the maximum destructive action), archives are restorable, and pinned skills are
exempt from every automatic transition.

---

## 7. What Hermes is *not*

Knowing the non-goals will save you a rejected PR.

- **Not a library.** There is no stable public Python API to build your app on.
  Extend it through plugins and skills.
- **Not a place for other people's products.** Observability backends, vendor
  SaaS connectors, analytics dashboards, and new memory backends are explicitly
  closed to in-tree contributions — they ship as standalone plugin repos. This
  is about maintenance coupling, not quality.
- **Not configured through environment variables.** `.env` is for **secrets
  only**. Every behavioral setting goes in `config.yaml`. A PR that tells users
  to "set `HERMES_FOO` in your .env" for a non-secret gets rejected.
- **Not a place for speculative extension points.** A hook with no concrete
  consumer is rejected — adding one is easy, removing one after plugins depend
  on it is not.

---

## 8. Check your understanding

If you can answer these from the above, you are ready for
[02 · Architecture](02-architecture.md).

1. Why is a skill slash-command injected as a user message instead of added to
   the system prompt?
2. You want a tool that only works when the user is in the desktop app. Why is
   `if os.getenv("HERMES_DESKTOP")` the wrong gate, and what is the right one?
3. Your feature needs a 300-token tool description. What question should you ask
   before adding it to `_HERMES_CORE_TOOLS`?
4. Name the one operation that is allowed to break the cached prefix, and why it
   is allowed.
5. Where does the agent's turn get persisted, and what happens *after* that?

<details>
<summary>Answers</summary>

1. Because the system prompt sits at the front of the cached prefix. Editing it
   invalidates every cached token behind it; a user message appends to the end
   and costs nothing cached.
2. Because the desktop client and the backend can be different machines — only
   locally-spawned backends carry that env var, so the gate is a silent no-op
   over SSH, plain URL+token, and Cloud topologies. The right gate is the
   session's own platform/source, folded in by the GUI gateway's toolset
   resolver.
3. "Does every user of every surface need to pay for these tokens on every API
   call?" If not, it belongs on a lower rung — a service-gated tool, a plugin,
   an MCP server, or a skill.
4. Context compaction. It is the only way to keep a conversation alive past the
   model's window, and it is done deliberately and once rather than incidentally
   every turn.
5. SQLite via `hermes_state.py`. Afterwards, out of band: memory providers get
   the turn, background review may propose memory/skill writes, the curator
   eventually reviews skill usage, and plugin lifecycle hooks fire.

</details>
