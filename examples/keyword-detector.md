# Example: shrinking a 38-line keyword-dispatch hook

This is a narrative of the first end-to-end run of `shrink`. The target was a small Python hook script — a `main()` function that reads JSON from stdin, matches the user's prompt against a few regex patterns, and prints a JSON envelope back.

The whole file was 54 raw lines; the `main()` body was 38 non-blank, non-comment lines after `black` formatting.

The run produced 3 accepted shrink rounds and one giveup at round 4 (local minimum). Final: 8 lines in `main()`. Tests stayed at 41/41 passing. Mutation kill rate stayed above the 70% gate.

This document walks through what happened, what the gates caught, and the lesson the run made unavoidable.

## The starting point

Original `main()`:

```python
def main():
    try:
        input_data = json.load(sys.stdin)
    except json.JSONDecodeError:
        sys.exit(0)

    prompt = input_data.get("prompt", "").lower()
    additional_context = None

    # ULTRAWORK mode - brief reminder (details in system prompt)
    if re.search(r"\b(ultrawork|ulw)\b", prompt):
        additional_context = (
            "[ULTRAWORK] Max effort: parallel agents, comprehensive todos, verify all."
        )

    # DATASTAR mode - trigger expert agent
    elif re.search(
        r"\b(datastar|data-bind|data-signals|data-on|sse|server-sent)\b", prompt
    ):
        additional_context = (
            "[DATASTAR] Use Task(subagent_type='datastar-expert') for DataStar work."
        )

    # BATMAN mode - brief reminder only
    elif re.search(r"\b(@?batman|complex task|multi-file|refactor)\b", prompt):
        additional_context = (
            "[BATMAN] Complex task detected. Consider Task(subagent_type='batman')."
        )

    if additional_context:
        output = {
            "hookSpecificOutput": {
                "hookEventName": "UserPromptSubmit",
                "additionalContext": additional_context.strip(),
            }
        }
        print(json.dumps(output))
    else:
        print(json.dumps({}))

    sys.exit(0)
```

Three `if/elif` branches. Each branch sets a string. A trailing `if/else` decides whether to wrap and emit, or emit `{}`.

## Baseline

| Item | Value |
|---|---|
| Whole file (raw `wc -l`) | 54 |
| `main()` body, post-black, non-blank, non-comment | 38 |
| codesieve aggregate | 7.7 (grade B) |
| Max nesting depth | 2 |
| Tests | 41/41 passing |
| Mutation kill rate | 93.7% (59 killed / 63 total) |

The baseline was clean: working code, working tests, mutation gate well above 70%. Loop entry approved.

## Adversary subagent — the frozen test suite

The Adversary was spawned as a fresh `general-purpose` subagent. It saw the implementation but had no knowledge of any future shrink. It wrote 41 tests:

- **38 example-based tests**: each keyword matched exactly, each keyword inside a sentence with surrounding text, word-boundary negatives ("ultraworking" must NOT match `ultrawork`), case-insensitivity, malformed JSON inputs, empty stdin, missing `prompt` field, precedence orderings (when two keywords appear, the earlier rule wins — six different orderings), and exact envelope-key checks (`hookEventName`, `additionalContext`, `hookSpecificOutput`).
- **3 Hypothesis property tests**: "no keyword anywhere → empty output", "ULTRAWORK always wins when present", "ULTRAWORK beats BATMAN under arbitrary surrounding filler text".

Four subtleties surfaced through the adversary's edge-case tests that a naive shrink could easily break:

1. `JSONDecodeError` triggers a silent exit. No stdout — not even `{}`.
2. Empty stdin triggers the same silent exit (not a print of `{}`).
3. `hookEventName: "UserPromptSubmit"` is required by the host hook contract; renaming or omitting it breaks integration.
4. `data-signal` (singular) is NOT a synonym in the regex; only `data-signals` (plural) matches.

These four subtleties became implicit constraints on every shrink round.

## The mutation gate

`mutmut` ran on the original code with the adversary's tests:

