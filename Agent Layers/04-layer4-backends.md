# Layer 4 — Backends: Who Actually Executes the Work

> **Source of truth:** `plugins/model-providers/` (35),
> `plugins/memory/` (8),
> `tools/environments/` (8),
> `plugins/web/` (8),
> `hermes_cli/config_defaults.py` (all of it).

Layers 1–3 are about capability, competence, and reach. Layer 4 is the substrate:
when the model calls `terminal`, *which machine runs the command*. When it calls
`web_search`, *which search engine*. When it thinks, *which model*.

Almost every engine in Hermes is swappable, and the swap is a config key rather
than a code change. This is the layer that determines your cost, your privacy
posture, and your blast radius.

---

## 1. The swappable inventory

| Backend class | Count | Config key | Default |
|---|---|---|---|
| Model providers | **35** | `model`, `providers` | resolved from credentials |
| Auxiliary model slots | **14** | `auxiliary.<task>.provider` | `auto` (= main model) |
| Execution environments | **8** | `terminal.backend` | `local` |
| Memory backends | **8** + built-in | `memory.provider` | `""` (built-in only) |
| Web search / extract | **8** | `web.backend` | `""` (auto) |
| Browser engines | 3 plugins + built-in | `browser.backend` | `""` (auto) |
| Image generation | **7** | via `hermes tools` | — |
| Video generation | **3** | via `hermes tools` | — |
| Text-to-speech | **11** | `tts.provider` | `edge` (free) |
| Speech-to-text | **6** | `stt.provider` | `local` (faster-whisper) |
| Wake word | **3** | `wake_word.provider` | `openwakeword` (free, local) |
| Context engine | pluggable | `context.engine` | `compressor` |
| Cron scheduler | 1 plugin + built-in | — | built-in |
| Dashboard auth | **4** | — | — |
| Observability | **2** | — | off |

---

## 2. Model providers — all 35

Each is a directory under `plugins/model-providers/`
with a `plugin.yaml`.

### Frontier labs

| Provider | Label |
|---|---|
| `anthropic` | Anthropic (Claude) |
| `gemini` | Google Gemini (API key + Cloud Code OAuth) |
| `vertex` | Google Vertex AI (Gemini via OpenAI-compatible endpoint, OAuth2) |
| `xai` | xAI Grok (Responses API) |
| `openai-codex` | OpenAI Codex (Responses API) |
| `deepseek` | DeepSeek |
| `nous` | Nous Research Portal |

### Aggregators and gateways

| Provider | Label |
|---|---|
| `openrouter` | OpenRouter aggregator |
| `ai-gateway` | Vercel AI Gateway |
| `commandcode` | CommandCode — unified multi-model API |
| `opencode-zen` | OpenCode (Zen + Go) |
| `huggingface` | HuggingFace Inference Providers |

### Hyperscalers

| Provider | Label |
|---|---|
| `bedrock` | AWS Bedrock |
| `azure-foundry` | Microsoft Foundry |
| `nvidia` | NVIDIA NIM |

### Open-model inference hosts

| Provider | Label |
|---|---|
| `deepinfra` | DeepInfra — 100+ open models, pay-per-use |
| `fireworks` | Fireworks AI |
| `novita` | NovitaAI |
| `gmi` | GMI Cloud |
| `arcee` | Arcee AI |
| `actual` | Actual Computer inference |
| `ollama-cloud` | Ollama Cloud |

### China-region

| Provider | Label |
|---|---|
| `alibaba` | Alibaba DashScope (international) |
| `alibaba-coding-plan` | Alibaba Cloud Coding Plan |
| `zai` | Z.AI / GLM |
| `kimi-coding` | Moonshot Kimi Coding (global + China) |
| `minimax` | MiniMax M-series (global + China + OAuth) |
| `qwen-oauth` | Qwen Portal (OAuth) |
| `stepfun` | StepFun Step Plan |
| `xiaomi` | Xiaomi MiMo |
| `upstage` | Upstage (Solar API) |

### Coding-subscription bridges

