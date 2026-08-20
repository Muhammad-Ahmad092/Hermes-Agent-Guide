# 03 · Dev Setup

**For:** getting from a clean machine to a running agent you can modify.
**Prerequisites:** [01 · What Hermes Is](01-what-hermes-is.md)
**After this file you will know:** how to install for development, run every
surface, put settings in the right place, use profiles to avoid wrecking your
own agent, and read logs when something breaks.
**Next:** [04 · Extending Hermes](04-extending-hermes.md)

---

## 1. Prerequisites

| Requirement | Notes |
|---|---|
| **Git** | With `git-lfs` installed |
| **Python 3.11–3.13** | `.python-version` pins 3.11; `uv` installs it if missing |
| **uv** | The package manager Hermes bundles and expects |
| **Node.js 20+** | Needed for the TUI, desktop, dashboard, browser tools, WhatsApp bridge. `.nvmrc` says 26 |

---

## 2. Install (recommended path)

Use the **same installer end users run**, then develop inside the repo it clones.
This keeps your environment on the layout the CLI, updater, lazy dependency
installer, gateway, and docs all assume.

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
cd "${HERMES_HOME:-$HOME/.hermes}/hermes-agent"

# dev + test extras on top of the standard install
uv pip install -e ".[all,dev]"

# optional: docs site + JS workspace deps
npm install
```

Windows (native, PowerShell):

```powershell
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

The installer handles uv, Python, Node, ripgrep, ffmpeg, and — on Windows — a
portable Git Bash (MinGit under `%LOCALAPPDATA%\hermes\git`, no admin, isolated
from any system Git).

> **If your antivirus quarantines `uv.exe`** from the Hermes `bin` folder, it is
> a known false positive on unsigned Rust binaries. `README.md` has the
> attestation-verification snippet and the whitelisting steps. Whitelist the
> *folder*, not the hash — `uv` updates.

### Manual clone (only if you deliberately don't want the managed layout)

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# Create the venv OUTSIDE the source tree
uv venv ~/.hermes/venvs/hermes-dev --python 3.11
export VIRTUAL_ENV="$HOME/.hermes/venvs/hermes-dev"
export PATH="$VIRTUAL_ENV/bin:$PATH"

uv pip install -e ".[all,dev]"
npm install   # optional
```

**Why outside the tree:** a venv inside the directory the agent operates from can
be destroyed by a relative-path command the agent runs against its own checkout
(`rm -rf venv`, `uv venv venv`). That silently kills the running runtime
mid-session. Keeping it outside means no relative path from the workspace
resolves to it.

---

## 3. Configure

```bash
mkdir -p ~/.hermes/{cron,sessions,logs,memories,skills}
cp cli-config.yaml.example ~/.hermes/config.yaml
touch ~/.hermes/.env

