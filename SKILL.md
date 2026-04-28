---
name: shrink
description: Iteratively minimize a Python function while preserving correctness via a frozen regression test suite. Generates adversarial tests upfront (Hypothesis + edge cases), gates the loop with mutation testing (mutmut), then runs fresh-context shrink rounds until tests fail, codesieve grade drops, or returns diminish. Each accepted round is its own git commit. Python 3.14 only on day one. Use when the user asks to "shrink", "minify a function", "reduce LOC", "code-golf with safety", "compress this function", "make it smaller without breaking it", or invokes `/shrink`.
domain: code-quality
type: workflow
frequency: occasional
---

# Shrink — Iterative Code Minimization with Regression Safety Net

Three subagent roles, never one model. Frozen test suite. Mutation testing as the gate. Git commit per accepted round.

## When to use

User asks to:
- "shrink this function"
- "minimize this code"
- "reduce LOC while keeping tests green"
- "code-golf this with safety"
- Make a function smaller without breaking behavior
- Invokes `/shrink <file>:<func>`

## When NOT to use

- General refactoring or quality cleanup → use a refactoring workflow
- Code quality grading only → use a static-analysis tool
- One-shot edits → just edit
- Performance optimization → smaller code can be slower; this skill has no perf budget
- Functions with no docstring or no spec → ask the user for intent first

## Architecture

Three roles. They do not share context.

| Role | Job | How |
|------|-----|-----|
| Implementer | Wrote the original code | (Already done — input to the skill) |
| Adversary | Writes the frozen test suite | Subagent. Sees code only to find missed edge cases. |
| Shrinker | Proposes smaller rewrites | Fresh subagent per round. Sees current code + spec, never the tests. |

**Why fresh contexts**: stops the Shrinker from learning patterns that game the tests across rounds. Each round is independent.

**Why frozen tests**: a yardstick that bends is no yardstick. Never regenerate mid-loop.

## Stack (Python 3.14)

All run via `uv run --with <pkg>` — no global installs. Verified working on Python 3.14.3 as of 2026-04-28.

**Required (PyPI):**
- `black` — formatter (the metric is LOC after black). Rewrites in place; no `--check` flag.
- `hypothesis` — property-based tests
- `pytest` — test runner
- `mutmut` (3.x) — mutation testing. **Config-driven only**, no CLI path flags. Requires `setup.cfg` with `[mutmut]` section in the working directory.
- `radon` — cyclomatic complexity for the readability floor (default).

**Optional:**
- `codesieve` — if installed and on PATH, used as the readability floor (A-F grade) instead of radon CC. Detected at runtime via `command -v codesieve`.

## State directory

Default: `${SHRINK_STATE_DIR:-.shrink-state}` resolved relative to the target file's git root. Override with the env var `SHRINK_STATE_DIR=<absolute path>`.

## Workflow

### 0. Preflight

```bash
git status --porcelain
```
- If output is non-empty → abort with: "shrink requires clean git state. Commit or stash first."
- Identify target: file path + function name. Ask user via `AskUserQuestion` if ambiguous.
- **Create a worktree.** Default:
  ```bash
  git worktree add ${SHRINK_STATE_DIR:-.shrink-state}/<func-name>/worktree -b claude/shrink/<func-name>
  ```
  All shrink work (file edits, commits) happens inside the worktree — the user's primary checkout stays on its current branch, untouched. Enables parallel shrinks on different functions simultaneously. Override with `--no-worktree` to fall back to `git checkout -b` on the primary checkout (rare; only if worktrees are unsupported in the repo).
- Record the starting branch from the **primary** checkout: `git -C <git-root> rev-parse --abbrev-ref HEAD` (run this before creating the worktree, from the repo root of the target file).
- The **target file path inside the worktree** is `${SHRINK_STATE_DIR:-.shrink-state}/<func-name>/worktree/<relative-path>`, where `<relative-path>` is the path of the target file relative to the git root. Use this worktree-internal path for all subsequent steps (black format, AST parse, codesieve, mutmut `paths_to_mutate`).
- **Parse model flags** from `$ARGUMENTS`: `--adversary-model=<name>` and `--shrinker-model=<name>`. Default both to `claude`. If either is non-`claude`, check that `OPENROUTER_API_KEY` is set; abort with "Set OPENROUTER_API_KEY to use non-Claude models." Print "Adversary model: <X>, Shrinker model: <Y>" for transparency.
- State dir: `${SHRINK_STATE_DIR:-.shrink-state}/<func-name>/`
- Record baseline:
  - Format the file: `uv run --with black black --quiet <file>` (rewrites in place — commit any whitespace-only diff first if present)
  - Count formatted-LOC of the function body using **non-blank, non-comment lines** via Python AST (slice the function body range, drop blank lines, drop lines whose lstrip() starts with `#`). Comment-only lines do NOT count toward the metric — otherwise the shrinker can win cheaply by deleting docstrings/comments, which is documentation removal, not code reduction.
  - **Readability baseline.** Detect available tool:
    - If `command -v codesieve` succeeds → run `codesieve scan <file>` and record the aggregate grade (A-F).
    - Else → run `uv run --with radon radon cc -s <file>` and record the function's cyclomatic complexity score (1-N, lower is simpler).
  - Save to `state.json` in state dir with: `{"original_loc": N, "current_loc": N, "readability_baseline": "<grade or CC>", "readability_tool": "codesieve|radon", "round": 0, "consecutive_rejects": 0}`.