| Provider | Label |
|---|---|
| `copilot` | GitHub Copilot |
| `copilot-acp` | GitHub Copilot via ACP subprocess |
| `kilocode` | Kilo Code |

### The escape hatch

| Provider | Label |
|---|---|
| `custom` | Custom / Ollama / local OpenAI-compatible endpoint |

`custom` is how you run entirely locally. Point it at Ollama, llama.cpp's server,
vLLM, or anything speaking the OpenAI wire format, and no request leaves your
machine. Combine with `terminal.backend: local` and `memory.provider: ""` for a
fully offline posture.

### Auth model matters

Notice the parenthetical labels: several providers support **OAuth in addition to
API keys** — `gemini` (Cloud Code OAuth), `vertex` (OAuth2), `minimax`,
`qwen-oauth`, `copilot`. This exists so you can use a subscription you already pay
for instead of provisioning a separate API key with separate billing.

### Resilience

```yaml
model: "..."
providers: {...}
fallback_providers: []            # ordered failover list
credential_pool_strategies: {}    # rotate across multiple keys per provider
```

Both empty by default. `fallback_providers` gives you ordered failover when a
provider errors or rate-limits; `credential_pool_strategies` rotates across
multiple credentials for one provider.

---

## 3. Auxiliary models — the detail that saves you the most money

Hermes does not use your main model for everything. There are **14 named auxiliary
task slots**, each independently routable:

```
vision           web_extract        compression       skills_hub
approval         mcp                title_generation  memory_query_rewrite
tts_audio_tags   triage_specifier   kanban_decomposer profile_describer
goal_judge       curator
```

Each takes a full provider block:

```yaml
auxiliary:
  monitor:
    provider: "openrouter"
    model: "google/gemini-3-flash-preview"
    reasoning_effort: ""     # none|minimal|low|medium|high|xhigh|max|ultra
  curator:
    provider: "auto"         # "auto" = use the main chat model
    timeout: 600
```

The config comment for the monitor slot explains the economics:

> "auto" = main chat model; override to a cheap fast model (e.g. openrouter
> google/gemini-3-flash-preview, haiku) since per-item scoring is high-volume and
> a small model is fine.

**This is the highest-leverage tuning knob in Hermes.** If you run the
important-mail monitor hourly, per-item urgency scoring is by far your highest
request volume, and it does not need a frontier model. Route
`title_generation`, `memory_query_rewrite`, and `monitor` to something small and
fast; leave the main model alone.

Every slot also accepts `reasoning_effort`, so you can dial thinking per task
rather than globally.

---

## 4. Execution environments — where `terminal` actually runs

Eight backends behind one interface (`BaseEnvironment`), selected by
`terminal.backend` and constructed by `_create_environment` in
`tools/terminal_tool.py`.

| Backend | Isolation | Persistence | Notes |
|---|---|---|---|
| `local` | **none** | your filesystem | Default. Spawn-per-call with a session snapshot. |
| `docker` | strong | bind mounts | Hardened: `cap-drop ALL`, `no-new-privileges`, PID limits, configurable CPU/memory/disk |
| `singularity` | strong | writable overlay dirs | Apptainer. `--containall`, `--no-home`, capability dropping |
| `ssh` | remote host | remote FS | ControlMaster connection persistence |
| `modal` | cloud sandbox | snapshots across sessions | Native Modal SDK, `Sandbox.create()` / `.exec()` |
| `managed_modal` | cloud sandbox | — | Modal via the Nous tool gateway |
| `daytona` | cloud sandbox | stop + resume preserves FS | Daytona SDK |
| `vercel_sandbox` | cloud sandbox | task-scoped snapshots under `HERMES_HOME` | Vercel SDK |

```yaml
terminal:
  backend: "local"        # local | docker | ssh | singularity | modal | daytona | vercel_sandbox
  modal_mode: "auto"
  degraded_mode: "warn"   # warn | fail
  cwd: "."
```

### The unified shell contract

Every backend implements the same model
(`tools/environments/base.py`):