# at minimum, one provider key
echo "OPENROUTER_API_KEY=***" >> ~/.hermes/.env
```

Then:

```bash
hermes doctor              # diagnose the install
hermes chat -q "Hello"     # smoke test
```

If you used the manual clone, run `./hermes` from the checkout or symlink its
venv entry point onto your PATH.

---

## 4. Run each surface

```bash
hermes                 # classic CLI (prompt_toolkit)
hermes --tui           # Ink/React TUI  (or HERMES_TUI=1)
hermes --cli           # force classic CLI when tui is your default
hermes -z "prompt"     # one-shot, no session UI
hermes gateway         # messaging gateway (Telegram, Discord, …)
hermes dashboard       # browser dashboard (serves the SPA, embeds the TUI)
hermes serve           # headless backend only — no web UI (what desktop spawns)
hermes acp             # ACP server for VS Code / Zed / JetBrains
```

Useful configuration entry points:

```bash
hermes model           # pick provider + model
hermes tools           # curses UI for toolsets
hermes setup           # full setup wizard
hermes config get|set  # individual values
hermes skills          # enable/disable skills and categories
hermes doctor          # diagnostics
hermes update          # update the install
```

### TUI development

```bash
cd ui-tui
npm install       # first time
npm run dev       # watch mode (rebuilds hermes-ink + tsx --watch)
npm start         # production
npm run build     # full build
npm run typecheck # tsc --noEmit
npm run lint
npm run fmt
npm test          # vitest
```

### Desktop development

Workspace dependencies install at the **repo root**, not in `apps/desktop`.
Read `apps/desktop/AGENTS.md` before editing. Example scoped test run:

```bash
cd apps/desktop
npx vitest run src/lib/desktop-slash-commands.test.ts
```

---

## 5. Where settings go — the rule that gets PRs rejected

**`.env` is for secrets only.** API keys, tokens, passwords. Nothing else.

**`config.yaml` is for behavior.** Timeouts, thresholds, feature flags, display
preferences, paths. All of it.

A PR that tells users to "set `HERMES_FOO=1` in your `.env`" for a non-secret is
rejected. If internal code needs an env-var mirror for back-compat, bridge it
*from* `config.yaml` in code — the way `gateway_timeout` and
`terminal.cwd → TERMINAL_CWD` do — and point the docs at the config key.

### config.yaml sections

`model` · `agent` · `terminal` · `compression` · `display` · `stt` · `tts` ·
`memory` · `security` · `delegation` · `smart_model_routing` · `checkpoints` ·
`auxiliary` · `curator` · `skills` · `gateway` · `logging` · `cron` · `profiles` ·
`plugins` · `honcho` (non-exhaustive)

`auxiliary` deserves a note: it holds per-task overrides for side-LLM work
(curator, vision, embedding, title generation, session_search). Each task can pin
its own provider, model, base_url, max_tokens, and reasoning_effort. Resolution
order lives in `agent/auxiliary_client.py::_resolve_auto`.

### The three config loaders — know which one you're in

| Loader | Used by | Location |
|---|---|---|
| `load_cli_config()` | CLI mode | `cli.py` — CLI defaults + user YAML |
| `load_config()` | `hermes tools`, `hermes setup`, most subcommands | `hermes_cli/config.py` — merges `DEFAULT_CONFIG` (defined in `config_defaults.py`) + user YAML |
| Direct YAML load | Gateway runtime | `gateway/run.py`, `gateway/config.py` — raw user YAML |

**Debugging heuristic:** if the CLI sees your new key but the gateway doesn't (or
vice versa), you are on the wrong loader. Check `DEFAULT_CONFIG` coverage first.

### Working directory

- **CLI** — the process's current directory (`os.getcwd()`).
- **Messaging** — `terminal.cwd` from `config.yaml`, bridged to `TERMINAL_CWD`
  for child tools.
- `MESSAGING_CWD` **has been removed**. `TERMINAL_CWD` in `.env` is also
  deprecated — the canonical setting is `terminal.cwd` in `config.yaml`. The
  loader warns if it finds the old forms.

---

## 6. Profiles — develop without wrecking your own agent

A profile is a fully isolated instance: its own config, keys, memory, sessions,
skills, and gateway.

```bash
hermes profile list
hermes profile create dev
hermes -p dev                    # run under the dev profile
hermes -p dev profile list       # profile ops still see every profile
```

Under a profile, `HERMES_HOME` becomes `~/.hermes/profiles/dev`. This is the
cheapest way to test destructive changes without touching your real setup.

There is also a dev sandbox helper:

```bash
scripts/dev-sandbox.sh           # see --help; supports --from DIR to seed HERMES_HOME
```

For the code rules that make profiles work (and the ways to break them), see
[02 § 8](02-architecture.md#8-profiles). The short version: never hardcode
`~/.hermes`; use `get_hermes_home()` for paths and `display_hermes_home()` for
messages.

---

## 7. Logs and debugging

Logs live in `$HERMES_HOME/logs/` — profile-aware:

| File | Contents |
|---|---|
| `agent.log` | INFO and above |
| `errors.log` | WARNING and above |
| `gateway.log` | Gateway-specific, when running |
| `desktop.log` | Desktop app, included in debug shares |

```bash
hermes logs                      # browse
hermes logs --follow             # tail
hermes logs --level WARNING      # filter by level
hermes logs --session <id>       # filter by session
hermes debug share               # upload a debug report to a pastebin
hermes dump                      # copy-pasteable setup summary
hermes prompt-size               # diagnose what's filling the context window
```

Setup lives in `hermes_logging.py` (`setup_logging()`).

### Debugging playbook

| Symptom | First thing to check |
|---|---|
| Tool exists but the model never calls it | Is its name in a toolset in `toolsets.py`? Discovery ≠ exposure. |
| Tool disappears on one surface only | The toolset the platform inherits, and the tool's `check_fn` |
| Config key ignored | Which loader that code path uses (§5); is it in `DEFAULT_CONFIG`? |
| Works in CLI, not in gateway | The two message guards in `gateway/run.py` and `gateway/platforms/base.py` |
| Works for default profile, fails for a second | Profile-scoped env reads — see the fail-closed rule |
| Plugin not discovered | Was `model_tools.py` imported? Call `discover_plugins()` explicitly |
| Cost exploded on a long chat | Cache invalidation. Something rewrote the prefix. `hermes prompt-size` |
| Tests pass locally, fail in CI | You ran bare `pytest`. Use `scripts/run_tests.sh` — see [06](06-testing.md) |

Runtime debug switches:

```bash
HERMES_PLUGINS_DEBUG=1 hermes      # plugin discovery logs
/verbose                           # toggle debug output at runtime
/statusbar                         # toggle the context bar
```

---

## 8. Dependency policy (you will hit this)

Every dependency needs an **upper bound**. This came from a real supply-chain
incident (the litellm compromise, PRs #2796/#2810) and was reinforced after the
Mini Shai-Hulud worm campaign in May 2026.

| Source | Treatment | Example |
|---|---|---|
| PyPI package | `>=floor,<next_major` | `"httpx>=0.28.1,<1"` |
| Git URL | Full commit SHA | `git+https://...@<40-char-sha>` |
| GitHub Action | SHA + version comment | `uses: actions/checkout@<sha>  # v4` |
| CI-only pip | Exact | `pyyaml==6.0.2` |

