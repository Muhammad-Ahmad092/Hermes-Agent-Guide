# 07 · Contributing & PRs

**For:** getting your work merged — and understanding why PRs here get closed.
**Prerequisites:** [04 · Extending Hermes](04-extending-hermes.md),
[06 · Testing](06-testing.md)
**After this file you will know:** the whole path from idea to merge, what the
reviewers are grading, and the specific traps that close otherwise-good PRs.

Some context before the process. This repo has merged 23,405 commits from 2,765
authors, with PR numbers past #88,000 — so **most opened PRs never merge.** That
is not hostility; it is volume plus a specific bar. This file is about clearing
the bar.

The ratio tells you what is valued: **12,149 `fix` commits vs 3,046 `feat`**. The
bulk of what lands is bug fixes against real reported symptoms.

---

## 1. Before you write anything: search

Duplicates are the single most common waste here. The PR template's duplicate
check fires at *review* time — after you have already done the work.

```bash
gh search issues --repo NousResearch/hermes-agent "<your terms>"
gh search prs --repo NousResearch/hermes-agent --state all "<your terms>"
```

Note `--state all`: search **merged** PRs too, not just open ones.

Then search the **source**, because the issue tracker lags the code — many
requested features are already implemented in-tree:

```bash
rg "<capability>" --type py
```

If an open PR already addresses it, **improve or review that one** instead of
opening a competitor. For larger work, comment on the issue to signal you are
working on it.

---

## 2. Pick the rung, then verify the premise

Two gates before any code:

**Pick the rung.** [04 · Extending Hermes](04-extending-hermes.md). Most closed
feature PRs are not badly written — they are on the wrong rung. You should be
able to defend your choice in one sentence.

**Verify the premise.** The most common reason a *well-written* PR gets closed is
a wrong premise, or treating an intentional design as a gap.

> If you cannot point to the exact line where the bug manifests **and** show
> that your change alters that line's behavior, you have not verified the
> premise.

Read the original intent before "fixing" anything:

```bash
git log -p -S "<symbol>" -- path/to/file.py
```

Four real closure patterns, worth memorizing:

| Pattern | Real example |
|---|---|
| **Intentional design, not a gap** | Live config inheritance between profiles was closed — profile isolation *is* the design, and `--clone` already covers "start from my default" |
| **Premise doesn't hold** | A rate-limit "re-probe during cooldown" PR: the breaker only trips on a *confirmed-empty* bucket, so re-probing hammers a bucket already proven empty |
| **Premise doesn't hold (dead branch)** | A usage-accumulation fix whose new branch never executes at runtime — an earlier guard already popped the state it depended on |
| **The omission was deliberate** | Restoring "missing" `__init__.py` files made a test tree importable as a dotted package that shadowed the real plugin, deleting its `register()` at import time |
| **Overreached** | Scope creep past an agreed base, or reviving a direction maintainers deliberately closed — rejected even when the code works |

When unsure about intent, **ask**. Cheaper than shipping a fix that fights the
design.

---

## 3. Scope the PR

**One logical change per PR.** Do not mix a bug fix with a refactor with a new
feature. This is checked and it is the easiest thing to get wrong.

A good bug fix:

- Reproduces the symptom on **current `main`**
- Points at the exact line where it manifests
- Fixes the whole **bug class** — sibling call paths included — not just the one
  site the reporter hit
- Ships a test that fails before and passes after

That "whole bug class" expectation is real. If `_get_scoped_secret()` is wrong in
one adapter, reviewers will ask about the other fourteen.

---

## 4. Branch and commit

### Branch naming

```
fix/description        # bug fixes
feat/description       # new features
docs/description       # documentation
test/description       # tests
refactor/description   # restructuring
```

### Conventional Commits

```
<type>(<scope>): <description>
```

| Type | For |
|---|---|
| `fix` | Bug fixes |
| `feat` | New features |
| `docs` | Documentation |
| `test` | Tests |
| `refactor` | Restructuring, no behavior change |
| `chore` | Build, CI, dependencies |
| `perf` | Performance |

Common scopes: `cli`, `tui`, `desktop`, `gateway`, `tools`, `skills`, `agent`,
`models`, `kanban`, `plugins`, `dashboard`, `cron`, `install`, `security`,
`compression`, `delegation`, plus per-platform ones (`telegram`, `discord`,
`slack`, `whatsapp`).

Real examples from the log:

```
fix(cli): prevent crash in save_config_value when model is a string
feat(gateway): add WhatsApp multi-user session isolation
fix(security): prevent shell injection in sudo password piping
test(tools): add unit tests for file_operations
feat(delegation): raise subagent iteration cap default 50 -> 250 (+migration)
```