> Unified spawn-per-call model: every command spawns a fresh `bash -c` process. A
> session snapshot (env vars, functions, aliases) is captured once at init and
> re-sourced before each command. CWD persists via in-band stdout markers (remote)
> or a temp file (local).

That is a genuinely hard problem solved cleanly. Because there is no persistent
shell, there is no shell state to corrupt or leak — but `cd` still works across
calls, and your aliases still exist, because both are reconstructed.

### File sync vs bind mounts

`tools/environments/file_sync.py` tracks
local changes by mtime+size, detects deletions, and syncs transactionally. Used by
**SSH, Modal, and Daytona**. Docker and Singularity skip it entirely — they bind-
mount and therefore already see a live host FS view.

### Graceful degradation

```yaml
terminal:
  degraded_mode: "warn"     # default
```

When a connection-class failure hits (SSH host unreachable, Docker daemon down),
`warn` returns a **structured degraded result with a reason and retry hint** so the
model can react intelligently. `fail` restores the old error-plus-traceback
behavior. The default is right: a model that receives "host unreachable, retry
hint: check VPN" can do something useful; one that receives a Python traceback
usually cannot.

### The recommendation

**Interactive work: `local`. Autonomous work: not `local`.**

A scheduled job that runs shell commands unattended against your real filesystem
is the highest-risk configuration in Hermes. Point cron jobs at `docker` or a
cloud sandbox. The eight backends exist precisely so this is a one-line change.

---

## 5. Memory — built-in plus 8 providers

### The built-in store

`tools/memory_tool.py` — two files, no external
service:

- `MEMORY.md` — the agent's own notes: environment facts, project conventions,
  tool quirks, things learned
- `USER.md` — what it knows about you: preferences, communication style, workflow

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200      # ~800 tokens at 2.75 chars/token
  user_char_limit: 1375        # ~500 tokens
  write_approval: false
  provider: ""                 # empty = built-in only
```

Two design details worth stealing:

**Character limits, not token limits.** *"because char counts are
model-independent."* A token budget is a lie the moment you switch tokenizers.

**The frozen-snapshot pattern.** Both files are injected into the system prompt at
session start. Mid-session writes hit disk immediately (durable) but do **not**
change the system prompt — the snapshot refreshes on the next session. That is
invariant #1 being defended: a mid-session prompt edit would invalidate the prefix
cache for the entire remaining session.

Entries are delimited by `§` and may be multiline. Edits use short unique
substring matching rather than IDs or full text.

### Write approval

```yaml
memory:
  write_approval: true
```

Applies to both foreground turns **and** the background self-improvement review
fork — which the config identifies as the source of *"unprompted 'wrong
assumption' saves users reported."* Foreground writes prompt inline (entries are
small enough to read in a chat bubble); background writes are **staged**, because a
daemon thread cannot block on a prompt.

```bash
/memory pending
/memory approve <id>
/memory reject <id>
```

Contrast Layer 2: skills always stage, never prompt, because a `SKILL.md` is too
big for a bubble. Same gate, different UX, driven by payload size.

### The 8 external providers

| Provider | What it adds |
|---|---|
| `mem0` | Server-side LLM fact extraction, semantic search, automatic deduplication |
| `hindsight` | Knowledge graph, entity resolution, multi-strategy retrieval |
| `holographic` | **Local** SQLite fact store, FTS5 search, trust scoring, HRR-based recall |
| `honcho` | AI-native cross-session user modeling, dialectic Q&A, semantic search |
| `openviking` | Context database with automatic extraction, tiered retrieval |
| `byterover` | Persistent knowledge tree, tiered retrieval, via the `brv` CLI |
| `retaindb` | Cloud memory API, hybrid search, 7 memory types |
| `supermemory` | Semantic long-term memory, profile recall, explicit memories |

```yaml
memory:
  provider: "holographic"
