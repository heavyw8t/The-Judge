---
name: judge
description: "The Judge — filter AI-generated web3 security findings for false positives via a multi-step adversarial pipeline. Accepts a single issue as text or a CSV file of findings. Usage: /judge <issue text> OR /judge (then select CSV mode)"
---

# Validate-Issue — Web3 Security Issue Validation Pipeline

> **Purpose**: Determine whether a web3 security issue report is valid, invalid, or should be severity-adjusted through a 4-step adversarial process with early exits.
> **Models**: opus for checkers/generator/judge, sonnet for selector.

---

## PIPELINE OVERVIEW

```
MODE SELECTION → PRE-STEP (once)
  ↓
[CSV mode]    BATCH PARALLEL: process ISSUE_LIST in batches of BATCH_SIZE (default 5);
              each issue runs as an independent subagent.
[Single mode] one issue, sequential.

Per-issue pipeline:
  STEP 1 (sweep) → STEP 2 (roles)
    ↓
  WAVE 1:  [STEP 1.5  ∥  STEP 3A  ∥  STEP 4A]            (research + selector + generator)
    ↓
  Orchestrator filters Step 4A reasons against Step 3A selections (drop duplicates)
    ↓
  WAVE 2:  [STEP 3B (3 opus) + STEP 4B (3 opus)]         (6 parallel checkers, ONE message)
    ↓
  STEP 3C: orchestrator confirms — EARLY EXIT if ≥2 Step 3B checkers HOLDS
    ↓
  STEP 4C: judge — only if no 3C exit AND any 4B checker HOLDS
    ↓
  OUTPUT
```

Each step can EARLY EXIT with a verdict at Steps 1, 2, or 3C.

---

## MODE SELECTION (runs before everything else)

Use `AskUserQuestion` to ask:
> "How would you like to provide issue(s) to validate?"

Options:
- "Single issue text (print result to console)"
- "CSV file of findings (write compact result files)"

**If "Single issue text"**:
- Set `INPUT_MODE = single`, `OUTPUT_MODE = console`.
- `ISSUE_TEXT = $ARGUMENTS`. If `$ARGUMENTS` is empty or fewer than 50 characters, use `AskUserQuestion` to ask the user to paste the full issue report text now (options: "I'll type it below" | "Actually use CSV mode instead").
- Set `ISSUE_LIST = [{ id: "1", title: "(extracted from report)", text: ISSUE_TEXT }]`.

**If "CSV file of findings"**:
- Set `INPUT_MODE = csv`, `OUTPUT_MODE = files`.
- Use `AskUserQuestion` with:
  > "Please type the path to your CSV file in the 'Other' field below, then submit."
  Options: `["None — cancel and use single issue mode instead"]`
  The text entered in the "Other" field = `CSV_PATH`.
- Use `Read` to load `CSV_PATH`. Parse all rows. Expected columns: `Number`, `Title`, `Summary`.
  - If file cannot be read → halt: "Could not read CSV at `{CSV_PATH}`. Please check the path and try again."
  - If required columns missing → halt: "CSV must have Number, Title, Summary columns."
  - Skip rows where Summary is empty (log a warning per skipped row).
  - Set `ISSUE_LIST = array of objects { id: row.Number, title: row.Title, text: row.Summary }`.
- Initialize `validation_notes.md` in the working directory: if file does not exist, `Write` it with empty content. This file accumulates inter-issue notes across the batch.
- Set `BATCH_SIZE = 5`. This controls cross-issue parallelism in EXECUTION DISPATCH below. Larger = faster wall clock but less inter-batch learning. Smaller = more learning, slower.

---

## PRE-STEP: Collect Context (runs once per session)

**This step runs once for the whole session, before the per-issue loop. Context collected here is shared across all issues.**

Collect three inputs using the two-round pattern described below for each. This pattern eliminates the path input bug — never offer "Enter path below" as a predefined option; always use the "Other" field mechanism to capture typed paths and URLs.

**The Two-Round Input Pattern (use for each item A, B, C below):**
- **Round 1**: AskUserQuestion asking what format they have, with options: `None / skip` | `I'll describe it as free text` | `I have a file path or URL`
- **If "I have a file path or URL"** → **Round 2**: AskUserQuestion with the prompt "Type the file path or URL in the 'Other' field below, then submit." Options: `["None — skip after all"]`. Whatever text is in the "Other" field = the path/URL. If the "Other" field is empty, re-prompt once; if still empty, treat as "None / skip".
- **If file path**: use `Read`. **If URL**: use `WebFetch`. On failure: warn user, set content to the fallback string.

**A. Protocol Documentation**

Round 1: "Do you have protocol documentation for this protocol?"
Options: `None / skip` | `I'll describe it as free text` | `I have a file path or URL`

- None/skip → `DOCS_RAW = "No documentation provided."`
- Free text → `AskUserQuestion` to ask user to describe protocol mechanics. Store response as `DOCS_RAW`.
- File/URL → Apply two-round pattern. Store loaded content as `DOCS_RAW`. On failure → `DOCS_RAW = "Docs fetch failed."`

