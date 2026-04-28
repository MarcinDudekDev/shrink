# shrink

A Claude Code skill for **iterative code minimization with regression safety**.

> Give it a Python function. It writes a frozen test suite, validates the suite via mutation testing, then runs a sequence of rewrite rounds — each in a fresh model context, gated on tests + readability + nesting + actual size reduction. Stops when no honest reduction is possible.

This is not code golf. The size metric ignores comments and blank lines so the model can't cheat by deleting documentation. Each accepted round is its own git commit, so any rejection is a one-line revert.

## Why

LLM refactoring loops typically converge on one of two failure modes:

1. **Test-passing junk** — the model rewrites code that passes the existing tests but breaks behavior the tests don't cover.
2. **Cryptic one-liners** — the model wins the LOC metric by collapsing readable code into dense ternary chains.

`shrink` defends against both:

- **Mutation testing as the loop entry gate.** Before any rewriting starts, `mutmut` runs on the original code with the auto-generated tests. If mutation kill rate is below 70%, the suite is too vacuous to gate a shrink loop and the skill aborts. This is the difference between "tests pass" and "tests are real."
- **Readability floor.** Each accepted shrink must hold codesieve grade (or radon CC, depending on what's installed) at or above the baseline. Density wins are rejected.
- **Three roles, one model family.** Implementer (the human, already done), Adversary (writes the frozen test suite, sees implementation), Shrinker (sees code + spec only, never the tests, fresh context per round). The Adversary doesn't see future shrinks; the Shrinker doesn't learn the test patterns across rounds.

## Install

```bash
# Required: Python 3.14 + uv
uv venv --python 3.14
# Skill uses `uv run --with <pkg>` so no global installs needed for the toolchain.
```

The skill itself is a `SKILL.md` and a slash-command file. Drop them into your Claude Code skills directory:

```
~/.claude/skills/shrink/SKILL.md
~/.claude/commands/shrink.md
```

Or, for a project-local skill, put them under `<your-project>/.claude/skills/shrink/SKILL.md`.

### Optional: codesieve

If `codesieve` is on your PATH, it's used as the readability floor (A-F grade). Otherwise the skill falls back to `radon cc` for a cyclomatic complexity floor. codesieve gives more nuanced grading (KISS, Nesting, Naming, ErrorHandling, TypeHints, etc.); radon is simpler and on PyPI.

## Usage

```
/shrink path/to/module.py:function_name
```

Optional flags:

```
/shrink src/parser.py:tokenize --rounds=4 --min-reduction=10
```

- `--rounds=N` — max shrink rounds (default 6)
- `--min-reduction=P` — stop when reduction falls below P% (default 5)
- `--no-mutation` — skip the mutation gate. **Dangerous.** Use only when the existing test suite is known-comprehensive (e.g., from coverage data).

## What it does, step by step

1. **Preflight.** Verifies clean git state. Records baseline: post-black non-blank-non-comment LOC of the function body, codesieve grade or radon CC, max nesting depth.
2. **Adversarial test generation.** A subagent writes a frozen test suite using Hypothesis property-based tests + edge cases. Tests must be in-process (not subprocess) so mutmut can trace coverage.
3. **Mutation gate.** `mutmut` runs the auto-generated tests against the original. If kill rate >= 70% → proceed. 50-70% → warn. < 50% → abort.
4. **Shrink loop.** Up to N rounds. Each round: fresh subagent, sees current code + spec only (never the tests). Proposes a rewrite. The skill applies it, formats with black, runs tests, checks codesieve/radon, checks nesting. Accept → git commit. Reject → restore + try next round.
5. **Stop conditions.** Tests fail | readability drops | nesting increases | reduction < min-reduction | rounds exhausted | 3 consecutive rejects.
6. **Final report.** Per-round table, mutation kill rate before and after, git log of accepted shrinks. Markdown summary in the state directory.

## State

Defaults to `.shrink-state/<func-name>/` in the target file's git root. Override with `SHRINK_STATE_DIR=<path>`.

```
.shrink-state/<func>/
├── state.json                # baseline + round counter
├── tests.py                  # frozen adversarial test suite
├── setup.cfg                 # mutmut config (paths_to_mutate, runner)
├── mutation_baseline.txt     # initial kill rate
├── mutation_final.txt        # post-shrink kill rate
├── round-1/                  # candidate.py, diff.patch, result.json
├── round-2/
├── ...
└── summary.md
```

## Limitations

- **Python 3.14 only on day one.** JavaScript (Stryker + Vitest + prettier + fast-check) is the next target language.
- **Performance regressions are NOT detected.** Smaller code can be slower or use more memory. If perf-sensitive, add a benchmark step manually.
- **Tests written by the same model family share blind spots with the implementer.** The mutation gate mitigates but does not eliminate this. For higher rigor, generate tests with a different model family.
- **Cannot improve a function it cannot test.** Pure side-effect functions with no return value and no observable state change are out of scope.
- **Single function per invocation.** File-level and class-level scopes are planned but not implemented. The current `<file>:<func>` parser doesn't handle them.

## Honest reduction numbers

The skill measures **non-blank, non-comment LOC after black formatting**. Comment deletion does not count as shrinkage. Black-induced expansion of the original baseline is normalized away by formatting both before measurement.

A first-run example on a 3-branch keyword-dispatch hook (see `examples/keyword-detector.md` for the full narrative):

| Metric | Original | Final | Reduction |
|--------|----------|-------|-----------|
| Whole file (raw `wc -l`) | 54 | 36 | -33% |
| Function body, code only, true source | 23 | 8 | -65% |
| Function body, code only, both post-black | 31 | 8 | -74% |

The middle column is what the skill reports.

## Why mutation testing is non-negotiable

If you skip the mutation gate, the entire workflow becomes "rewrite this until tests pass." That's exactly the failure mode that makes LLM refactoring loops produce subtly broken code. The mutation gate is what makes the test suite into a real correctness oracle, not just a syntax check.

If your test suite has a kill rate below 70%, fix the test suite first.

## Status

Functional. One end-to-end run to date. Not battle-tested. Treat it as alpha.

## License

MIT. See `LICENSE`. Attribution welcome.
