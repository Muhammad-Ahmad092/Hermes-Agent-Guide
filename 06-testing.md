# 06 · Testing

**For:** anyone about to open a PR. Tests are required for bug fixes and
strongly encouraged for features.
**After this file you will know:** how to run the suite the way CI does, where
your test belongs, and the four test patterns that are **banned outright** here.
**Next:** [07 · Contributing & PRs](07-contributing-prs.md)

There are 3,130 Python test files under `tests/`. This codebase has opinions about tests that are
stronger than most — and reviewers enforce them.

---

## 1. Always use `scripts/run_tests.sh`

**Do not call `pytest` directly.**

```bash
scripts/run_tests.sh                                   # full suite, CI parity
scripts/run_tests.sh tests/gateway/                    # one directory
scripts/run_tests.sh tests/agent/test_foo.py -k test_x # one test
scripts/run_tests.sh -v --tb=long                      # pytest flags pass through
```

The wrapper enforces hermetic parity with CI. Without it, a 16-core dev machine
with API keys in the environment diverges from CI in ways that have caused
multiple "works locally, fails in CI" incidents — **and the reverse**.

| | Bare `pytest` | `run_tests.sh` |
|---|---|---|
| Provider API keys | Whatever is in your env (auto-detects a pool) | All env vars unset except a specific few |
| `HOME` / `~/.hermes/` | Your real config and `auth.json` | Temp dir per test |
| Timezone | Your local TZ | `UTC` |
| Locale | Whatever is set | `C.UTF-8` |

The runner probes `.venv`, then `venv`, then
`$HOME/.hermes/hermes-agent/venv` — so worktrees sharing a venv with the main
checkout work.

> The PR template's checklist says `pytest tests/ -q`. `AGENTS.md` says always
> use the wrapper. **Use the wrapper** — it is the one that matches CI, and it is
> the stricter of the two.

### Subprocess-per-file isolation

Every test **file** runs in a freshly-spawned Python subprocess via
`scripts/run_tests_parallel.py` (not xdist; worker count auto-scales from CPU
count). Module-level dicts, sets, and ContextVars therefore cannot leak between
files.

Practical consequence: if your test only passes when run after another file, you
have found a real bug in your test, not a runner quirk.

### Flake policy

The runner auto-retries a failing test **file** once in a fresh subprocess
(`--file-retries`, default 1; `HERMES_TEST_FILE_RETRIES=0` disables).
Pass-on-retry counts as green but prints in a `⚠ FLAKY` summary with both
attempts' output.

**A FLAKY report is a bug to fix, not noise to ignore.** Timing-sensitive tests
must not assume a quiet runner:

- Loose wall-clock bounds (≥ 2s), not tight ones.
- Event-based synchronization, not sleeps.
- No `assert not _wait_until(...)` negative-timing races.

### JavaScript

```bash
cd ui-tui && npm test          # vitest
cd apps/desktop && npx vitest run src/lib/foo.test.ts
```

Workspace dependencies install at the **repo root**.

---

## 2. Where your test belongs

Mirror the source tree:

```
tests/agent/          tests/gateway/       tests/hermes_cli/
tests/tools/          tests/skills/        tests/plugins/
tests/providers/      tests/cron/          tests/hermes_state/
tests/acp/            tests/dashboard/     tests/e2e/
tests/integration/    tests/conformance/   tests/install/
```

### The CI-classifier trap

`scripts/ci/classify_changes.py` decides which jobs run based on which files
changed. So:

> A Python test that asserts about the contents of `package.json`,
> `package-lock.json`, `.ts`/`.tsx` source, or any other JS-side artifact **will
> not run** on a PR that only touches those files.

That means a regression can go green on your PR and red on `main`, where the
classifier fails open and runs everything.

**Rule:** any test that reads or asserts about `package.json`,
`package-lock.json`, `tsconfig.json`, or `.ts`/`.tsx`/`.js`/`.mjs`/`.cjs` source
belongs in the **vitest** suite, not in `tests/*.py`.

---

## 3. Don't fake the host OS

Hermes ships on Linux, macOS, and native Windows, and plenty of its behavior
genuinely differs per host. Those differences are tested **by running on the
host**, not by patching `sys.platform`.

```python
@pytest.mark.linux_only
@pytest.mark.macos_only
@pytest.mark.windows_only
```