**B. Role Trust File**

Round 1: "Do you have a file describing privileged roles and their trust levels?"
Options: `None / skip` | `I'll describe it as free text` | `I have a file path or URL`

Same pattern. Store as `ROLE_TRUST_RAW`. Fallback: `"No role trust file provided."` / `"Role trust file not readable."`

**C. Audit Scope**

Round 1: "Do you have an audit scope file listing in-scope contracts?"
Options: `None / skip — treat all .sol files as in scope` | `I'll list them as free text` | `I have a file path or URL`

Same pattern. Store as `SCOPE_RAW`.

After loading `SCOPE_RAW`:
- Use `Glob("**/*.sol")` (and `**/*.vy` if Vyper) to find all source files in the working directory.
- If "None / skip": `SCOPE_FILES` = all found .sol files.
- If text or file was provided: match the contract names/paths listed in `SCOPE_RAW` against the Glob results. `SCOPE_FILES` = resolved matches. Warn on any listed name not found (non-fatal).
- If `SCOPE_FILES` is empty → warn: "No source files found in working directory. Text-only analysis will be used."

**D. Summarization**

After collecting all three inputs, write summarized versions to `.validation_context/` in the working directory. Create the directory if needed (`mkdir -p .validation_context`).

**1. `.validation_context/docs_summary.md`** (max 1000 words)
Summarize `DOCS_RAW` covering: protocol purpose, core mechanics, key invariants, security-relevant behaviors.
If `DOCS_RAW = "No documentation provided."` → write that string verbatim.

**2. `.validation_context/roles_summary.md`** (max 300 words)
Summarize `ROLE_TRUST_RAW` as a structured list: Role → trust level (TRUSTED / SEMI-TRUSTED / UNTRUSTED) + one-sentence description of capabilities.
If `ROLE_TRUST_RAW = "No role trust file provided."` → write that string verbatim.

**3. `.validation_context/scope_summary.md`** (concise list)
Write `SCOPE_FILES` as a list, one path per line.
If empty → write "No source files in scope identified."

**4. `.validation_context/external_integrations.md`**
Grep `SCOPE_FILES` for these external protocol names: `Pendle`, `Curve`, `Chainlink`, `Aave`, `Uniswap`, `Lido`, `Ethena`, `Compound`, `Balancer`, `Convex`, `Frax`, `GMX`, `Morpho`, `EigenLayer`, `Renzo`, `Kelp`.
For each found: list the protocol name, which files it appears in, and the apparent integration type (import, interface call, oracle, etc.).
Also scan `DOCS_RAW` for these names and add any additionally mentioned.
If none found: write "No external protocol integrations detected."

**E. Variable Assignments**

```
DOCS_SUMMARY    = content of .validation_context/docs_summary.md
ROLES_SUMMARY   = content of .validation_context/roles_summary.md
SCOPE_SUMMARY   = content of .validation_context/scope_summary.md
EXTERNAL_INTEGRATIONS = content of .validation_context/external_integrations.md
EXTERNAL_RESEARCH_CACHE = {}   ← session-level cache; keyed by "{protocol}::{claim_fingerprint}"
                                  where claim_fingerprint = first 80 chars of the normalized claim text
```

All downstream pipeline steps use `DOCS_SUMMARY` and `ROLES_SUMMARY` (never the raw content). Code reads in Steps 1–4 are restricted to files in `SCOPE_FILES`. If an issue references an out-of-scope file, read it but flag it as out-of-scope in the pipeline trace.

---

## EXECUTION DISPATCH

### Single mode (`INPUT_MODE = single`)

Process the one issue sequentially in the orchestrator. Run the per-issue pipeline (Steps 1 → 2 → Wave 1 → Wave 2 → 3C → 4C → OUTPUT) once and stop.

Set `BATCH_NOTES = ""` (no batch context in single mode).

Scan `ISSUE_TEXT` against `EXTERNAL_INTEGRATIONS`; populate `RELEVANT_INTEGRATIONS`.

### CSV mode (`INPUT_MODE = csv`) — BATCH PARALLEL

Process `ISSUE_LIST` in waves of `BATCH_SIZE` issues. Each issue in a batch runs as an independent subagent in parallel.

```
While ISSUE_LIST is not empty:
  batch = first BATCH_SIZE items of ISSUE_LIST (or fewer if remaining < BATCH_SIZE)
  remove batch from ISSUE_LIST

  Snapshot once per batch (NOT per issue):
    BATCH_NOTES_SNAPSHOT = current contents of validation_notes.md
    CACHE_SNAPSHOT       = current EXTERNAL_RESEARCH_CACHE

  Spawn |batch| general-purpose agents in ONE message (parallel Task calls).
  Each agent receives the prompt template below, with its own CURRENT_ISSUE.

  Await all agents in the batch.

  For each returned result:
    - Append the agent's notes line to validation_notes.md (in order of issue id)
    - Merge the agent's new external research entries into EXTERNAL_RESEARCH_CACHE
    - Print one-liner progress to console
```

