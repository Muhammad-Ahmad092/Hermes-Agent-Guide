# Layer 2 — Skills: Procedural Knowledge, Loaded on Demand

> **Source of truth:** `skills/` (82 `SKILL.md` files),
> `tools/skills_tool.py`,
> `tools/skill_ledger.py`.
> **Runtime location:** `~/.hermes/skills/` — seeded from bundled `skills/` on install.

Layer 1 decides *whether* the model may run a shell command. Layer 2 decides
*whether it knows the right way* to run one for a specific job. A skill is a
folder containing a `SKILL.md` — instructions, conventions, gotchas, and often
helper scripts — that the model pulls into context only when it becomes relevant.

At baseline `cf64ca20c`: **82 skills** across **15 categories**.

---

## 1. Why skills are a separate layer

The obvious alternative would be a much longer system prompt. That is forbidden by
the first invariant.

Per-conversation prompt caching requires a byte-stable system prompt for the life
of a session. If all 82 skills were pasted into the prompt you would pay for tens
of thousands of tokens on every turn of every session — including the ones where
you asked what time it is.

So skills use **progressive disclosure**:

```
Turn 1:  model sees a short INDEX of skill names + one-line descriptions
Turn 2:  model calls skill_view("notion") when Notion actually comes up
         → full SKILL.md enters the conversation, not the system prompt
```

The one-line `description` in the frontmatter is therefore doing real work: it is
the *only* thing the model sees when deciding whether a skill is relevant. A vague
description means the skill is never loaded. This is the single most important
thing to get right when authoring one.

---

## 2. Anatomy of a skill

A real example, `skills/productivity/notion/SKILL.md`:

```yaml
---
name: notion
description: "Notion API + ntn CLI: pages, databases, markdown, Workers."
version: 2.0.0
author: community
license: MIT
platforms: [linux, macos, windows]
prerequisites:
  env_vars: [NOTION_API_KEY]
metadata:
  hermes:
    tags: [Notion, Productivity, Notes, Database, API, CLI, Workers]
    homepage: https://developers.notion.com
---

# Notion

Talk to Notion two ways. Same integration token works for both — pick by what's
available.

◆ **`ntn` CLI** — Notion's official CLI. Shorter syntax, one-line file uploads,
required for Workers. macOS + Linux only as of May 2026 (Windows support
"coming soon"). **Default when installed.**
```

### Frontmatter fields that change behavior

| Field | Purpose |
|---|---|
| `name` | Identity. Must match what `skill_view` is called with. |
| `description` | **The relevance signal.** One line, specific, names the concrete nouns a user would say. |
| `platforms` | `[linux, macos, windows]` — filters the skill out where it cannot work. This is why the Apple skills never surface on your Windows box. |
| `prerequisites.env_vars` | Declares required credentials. Lets the surface tell you *why* a skill isn't usable. |
| `version`, `author`, `license` | Provenance — matters for hub-installed skills. |
| `metadata.hermes.tags` | Search/discovery. |
| `metadata.hermes.homepage` | Upstream docs link. |