**Use the marker, never a bare `skipif`.** `scripts/ci/list_os_marked_tests.py`
decides which files the macOS and Windows lanes import by grepping for the marker
*name*, then the lane filters with `-m <marker>`. So:

- `@pytest.mark.skipif(sys.platform != "win32")` skips on Linux **and** is never
  imported on the Windows lane. It runs on **no host at all**, silently.
- A file-local alias (`windows_only = pytest.mark.skipif(...)`) is worse: the
  grep matches the name so the file *is* listed, but `-m windows_only` deselects
  every test in it — the lane reports green over zero coverage.
- Don't `pytest.skip()` the non-host rows of a `@parametrize` over platforms.
  Split into one marked test per OS, or only the host's row ever runs.

**What can stay unmarked:**

- **Pure functions taking platform as data.** `hidden_windows_child_options(opts,
  is_windows=True)` is input→output, not a fake host. (Contrast: setting a
  module-level `IS_WINDOWS` flag then calling `windows_detach_flags()` *is* a
  fake.)
- **Declaration/packaging invariants.** "pyproject declares `tzdata` with a
  `sys_platform == 'win32'` marker" asserts about a file, not about runtime.

**The line:** if the test needs the interpreter to believe it is on another OS in
order to pass, it belongs on that OS. When one test body walks several platforms
in sequence, split it — keep the host-native arm on the Linux lane and move the
others into their own marked tests.

---

## 4. Don't write change-detector tests

A test is a **change-detector** if it fails whenever data that is *expected to
change* gets updated: model catalogs, config version numbers, enumeration counts,
hardcoded provider model lists. These add no behavioral coverage — they just
guarantee that routine source updates break CI and cost engineering time to
"fix."

**Do not write:**

```python
# catalog snapshot — breaks every model release
assert "gemini-2.5-pro" in _PROVIDER_MODELS["gemini"]
assert "MiniMax-M2.7" in models

# config version literal — breaks every schema bump
assert DEFAULT_CONFIG["_config_version"] == 21

# enumeration count — breaks every time a skill/provider is added
assert len(_PROVIDER_MODELS["huggingface"]) == 8
```

**Do write:**

```python
# behavior: does the catalog plumbing work at all?
assert "gemini" in _PROVIDER_MODELS
assert len(_PROVIDER_MODELS["gemini"]) >= 1

# behavior: does migration bump the user's version to current latest?
assert raw["_config_version"] == DEFAULT_CONFIG["_config_version"]

# invariant: no plan-only model leaks into the legacy list
assert not (set(moonshot_models) & coding_plan_only_models)

# invariant: every model in the catalog has a context-length entry
for m in _PROVIDER_MODELS["huggingface"]:
    assert m.lower() in DEFAULT_CONTEXT_LENGTHS_LOWER
```

**The rule:** if the test reads like a *snapshot of current data*, delete it. If
it reads like a *contract about how two pieces of data must relate*, keep it.

When your PR adds a provider or model and you want a test, assert the
relationship, not the names. Reviewers reject new change-detector tests; authors
should convert them into invariants before re-requesting review.

---

## 5. Never read source code in tests

**Banned outright.** A test that reads a source file's text tests *the shape of
the source code*, not its behavior. Any test that opens a `.py`, `.ts`, or `.tsx`
file is suspect.

Why it is actively harmful, not merely weak:

- **It fails in both wrong directions.** It passes when the implementation is
  subtly broken (the regex matches a call site that exists but is wired wrong),
  and fails when a correct refactor changes formatting, variable names, or
  control flow with identical runtime behavior.
- **It stops testing anything silently** the moment code moves, gets renamed, or
  a dependency reformats it — and it can't run against a built or bundled
  artifact at all.
- **It blocks refactors.** Reviewers watch "keeps a pattern intact" tests fail
  during pure structural cleanup and either hand-wave the failure (dangerous) or
  waste time updating regexes that add nothing.
- **It gives false confidence.** A green suite full of source-regex tests looks
  like coverage but has never once executed the path it claims to guard.

**Do not write:**

```ts
const source = fs.readFileSync(path.join(__dirname, 'main.ts'), 'utf8')

test('backend spawn hides the Windows console', () => {
  assert.match(source, /spawn\(\s*backend\.command,\s*backend\.args[\s\S]{0,300}hiddenWindowsChildOptions/)
})
```