### Per-Issue Subagent Prompt (CSV mode)

Spawn each issue as:

```
Task(subagent_type="general-purpose", model="opus", prompt="
You are a Validate-Issue Per-Issue Agent. Run the per-issue pipeline for ONE issue.

## Issue
- ID: {CURRENT_ISSUE.id}
- Title: {CURRENT_ISSUE.title}
- Text:
{CURRENT_ISSUE.text}

## Shared Context (read-only)
- DOCS_SUMMARY: contents of .validation_context/docs_summary.md
- ROLES_SUMMARY: contents of .validation_context/roles_summary.md
- SCOPE_SUMMARY: contents of .validation_context/scope_summary.md
- EXTERNAL_INTEGRATIONS: contents of .validation_context/external_integrations.md
- BATCH_NOTES (snapshot — NON-AUTHORITATIVE, must re-verify any factual claim):
{BATCH_NOTES_SNAPSHOT}
- EXTERNAL_RESEARCH_CACHE (snapshot, JSON-like):
{CACHE_SNAPSHOT}

## Your Task
Execute the per-issue pipeline from ~/.claude/agents/skills/judge/SKILL.md
(or skill/judge/SKILL.md if working locally) for THIS one issue:

  STEP 1 → STEP 2 → WAVE 1 (1.5 ∥ 3A ∥ 4A) → filter → WAVE 2 (3B + 4B in one message)
    → STEP 3C → STEP 4C (if needed) → OUTPUT

You MUST:
- Spawn checker/generator/judge subagents exactly as the SKILL describes.
- Treat BATCH_NOTES as advisory only.
- Use the provided EXTERNAL_RESEARCH_CACHE before spawning Step 1.5; if a cache hit
  exists for your issue's external claim, skip the WebSearch entirely.
- Write `validation_results/ISSUE-{id}.md` per the OUTPUT section.

You MUST NOT:
- Modify validation_notes.md, .validation_context/*, or any shared file other than your
  own validation_results/ISSUE-{id}.md.
- Spawn nested judge pipelines.
- Process issues other than the one assigned to you.

## Return
Return a JSON-like block:
  VERDICT: VALID / INVALID / DOWNGRADED
  EXIT_STEP: 1|2|3|4
  SEVERITY: {final severity or N/A}
  NOTES_LINE: one line, ≤50 words, suitable to append to validation_notes.md
  NEW_CACHE_ENTRIES: list of {protocol, claim_fingerprint, result_summary} added by your
                     Step 1.5 (empty list if you only had cache hits or no external claims)

SCOPE: One issue, one output file, return and stop.
")
```

### Cross-Issue Parallelism Notes

- Issues in the SAME batch do not see each other's notes or cache updates. This is acceptable: `BATCH_NOTES` is non-authoritative by design (the rule below), and the cache only affects *later* batches.
- BATCH_NOTES rule: BATCH_NOTES contains observations and context hints from prior issues — **NOT verdicts**. Do NOT treat a prior note that says an issue is valid/invalid as authoritative. Re-verify any factual claim in BATCH_NOTES against the actual code before relying on it.
- Each per-issue subagent itself spawns 4–7 opus subagents during its run. At BATCH_SIZE=5 you have ~25–35 opus agents in flight simultaneously. Reduce BATCH_SIZE if you hit rate limits.

---

## STEP 1: Initial Sweep (orchestrator-direct)

Parse `ISSUE_TEXT` for three required components:

| Component | What to look for |
|-----------|-----------------|
| **Code location** | File name, line numbers, function names, contract names |
| **Description** | What the vulnerability is — the mechanism |
| **Impact** | What harm results — fund loss, DoS, griefing, etc. |

**If any component is missing**: Use `AskUserQuestion` to request the specific missing piece. Example: "The issue report is missing a code location. Please provide the affected file(s), function(s), and line number(s)."

**Once all three are present**, verify:
1. Use `Grep`/`Glob` to confirm the referenced file(s) exist in the working directory.
2. Use `Read` to confirm the referenced function/line exists in that file.
3. Check that the description is internally consistent (mechanism matches claimed impact).

**Scope check**: When verifying referenced files exist, check against `SCOPE_FILES` first. If a file is referenced but not in `SCOPE_FILES`:
- If the file exists in the working directory → read it but note "Out-of-scope file referenced" in the pipeline trace.
- If the file does not exist at all → EARLY EXIT `INVALID`: "Referenced file `{file}` not found in codebase."

**EARLY EXIT conditions**:
- Referenced function does not exist at claimed location → `INVALID` — "Function `{func}` not found in `{file}`."
- Description is internally contradictory → `INVALID` with explanation.
- If no codebase is available (no matching files at all) → warn user, continue with text-only analysis. Checker agents in later steps will return `UNCERTAIN` instead of `HOLDS`/`FAILS`.

On early exit, jump to **OUTPUT** with `EXIT_STEP = 1`.

---

## STEP 1.5: External Protocol Research (orchestrator-direct)