Notice the body itself is honest about platform gaps ("macOS + Linux only as of
May 2026"). Good skills date their claims, because a skill is documentation that
ships as code and rots the same way.

---

## 3. The three tools that drive this layer

From the `skills` toolset (Layer 1):

| Tool | Does |
|---|---|
| `skills_list` | Enumerate available skills with descriptions |
| `skill_view` | Load one skill's full content into the conversation |
| `skill_manage` | Create, edit, patch, delete skills — the agent authoring its own |

`skill_manage` is why this layer is unusual: **the agent can write its own
skills.** That capability comes with three separate safety systems, covered in §6.

---

## 4. Discovery and precedence

```python
# tools/skills_tool.py:140
# All skills live in ~/.hermes/skills/ (seeded from bundled skills/ on install).
SKILLS_DIR = HERMES_HOME / "skills"
```

Three things follow from this that surprise people:

1. **The bundled `skills/` directory in the repo is a seed, not the runtime
   source.** Editing `skills/foo/SKILL.md` in a checkout does nothing to a running
   Hermes until it is re-seeded or you edit the copy under `~/.hermes/skills/`.

2. **`HERMES_HOME` is profile-scoped.** Different profiles get different skill
   sets. `SKILLS_DIR` is re-resolved rather than trusted as a module constant,
   precisely because it can be stale in long-lived runtimes
   (`tools/skills_tool.py:152`).

3. **Skills outside the trusted directory are flagged.** Loading one produces a
   warning: `skill file is outside the trusted skills directory (~/.hermes/skills/)`
   (`tools/skills_tool.py:1388`).

### Adding your own directories

```yaml
skills:
  external_dirs: ["~/.agents/skills", "/shared/team-skills"]
```

This is the sanctioned way to keep a team skill library in its own git repo
without vendoring it into `~/.hermes/`.

### Category comes from the path

```
~/.hermes/skills/mlops/axolotl/SKILL.md  ->  category "mlops"
```

Search is recursive, so nesting deeper than one level works — `mlops` contains 5
skills across 4 directories at this baseline.

---

## 5. Two body features worth knowing

### Template variables (on by default)

```yaml
skills:
  template_vars: true
```

`${HERMES_SKILL_DIR}` and `${HERMES_SESSION_ID}` are substituted before the agent
sees the content. This lets a skill reference its own bundled scripts without the
model having to reason about path joining:

```markdown
Run the helper: `python ${HERMES_SKILL_DIR}/scripts/extract.py input.pdf`
```

### Inline shell (off by default — and it should stay off)

```yaml
skills:
  inline_shell: false           # default
  inline_shell_timeout: 10
```

When enabled, snippets written as `` !`cmd` `` in a skill body are **executed on
your host** and their stdout is inlined before the agent reads the skill. That
lets a skill inject live context — today's date, git state, detected tool
versions.

The config comment states the risk plainly:

> Off by default because any content from the skill author runs on the host
> without approval; only enable for skill sources you trust.

If you install a community skill from a hub with `inline_shell: true` set, you
have granted that skill author arbitrary code execution on your machine. Leave it
off unless every skill directory is one you wrote.

---

## 6. The three safety systems around agent-authored skills

Because `skill_manage` lets the model write files that later become instructions
to itself, this layer carries more machinery than the others.

### 6a. The security scanner

```yaml
skills:
  guard_agent_created: false    # default
```

A keyword/pattern scanner (`tools/skills_guard.py`,
`tools/threat_patterns.py`) runs on skills the
agent writes. Off by default, with a candid rationale:

> Off by default because the agent can already execute the same code paths via
> `terminal()` with no gate, so the scan adds friction (blocks skills that mention
> risky keywords in prose) without meaningful security.

Note the asymmetry that *is* enforced:

> External hub installs (trusted/community sources) are always scanned regardless
> of this setting.

Your own agent writing a skill is not a new privilege. A stranger's skill arriving
over the network is.

### 6b. The approval gate

```yaml
skills:
  write_approval: false         # default
```

Turn it on and `skill_manage` writes are **staged** rather than committed. The
design note explains why staging rather than an inline prompt:

> a SKILL.md is too large to review inline, so skills always stage rather than
> prompt

Review flow:

```bash
/skills pending          # list staged writes
/skills diff <id>        # full diff — CLI/dashboard/file, never a chat bubble
/skills approve <id>
/skills reject <id>
```

This applies to **both** foreground turns and the background self-improvement
review fork — which is where unprompted writes come from.

### 6c. The curator ledger

Every mutation, from any actor, appends one JSONL line to
`~/.hermes/skills/.curator_ledger.jsonl`, with before/after file manifests stored
content-addressed (sha256-deduped) under `~/.hermes/.curator_backups/blobs/`
(`tools/skill_ledger.py`).

Recovery:

```bash
hermes curator rollback <entry-id>
```

Three design decisions from that file are worth internalizing as general
engineering practice:

- **JSONL, not the state DB** — a durable, human-greppable audit trail that
  survives DB resets and is rsync-friendly.
- **Content-addressed per-file blobs, not tarballs** — a mutation usually touches
  one file, so per-mutation tarballs would be wasteful, and identical content
  dedupes to one blob.
- **The ledger is telemetry, not a gate** — a ledger failure must never block the
  mutation it describes. Every write path swallows exceptions.

With one deliberate exception: `rollback_entry` **fails closed** when its own
pre-rollback safety capture fails. Writing an audit record is best-effort;
destroying current state to restore an old one is not.

The curator invariant — never hard-delete autonomously — applies to autonomous
actors only. A foreground user delete is a real delete, but it is still ledgered
and therefore still recoverable.

---

## 7. The complete catalog — 82 skills

### productivity (17)

| Skill | Description |
|---|---|
| `google-workspace` | Gmail, Calendar, Drive, Docs, Sheets via `gws` CLI or Python |
| `notion` | Notion API + `ntn` CLI: pages, databases, markdown, Workers |
| `airtable` | Airtable REST API via curl. Records CRUD, filters, upserts |
| `box` | Cloud files, sharing, search, metadata |
| `xlsx` | Create, read, edit Excel `.xlsx` workbooks and CSVs |
| `docx` | Create, read, edit, template, review Word `.docx` |
| `powerpoint` | Create, read, edit `.pptx` decks with python-pptx |
| `pdf` | Create, read, merge, fill, secure PDFs |
| `nano-pdf` | Edit text in existing PDFs via natural-language prompts |
| `ocr-and-documents` | Extract text from PDFs/scans (pymupdf, marker-pdf) |
| `maps` | Geocode, POIs, routes, timezones via OpenStreetMap/OSRM |
| `meeting-action-items` | Meeting notes → cited decisions, owners, tickets |
| `document-to-action-items` | Extract cited obligations, deadlines, tasks |
| `weekly-review-planning` | Weekly reset: commitments, stalled work, next-week plan |
| `product-price-monitor` | Watch product/flight/listing prices; alert on target |
| `session-librarian` | Organize sessions by prompt: find, rename, archive, prune |
| `teams-meeting-pipeline` | Teams summaries, job replay, Graph subscriptions |

### creative (16)

| Skill | Description |
|---|---|
| `manim-video` | Manim CE animations: 3Blue1Brown-style math/algo videos |
| `p5js` | p5.js sketches: generative art, shaders, interactive, 3D |
| `excalidraw` | Hand-drawn Excalidraw JSON diagrams (arch, flow, seq) |
| `architecture-diagram` | Dark-themed SVG architecture/cloud/infra diagrams as HTML |
| `comfyui` | Images, video, audio via diffusion workflows |
| `claude-design` | Design one-off HTML artifacts (landing, deck, prototype) |
| `popular-web-designs` | 54 real design systems (Stripe, Linear, Vercel) as HTML/CSS |
| `sketch` | Throwaway HTML mockups: 2–3 variants to compare |
| `design-md` | Author/validate/export Google's DESIGN.md token spec |
| `baoyu-infographic` | Infographics: 21 layouts × 21 styles |
| `ascii-art` | pyfiglet, cowsay, boxes, image-to-ascii |
| `ascii-video` | Convert video/audio to colored ASCII MP4/GIF |
| `humanizer` | Strip AI-isms, add real voice |
| `songwriting-and-ai-music` | Songwriting craft and Suno prompts |
| `pretext` | Creative browser demos with DOM-free text layout |
| `touchdesigner-mcp` | Control TouchDesigner via twozero MCP |

### software-development (11)

| Skill | Description |
|---|---|
| `test-driven-development` | Enforce RED-GREEN-REFACTOR, tests before code |
| `systematic-debugging` | 4-phase root-cause debugging |
| `plan` | Write a markdown plan to `.hermes/plans/`; no execution |
| `spike` | Throwaway experiments to validate an idea before build |
| `simplify-code` | Parallel 4-agent cleanup of recent changes |
| `requesting-code-review` | Pre-commit security scan, quality gates, auto-fix |
| `python-debugpy` | pdb REPL + debugpy remote (DAP) |
| `node-inspect-debugger` | Node `--inspect` + Chrome DevTools Protocol CLI |
| `dogfood` | Exploratory QA of web apps: bugs, evidence, reports |
| `hermes-agent-skill-authoring` | Author in-repo `SKILL.md`: frontmatter and structure |
| `inspecting-hermes-desktop-dom` | Read the live desktop DOM/CSS over CDP |

### github (7)

`github-issue-to-pr` (carry an issue to a verified PR with honest CI state) ·
`github-pr-workflow` (branch, commit, open, CI, merge) · `github-code-review`
(diffs, inline comments via `gh` or REST) · `github-issues` (create, triage,
label, assign) · `github-repo-management` (clone/create/fork, remotes, releases) ·
`codebase-inspection` (pygount: LOC, languages, ratios) · `github-auth` (HTTPS
tokens, SSH keys, `gh` login)

### research (7)

`arxiv` · `competitor-news-monitor` (named companies, material news, cited
digests) · `blogwatcher` (RSS/Atom via blogwatcher-cli) · `grounded-citations`
(ground answers in verifiable sources) · `research-paper-writing`
(NeurIPS/ICML/ICLR: design→submit) · `blocked-page-recovery`
(paywalled/WAF'd pages via fallbacks) · `llm-wiki` (Karpathy's interlinked
markdown KB)

### autonomous-ai-agents (6)

`hermes-agent` (use, configure, theme, extend, orchestrate Hermes itself — **18
reference pages, the most useful skill in the tree for contributors**) ·
`claude-code` · `codex` · `opencode` (delegate coding to other agent CLIs) ·
`computer-use` (drive the desktop without stealing focus) · `merge-reconciler`
(neutral third-party resolution of agent merge conflicts)

### mlops (5)

`evaluating-llms-harness` (lm-eval-harness: MMLU, GSM8K) · `weights-and-biases`
(experiments, sweeps, model registry) · `huggingface-hub` (`hf` CLI:
search/download/upload) · `llama-cpp` (local GGUF inference) ·
`serving-llms-vllm` (high-throughput serving, OpenAI API, quantization)

### apple (4) — macOS only

`apple-notes` (via `memo`) · `apple-reminders` (via `remindctl`) · `imessage`
(via `imsg`) · `findmy` (devices/AirTags via FindMy.app)

### media (3)

`youtube-content` (transcripts → summaries, threads, blogs) · `songsee` (audio
spectrograms/features: mel, chroma, MFCC) · `gif-search` (Tenor via curl + jq)

### email (2)

`email-inbox-triage` (prioritize threads, draft replies safely) · `himalaya`
(IMAP/SMTP from the terminal)

### Single-skill categories (5)

| Category | Skill | Description |
|---|---|---|
| `note-taking` | `obsidian` | Read, search, create, edit vault notes |
| `smart-home` | `openhue` | Philips Hue lights, scenes, rooms via OpenHue CLI |
| `social-media` | `xurl` | X/Twitter via `xurl`: raw post search, posting, DM, media |
| `devops` | `sdlc-review` | Review Kanban handoffs, route verified outcomes |
| `research`* | — | *(see research above)* |

*(`index-cache` is a build artifact directory, not a skill category.)*

---

## 8. Reading the catalog strategically

Three patterns worth noticing, because they tell you how to *use* this layer
rather than just what's in it.

**Most skills wrap a CLI, not an API.** `himalaya`, `openhue`, `xurl`, `ntn`,
`gws`, `memo`, `remindctl`, `hf`. This is a deliberate architectural choice: a
skill that teaches the model to drive an existing well-tested CLI through
`terminal` needs no new tool, no new plugin, no new dependency, and no new
credential path. It is rung 2 of the Footprint Ladder, and it is why the ladder
tells you to look there before writing code.

**Several skills are pure process, with no integration at all.**
`systematic-debugging`, `test-driven-development`, `plan`, `spike`,
`weekly-review-planning`, `grounded-citations`. These add zero capability and
change behavior substantially. They are the clearest demonstration that Layer 2 is
about competence, not access.

**A few skills are the recommended *replacement* for a tool.** The `x_search`
toolset description says it outright: use `x_search` for read-only public
discovery, but use the `xurl` **skill** for authenticated X API reads and account
actions. When a capability is broad and evolving, a skill over a CLI ages better
than a tool with a frozen schema.

---

## 9. Authoring your own

Use the in-tree skill for this: `hermes-agent-skill-authoring` documents the
frontmatter and structure conventions the maintainers actually enforce.

Minimum viable skill:

```bash
mkdir -p ~/.hermes/skills/personal/my-workflow
cat > ~/.hermes/skills/personal/my-workflow/SKILL.md <<'EOF'
---
name: my-workflow
description: "Deploy the staging app: build, migrate, smoke-test, notify."
version: 1.0.0
platforms: [linux, macos, windows]
---

# My Deploy Workflow

## Preconditions
- `STAGING_TOKEN` in env
- On branch `main`, clean tree

## Steps
1. `npm run build` — fails loudly on type errors; do not `--force`
2. `./scripts/migrate.sh --dry-run` first, always read the diff
3. Smoke test: `curl -sf https://staging.example.com/healthz`
4. On failure, roll back with `./scripts/rollback.sh` — do NOT retry the deploy

## Gotchas
- The migration is not reversible after step 3. Confirm before proceeding.
EOF
```

That's it — no registration, no restart of anything but the session, no code.
`skills_list` will pick it up.

### Checklist for a skill that actually gets used

- [ ] `description` names the concrete nouns a user would say ("Notion pages,
      databases"), not a category ("productivity helper")
- [ ] `platforms` set honestly — a macOS-only skill claiming `windows` is worse
      than useless
- [ ] `prerequisites.env_vars` declared, so the failure mode is "missing
      credential" not "mysterious error"
- [ ] Body includes **gotchas and failure modes**, not just the happy path. This
      is where skills earn their keep over the model's own knowledge.
- [ ] Version-dated claims where the upstream tool is moving
- [ ] No `!`cmd`` inline shell unless you also control the trust boundary

---

## 10. Where skills fit on the ladder

Rung 2: **CLI command + skill**. Second-cheapest thing you can build, right after
"extend existing code."

A skill costs nothing in the system prompt beyond one index line. It requires no
review of core code. It cannot break prompt caching. It has no schema to
maintain. If your feature can be expressed as "teach the model how to do X with
tools it already has," this is where it belongs — and reviewers will send you
back here from higher rungs.

Contrast the top rung: a new core tool costs schema tokens in every session on
every surface forever.

---

## 11. Self-check

<details>
<summary>1. Why aren't all 82 skills in the system prompt?</summary>

Prompt caching. A byte-stable system prompt is invariant #1; loading 82 full
skill bodies would cost tens of thousands of tokens on every turn of every
session. Skills use progressive disclosure — an index by default, full bodies on
`skill_view`.
</details>

<details>
<summary>2. You edited <code>skills/foo/SKILL.md</code> in your checkout and nothing changed. Why?</summary>

`skills/` in the repo is a **seed**. The runtime reads `~/.hermes/skills/`, which
was populated at install. Edit the runtime copy, or re-seed. And remember
`HERMES_HOME` is profile-scoped, so "the runtime copy" depends on the active
profile.
</details>

<details>
<summary>3. Why is <code>guard_agent_created</code> off by default but hub installs always scanned?</summary>

Threat model. Your agent writing a skill grants it nothing it didn't already have
— it can run the same code through `terminal` with no gate, so scanning adds
friction without security. A skill arriving from a stranger over the network is a
genuinely new trust boundary, so it is always scanned.
</details>

<details>
<summary>4. Why does <code>skill_manage</code> stage writes instead of prompting inline?</summary>

Size. A `SKILL.md` is too large to review in a chat bubble. Staging routes the
diff to a surface that can actually display it (`/skills diff <id>` in
CLI/dashboard/file). Compare `memory` writes, which *do* prompt inline — memory
entries are small enough to read in place.
</details>

<details>
<summary>5. What's the risk of <code>inline_shell: true</code>?</summary>

Any `` !`cmd` `` in any skill body executes on your host with no approval prompt,
before the agent even reads the skill. Combined with a community skill install,
that is arbitrary code execution granted to the skill author. Only enable it if
you wrote every skill directory yourself.
</details>

---

**Next:** [03-layer3-surfaces.md](03-layer3-surfaces.md) — the 22 platform
adapters, the gateway, and where a turn actually comes from.