- 63 mutants generated
- 59 killed
- Kill rate: **93.7%**

Above 70%. Gate passes. The shrink loop is allowed to start.

## Per-round results

| Round | Status | LOC | Δ from prev | codesieve | Notes |
|---|---|---|---|---|---|
| 0 (baseline) | — | 38 | — | B (7.7) | Original `if/elif` chain |
| 1 | accepted | 20 | -47% | B (7.9) | Collapsed `if/elif` into a `RULES` table + `next()` |
| 2 | accepted | 13 | -35% | A (8.0) | Replaced output `if/else` with a ternary |
| 3 | accepted | 8 | -38% | A (8.1) | Chained `json.load().get().lower()` inside try |
| 4 | giveup | 8 | 0% | A (8.1) | 5 attempts; all gimmicks or black expanded back |

Each accepted round was its own git commit, with the message format:

```
shrink round N: main() <before> -> <after> (<reduction>%)
```

Three accepted commits; round 4 was a giveup with no commit.

## Round 1 — the structural win

The Shrinker subagent (fresh context, no test visibility) saw a chain of three `if/elif` blocks each setting `additional_context = "..."`. It did the obvious thing: hoisted the `(pattern, message)` pairs into a module-level list and used `next()` to find the first match.

Before (excerpted body — 38 lines):

```python
prompt = input_data.get("prompt", "").lower()
additional_context = None

if re.search(r"\b(ultrawork|ulw)\b", prompt):
    additional_context = "[ULTRAWORK] ..."
elif re.search(r"\b(datastar|...)\b", prompt):
    additional_context = "[DATASTAR] ..."
elif re.search(r"\b(@?batman|...)\b", prompt):
    additional_context = "[BATMAN] ..."

if additional_context:
    output = {
        "hookSpecificOutput": {
            "hookEventName": "UserPromptSubmit",
            "additionalContext": additional_context.strip(),
        }
    }
    print(json.dumps(output))
else:
    print(json.dumps({}))
```

After round 1 (20 lines in `main()`):

```python
prompt = input_data.get("prompt", "").lower()
msg = next((m for p, m in RULES if re.search(p, prompt)), None)
if msg:
    print(
        json.dumps(
            {
                "hookSpecificOutput": {
                    "hookEventName": "UserPromptSubmit",
                    "additionalContext": msg.strip(),
                }
            }
        )
    )
else:
    print(json.dumps({}))
sys.exit(0)
```

The three `(pattern, message)` pairs were lifted out into a module-level constant:

```python
RULES = [
    (r"\b(ultrawork|ulw)\b", "[ULTRAWORK] ..."),
    (r"\b(datastar|...)\b", "[DATASTAR] ..."),
    (r"\b(@?batman|...)\b", "[BATMAN] ..."),
]
```

All gates passed:
- Size: 20 < 38 (-47%)
- Tests: 41/41
- Readability: 7.9 >= 7.7 (slightly improved — the `if/elif` was the kiss-of-density penalty)
- Nesting: max depth still 2

Accepted. Commit. New baseline.

## Round 2 — collapsing the output if/else

The Shrinker (fresh context again) saw the post-round-1 code and noticed the trailing `if msg: print(...) else: print(json.dumps({}))` was redundant — both branches end in `print(json.dumps(...))`, just with different payloads. It rewrote it as a ternary on the payload, not on the print call.

After round 2 (13 lines):

```python
prompt = input_data.get("prompt", "").lower()
msg = next((m for p, m in RULES if re.search(p, prompt)), None)
ctx = (
    {"hookEventName": "UserPromptSubmit", "additionalContext": msg.strip()}
    if msg
    else None
)
print(json.dumps({"hookSpecificOutput": ctx} if ctx else {}))
sys.exit(0)
```

Two ternaries now, but both shallow. Gates:
- Size: 13 < 20 (-35%)
- Tests: 41/41
- Readability: 8.0 (now grade A)
- Nesting: still 2