> **Wave 1**: launched in parallel with Step 3A AND Step 4A immediately after Step 2 completes. All three fire in the same message.

Scan `ISSUE_TEXT` for references to **external protocols** the in-scope code integrates with (e.g., Pendle, Aave, Uniswap, Chainlink, Curve, Lido, Ethena, Compound, MakerDAO, Balancer, etc.).

**If no external protocol is referenced in the issue** → set `EXTERNAL_RESEARCH = ""` and complete immediately (Step 3A continues unblocked).

**If the issue's validity depends on the behavior of an external protocol** (e.g., "Pendle's getPtToSyRate returns X", "expired PT redeems 1:1", "oracle always returns 18 decimals"):

1. **Identify the external claim**: Extract the specific claim about external protocol behavior that the issue depends on.

2. **Cache check**: Before spawning a research agent, compute `cache_key = "{EXTERNAL_PROTOCOL_NAME}::{first 80 chars of normalized claim}"`. If `EXTERNAL_RESEARCH_CACHE[cache_key]` exists → set `EXTERNAL_RESEARCH = EXTERNAL_RESEARCH_CACHE[cache_key]` and skip the agent spawn entirely.

3. **Research via WebSearch** (only if cache miss): Spawn a research agent to verify the claim:
   ```
   Agent(subagent_type="general-purpose", model="sonnet", prompt="
   You are an External Protocol Research Agent.

   ## External Integrations Detected in Scope Codebase
   {EXTERNAL_INTEGRATIONS}

   ## Relevant Integrations for This Issue
   {RELEVANT_INTEGRATIONS or "None detected."}

   ## Your Task
   Verify the following claim about an external protocol's behavior by searching
   the web for authoritative documentation, forum posts, or on-chain evidence.

   ## Claim to Verify
   {EXTERNAL_CLAIM}

   ## Protocol
   {EXTERNAL_PROTOCOL_NAME}

   ## Instructions
   1. Use WebSearch to find the protocol's official documentation on this behavior.
      Search queries to try:
      - '{protocol} {function_name} documentation'
      - '{protocol} {function_name} return value decimals'
      - 'site:docs.{protocol}.finance {topic}'
      - '{protocol} {topic} audit finding'
   2. Look for on-chain evidence (Etherscan links, Dune queries cited in forums).
   3. Check audit reports or forum discussions about this specific behavior.
   4. If WebSearch fails, use WebFetch on the protocol's official docs URL if known.

   ## Output Format
   **Claim**: {the claim being verified}
   **Verified**: TRUE / FALSE / UNVERIFIABLE
   **Evidence**: {links, quotes from docs, on-chain data}
   **Summary**: {2-3 sentences on what the external protocol actually does}

   SCOPE: Research only. Return findings and stop.
   ")
   ```

4. **Store result and populate cache**: Set `EXTERNAL_RESEARCH = agent result`. Store `EXTERNAL_RESEARCH_CACHE[cache_key] = EXTERNAL_RESEARCH` for reuse by subsequent issues in this session.

5. **Inject into all checker prompts**: All subsequent checker agents receive `EXTERNAL_RESEARCH` as additional context, with the instruction: *"The following external protocol research was conducted. Use these VERIFIED facts over your own assumptions about external protocol behavior. If this research contradicts a common assumption, trust the research."*

**CRITICAL RULE**: If the external claim is UNVERIFIABLE (WebSearch returned no authoritative source), instruct all checker agents: *"Claims about {protocol}'s behavior could not be independently verified. Any invalidation reason that depends on assumptions about {protocol}'s behavior MUST return UNCERTAIN, not HOLDS. UNCERTAIN does not trigger early exit."*

---

## STEP 2: Privileged Roles Check (orchestrator-direct, runs before Step 1.5 and Step 3A)

Scan `ISSUE_TEXT` for mentions of these roles in the **attack path** (not just mentioned in context):
`owner`, `admin`, `governance`, `operator`, `keeper`, `multisig`, `timelock`, `guardian`, `manager`, `deployer`, `dao`, `council`

**If no privileged role is part of the attack path** → skip to Step 3.

**If a privileged role IS required for the attack**:

1. **Role trust determination** (in priority order):
   - If `ROLES_SUMMARY` available → search for the role name and extract trust level.
   - If `DOCS_SUMMARY` available → search for role description, infer trust level.
   - If neither available → apply default heuristic:
     - `owner`, `admin`, `governance`, `timelock`, `multisig`, `dao`, `council` = **TRUSTED** (fully trusted by default)
     - `operator`, `keeper`, `guardian`, `manager`, `deployer` = **SEMI-TRUSTED** (flag but don't auto-downgrade)

2. **If role is TRUSTED and attack requires it to act maliciously**:
   - Set `MAX_SEVERITY = Low`
   - Set `DOWNGRADE_REASON = "Attack requires trusted role [{role}] to act maliciously."`
   - **CRITICAL DISTINCTION**: Only apply this cap when the vulnerability REQUIRES the trusted role to take a deliberate harmful action. Do NOT apply if:
     - The trusted role makes a legitimate, correct administrative call but the code behaves unexpectedly (that is a code bug, not admin abuse)
     - The issue is about unexpected protocol behavior triggered by normal admin operations
     - The harm falls on users without any malicious intent from the admin

3. **EARLY EXIT**: Only if the ENTIRE issue reduces to "trusted role can rug/steal" with no other dimension (no external precondition manipulation, no path via non-privileged actor). In this case → `INVALID` or `Low` with justification. Jump to **OUTPUT** with `EXIT_STEP = 2`.

4. If role is SEMI-TRUSTED or issue has additional dimensions beyond role abuse → continue to Step 3 (with `MAX_SEVERITY` cap applied if set).

**After Step 2 completes without early exit → launch WAVE 1 (Step 1.5, Step 3A, and Step 4A) in parallel in a SINGLE message. Do not wait for any one before starting the others. Wave 1 wall clock is bounded by the slowest of the three (typically Step 4A opus generator, ~1.5 min).**

---

## STEP 3: Generic Invalidation Check (1 selector + 3 checkers)

### 3A — Selector Agent

> **Wave 1**: launched in parallel with Step 1.5 and Step 4A. Sonnet selector reads the invalidation library and picks 3 generic reasons. Does not depend on Step 1.5 or Step 4A output.

Spawn one agent to select the 3 most applicable invalidation reasons:

```
Agent(subagent_type="general-purpose", model="sonnet", prompt="
You are an Invalidation Selector Agent.

## Your Task
Read the invalidation library below and select the 3 reasons MOST APPLICABLE to the
security issue under review. Do NOT generate new reasons — only select from the library.

## Issue Report
{ISSUE_TEXT}

## Protocol Documentation
{DOCS_SUMMARY}

## Audit Scope
{SCOPE_SUMMARY}

## Prior Batch Discoveries
{BATCH_NOTES or "None."}

## Invalidation Library
{Read and paste the full content of ~/.claude/agents/skills/judge/references/invalidation-library.md}

## Output Format
For each of your 3 selections:
### Selection {N}
- **ID**: {e.g., CP-2}
- **Title**: {from library}
- **Why this applies**: {2-3 sentences explaining why this specific reason is relevant to THIS issue}

SCOPE: Select 3 reasons from the library. Do NOT verify them against code. Return selections and stop.
")
```

### 3B — Parallel Checkers (part of WAVE 2, fires together with Step 4B)

**After WAVE 1 completes** (Steps 1.5, 3A, and 4A have all returned):

1. **Filter Step 4A's reasons against Step 3A's selections.** For each reason from Step 4A, compare against Step 3A's 3 selections. Drop any reason whose mechanism falls into the same vulnerability category as a Step 3A selection (e.g., "missing slippage check" overlaps with library entry CP-2 "input validation"). Keep up to 3 surviving reasons, ranked by the generator's confidence. If Step 4A produced fewer than 3 unique surviving reasons, proceed with what remains (Wave 2 may have <6 agents).

2. **Launch WAVE 2 in a SINGLE message**: spawn 3 Step 3B checkers (one per Step 3A selection) AND up to 3 Step 4B checkers (one per surviving Step 4A reason) — up to 6 opus agents in parallel.

Step 3B agents use this prompt (one agent per selected reason):

```
Agent(subagent_type="general-purpose", model="opus", prompt="
You are an Invalidation Checker Agent.

## Your Task
Determine whether the following invalidation reason HOLDS against the actual code.

## Issue Report
{ISSUE_TEXT}

## Invalidation Reason to Check
**{REASON_ID}**: {REASON_TITLE}
{REASON_FULL_TEXT_FROM_LIBRARY}
**Why it might apply**: {SELECTOR_JUSTIFICATION}

## Protocol Documentation
{DOCS_SUMMARY}

## Audit Scope
{SCOPE_SUMMARY}

## Prior Batch Discoveries
{BATCH_NOTES or "None."}

## External Protocol Research (from Step 1.5)
{EXTERNAL_RESEARCH or "No external protocol research was conducted."}

## Instructions
1. Read the source code files referenced in the issue report.
2. Trace the specific claim made by this invalidation reason against the actual code.
3. Look for concrete evidence: does a guard exist? Is the state reachable? Does the math check out?
4. Determine your verdict.

## ANTI-HALLUCINATION RULE (CRITICAL)
Do NOT make confident claims about external protocol behavior (return values, scaling,
exchange rates, redemption mechanics) based on your training data. If your verdict depends
on how an external protocol's function behaves:
- Use the External Protocol Research above if available — trust it over your assumptions.
- If no research is available, and you cannot verify the behavior from IN-SCOPE code alone,
  your verdict MUST be UNCERTAIN, not HOLDS.
- "I believe Pendle's X always returns Y" is NOT evidence. On-chain data or official docs ARE.

## Output Format
**Reason**: {reason ID and title}
**Verdict**: HOLDS / FAILS / UNCERTAIN
**Evidence**: {specific code references — file:line, function names, snippets}
**Explanation**: {3-5 sentences on why this verdict}
**Severity Impact**: INVALIDATE / DOWNGRADE_TO_{severity} / NO_CHANGE

SCOPE: Check ONLY this one invalidation reason. Return your verdict and stop.
")
```

### 3C — Orchestrator Confirmation

> **Note**: by the time Step 3C runs, WAVE 2 has fully completed — both Step 3B and Step 4B checker outputs are already available. Step 3C operates ONLY on Step 3B's results. If Step 3C triggers an early exit, Step 4B's results are discarded (Step 4C judge is skipped, saving ~1.5 min). If no early exit, Step 4B's results are consumed by Step 4C.

**EARLY EXIT requires AT LEAST 2 of the 3 Step 3B checkers to return `HOLDS`** (a single HOLDS is not sufficient — this prevents false negatives from isolated checker mistakes).

If at least 2 Step 3B checkers returned `HOLDS`:
1. Read each holding checker's evidence and explanation.
2. Use `Read` to independently verify the cited code references.
3. If the evidence from at least 2 checkers is solid and the logic is sound → **EARLY EXIT**: set verdict to `INVALID` or `DOWNGRADED` based on `Severity Impact`. Discard Step 4B results entirely. Jump to **OUTPUT** with `EXIT_STEP = 3`.
4. If a checker's evidence is weak or has a logical flaw → discard that invalidation, note it as `OVERRULED` in the pipeline trace. Early exit only if 2+ remain confirmed after overrule.

If fewer than 2 Step 3B checkers returned `HOLDS` (or insufficient confirmed after overrule) → consume Step 4B results and proceed to Step 4C.

---

## STEP 4: Issue-Specific Adversarial Check (1 generator + 3 checkers + 1 judge)

### 4A — Generator Agent

> **Wave 1**: launched in parallel with Step 1.5 and Step 3A immediately after Step 2 completes — BEFORE the selector has picked its 3 reasons. The generator therefore does not know what Step 3A will choose; instead, it receives the FULL invalidation library and is told to generate reasons that go BEYOND the library categories.
>
> Duplicate filtering against Step 3A's actual selections happens after Wave 1 completes, in the orchestrator (see Step 3B "Filter rule"), before Wave 2 fires.

```
Agent(subagent_type="general-purpose", model="opus", prompt="
You are an Adversarial Invalidation Generator. The issue below survived generic
invalidation checking. Your job is to find ISSUE-SPECIFIC reasons it might still be invalid.

## Issue Report
{ISSUE_TEXT}

## Protocol Documentation
{DOCS_SUMMARY}

## Audit Scope
{SCOPE_SUMMARY}

## Prior Batch Discoveries
{BATCH_NOTES or "None."}

## Generic Invalidation Library (DO NOT generate reasons that fall into these categories)
{Read and paste the full content of ~/.claude/agents/skills/judge/references/invalidation-library.md}

The library above is being checked in parallel by Step 3 (selector + checkers). Your job
is ISSUE-SPECIFIC adversarial analysis that goes BEYOND these generic categories. You MUST
NOT propose reasons that fall into the library's categories. Focus on:
- Implicit invariants enforced by code outside the issue's narrow scope
- Concrete economic edge cases with real numbers
- Atomicity / MEV / frontrunning constraints on the attack sequence
- Intentional design (NatSpec, calling context, internal-only helpers)
- Interaction effects between the affected function and other protocol components

## Your Approach
Think adversarially. Read the actual source code. Consider:
- Implicit invariants maintained by other functions not mentioned in the report
- Exact economic analysis: compute attacker profit vs. cost with real numbers
- Gas costs for the full attack sequence
- MEV/frontrunning that would prevent or front-run the attack
- Timelock or delay mechanisms that give defenders time to respond
- Interaction effects with other protocol components
- Whether the described transaction sequence is actually executable atomically
- State changes from other users' normal operations that would disrupt the attack
- **Was this behavior intentional?** For each reason you generate, explicitly ask: is there a legitimate design intent behind this behavior? Look for evidence in NatSpec comments, protocol docs, or the broader calling context (e.g., a function that looks broken in isolation may be correct when used as an internal step by its only caller). If there is a plausible design intent, state it as a high-confidence invalidation reason.

## Output Format
Generate 3-5 NEW issue-specific invalidation reasons. For each:
### Reason {N}
- **Title**: {concise title}
- **Explanation**: {3-5 sentences with specific code references}
- **Code to check**: {file:line references the checker should read}
- **Confidence**: HIGH / MEDIUM / LOW

SCOPE: Generate invalidation hypotheses only. Do NOT verify them yourself. Return and stop.
")
```

### 4B — Parallel Checkers (part of WAVE 2, fires together with Step 3B)

**Wave 2**: launched in the SAME message as Step 3B's checkers — up to 6 opus agents in parallel. Spawn checkers for the top 3 surviving (post-filter) adversarial reasons by confidence. Same prompt pattern as Step 3B, one agent per reason.

Step 4B output is consumed by Step 4C ONLY if Step 3C did not early-exit. If Step 3C early-exits, Step 4B's results are discarded.

```
Agent(subagent_type="general-purpose", model="opus", prompt="
You are an Adversarial Invalidation Checker.

## Your Task
Verify whether this issue-specific invalidation reason holds up against the code.

## Issue Report
{ISSUE_TEXT}

## Invalidation Reason to Check
**{REASON_TITLE}**
{REASON_EXPLANATION}

## Code References to Check
{CODE_REFS_FROM_GENERATOR}

## Protocol Documentation
{DOCS_SUMMARY}

## Audit Scope
{SCOPE_SUMMARY}

## Prior Batch Discoveries
{BATCH_NOTES or "None."}

## External Protocol Research (from Step 1.5)
{EXTERNAL_RESEARCH or "No external protocol research was conducted."}

## Instructions
1. Read the specific code files and lines referenced.
2. Trace the invalidation claim step by step.
3. Look for concrete evidence supporting or refuting the claim.
4. Return your verdict with evidence.

## ANTI-HALLUCINATION RULE (CRITICAL)
Do NOT make confident claims about external protocol behavior based on your training data.
If your verdict depends on how an external protocol's function behaves, use the External
Protocol Research above. If unavailable and unverifiable from in-scope code, verdict = UNCERTAIN.

## Output Format
**Reason**: {reason title}
**Verdict**: HOLDS / FAILS / UNCERTAIN
**Evidence**: {specific code references — file:line, snippets}
**Explanation**: {3-5 sentences}
**Severity Impact**: INVALIDATE / DOWNGRADE_TO_{severity} / NO_CHANGE

SCOPE: Check ONLY your assigned reason. Return verdict and stop.
")
```

### 4C — Neutral Judge (only if Step 3C did NOT early-exit AND any 4B checker returns HOLDS)

```
Agent(subagent_type="general-purpose", model="opus", prompt="
You are the Neutral Judge. An adversarial checker has determined that a security issue
should be invalidated or downgraded. You must make the FINAL impartial assessment.

## Issue Report
{ISSUE_TEXT}

## Proposed Invalidation
**Reason**: {reason title}
**Checker Evidence**: {full evidence from checker agent}
**Checker Verdict**: HOLDS
**Severity Impact**: {from checker}

## Protocol Documentation
{DOCS_SUMMARY}

## Instructions
1. Read the original issue report carefully and understand the claimed attack.
2. Read the checker's evidence and trace their logic.
3. Independently read the relevant source code.
4. Seek counterarguments: is there a way the attack still works despite this reason?
5. Also seek design-intent arguments: is the described behavior intentional? Check NatSpec, docs, and calling context.
6. Render your final verdict impartially — do not bias toward VALID or INVALID. Follow the evidence.

## Balanced Standard
- INVALID: the invalidation reason clearly holds AND there is no plausible way the issue manifests despite it.
- DOWNGRADE: the issue is real but the invalidation reason significantly limits severity or scope.
- VALID: the invalidation reason does not hold or is insufficient to negate the issue.
- Apply your best judgment. Do not default to VALID out of caution — a well-evidenced INVALID is correct.

## Output Format
**Final Verdict**: VALID (issue is real, invalidation reason does not hold) / INVALID (issue should be rejected) / DOWNGRADE (severity should be lowered)
**If DOWNGRADE, New Severity**: Critical / High / Medium / Low / Informational
**Justification**: {5-10 sentences, detailed reasoning}
**Confidence**: HIGH / MEDIUM / LOW

SCOPE: Render your verdict and stop. Do not proceed to other pipeline steps.
")
```

### 4D — Final Severity Assessment (orchestrator inline)

After all Step 4 agents complete:
1. If neutral judge says `INVALID` → set verdict to `INVALID`.
2. If neutral judge says `DOWNGRADE` → apply new severity (but not below `MAX_SEVERITY` from Step 2 if set).
3. If no adversarial reason held, or judge says `VALID` → issue is **confirmed VALID**.
4. Final severity = `min(original_claimed_severity, MAX_SEVERITY, any_step4_downgrade)`.
5. If no severity was claimed in the original report → assign severity based on Impact × Likelihood:
   - **Impact High** + **Likelihood High** = Critical
   - **Impact High** + **Likelihood Medium** = High
   - **Impact High** + **Likelihood Low** = Medium
   - **Impact Medium** + **Likelihood High** = High
   - **Impact Medium** + **Likelihood Medium** = Medium
   - **Impact Low** + **Any** = Low
   - **Informational** = Informational

---

## OUTPUT

### A. Finalize Verdict and Severity

Apply final severity logic (same as Step 4D):
1. If neutral judge (Step 4C) says `INVALID` with HIGH confidence → verdict = `INVALID`. If MEDIUM or LOW confidence → treat as `DOWNGRADE` to Low instead.
2. If judge says `DOWNGRADE` → apply new severity (but not below `MAX_SEVERITY` from Step 2 if set).
3. If no adversarial reason held, or judge says `VALID` → verdict = `VALID`.
4. Final severity = `min(original_claimed_severity, MAX_SEVERITY, any_step4_downgrade)`.
5. If no severity was claimed → assign via Impact × Likelihood matrix:
   - Impact High + Likelihood High = Critical
   - Impact High + Likelihood Medium = High
   - Impact High + Likelihood Low = Medium
   - Impact Medium + Likelihood High = High
   - Impact Medium + Likelihood Medium = Medium
   - Impact Low + Any = Low
   - Informational = Informational

### B. If OUTPUT_MODE = console (single issue)

Print the following formatted block to the console:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VALIDATION RESULT
Issue:    {CURRENT_ISSUE.title}
Verdict:  {VALID / INVALID / DOWNGRADED}
Severity: {Final Severity, or N/A if INVALID}
Exit:     Step {EXIT_STEP}

Reasoning:
{2-3 sentences summarizing why this verdict was reached}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Also write the full detailed trace to `validated_issues/ISSUE-{N}.md`. Determine N by checking `ls validated_issues/ISSUE-*.md 2>/dev/null | sort -V | tail -1`, extract N, use N+1. If no files exist, start at 1.

Full trace format:
```markdown
# ISSUE-{N}: {Issue Title}

## Pipeline Result
**Verdict**: VALID / INVALID / DOWNGRADED
**Final Severity**: Critical / High / Medium / Low / Informational / N/A
**Original Claimed Severity**: {from report, or "Not specified"}
**Pipeline Exit Point**: Step {1|2|3|4}
**Confidence**: HIGH / MEDIUM / LOW

## Summary
{1-3 sentence summary of the issue and the validation result}

## Location
{File:Line references from the issue}

## Justification
{Detailed explanation of verdict. Include which pipeline steps executed, key evidence for/against validity, severity adjustment reasoning.}

## Invalidation Reasons Tested
| # | Reason | Source | Verdict | Evidence Summary |
|---|--------|--------|---------|-----------------|
| 1 | {reason} | Step 3 (Generic) | FAILS | {brief} |
| ... | | | | |

## Pipeline Trace
- **Step 1 (Initial Sweep)**: {PASSED / FAILED — reason}
- **Step 2 (Privileged Roles)**: {SKIPPED / DOWNGRADED to Low / NO_ISSUE}
- **Step 3 (Generic Check)**: {N} reasons selected, {M} checked, {K} held, {J} confirmed
- **Step 4 (Adversarial Check)**: {N} reasons generated, {M} checked, judge: {verdict}
- **Final Severity**: {severity} {(adjusted from X) if changed}
```

### C. If OUTPUT_MODE = files (CSV batch)

Create directory `validation_results/` if it does not exist.

Write `validation_results/ISSUE-{id}.md` where `{id}` = `CURRENT_ISSUE.id`:

```markdown
# {CURRENT_ISSUE.title}
**Issue ID**: {CURRENT_ISSUE.id}
**Final Severity**: {severity or INVALID}
**Verdict**: {VALID / INVALID / DOWNGRADED}
**Reasoning**: {max 50 words explaining the decisive factor}
**Exit Step**: Step {EXIT_STEP}
```

Print one-liner progress to console:
```
[{CURRENT_ISSUE.id}/{total issues in ISSUE_LIST}] {CURRENT_ISSUE.title} → {VERDICT} (Step {EXIT_STEP})
```

### D. NOTES APPEND (CSV mode only — runs after every issue's output)

After writing the output file, generate a ≤50-word notes entry for the batch notes file. Content should capture:
- Any role trust levels discovered or confirmed for this protocol
- Any external protocol behaviors verified or refuted by the pipeline
- Any contract invariants confirmed (e.g., "rounding always floors in X contract")
- Any pattern that might help validate related issues faster

Append to `validation_notes.md` in the working directory:
```markdown
### Issue {CURRENT_ISSUE.id} — {CURRENT_ISSUE.title}
{≤50 words of notes}
```

In single mode (`INPUT_MODE = single`): skip this step entirely.

---

## EDGE CASES

| Case | Handling |
|------|----------|
| No codebase in working directory | Warn user, run text-only; checkers return UNCERTAIN |
| WebFetch fails | Warn user, continue without docs |
| Subagent fails/times out | Record reason as UNCERTAIN, continue pipeline |
| Issue text < 50 chars | Ask user for full report before starting |
| Multiple issues in one invocation | Use CSV mode; single mode processes exactly one issue |
| No severity claimed in report | Assign via Impact × Likelihood matrix in Step 4D |
| CSV file missing required columns | Halt: "CSV must have Number, Title, Summary columns." |
| CSV row has empty Summary | Skip that row, log: "Issue {N} has empty summary — skipped." |
| `.validation_context/` write fails | Warn user; fall back to raw content directly (original behavior) |
| `validation_notes.md` unreadable mid-batch | Warn, continue with empty BATCH_NOTES for this issue |
| `SCOPE_FILES` is empty | Text-only analysis; checker code reads return UNCERTAIN |
| "Other" field empty after file/URL round | Re-prompt once; if still empty, treat as None/skip |