### 1. Adversarial test generation

**Model routing**: If `--adversary-model` is not `claude`, route through an OpenRouter-compatible agent instead of spawning `general-purpose`. Pass the full adversary prompt below plus an explicit "write the file to `<state-dir>/tests.py`" instruction and the target model name. Verify the file exists after the agent returns.

If `--adversary-model` is `claude` (default), spawn `general-purpose` subagent with:

> Read `<file>`. Write a pytest test file at `<state-dir>/tests.py` that aggressively tests `<func>`. Requirements:
> - At least 10 example-based tests covering the obvious behavior, all argument types in the signature, and edge cases the original author may have missed (empty inputs, extremes, type variations, boundary conditions, off-by-one).
> - At least 2 Hypothesis property-based tests using `@given` strategies that match the function's expected input domain.
> - Tests must import the function from its actual path. No mocks of the function under test.
> - **Tests must run in-process** — no `subprocess.run([sys.executable, ...])`. mutmut's stats phase uses `sys.settrace` and cannot follow trace into a child process; subprocess tests cause the entire mutation gate to abort with `'NoneType' object has no attribute 'max_stack_depth'`. If the target reads `sys.stdin` or writes `sys.stdout`, redirect them in-process via `io.StringIO` + `contextlib.redirect_stdout/stderr`, and use `importlib.reload(target_module)` at the start of each test helper so mutmut-applied source mutations are picked up.
> - Do not invent invariants the docstring or behavior doesn't actually support.
> Return only when the file is written and `uv run --with hypothesis --with pytest pytest <state-dir>/tests.py -q` passes against the original.

Run that pytest invocation yourself after the agent returns. If it fails → abort. The adversary wrote bad tests; that's a problem to surface, not to paper over.

### 2. Mutation gate

mutmut 3.x is config-driven. Write `<state-dir>/setup.cfg`:

```ini
[mutmut]
paths_to_mutate=<absolute path to target file>
runner=uv run --with pytest --with hypothesis pytest <absolute path to state-dir>/tests.py
```

Then run from the state dir:

```bash
cd <state-dir>
uv run --with mutmut --with pytest --with hypothesis mutmut run
uv run --with mutmut mutmut results
```

mutmut output uses emoji counters: killed (good), survived (bad), no tests, timeout, suspicious.

Compute kill rate = killed / (killed + survived). Ignore skipped/timeout/no-test mutants in the denominator.

| Kill rate | Action |
|-----------|--------|
| >= 70% | Proceed |
| 50-70% | Warn user via `AskUserQuestion`. Tests are weak. Continue only if user confirms. |
| < 50% | Abort. Tests are too vacuous to gate the shrink. |

Save `mutation_baseline.txt` to state dir.

### 3. Shrink loop

Defaults: `max_rounds=6`, `min_reduction_pct=5`, `max_consecutive_rejects=3`.

For each round N from 1 to max_rounds:

**3a. Spawn Shrinker subagent** (fresh context — `general-purpose` for `--shrinker-model=claude`, OpenRouter-compatible agent otherwise with the same prompt + explicit Write-tool instruction):

> Read `<current code>`. Read `<original docstring or user-provided spec>`. Rewrite `<func>` to be **strictly smaller** when formatted with black. Constraints:
> - You may not change the public signature.
> - You may not delete the docstring (you may shorten it to one line if it's longer).
> - You may not increase the maximum nesting depth of the original.
> - You may not introduce new external dependencies.
> Return only the new function definition (def line through end of body), nothing else.

**3b. Apply candidate**: write the rewrite to `<state-dir>/round-N/candidate.py` and merge into a copy of the source file at `<state-dir>/round-N/source-with-candidate.py`.

**3c. Format + measure**: `uv run --with black black --quiet <candidate-source>`. Re-parse the file with Python's `ast` module, find the target function definition, count its body lines that are **non-blank AND not starting with `#`**. Comment-only lines do not count. (Don't grep — function boundaries from AST only.)

**3d. Acceptance gates** (all must pass):