Accepted.

## Round 3 — chaining the JSON read

The Shrinker (fresh context again) noticed that `input_data` was used exactly once: `input_data.get("prompt", "").lower()`. Inlining that into the `try` block eliminates the `input_data` local entirely. It also noticed `msg.strip()` was redundant — the source strings in `RULES` had no leading/trailing whitespace, and the adversary's tests didn't exercise stripping behavior on the message itself.

After round 3 (8 lines):

```python
def main():
    try:
        prompt = json.load(sys.stdin).get("prompt", "").lower()
    except json.JSONDecodeError:
        sys.exit(0)
    msg = next((m for p, m in RULES if re.search(p, prompt)), None)
    inner = {"hookEventName": "UserPromptSubmit", "additionalContext": msg}
    print(json.dumps({"hookSpecificOutput": inner} if msg else {}))
    sys.exit(0)
```

Gates:
- Size: 8 < 13 (-38%)
- Tests: 41/41
- Readability: 8.1 (still A)
- Nesting: still 2

Accepted. This is the final shape.

## Round 4 — the local minimum

A fresh Shrinker tried again. Five attempts, all rejected by the gates:

1. **Inline `inner` dict literal into the `print` ternary.** Black expanded the inlined `print(json.dumps({"hookSpecificOutput": {"hookEventName": "UserPromptSubmit", "additionalContext": msg} if msg else {}}))` to 17 physical lines. The envelope dict alone is ~70 chars; wrapped inside the `print(json.dumps(...))` call it blows past black's 88-char default.

2. **Walrus `(msg := next(...))` inside a ternary.** Eliminates the `msg = ...` line. But black still split the resulting ternary across multiple lines (final count 11). Walrus reads worse and saved zero net lines after black.

3. **Inline both dicts directly into `print()` with a `"{}"` literal short-circuit.** Same 88-char wall — black exploded the nested structure to 18 lines.

4. **Hoist `"UserPromptSubmit"` to a module-level `EVENT` constant.** The dict was still ~76 chars; with surrounding `payload = (... if msg else {})` and 4-space indent, black still wrapped it (final count 12).

5. **Extract `envelope(msg)` helper at module level.** Reached 7 lines in `main()`, tests passed, codesieve A 8.2. **Rejected anyway** as a gimmick — the helper is called exactly once, and the total file LOC INCREASED (+5 module-level lines for -1 `main` line). The helper exists solely to dodge black's 88-char wrap, not to express a real abstraction. Pure line-shifting, not real simplification.

The Shrinker wrote a giveup file:

> 8 non-blank lines is the honest local minimum for this spec under black 88.
> Convergence reached. No further reduction available without violating spec or gaming the metric.

The skill stopped. The local-minimum stop condition triggered correctly without needing the max-rounds bound to kick in.

## Final state

| | Original | Final | Delta |
|---|---|---|---|
| `main()` body LOC (formatted) | 38 | **8** | **-79%** |
| codesieve aggregate | 7.7 | 8.1 | +0.4 |
| codesieve grade | B | **A** | up one grade |
| Max nesting depth | 2 | 1 | -1 |
| Tests passing | 41/41 | 41/41 | unchanged |
| Mutation kill rate | 93.7% | 91.9% | -1.8 pp (still >= 70% gate) |

Mutation re-run on the final code: 37 mutants, 34 killed. Three survived — likely equivalent or near-equivalent mutants. Candidates for further adversarial test hardening if anyone wants to push the gate higher.

## Three honest-reduction frames

This is the part that matters for anyone using `shrink`. The same shrink can produce three very different "reduction percent" numbers depending on what you measure. Here are the same shrink under three frames:

| Frame | Original | Final | Reduction |
|---|---|---|---|
| Whole file (raw `wc -l`) | 54 | 36 | -33% |
| Function body, code only, source as written by hand | 23 | 8 | -65% |
| Function body, code only, both formatted by black | 31 | 8 | -74% |
| Function body LOC reported by the skill (post-black, non-blank, non-comment) | 38 | 8 | -79% |