```

**Only one external provider at a time** — the config states this explicitly.
Providers inject their tools via `MemoryManager`, not through the toolset system,
which is why there is no `honcho` toolset any more (there used to be; the comment
in `toolsets.py` records its removal).

`holographic` is the one to look at first if you care about privacy: local SQLite,
no network.

> **Closed list (May 2026):** no new in-tree memory providers. Eight is the
> ceiling. Write it as an external plugin.

---

## 6. Web, browser, and generation backends

### Web search / extract — 8

`brave_free` · `ddgs` · `exa` · `firecrawl` · `parallel` · `searxng` · `tavily` ·
`xai`

```yaml
web:
  backend: ""                # shared fallback for both capabilities
  search_backend: ""         # per-capability override, e.g. "searxng"
  extract_backend: ""        # e.g. "native"
  extract_char_limit: 15000  # per page; larger pages truncate + cache full text
```

Search and extract are independently routable — a common real setup is `searxng`
for search (self-hosted, private) with `native` extraction.

`searxng` self-hosted is the privacy-preserving option; `brave_free` and `ddgs`
need no key.

### Browser — 3 plugins plus built-in

Plugins: `browserbase` · `browser_use` · `firecrawl`.
Built-in: native CDP (`tools/browser_cdp_tool.py`)
and Camofox (`tools/browser_camofox.py`).

```yaml
browser:
  backend: ""              # "" auto | "browser-use" | "off"
  inactivity_timeout: 120
  command_timeout: 30
  record_sessions: false   # auto-record as WebM
  headed: false            # visible window; also persists between turns
  allow_private_urls: false
  # engine: auto | lightpanda | chrome   (lightpanda: 1.3–5.8x faster nav, no screenshots)
```

`allow_private_urls: false` is a default worth understanding: without it the
browser will not navigate to `localhost` or RFC1918 addresses. That blocks SSRF
against your own network from a page the agent was told to visit.

Remember from Layer 1 that `backend: "browser-use"` **replaces** the 13 granular
browser tools with one `browser_exec`.

### Image generation — 7

`deepinfra` · `fal` · `krea` · `openai` · `openai-codex` · `openrouter` · `xai`

### Video generation — 3

`deepinfra` · `fal` · `xai` — plus the separate `bfl` toolset for FLUX 3 through
the Nous tool gateway.

---

## 7. Voice — TTS, STT, wake word

### Text-to-speech — 11 providers

```yaml
tts:
  provider: "edge"     # default, free
```

Cloud: `edge` (free) · `elevenlabs` · `openai` · `xai` · `minimax` · `mistral` ·
`gemini` · `deepinfra`
**Local:** `neutts` · `kittentts` · `piper`

Per-provider char limits differ substantially — Gemini 32,000, Edge 5,000, Mistral
4,000, NeuTTS/KittenTTS 2,000 — so the same text may need chunking on one provider
and not another.

### Speech-to-text — 6 providers

```yaml
stt:
  provider: "local"    # faster-whisper, free
```

`local` (faster-whisper) · `groq` · `openai` (Whisper API) · `mistral` (Voxtral
Transcribe) · `elevenlabs` (Scribe) · `deepinfra`

Recall from Layer 1 that `stt` is a **config-only** toolset — it ships zero model
tools and instead powers gateway voice messages and voice mode.

### Wake word — 3 providers

```yaml
wake_word:
  provider: "openwakeword"
```

`openwakeword` (free, local) · `sherpa` (free, **any phrase, no training**) ·
`porcupine` (premium, needs `PORCUPINE_ACCESS_KEY`)

`sherpa` is the interesting one — arbitrary wake phrases with no model training
step.

---

## 8. The remaining engines

### Context engine

```yaml
context:
  engine: "compressor"    # default, built-in
