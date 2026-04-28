---
description: Iteratively shrink a Python function while preserving correctness via a frozen regression test suite and mutation testing. Each accepted round is its own git commit.
---

Run the shrink workflow on `$ARGUMENTS`.

`$ARGUMENTS` should be `<file path>:<function name>`. Examples:
- `/shrink src/parser.py:tokenize`
- `/shrink utils/math_helpers.py:compute_stats`

If `$ARGUMENTS` is empty or doesn't match `path:func`, ask the user via `AskUserQuestion` for the file and function.

Follow the `shrink` SKILL.md exactly. Do not skip:

1. Preflight: clean git state, identify function, record baseline LOC + codesieve/radon grade
2. Spawn adversarial test generator (general-purpose subagent)
3. Mutation gate (`mutmut` via `uv run --with`); abort if kill rate < 50%, warn if 50-70%
4. Shrink loop: max 6 rounds, fresh subagent each round, gates on size/tests/readability/nesting
5. Git commit each accepted round with format: `shrink round N: <func> <before> -> <after> (<reduction>%)`
6. Final report at `<state-dir>/summary.md`, plus 5-line terminal summary

Defaults are in the SKILL. The user may override via flags in `$ARGUMENTS`:
- `--rounds=N` (default 6)
- `--min-reduction=P` (default 5, percent)
- `--no-mutation` (skip mutation gate — DANGEROUS, warn the user)
- `--no-worktree` (fall back to `git checkout -b` instead of `git worktree add`; use only if worktrees unsupported)

State directory: `${SHRINK_STATE_DIR:-.shrink-state}/<func-name>/`