Adding a dependency:

1. Post-1.0 → `>=current,<next_major` (e.g. `>=1.5.0,<2`).
2. Pre-1.0 → `>=current,<0.(minor+2)` (e.g. `>=0.29,<0.32`).
3. Never a bare `>=X.Y.Z`. CI and reviewers reject it.
4. Run `uv lock` to regenerate `uv.lock` with hashes.

```python
# ✅ accepted
"aiosqlite>=0.20,<0.23"

# ❌ no upper bound
"some-package>=1.2.3"

# ❌ too tight — blocks legitimate patches
"some-package==1.2.3"

# ❌ too loose for pre-1.0 — allows 80 minor versions
"some-package>=0.20,<1"
```

There is a `lockfile-diff` CI lane that posts a semantic `package-lock.json`
diff on your PR, plus `osv-scanner` and `supply-chain-audit` lanes.

---

## 9. Cross-platform reality

Hermes supports Linux, macOS, and **native** Windows (not just WSL2). Behavior
genuinely differs per host, and CI has per-OS lanes.

Things that bite:

- `.gitattributes` enforces LF on `*.sh` and `Dockerfile` — a CRLF checkout
  breaks `exec` in container entrypoints with "no such file or directory".
- POSIX-only primitives (`fcntl`, `termios`, `os.setsid`, `/proc`, hardcoded
  `/tmp`, `signal.SIGKILL`, bash heredocs, `osascript`, `apt`, `systemctl`)
  need either a cross-platform fix or explicit platform gating.
- Prefer `tempfile.gettempdir()`, `pathlib.Path`, `psutil.pid_exists()`, and
  Python-level filtering over shelling out.
- `scripts/check-windows-footguns.py` exists for a reason — run it if you touch
  process or path handling.
- The dashboard's `/chat` tab uses `ptyprocess` on POSIX and ConPTY
  (`win_pty_bridge`) on Windows.

**Do not fake the host OS in tests.** See [06 · Testing](06-testing.md#3-dont-fake-the-host-os).

---

## 10. A first change, end to end

A five-minute loop to prove your environment works:

```bash
hermes profile create dev            # 1. isolate
git checkout -b docs/first-change    # 2. branch

# 3. make a tiny change — e.g. add an alias to an existing CommandDef
#    in hermes_cli/commands.py

scripts/run_tests.sh tests/hermes_cli/  # 4. test the area you touched
hermes -p dev                            # 5. exercise it by hand
```

If all five steps work, you are set up. Move on to
[04 · Extending Hermes](04-extending-hermes.md) to decide *where* your real
feature belongs before you write it.