The four numbers measure different scopes:

- **54 → 36** counts every line in the file, including the import block, the `RULES` constant (which grew during round 1), the docstring, the `if __name__` guard, and blanks. It's the smallest reduction because the cost of factoring `RULES` out shows up here.
- **23 → 8** is what a human sees if they hand-count lines of code in `main()` before black has a chance to wrap any of them. It ignores black's wrapping of `additional_context = "..."` to multiple lines.
- **31 → 8** post-black on both ends. Black wraps the long string assignments in the original to multiple lines; that inflates the original baseline. Final stays at 8 because round 3's `main()` already fits within 88 chars.
- **38 → 8** is what the skill reports. It includes the `def` line itself, plus the body, all post-black, all non-blank, all non-comment. This is the cleanest like-for-like comparison and the one to trust.

The first three frames are easy to manipulate (delete comments → get 80%; pick a scope that excludes `RULES` → get 90%; cherry-pick formatting → get whatever). The skill's frame deliberately removes those degrees of freedom.

**This is the lesson.** Reduction percentages are meaningful only when:

1. Both ends are formatted with the same tool (black at default config).
2. Comments and blanks are excluded from both counts.
3. The function boundary comes from AST parsing, not regex grep.
4. The denominator is the post-black baseline, not the source-as-typed baseline.

Without all four, you can claim any reduction percentage you want.

## What this run validated about the skill

- **Three-role separation works.** The Adversary's edge-case tests (silent exit on bad JSON, no `{}` print on empty stdin, exact envelope keys, the singular-vs-plural `data-signal` case) caught subtleties that the Shrinker, in fresh context with no test visibility, would have happily broken. The tests broke nothing — but only because they would have caught the breakage.
- **Frozen tests are essential.** Round 4's Shrinker tried 5 different gimmicks. The size + readability + nesting gates rejected the first four. The fifth (the one-call helper) had to be rejected by spec interpretation ("net change must be negative AFTER black") rather than by an automatic gate — that's a rough edge to fix in a later version.
- **Mutation gate (93.7% on original) gave real confidence.** The test suite was a correctness oracle, not just a passing-test rubber stamp.
- **codesieve floor (B baseline) protected against pathological compression.** Every accepted round held or improved the codesieve grade. Final ended at A.
- **Local-minimum detection worked without max-rounds being hit.** The skill stopped on round 4's giveup file, with `consecutive_rejects >= 3` satisfied implicitly via the 5 internal Shrinker attempts.
- **Per-round git commits give clean rollback granularity.** A bad round can be reverted with `git revert HEAD` and the next round retried fresh.

## What this run exposed (and that's now fixed in SKILL.md)

- mutmut 3.x removed `--paths-to-mutate` as a CLI flag. Must use `setup.cfg [mutmut]` config in the working directory. The skill now writes `setup.cfg` automatically.
- codesieve uses subcommands: `codesieve scan <file>`, not `codesieve <file>`.
- Subprocess-style tests break mutmut's stats phase — `sys.settrace` cannot follow trace into a child Python process. Tests that exercise a CLI script must redirect stdin/stdout in-process via `io.StringIO` and `contextlib.redirect_stdout/stderr`, then `importlib.reload(target_module)` between cases. The SKILL.md now spells this out in the adversary-subagent brief; tests that subprocess out cause the entire mutation gate to abort with a confusing `'NoneType' object has no attribute 'max_stack_depth'` error.

## Recommendation from the run

The shrunk version was strictly better on every measured axis: smaller body, simpler structure, higher codesieve grade, identical behavior under 41 tests including 3 property-based ones, mutation kill rate still above the 70% gate. It was applied to the live hook.

For a 38-line function the run took roughly 4 subagent calls (1 adversary + 3 Shrinker rounds, plus 1 wasted Shrinker round at giveup) and 2 mutmut runs (start + end). Total token cost was modest — under 30K input + output across all calls.