**Breaking changes are vanishingly rare** — exactly two `feat!` commits exist in
3,046 features (`feat(docker)!: replace tini with s6-overlay as PID 1`, May 2026;
`feat(runtime)!: require Node 26 across all installers`, August 2026). If you think you need one, that is a strong signal to find a
backward-compatible path, usually a config key that defaults to the old behavior.

---

## 5. Before you push

The template's checklist, in the order that catches the most problems:

1. **Run tests** — `scripts/run_tests.sh` (see [06](06-testing.md)). Not bare
   `pytest`.
2. **Test manually** — run `hermes` and exercise the path you changed. Use a
   `dev` profile so you don't wreck your own setup.
3. **Check cross-platform impact** — if you touched file I/O, process
   management, or terminal handling, think about macOS, Linux, native Windows,
   and WSL2.
4. **Update `cli-config.yaml.example`** if you added or changed config keys.
5. **Update `AGENTS.md` / `CONTRIBUTING.md`** if you changed architecture or
   workflows — and this guide, if you changed something it describes.
6. **Update tool descriptions/schemas** if you changed tool behavior.
7. **Check your branch is current with `main`.**

### That last one is not optional

> **Squash merges from stale branches silently revert recent fixes.**

A stale branch's version of an *unrelated* file overwrites recent fixes on `main`
when squashed. Before your PR is merged:

```bash
git fetch origin main
git rebase origin/main        # or reset --hard + reapply in a worktree
```

After merging, `git diff HEAD~1..HEAD` — unexpected deletions are a red flag.
There is a `history-check` CI lane for this, and the practice of "salvaging" old
PRs exists partly because of it.

---

## 6. The attribution gate

`contributor-check.yml` runs on every PR and **fails on unmapped contributor
emails.** It computes the merge base with `main` and looks for author emails it
does not recognize.

The mechanism is `contributors/emails/` — 651 files, one per known email — plus
`.mailmap` and a frozen legacy `AUTHOR_MAP` in `scripts/release.py`.

If the lane fails on your email, the fix is `scripts/add_contributor.py`. Do
**not** append to the frozen `AUTHOR_MAP` — the script refuses ambiguous cases
with exit 1 specifically so a typo can't silently reassign someone else's
commits.