```

Selection is config-driven exactly like memory providers
(`agent/agent_init.py:2524`): if `engine` is
anything other than `"compressor"`, Hermes tries
`plugins/context_engine/<name>/`. That directory ships **empty** at this baseline —
the extension point exists, with no alternative implementations yet. The
`context_engine` toolset is correspondingly empty in the static dict, with tools
injected at runtime by whatever engine is active.

### Cron scheduler

Built-in (`cron/scheduler.py`) plus one provider
plugin: `plugins/cron_providers/chronos/`.
Selected via `cron/scheduler_provider.py`.

### Dashboard auth — 4

`basic` · `drain` · `nous` · `self_hosted`
(`plugins/dashboard_auth/`)

### Observability — 2

`langfuse` · `nemo_relay` (`plugins/observability/`).
Off by default — consistent with the repo rule that there is no outbound telemetry
without an explicit opt-in gate.

### Database

```yaml
database:
  journal_mode: "wal"        # DELETE for weak-fsync/shared filesystems
  wal_autocheckpoint: null
  journal_size_limit: null
```

Set `DELETE` on macOS virtiofs, NFS, or SMB, where WAL is not crash-safe. If you
run Hermes with `HERMES_HOME` on a network share, this is the setting that
prevents corruption.

---

## 9. Three worked configurations

### Maximum privacy — nothing leaves the machine

```yaml
model: "..."                    # a local model
providers:
  custom:
    base_url: "http://localhost:11434/v1"
terminal:
  backend: "local"
web:
  search_backend: "searxng"     # self-hosted
  extract_backend: "native"
memory:
  provider: "holographic"       # local SQLite
tts:
  provider: "piper"             # local
stt:
  provider: "local"             # faster-whisper
wake_word:
  provider: "openwakeword"      # local
```

Every engine local or self-hosted. Observability stays off by default.

### Minimum cost with a good main model

```yaml
model: "<frontier model>"
auxiliary:
  title_generation:     { provider: "openrouter", model: "<small fast model>" }
  memory_query_rewrite: { provider: "openrouter", model: "<small fast model>" }
  web_extract:          { provider: "openrouter", model: "<small fast model>" }
  compression:          { provider: "openrouter", model: "<small fast model>" }
  monitor:              { provider: "openrouter", model: "<small fast model>" }
fallback_providers: ["openrouter"]
```

The frontier model handles reasoning; the high-volume mechanical slots don't.

### Safe autonomy — for anything on a schedule

```yaml
terminal:
  backend: "docker"
  degraded_mode: "warn"
browser:
  allow_private_urls: false
memory:
  write_approval: true
skills:
  write_approval: true
  inline_shell: false
```

Sandboxed shell, no SSRF into your LAN, and both self-modification paths gated.

---

## 10. Self-check

<details>
<summary>1. Your automation bill is dominated by one hourly monitor job. Cheapest fix?</summary>

Route `auxiliary.monitor` to a small fast model. Per-item urgency scoring is
high-volume and does not need a frontier model — the config comment says so
directly. Leave your main model alone.
</details>

<details>
<summary>2. Why char limits instead of token limits on memory?</summary>

*"because char counts are model-independent."* A token budget assumes a tokenizer;
switch providers and the budget silently changes meaning.
</details>

<details>
<summary>3. Why does a mid-session memory write not change the system prompt?</summary>

Invariant #1. The prompt is a frozen snapshot taken at session start; editing it
mid-session would invalidate the prefix cache for every remaining turn. Writes are
durable on disk immediately and appear in the prompt at the next session start.
</details>

<details>
<summary>4. Docker and Singularity skip <code>file_sync</code>. Why?</summary>

They bind-mount, so they already have a live host filesystem view. SSH, Modal, and
Daytona are genuinely remote and need transactional mtime+size-based syncing.
</details>

<details>
<summary>5. You put <code>HERMES_HOME</code> on an SMB share and see DB corruption. Fix?</summary>

`database.journal_mode: "DELETE"`. WAL is not crash-safe on weak-fsync or shared
filesystems — macOS virtiofs, NFS, SMB — and the default is WAL.
</details>

<details>
<summary>6. Why is <code>allow_private_urls</code> false by default?</summary>

SSRF. Without it, a page the agent was asked to visit could steer the browser at
`localhost` or `192.168.x.x` and reach services on your network that assume they
are unreachable from outside.
</details>

---

**Next:** [05-composing-automations.md](05-composing-automations.md) — how all four
layers combine into automations that actually run.