**Do write — extract the logic and call it for real:**

```ts
// backend-spawn.ts
export function hiddenWindowsChildOptions(
  options: SpawnOptionsLike = {},
  isWindows = process.platform === 'win32',
) {
  if (!isWindows || 'windowsHide' in options) return options
  return { ...options, windowsHide: true }
}

// backend-spawn.test.ts
test('windowsHide defaults to true on Windows, is left alone elsewhere', () => {
  assert.equal(hiddenWindowsChildOptions({}, true).windowsHide, true)
  assert.equal(hiddenWindowsChildOptions({}, false).windowsHide, undefined)
  assert.equal(hiddenWindowsChildOptions({ windowsHide: false }, true).windowsHide, false)
})
```

If the logic lives inline in a god-file (`main.ts`, `cli.py`,
`gateway/run.py`) and extracting it feels disruptive — **that is the signal to do
the extraction**, not to regex around it.

---

## 6. Never write to `~/.hermes/`

The autouse `_isolate_hermes_home` fixture in `tests/conftest.py` redirects
`HERMES_HOME` to a temp dir. Never hardcode `~/.hermes/` paths in tests.

**Profile tests need more:** also mock `Path.home()`, so that
`_get_profiles_root()` and `_get_default_hermes_home()` resolve inside the temp
dir. The canonical pattern from `tests/hermes_cli/test_profiles.py`:

```python
@pytest.fixture
def profile_env(tmp_path, monkeypatch):
    home = tmp_path / ".hermes"
    home.mkdir()
    monkeypatch.setattr(Path, "home", lambda: tmp_path)
    monkeypatch.setenv("HERMES_HOME", str(home))
    return home
```

And when you mock `Path.home()` for any other reason, **also** set `HERMES_HOME`
— production code reads the env var through `get_hermes_home()`, not
`Path.home()`:

```python
with patch.object(Path, "home", return_value=tmp_path), \
     patch.dict(os.environ, {"HERMES_HOME": str(tmp_path / ".hermes")}):
    ...
```

---

## 7. Don't wire in dead code without E2E proof

Unused code that never shipped was dead for a reason. Before wiring an unused
module into a live path, E2E test the **real** resolution chain — actual imports,
not mocks — against a temp `HERMES_HOME`.

---

## 8. What a good test looks like here

Checklist for a test you are about to write:

- [ ] It executes the code path it claims to guard (no source regexes)
- [ ] It asserts a **contract or invariant**, not a snapshot of changeable data
- [ ] If it's OS-specific, it uses a marker and runs on that OS
- [ ] It doesn't touch the real `~/.hermes/`
- [ ] It doesn't need another test file to run first
- [ ] It has loose timing bounds, or none
- [ ] For a bug fix: it **fails before your change and passes after**
- [ ] If it's about JS artifacts, it lives in vitest

That last point about bug fixes is the strongest one. A bug-fix PR whose test
passes on unmodified `main` has not demonstrated anything.

---

## 9. The CI lanes

27 workflows in `.github/workflows/`. The ones you will interact with:

| Lane | What it does |
|---|---|
| `ci.yml`, `tests.yml` | Main Python suite |
| `tests-os.yml` | macOS and Windows lanes (marker-selected — see §3) |
| `js-tests.yml`, `js-autofix.yml` | vitest + TS |
| `lint.yml` | Python/TS lint |
| `contributor-check.yml` | **Attribution** — see [07 § 6](07-contributing-prs.md#6-the-attribution-gate) |
| `history-check.yml` | Guards against stale-branch reverts |
| `lockfile-diff.yml` | Posts a semantic `package-lock.json` diff on your PR |
| `osv-scanner.yml`, `supply-chain-audit.yml` | Dependency vulnerability scans |
| `docker.yml`, `docker-lint.yml` | Container build + hadolint |
| `install-e2e.yml`, `installer-tests.yml` | Installer end to end |
| `e2e-desktop.yml` | Desktop app E2E |
| `skills-index.yml`, `skills-index-freshness.yml` | Keeps the skills index current |
| `ci-review-comment.yml` | Live-updating review comment; polls up to 40 min |
| `uv-lockfile-check.yml` | `uv.lock` consistency |

Before pushing, the highest-value local check is `scripts/run_tests.sh` on the
directories you touched, plus `npm run typecheck` if you touched TypeScript.