Related: if you drafted work *using* Hermes, your commits may show "Hermes Agent"
as author. Fix that — for skills it is an explicit standard ("credit the human,
not the tool"), and it matters everywhere else too.

---

## 7. Write the PR description

The template (`.github/PULL_REQUEST_TEMPLATE.md`) asks for:

**What does this PR do?** The problem it solves, and *why this approach is the
right one*. That second half is where rung justification goes.

**Related issue.** `Fixes #NNNN`.

**Type of change.** One box: bug fix / feature / security / docs / tests /
refactor / new skill.

**Changes made.** Specific changes with file paths.

**How to test.** For bugs: reproduction steps **plus proof the fix works**. For
features: usage examples.

**Checklist.** Code (contributing guide read, Conventional Commits, duplicate
search done, only related changes, tests pass, tests added, platform tested) and
Documentation & Housekeeping (docs, `cli-config.yaml.example`, `AGENTS.md`,
cross-platform, tool schemas — each with an "or N/A").

**For new skills**, an extra block: broadly useful (if bundled), SKILL.md format
followed, no new external dependencies, tested end to end with
`hermes --toolsets skills -q "Use the X skill to do Y"`.

**Screenshots / logs** if applicable. For desktop or TUI changes, include them —
they materially speed up review.

### What a strong description contains that the template doesn't ask for

- The **rung** and one sentence on why (`"rung 2 — CLI command + skill, so it
  adds zero model-tool schema"`).
- The **exact line** where the bug manifests, for fixes.
- Evidence you checked **sibling call paths** for the same bug class.
- For anything touching message assembly: a note on **prompt-cache impact**.
- For anything surface-dependent: a note that the gate is **session-scoped**, and
  the test that proves it with the env var absent.

---

## 8. What happens after you open it

- CI runs the lanes in [06 § 9](06-testing.md#9-the-ci-lanes). `ci-review-comment`
  posts a live-updating comment and polls for up to 40 minutes.
- An automated triage sweeper may close PRs on exactly three grounds:
  `implemented_on_main`, `cannot_reproduce`, or `incoherent`. Taste-based "we
  don't want this / out of scope" closes are **not** automated — those stay with
  a human maintainer.
- A human reviews. Expect questions about rung, bug class, and cache impact.

If your PR goes stale, it may later be **salvaged** — rebuilt on current `main`
by a maintainer, usually titled `feat(x): … (salvage #NNNN)` or
`… (supersedes #NNNN)`. Dozens of merged features arrived that way. Your work
isn't necessarily lost if the PR itself goes cold.

---

## 9. The complete rejection list

Consolidated from `AGENTS.md`. Any one of these closes a PR **even when the code
is good**.

### Architecture and footprint

- **Speculative infrastructure** — hooks, callbacks, or extension points with no
  concrete consumer. (Not speculative if you state a real use case, even if the
  consumer ships separately.)
- **A new core tool when `terminal` + `read_file` already do the job**, or when a
  skill would. If the only barrier is file visibility on a remote backend, **fix
  the mount, not the toolset.**
- **Lazy-reading escape hatches on instructional tools** — no `offset`/`limit`
  on skills, prompts, or playbooks.
- **Plugins that touch core files.** Widen the generic plugin surface instead.
- **Third-party products in-tree** — observability backends, vendor SaaS
  connectors, analytics dashboards. Ship as a standalone plugin repo.
- **New in-tree memory providers.** That set is closed (May 2026).

### Configuration

- **New `HERMES_*` env vars for non-secret config.** `.env` is credentials only;
  behavior goes in `config.yaml`.

### Correctness and process

- **Cache-breaking mid-conversation.** Mutating past context, swapping toolsets,
  or rebuilding the system prompt mid-conversation.
- **"Fixes" that destroy the feature they secure.** A mitigation that kills the
  feature's purpose is the wrong mitigation. Read the original commit's intent
  before restricting behavior.
- **Dead code wired in without E2E proof.**
- **Change-detector tests** — see [06 § 4](06-testing.md#4-dont-write-change-detector-tests).
- **Tests that read source code** — banned outright.
- **Faking the host OS** instead of using platform markers.
- **Unpinned dependencies** — no upper bound gets rejected.

### Product

- **Outbound telemetry or usage attribution without an opt-in gate** (config gate
  + setup prompt + `hermes tools` toggle).

---

## 10. What gets merged readily

The other side of the ledger, in the maintainers' own priority order:

1. **Bug fixes** — crashes, incorrect behavior, data loss. Always top priority.
2. **Cross-platform compatibility** — macOS, Linux distros, native Windows, WSL2.
3. **Security hardening** — shell injection, prompt injection, path traversal,
   privilege escalation.
4. **Performance and robustness** — retries, error handling, graceful degradation.
5. **Reach at the edges** — new platform adapters, channels, providers, models,
   and desktop/TUI/dashboard features land routinely, **including large ones**.
6. **New skills** — broadly useful ones.
7. **New core tools** — rarely needed.
8. **Documentation** — fixes, clarifications, examples.

Read the balance correctly: **Hermes ships a lot.** The product surface expands
aggressively and on purpose. The restraint is aimed at the core agent and the
model tool schema — the one place where every addition is paid for on every API
call. Expansive at the edges, conservative at the waist.

---

## 11. Security-sensitive contributions

If your change touches shell execution, credential handling, path resolution, or
prompt-injection surfaces:

- Read `SECURITY.md` and the security section of `CONTRIBUTING.md` first.
- Existing protections you should not duplicate or accidentally weaken: command
  approval with a hardline blocklist for unrecoverable commands, secret
  redaction, promptware defense (shared threat patterns + memory load-time scan +
  tool-result delimiters), supply-chain advisory checking, an iron-proxy
  credential-injection firewall for sandboxes, and website blocklist enforcement
  for web/browser tools.
- **Report vulnerabilities privately**, not as a public issue or PR.
- Remember the rule: a mitigation that kills the feature's purpose is the wrong
  mitigation. Find the fix that preserves the feature.

---

## 12. The one-page version

```
1.  Search open AND merged PRs, issues, and the source.
2.  Pick the footprint rung. Defend it in one sentence.
3.  Verify the premise: point at the line; read the original intent.
4.  Branch: fix/… feat/… docs/…
5.  One logical change. Fix the whole bug class.
6.  Config → config.yaml. Secrets → .env. Never the reverse.
7.  Write a test that fails before and passes after.
8.  No change-detectors, no source-reading, no fake OS, no ~/.hermes writes.
9.  scripts/run_tests.sh — not bare pytest.
10. Exercise it by hand under a dev profile.
11. Rebase onto current main. Stale branches silently revert fixes.
12. Conventional Commits. Real name in the author field.
13. Fill the template: what, why this approach, how to test, platforms.
14. Expect questions about rung, bug class, and prompt-cache impact.
```

---

## Where to ask

- **Discord** — [discord.gg/NousResearch](https://discord.gg/NousResearch).
  `#plugins-skills-and-skins` is where standalone plugins get promoted.
- **GitHub Discussions** — design proposals and architecture questions.
- **GitHub Issues** — bugs. Include OS, Python version, `hermes version`, the
  full traceback, and reproduction steps.

Asking before building is cheaper than a rejected PR, and the maintainers say so
explicitly: *"When in doubt about intent, it is cheaper to ask than to ship a fix
that fights the design."*

By contributing you agree your contributions are licensed under Hermes Agent's
[MIT License](https://github.com/NousResearch/hermes-agent/blob/main/LICENSE).