| Gate | Condition | On fail |
|------|-----------|---------|
| Size | `formatted_loc < previous_loc` (non-blank, non-comment) | Reject (no reduction) |
| Tests | `pytest <state-dir>/tests.py` exits 0 | Reject (broke behavior) |
| Readability | If `readability_tool=codesieve`: grade >= baseline. If `readability_tool=radon`: candidate CC <= baseline CC. | Reject (too cryptic) |
| Nesting | Max nesting depth <= original's max | Reject (gamed by inlining) |

Record reason in `<state-dir>/round-N/result.json`.

**3e. If accepted**:
- Apply candidate to the target file inside the worktree (`<state-dir>/worktree/<relative-path>`).
- From `<state-dir>/worktree/`: `git add -A && git commit -m "shrink round N: <func> <baseline_loc> -> <new_loc> (<reduction>%)"`
- Update `state.json` with new baseline.
- Reset consecutive_rejects = 0.

**3f. If rejected**: increment consecutive_rejects.

**3g. Stop conditions**:
- Tests broke (any round): stop. Last accepted commit is the result.
- consecutive_rejects >= max_consecutive_rejects: stop.
- Last reduction < min_reduction_pct: stop (diminishing returns).
- N == max_rounds: stop.

### 4. Final report

Re-run mutmut on the final shrunk version (sanity check the suite still kills mutants on the new code).

Write `<state-dir>/summary.md`:
- Original LOC vs final LOC, % reduction (using the non-blank, non-comment metric)
- Per-round table: round, status (accepted/rejected/why), LOC, readability score, mutation kill rate
- Git log of shrink commits (`git log --oneline <starting-branch>..HEAD`)
- Mutation kill rate before/after

**Worktree handoff.** The shrink ran in a dedicated worktree — the user's primary checkout was never touched. Print these options to the terminal:

```
Worktree: <state-dir>/worktree/
Branch:   claude/shrink/<func-name>  (N commits ahead of <starting-branch>)
Primary checkout is still on: <starting-branch>

To keep all shrink rounds in history (each round as its own commit):
  git merge --no-ff claude/shrink/<func-name>

To squash everything into one clean commit:
  git merge --squash claude/shrink/<func-name> && git commit -m "shrink <func>: <orig-loc> -> <final-loc> (<reduction>%)"

To abandon the shrink entirely (removes worktree + branch):
  git worktree remove <state-dir>/worktree && git branch -D claude/shrink/<func-name>
```

(The merge commands above run from the primary checkout, already on `<starting-branch>`.)

Print a 5-line summary to terminal pointing at the report file.

## Safety constraints

- **Never modify a dirty working tree.** Preflight check #0.
- **Never regenerate tests mid-loop.** The yardstick is frozen.
- **Never accept a shrink that drops codesieve grade.** Readability floor.
- **Never run on a function with no docstring or no spec.** Ask the user for intent first.
- **Never skip the mutation gate.** Without it, this skill is dangerous.
- **Never auto-install dependencies into a project venv.** Use `uv run --with` for ephemeral envs.

## State files

`${SHRINK_STATE_DIR:-.shrink-state}/<func-name>/`

```
worktree/                       # git worktree — full repo copy, shrink branch checked out
state.json                      # baseline + current LOC, grade, round count
tests.py                        # frozen adversarial test suite
mutation_baseline.txt           # initial mutmut output
mutation_final.txt              # post-shrink mutmut output
round-1/
  candidate.py                  # the proposed rewrite
  source-with-candidate.py      # full file with candidate applied
  result.json                   # accept/reject + reason + metrics
round-2/
...
summary.md                      # final report
```

## Cost note

Each round runs: 1 Sonnet-class rewrite + 1 pytest + 1 codesieve (mutmut only at start and end). A 6-round shrink on a 200-LOC function is ~50K tokens total. Print a budget warning upfront if the function is over 300 LOC.

## Limitations (call these out at start of run)

- Python 3.14 only. JavaScript/TypeScript (Stryker + Vitest + prettier) is the next target.
- Performance regressions are NOT detected. Smaller code can be slower or use more memory. If perf-sensitive, the user should add a benchmark step manually.
- Tests written by the same model family share blind spots with the implementer. The mutation gate mitigates but does not eliminate.
- The skill cannot improve a function it cannot test. Pure side-effect functions (no return value, no observable state change without I/O) are out of scope.

## Failure modes and recovery

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Adversary's tests fail on original | Tests assume invariants the function doesn't actually have | Abort. Surface to user. Don't try to "fix" the tests. |
| Mutation kill rate < 50% | Tests are vacuous | Abort. Suggest user write tests manually first. |
| Every round rejected | Function may already be near-minimal | Stop after 3 rejects. Report "function is at or near a local minimum." |
| codesieve grade drops every round | Shrinker is gaming via density | Stop. Report needed. The metric/floor combo failed. |
| Tests break after acceptance | Test suite was incomplete despite mutation gate | `git revert` last shrink commit. Surface to user. |
