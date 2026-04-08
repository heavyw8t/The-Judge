# The Judge

> **Author:** [heavyw8t](https://github.com/heavyw8t)

A high-accuracy false-positive filter for AI-generated web3 security findings, runs each finding through a multi-stage adversarial validation pipeline and returns `VALID`, `INVALID`, or `DOWNGRADED` with evidence.

---

## What is The Judge?

Modern AI auditing tools generate large volumes of candidate findings, but a significant fraction are false positives, hallucinated invariants, or noise. The Judge is the jury and "executioner" for those findings.

Point The Judge at a finding (or a CSV batch of findings) and a target codebase. It will:

1. Verify the finding's location, code, and internal consistency.
2. Check whether the attack reduces to "trusted role abuse" (cap severity).
3. Test 3 generic invalidation reasons against the actual code in parallel.
4. Generate 3 issue-specific adversarial counter-arguments and test those in parallel.
5. If anything sticks, route the result through a neutral judge agent.
6. Return a final verdict + severity, with code-line evidence.

In benchmarking against ground-truth audit reports, The Judge significantly reduces false-positive rates from upstream AI auditors while preserving real findings.

---

## Setup

The Judge ships as a Claude Code skill plus a slash command. It works with Claude Code (CLI / IDE extensions) and any Cursor/IDE that talks to the Anthropic API through Claude Code.

### Manual install (3 commands)

```bash
# 1. Skill (methodology + invalidation library)
mkdir -p ~/.claude/agents/skills/judge/references
cp skill/judge/SKILL.md ~/.claude/agents/skills/judge/SKILL.md
cp skill/judge/references/invalidation-library.md ~/.claude/agents/skills/judge/references/invalidation-library.md

# 2. Slash command
mkdir -p ~/.claude/commands
cp commands/judge.md ~/.claude/commands/judge.md

# 3. Verify
ls ~/.claude/agents/skills/judge/ ~/.claude/commands/judge.md
```

That's it. Open Claude Code and type `/judge` to invoke.

### One-shot install via Claude / Cursor

Paste this prompt into Claude Code (or Cursor with Claude) and it will install The Judge for you:

> Install The Judge skill from the current working directory. Run these steps exactly:
> 1. `mkdir -p ~/.claude/agents/skills/judge/references ~/.claude/commands`
> 2. Copy `skill/judge/SKILL.md` to `~/.claude/agents/skills/judge/SKILL.md`
> 3. Copy `skill/judge/references/invalidation-library.md` to `~/.claude/agents/skills/judge/references/invalidation-library.md`
> 4. Copy `commands/judge.md` to `~/.claude/commands/judge.md`
> 5. Confirm all 3 files exist at their destinations and report back.
> Do not modify the file contents. Do not add anything else. Then tell me how to invoke `/judge`.

---

## Usage

### Single finding

```
/judge <paste full finding text here>
```

The Judge will prompt you (once) for:
- Protocol documentation (file, URL, free text, or skip)
- Role trust file (which roles are trusted vs semi-trusted)
- Audit scope (which files are in-scope)

Then it runs the per-issue pipeline and prints the verdict to console. A full trace is written to `validated_issues/ISSUE-{N}.md`.

### Batch mode (CSV)

```
/judge
```

When prompted, choose "CSV file of findings" and provide the path. The CSV must have columns `Number`, `Title`, `Summary`.

The Judge will process the batch in parallel waves of 5 issues at a time. Per-issue results are written to `validation_results/ISSUE-{id}.md`. Inter-issue observations accumulate (non-authoritatively) in `validation_notes.md`.

**Working directory**: run `/judge` from a directory that contains the target codebase (or a submodule pointing at it). The Judge reads source files via Glob/Grep/Read, so the code must be reachable from your CWD.

---

## Pipeline / Strategy

The Judge's accuracy comes from **adversarial breadth** + **hard early exits**. Each finding is challenged from multiple angles in parallel; only findings that survive every angle are marked VALID.

```
STEP 1 (sweep)    Verify file/function/line exists, description is internally consistent.
                  EARLY EXIT INVALID if location is wrong.
   ↓
STEP 2 (roles)    If the attack reduces to "trusted role acts maliciously", cap severity
                  at Low or early-exit INVALID.
   ↓
WAVE 1            Three things in parallel (one message):
                    • Step 1.5  external protocol research (WebSearch, cached per session)
                    • Step 3A   selector picks 3 best generic invalidation reasons from a library
                    • Step 4A   adversarial generator produces 3-5 issue-specific counter-arguments
   ↓
   filter         Drop Step 4A reasons that overlap with Step 3A selections.
   ↓
WAVE 2            Up to six opus checkers in parallel (one message):
                    • Step 3B  3 checkers verify the generic reasons against actual code
                    • Step 4B  3 checkers verify the adversarial reasons against actual code
                  Each checker reads source, traces logic, returns HOLDS / FAILS / UNCERTAIN
                  with code-line evidence.
   ↓
STEP 3C           If ≥2 Step 3B checkers HOLDS with solid evidence → EARLY EXIT INVALID
                  (Step 4B results discarded). Otherwise consume Step 4B and continue.
   ↓
STEP 4C (judge)   Only runs if any Step 4B checker HOLDS. A neutral opus judge reads the
                  proposed invalidation, the original finding, and the code, then renders
                  the final verdict: VALID / INVALID / DOWNGRADE.
   ↓
OUTPUT            Verdict + severity + evidence trace.
```

### Why this works

- **Adversarial duality** — every finding is tested by both a *generic* checker (covering common false-positive patterns from a library) and a *specific* generator that reads the actual code looking for issue-specific defenses, intentional design, MEV/atomicity constraints, implicit invariants, and economic edge cases.
- **Two-checker confirmation** — early exit requires at least 2 of 3 checkers to independently confirm the same invalidation, preventing isolated checker mistakes from killing real findings.
- **Anti-hallucination guard** — checker prompts forbid making confident claims about external protocol behavior from training data. Step 1.5 does live WebSearch verification of any external claim and caches results per session.
- **Neutral judge** — when checkers disagree or evidence is mixed, a separate impartial judge agent renders the final call without bias toward `VALID` or `INVALID`.
- **Parallel waves** — Wave 1 (3 agents) and Wave 2 (up to 6 agents) keep wall-clock time under ~7 minutes per finding even on the deepest path.
- **Cross-issue parallelism** — CSV mode batches issues into waves of 5, so 30 findings complete in ~25-35 minutes instead of ~3 hours sequential.

### What gets written where

| Mode | Output |
|------|--------|
| Single finding | Console summary + `validated_issues/ISSUE-{N}.md` (full trace) |
| CSV batch | `validation_results/ISSUE-{id}.md` per issue + `validation_notes.md` (accumulated observations) |
| Both | `.validation_context/` (cached docs/roles/scope summaries — reused across runs in the same CWD) |

---

## Requirements

- Claude Code installed (CLI, IDE extension, or desktop app)
- Anthropic API access with **Opus 4.x** model availability (The Judge uses opus for the checker/generator/judge agents and sonnet for the selector)
- A working directory containing the target codebase

---

## Repository Layout

```
The-Judge/
├── README.md
├── skill/
│   └── judge/
│       ├── SKILL.md                       ← the methodology, ~800 lines
│       └── references/
│           └── invalidation-library.md    ← generic invalidation reason catalog
└── commands/
    └── judge.md                           ← Claude Code slash command stub
```

---

## License

MIT License

Copyright (c) [2026] [Heeavyw8t]

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
