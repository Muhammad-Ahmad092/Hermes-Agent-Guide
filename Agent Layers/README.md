# The Four Layers — What Hermes Can Actually Do

> **Baseline:** `NousResearch/hermes-agent` @ `cf64ca20c` (committed 2026-08-17).
> Every count, key, and file path in this folder was read out of the tree at that
> commit. Numbers drift — the method for re-checking them is given in each file.

Everything Hermes can do resolves into four layers. They are not a marketing
grouping; they are four genuinely different extension mechanisms with different
registration paths, different config, different failure modes, and different
review bars if you want to contribute one upstream.

Read them in order the first time. After that, jump straight to the layer you're
working in.

| # | File | Layer | What it governs | Verified size |
|---|---|---|---|---|
| 1 | [01-layer1-toolsets.md](01-layer1-toolsets.md) | **Toolsets** | What the model is allowed to call this turn | 59 toolsets, 135 tool names, 121 modules |
| 2 | [02-layer2-skills.md](02-layer2-skills.md) | **Skills** | Procedural knowledge loaded on demand | 82 skills, 15 categories |
| 3 | [03-layer3-surfaces.md](03-layer3-surfaces.md) | **Surfaces** | Where a turn comes from and where output goes | 22 platform plugins + 9 in-tree adapters |
| 4 | [04-layer4-backends.md](04-layer4-backends.md) | **Backends** | Who executes the work underneath | 35 model providers, 8 memory, 8 environments |
| 5 | [05-composing-automations.md](05-composing-automations.md) | **Composition** | How the four combine into real automations | 16 cron blueprints, 4 starter jobs |

**Reading these alongside a checkout.** Every source path is written as inline
code (`tools/registry.py:213`) rather than a link, so this folder stays valid when
published on its own. Open a Hermes checkout beside it and the paths resolve
verbatim from the repo root. The two files worth having open are `AGENTS.md` (the
project's own rules and invariants) and `CONTRIBUTING.md`.

---

## The one-diagram version

```
                    ┌─────────────────────────────────────┐
   LAYER 3          │  SURFACE: CLI · TUI · desktop ·     │
   Where the turn   │  dashboard · ACP · API server ·     │
   comes from       │  22 messaging platforms · cron      │
                    └──────────────────┬──────────────────┘
                                       │ one turn
                    ┌──────────────────▼──────────────────┐
   LAYER 1          │  THE AGENT LOOP                     │
   What the model   │  system prompt + history + TOOLSETS │
   may call         │  → model → tool calls → repeat      │
                    └───────┬──────────────────┬──────────┘
                            │                  │
   LAYER 2      ┌───────────▼──────┐   ┌───────▼──────────┐   LAYER 4
   Procedural   │  SKILLS          │   │  BACKENDS        │   Who executes
   knowledge    │  82 packs,       │   │  model provider  │
                │  loaded on       │   │  memory store    │
                │  demand          │   │  shell env       │
                └──────────────────┘   └──────────────────┘
```

**The load-bearing distinction people get wrong:** Layer 1 is *capability* — may
the model call `terminal` at all. Layer 2 is *competence* — does it know the
right way to use `terminal` for a Notion migration. Layer 4 is *execution* —
does `terminal` run on your laptop or in a throwaway Docker container. You can
change any one of the three without touching the other two, and that separation
is deliberate.

---

## Why this split exists

Two invariants in `AGENTS.md` explain almost every design
decision in the tree:

1. **Per-conversation prompt caching is sacred.** The system prompt must stay
   byte-stable for the life of a session. This is why skills load *on demand*
   instead of all 82 being pasted into the prompt, and why mid-session memory
   writes hit disk but do **not** change the prompt until the next session
   (`tools/memory_tool.py`).

2. **The core is a narrow waist; capability lives at the edges.** The agent loop
   knows almost nothing about Slack, Notion, Modal, or Grok. Every one of those
   arrives through one of the four layers as a plugin, a skill, an adapter, or a
   provider.

If you internalize only one thing: **adding a feature almost never means editing
the loop.** It means picking the right layer. That decision procedure is the
Footprint Ladder, documented at `AGENTS.md:182` ("The Footprint Ladder")
and cross-referenced from each file here.

---

## Verified headline counts

Reproduce all of these from a clean checkout:

```bash
ls tools/*.py | wc -l                    # 121  tool modules
find skills -name SKILL.md | wc -l       #  82  skills
ls -d plugins/platforms/*/ | wc -l       #  22  platform plugins
ls -d plugins/model-providers/*/ | wc -l #  35  model providers
ls -d plugins/memory/*/ | wc -l          #   8  memory backends
ls -d plugins/web/*/ | wc -l             #   8  web search backends
ls -d plugins/image_gen/*/ | wc -l       #   7  image generation backends
ls -d plugins/video_gen/*/ | wc -l       #   3  video generation backends
```

For the toolset count, the registry is a Python dict rather than a directory:

```bash
python -c "import re; s=open('toolsets.py',encoding='utf-8').read(); \
b=s[s.index('TOOLSETS = {'):]; print(len(re.findall(r'^    \"([a-z_0-9-]+)\":', b, re.M)))"
# 59
```

---

## What is *not* in these four layers

Worth naming explicitly so you don't go looking:

- **Slash commands** — a surface concern, not a capability. Some live in the
  Python `COMMAND_REGISTRY`, some are TUI-side only (`/rewind` is TUI-side).
  See `AGENTS.md`.
- **Hooks** — 27 named lifecycle hooks that let a plugin observe or veto rather
  than add capability. Covered in `CONTRIBUTING.md`.
- **Kanban and delegation** — these *use* Layer 1 toolsets (`kanban`,
  `delegation`) but they are orchestration patterns. Treated in
  [05-composing-automations.md](05-composing-automations.md).
- **MCP** — the deliberate escape hatch that sidesteps all four layers. Also in
  file 05.

---

## A note on trusting this folder

These files were written by reading source, not documentation. Where the
repository's own docs disagree with the code, **the code wins and the drift is
recorded inline**. Five such cases are known at this baseline; each is flagged
where it appears with a `⚠ Doc drift` marker.

That is not a criticism of the project. A tree taking ~50 merged features a week
will always have prose trailing behind it. It is a warning about method: when you
plan work against this codebase, verify the number in the source before you
depend on it.
