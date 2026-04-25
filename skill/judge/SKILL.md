---
name: judge
description: "The Judge — filter AI-generated web3 security findings for false positives via a multi-step adversarial pipeline. Accepts a single issue as text or a CSV file of findings. Usage: /judge <issue text> OR /judge (then select CSV mode)"
---

# Validate-Issue — Web3 Security Issue Validation Pipeline

> **Purpose**: Determine whether a web3 security issue report is valid, invalid, or should be severity-adjusted through a 4-step adversarial process with early exits.
> **Models**: opus for generator/adversarial checkers (4B)/judge, sonnet for selector/generic checkers (3B)/mitigation check (2.5)/external research (1.5)/severity calibrator (5).

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
  WAVE 1:  [STEP 1.5  ∥  STEP 2.5  ∥  STEP 3A  ∥  STEP 4A]  (research + mitigation + selector + generator)
    ↓
  Orchestrator filters Step 4A reasons against Step 3A's 4 selections (drop duplicates)
    ↓
  WAVE 2:  [STEP 3B (2 sonnet) + STEP 4B (2 opus)]       (4 parallel checkers, ONE message)
    ↓
  STEP 3C: orchestrator confirms — EARLY EXIT if ≥2 Step 3B checkers HOLDS
    ↓
  STEP 4C: judge — fires on HOLDS, UNCERTAIN, or HIGH-confidence-reason FAILS (symmetric)
    ↓
  STEP 4D: orchestrator aggregates verdicts; INVALID short-circuits to OUTPUT
    ↓
  STEP 5: severity calibrator — runs on every VALID/DOWNGRADED outcome, sets severity from
          the verified attack path (independent of claimed severity)
    ↓
  OUTPUT
```

Each step can EARLY EXIT with a verdict at Steps 1, 2, or 3C. Step 2.5 runs in background and can cap severity if the issue is a design trade-off. Step 5 only runs for non-INVALID outcomes.

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

### Session Reuse Check

Before collecting anything, check if `.validation_context/` already exists in the working directory with valid summaries from a prior run:

1. Check if all four files exist: `docs_summary.md`, `roles_summary.md`, `scope_summary.md`, `external_integrations.md` in `.validation_context/`.
2. If ALL four exist and are non-empty:
   - Read each file and display a compact preview (first 3 lines of each).
   - Use `AskUserQuestion`:
     > "Found existing validation context from a prior session. What would you like to do?"
     Options: `Reuse as-is` | `Reuse but let me update some fields` | `Start fresh — re-detect everything`
   - **"Reuse as-is"**: Load all four files into their variables (Section E below) and skip directly to **E. Variable Assignments**. Print: "Reusing existing context."
   - **"Reuse but let me update some fields"**: Load all four files, then use `AskUserQuestion`:
     > "Which fields do you want to update?"
     Options: `Docs` | `Roles` | `Scope` | `Multiple / all of them`
     Run ONLY the selected section(s) (A, B, and/or C below) using the standard flow. Keep the rest as-is.
   - **"Start fresh"**: Proceed to auto-detection below as if no context exists.
3. If any file is missing or empty → proceed to auto-detection (no prompt needed).

### Auto-Detection Phase

Before prompting the user for anything, scan the working directory to auto-detect docs, roles, and scope. This runs silently and populates `AUTO_DOCS`, `AUTO_ROLES`, and `AUTO_SCOPE` candidates.

**Auto-detect documentation** — search for common doc files in priority order (stop at first hit per category):

| Priority | Glob Pattern | Notes |
|----------|-------------|-------|
| 1 | `**/WHITEPAPER.md`, `**/whitepaper.md` | Dedicated whitepaper |
| 2 | `**/SPEC.md`, `**/spec.md`, `**/SPECIFICATION.md` | Formal spec |
| 3 | `**/DESIGN.md`, `**/design.md`, `**/ARCHITECTURE.md` | Design doc |
| 4 | `**/docs/README.md`, `**/docs/overview.md`, `**/docs/protocol.md` | Docs folder |
| 5 | `**/README.md` (root only — not nested in node_modules/lib) | Project readme |
| 6 | `**/*.pdf` (exclude `node_modules/`, `lib/`, `.git/`) | PDF docs |

Collect ALL hits (not just first). Set `AUTO_DOCS = list of {path, priority}`. If no hits → `AUTO_DOCS = []`.

**Auto-detect scope** — look for explicit scope files and common contract directories:

| Priority | Pattern | Interpretation |
|----------|---------|---------------|
| 1 | `**/scope.txt`, `**/scope.md`, `**/SCOPE.md`, `**/scope.json` | Explicit scope file |
| 2 | `**/audit-scope.md`, `**/audit_scope.md` | Audit scope file |
| 3 | `foundry.toml` → read `src` field; `hardhat.config.*` → read `paths.sources` | Build config points to source dir |
| 4 | `**/src/**/*.sol`, `**/contracts/**/*.sol` | Standard Solidity source dirs |
| 5 | `**/*.sol` excluding `node_modules/`, `lib/`, `test/`, `tests/`, `script/`, `scripts/`, `mock/`, `mocks/` | All non-test Solidity files |
| 6 | `**/programs/*/src/**/*.rs` (Anchor), `**/*.move` (Aptos/Sui) | Non-EVM source patterns |

Set `AUTO_SCOPE_FILE` = first hit from priorities 1-2 (explicit scope file), or `null`.
Set `AUTO_SCOPE_FILES` = resolved file list from priorities 3-6.
Detect language: if `.sol` files found → `LANG = solidity`; if `.rs` + `Anchor.toml` → `LANG = solana`; if `.move` + `Move.toml` → `LANG = move`. Adjust Glob patterns in scope resolution and later steps accordingly.

**Auto-detect roles** — look for role/trust files:

| Priority | Pattern |
|----------|---------|
| 1 | `**/roles.md`, `**/ROLES.md`, `**/trust.md`, `**/TRUST.md` |
| 2 | `**/access-control.md`, `**/access_control.md` |

Set `AUTO_ROLES = first hit path` or `null`.

### User Confirmation (per-field, always required)

Auto-detection results MUST be confirmed by the user for each field. Present each detected value and offer an override. This runs as three sequential confirmations (one per field), keeping each focused.

**Confirm Docs:**

If `AUTO_DOCS` is non-empty, show the top hit(s) and ask:
> "Auto-detected documentation:\n{list of AUTO_DOCS paths, one per line}\n\nIs this correct?"
Options: `Yes — use {AUTO_DOCS[0].path}` | `Use a different file / URL` | `I'll describe it as free text` | `Skip — no docs`

- **"Yes"**: `DOCS_RAW` = concatenated content of `AUTO_DOCS` files (read each, cap total at 10000 words).
- **"Use a different file / URL"**: Apply the Two-Round Input Pattern (Round 2 below) to get the path/URL. Read/fetch it into `DOCS_RAW`.
- **"Free text"**: `AskUserQuestion` to ask user to describe protocol mechanics. Store as `DOCS_RAW`.
- **"Skip"**: `DOCS_RAW = "No documentation provided."`

If `AUTO_DOCS` is empty:
> "No documentation files auto-detected. Do you have protocol documentation?"
Options: `I have a file path or URL` | `I'll describe it as free text` | `Skip — no docs`
Handle as above.

**Confirm Scope:**

If `AUTO_SCOPE_FILE` is found, show it:
> "Auto-detected scope file: `{AUTO_SCOPE_FILE}`\n\nIs this correct?"
Options: `Yes — use this scope file` | `Use a different file` | `I'll list them as free text` | `Skip — use all {N} detected source files`

If no explicit scope file but `AUTO_SCOPE_FILES` is non-empty:
> "No explicit scope file found, but detected {N} source files in `{directories}`:\n{first 10 files, then '... and N more' if >10}\n\nIs this the correct scope?"
Options: `Yes — use these {N} files` | `Use a scope file instead` | `I'll list the in-scope files` | `Let me adjust the list`

- **"Yes"**: Use `AUTO_SCOPE_FILE` content as `SCOPE_RAW`, or set `SCOPE_FILES = AUTO_SCOPE_FILES` directly.
- **"Use a different/scope file"**: Apply Two-Round Input Pattern. Store as `SCOPE_RAW`.
- **"Free text" / "List"**: `AskUserQuestion` to ask user to list contracts. Store as `SCOPE_RAW`.
- **"Skip"**: `SCOPE_FILES` = all detected source files. `SCOPE_RAW = null`.
- **"Adjust"**: `AskUserQuestion`: "List the files to ADD or REMOVE (prefix with - to remove)." Apply edits to `AUTO_SCOPE_FILES`.

If nothing found:
> "No source files detected in the working directory. Do you have a scope file or file list?"
Options: `I have a file path or URL` | `I'll list them as free text` | `Skip — text-only analysis`

**Confirm Roles:**

If `AUTO_ROLES` is found:
> "Auto-detected role/trust file: `{AUTO_ROLES}`\n\nIs this correct?"
Options: `Yes — use this file` | `Use a different file` | `I'll describe roles as free text` | `Skip — no role file`

If not found:
> "No role trust file auto-detected. Do you have one?"
Options: `I have a file path or URL` | `I'll describe roles as free text` | `Skip — no role file`

Handle identically to the manual flow for each response.

**After all three confirmations**, proceed to **D. Summarization**.

### Manual Input Flow (for fields not auto-detected or user wants to override)

**The Two-Round Input Pattern (use for each item below when manual input is needed):**
- **Round 1**: AskUserQuestion asking what format they have, with options: `None / skip` | `I'll describe it as free text` | `I have a file path or URL`
- **If "I have a file path or URL"** → **Round 2**: AskUserQuestion with the prompt "Type the file path or URL in the 'Other' field below, then submit." Options: `["None — skip after all"]`. Whatever text is in the "Other" field = the path/URL. If the "Other" field is empty, re-prompt once; if still empty, treat as "None / skip".
- **If file path**: use `Read`. **If URL**: use `WebFetch`. On failure: warn user, set content to the fallback string.

**A. Protocol Documentation** (manual)

Round 1: "Do you have protocol documentation for this protocol?"
Options: `None / skip` | `I'll describe it as free text` | `I have a file path or URL`

- None/skip → `DOCS_RAW = "No documentation provided."`
- Free text → `AskUserQuestion` to ask user to describe protocol mechanics. Store response as `DOCS_RAW`.
- File/URL → Apply two-round pattern. Store loaded content as `DOCS_RAW`. On failure → `DOCS_RAW = "Docs fetch failed."`

**B. Role Trust File** (manual)

Round 1: "Do you have a file describing privileged roles and their trust levels?"
Options: `None / skip` | `I'll describe it as free text` | `I have a file path or URL`

Same pattern. Store as `ROLE_TRUST_RAW`. Fallback: `"No role trust file provided."` / `"Role trust file not readable."`

**C. Audit Scope** (manual)

Round 1: "Do you have an audit scope file listing in-scope contracts?"
Options: `None / skip — treat all source files as in scope` | `I'll list them as free text` | `I have a file path or URL`

Same pattern. Store as `SCOPE_RAW`.

### Scope Resolution (runs regardless of auto vs manual)

After `SCOPE_RAW` and/or `AUTO_SCOPE_FILES` are set:
- Detect language if not already detected: Use `Glob("**/*.sol")`, `Glob("**/*.vy")`, `Glob("**/programs/*/src/**/*.rs")`, `Glob("**/*.move")` to find source files.
- If `SCOPE_RAW` is set (from explicit scope file or user input): match the contract names/paths listed in `SCOPE_RAW` against the Glob results. `SCOPE_FILES` = resolved matches. Warn on any listed name not found (non-fatal).
- If `SCOPE_RAW` is `null` and `AUTO_SCOPE_FILES` is populated: `SCOPE_FILES` = `AUTO_SCOPE_FILES`.
- If neither: `SCOPE_FILES` = all found source files (`.sol`, `.vy`, `.rs`, `.move`).
- If `SCOPE_FILES` is empty → warn: "No source files found in working directory. Text-only analysis will be used."

**D. Summarization**

After all inputs are resolved, write summarized versions to `.validation_context/` in the working directory. Create the directory if needed (`mkdir -p .validation_context`).

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

  STEP 1 → STEP 2 → WAVE 1 (1.5 ∥ 2.5 ∥ 3A ∥ 4A) → filter → WAVE 2 (3B + 4B in one message)
    → STEP 3C → STEP 4C (symmetric trigger: HOLDS, UNCERTAIN, or rejected HIGH-confidence reason)
    → STEP 4D → STEP 5 (severity calibrator, only if verdict is VALID or DOWNGRADED) → OUTPUT

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
- Each per-issue subagent itself spawns up to 8 subagents in the worst case (5 sonnet + 3 opus): Step 1.5, Step 2.5, Step 3A, two Step 3B, Step 4A, two Step 4B, Step 4C, Step 5. Best case is ~5 (when 3C early-exits or 4C is skipped, Step 5 still adds one if not INVALID). At BATCH_SIZE=5 you have ~15 opus + ~25 sonnet agents in flight simultaneously. Reduce BATCH_SIZE if you hit rate limits.

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

**Scope check**: When verifying referenced files exist, check against `SCOPE_FILES` first. Distinguish two reference roles:

- PRIMARY reference: the file/function cited as the location of the bug (the "Code location" component of the issue, the function whose logic is claimed to be vulnerable).
- SECONDARY reference: a file mentioned only as an interaction point (e.g., an external contract the in-scope code calls, or a library the vulnerable function relies on).

If a PRIMARY referenced file is out of scope:
- File exists in the working directory but is NOT in `SCOPE_FILES` → EARLY EXIT `INVALID`: "Primary vulnerability location `{file}` is out of scope. The bug is reported in code that is not part of this audit's scope."
- File does not exist anywhere → EARLY EXIT `INVALID`: "Referenced file `{file}` not found in codebase."

If only SECONDARY references are out of scope:
- Read the out-of-scope file as needed for context, but mark it as `OUT_OF_SCOPE_DEPENDENCY` in the pipeline trace.
- Inject a note into the Step 4A generator and Step 4C judge prompts: *"The issue depends on out-of-scope file(s) `{files}`. The behavior of out-of-scope code is not auditable; treat assumptions about it the way you would treat assumptions about an external protocol — they need evidence (docs, on-chain data) before they can support a HOLDS verdict."*
- Continue with the pipeline.

If `SCOPE_FILES` is empty (text-only mode, no codebase available) → skip the scope check entirely; the existing "no codebase available" handling below applies.

**EARLY EXIT conditions**:
- Referenced function does not exist at claimed location → `INVALID` — "Function `{func}` not found in `{file}`."
- Description is internally contradictory → `INVALID` with explanation.
- If no codebase is available (no matching files at all) → warn user, continue with text-only analysis. Checker agents in later steps will return `UNCERTAIN` instead of `HOLDS`/`FAILS`.

On early exit, jump to **OUTPUT** with `EXIT_STEP = 1`.

---

## STEP 1.5: External Protocol Research (orchestrator-direct)

> **Wave 1**: launched in parallel with Step 2.5, Step 3A, AND Step 4A immediately after Step 2 completes. All four fire in the same message.

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
   - Set `MAX_SEVERITY = Informational`
   - Set `DOWNGRADE_REASON = "Attack requires trusted role [{role}] to act maliciously."`
   - **CRITICAL DISTINCTION**: Only apply this cap when the vulnerability REQUIRES the trusted role to take a deliberate harmful action. Do NOT apply if:
     - The trusted role makes a legitimate, correct administrative call but the code behaves unexpectedly (that is a code bug, not admin abuse)
     - The issue is about unexpected protocol behavior triggered by normal admin operations
     - The harm falls on users without any malicious intent from the admin

3. **EARLY EXIT**: Only if the ENTIRE issue reduces to "trusted role can rug/steal" with no other dimension (no external precondition manipulation, no path via non-privileged actor). In this case → `INVALID` or `Informational` with justification. Jump to **OUTPUT** with `EXIT_STEP = 2`.

4. If role is SEMI-TRUSTED or issue has additional dimensions beyond role abuse → continue to Step 3 (with `MAX_SEVERITY` cap applied if set).

**After Step 2 completes without early exit → launch WAVE 1 (Step 1.5, Step 2.5, Step 3A, and Step 4A) in parallel in a SINGLE message. Do not wait for any one before starting the others. Wave 1 wall clock is bounded by the slowest of the four (typically Step 4A opus generator, ~1.5 min).**

---

## STEP 2.5: Mitigation Viability Check (Wave 1, background)

> **Wave 1**: launched in parallel with Steps 1.5, 3A, and 4A immediately after Step 2 completes. Runs as a background agent — does NOT block the pipeline.

**Purpose**: Determine whether the reported issue has a clean fix or represents an inherent design trade-off where any mitigation introduces equivalent downsides. Trade-off issues (where the cure is as costly as the disease) are capped at Low/Informational.

**Mitigation extraction**: Scan `ISSUE_TEXT` for a recommended fix/mitigation section. Common markers: "Recommendation", "Mitigation", "Fix", "Remediation". Extract the mitigation text as `MITIGATION_TEXT`. If no mitigation is found in the report, set `MITIGATION_TEXT` to `"No mitigation was proposed in the report. Identify the most obvious fix for this vulnerability class."`.

**Spawn agent**:

```
Agent(subagent_type="general-purpose", model="sonnet", run_in_background=true, prompt="
You are a Mitigation Viability Checker.

## Your Task
Determine whether the security issue below has a clean fix, or whether it represents
an inherent design trade-off where any mitigation introduces equivalent downsides.

## Issue Report
{ISSUE_TEXT}

## Proposed Mitigation
{MITIGATION_TEXT}

## Protocol Documentation
{DOCS_SUMMARY}

## Audit Scope
{SCOPE_SUMMARY}

## Instructions
1. Read the source code at the referenced location(s).
2. If a mitigation was proposed, evaluate it. If not, identify the most natural fix
   for this vulnerability class.
3. Evaluate the mitigation:
   - Does it fully resolve the root cause without introducing new problems of
     comparable magnitude?
   - Or does it create a trade-off of similar severity? Examples of trade-offs:
     * Fixing front-running by adding commit-reveal that adds heavy UX friction
     * Fixing griefing by adding a whitelist that centralizes control
     * Fixing a rounding issue by adding precision that proportionally increases gas costs
     * Fixing MEV by adding a delay that equally degrades user experience
   - A trade-off means: the cure is roughly as costly/risky as the disease.
4. These are NOT trade-offs (clean fixes with negligible downside):
   - Standard input validation, access control checks, reentrancy guards
   - Event emissions, proper error messages
   - Bounds checking, overflow protection
   - Correct arithmetic or ordering fixes

## Output Format
**Mitigation Evaluated**: {1-2 sentence description of the mitigation}
**Verdict**: SOLVABLE / TRADE_OFF / UNCERTAIN
**Confidence**: HIGH / MEDIUM / LOW
**Reasoning**: {3-5 sentences explaining why this is solvable or a trade-off}
**If TRADE_OFF, Suggested Severity**: Low / Informational
**If TRADE_OFF, Downside of Fix**: {what the mitigation costs — the equivalent downside}

SCOPE: Evaluate the mitigation only. Return verdict and stop.
")
```

**Result variable**: Store the agent result as `MITIGATION_CHECK`. This is consumed in Step 4D / OUTPUT.

**Result handling** (applied in Step 4D and OUTPUT Section A):
- `TRADE_OFF` + HIGH confidence → apply `MAX_SEVERITY = max(Low, existing MAX_SEVERITY)` (cap at Low, but don't override a stricter cap from Step 2)
- `TRADE_OFF` + MEDIUM confidence → inject into Step 4C judge prompt as advisory context (let the judge weigh it), no auto-cap
- `SOLVABLE` or `UNCERTAIN` → no severity impact
- Agent timeout or failure → treat as `UNCERTAIN`, no severity impact

---

## STEP 3: Generic Invalidation Check (1 selector + 2 checkers + 2 advisory)

### 3A — Selector Agent

> **Wave 1**: launched in parallel with Step 1.5, Step 2.5, and Step 4A. Sonnet selector reads the invalidation library and picks 4 ranked reasons. Top 2 are checked in Wave 2; bottom 2 are passed to Step 4C as "considered alternatives" if the judge fires. Does not depend on the other Wave 1 outputs.

Spawn one agent to select the 4 most applicable invalidation reasons, ranked by confidence:

```
Agent(subagent_type="general-purpose", model="sonnet", prompt="
You are an Invalidation Selector Agent.

## Your Task
Read the invalidation library below and select the 4 reasons MOST APPLICABLE to the
security issue under review, ranked from most-likely-to-hold to least. Do NOT generate
new reasons — only select from the library.

The top 2 of your 4 picks will be verified against the actual code by checker agents.
The bottom 2 will be shown to the final judge as "considered alternatives" so the judge
can see what other invalidation pathways were on the table even if they were not directly
checked. Pick alternatives that are plausibly applicable, not filler.

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
For each of your 4 selections, ordered from rank 1 (highest confidence) to rank 4:
### Selection {N}
- **Rank**: {1, 2, 3, or 4 — 1 and 2 will be checked, 3 and 4 are advisory only}
- **ID**: {e.g., CP-2}
- **Title**: {from library}
- **Confidence**: HIGH / MEDIUM / LOW
- **Why this applies**: {2-3 sentences explaining why this specific reason is relevant to THIS issue}

SCOPE: Select 4 reasons from the library, ranked. Do NOT verify them against code. Return selections and stop.
")
```

### 3B — Parallel Checkers (part of WAVE 2, fires together with Step 4B)

**After WAVE 1 completes** (Steps 1.5, 2.5, 3A, and 4A have all returned):

1. **Split Step 3A's 4 selections.** Take Selections rank 1-2 as `STEP_3A_TOP` (will be checked by 3B). Take Selections rank 3-4 as `STEP_3A_ADVISORY` (passed to Step 4C as considered alternatives, never independently verified).

2. **Filter Step 4A's reasons against ALL 4 of Step 3A's selections.** For each reason from Step 4A, compare against all 4 selections (top + advisory). Drop any reason whose mechanism falls into the same vulnerability category as ANY Step 3A selection (e.g., "missing slippage check" overlaps with library entry CP-2 "input validation"). Keep up to 2 surviving reasons, ranked by the generator's confidence. If Step 4A produced fewer than 2 unique surviving reasons, proceed with what remains (Wave 2 may have <4 agents).

3. **Launch WAVE 2 in a SINGLE message**: spawn 2 Step 3B checkers (one per `STEP_3A_TOP` selection, sonnet) AND up to 2 Step 4B checkers (one per surviving Step 4A reason, opus) — up to 4 agents in parallel.

Step 3B agents use this prompt (one agent per selected reason):

```
Agent(subagent_type="general-purpose", model="sonnet", prompt="
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
Numeric and behavioral claims must be grounded in EVIDENCE, not training-data assumptions.
If your verdict depends on any of the following and you cannot cite a concrete source for it,
your verdict MUST be UNCERTAIN (not HOLDS, not FAILS):

1. External protocol behavior — return values, scaling, exchange rates, redemption mechanics,
   slippage curves. Use the External Protocol Research above if available; trust it over your
   own assumptions. If no research is available and you cannot verify from in-scope code,
   verdict = UNCERTAIN.
2. Numeric claims that must be grounded in actual data:
   - Gas costs ("the attack costs ~50k gas") — must be reasoned from the specific opcodes /
     storage writes in the code, not asserted.
   - Real-world DEX liquidity ("the pool has $X liquidity") — requires on-chain data, not
     a guess.
   - Token decimals ("USDC is 6 decimals on chain Y") — must reference the actual deployed
     token at the relevant address. Do not assume; many tokens have non-standard decimals
     across chains.
   - Oracle parameters ("Chainlink ETH/USD heartbeat is 3600s") — must reference the actual
     feed's documented or on-chain configuration.
   - Fee rates, swap fees, flash-loan fees — must be either documented or read from code.
3. Market or economic claims ("attacker can move price by X%", "MEV searchers will frontrun
   this") — must be supported by liquidity data, a documented invariant, or in-scope code.

"I believe X is Y" or "typically Z" is NOT evidence. On-chain data, official docs, code
citations from in-scope files, or numbers explicitly listed in DOCS_SUMMARY ARE evidence.
When in doubt: UNCERTAIN. UNCERTAIN does not bias the pipeline toward VALID or INVALID;
it correctly signals that this checker could not resolve the question.

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

**EARLY EXIT requires BOTH Step 3B checkers to return `HOLDS`** (unanimous agreement — this prevents false negatives from a single checker mistake).

If both Step 3B checkers returned `HOLDS`:
1. Read each holding checker's evidence and explanation.
2. Use `Read` to independently verify the cited code references.
3. If BOTH checkers' evidence is solid and the logic is sound → **EARLY EXIT**: set verdict to `INVALID` or `DOWNGRADED` based on `Severity Impact`. Discard Step 4B results entirely. Jump to **OUTPUT** with `EXIT_STEP = 3`.
4. If EITHER checker's evidence is weak or has a logical flaw → discard that invalidation, note it as `OVERRULED` in the pipeline trace. With 2 checkers, any overrule breaks unanimity — no early exit.

If fewer than 2 Step 3B checkers returned `HOLDS` (or unanimity broken by overrule) → consume Step 4B results and proceed to Step 4C.

---

## STEP 4: Issue-Specific Adversarial Check (1 generator + 2 checkers + 1 judge)

### 4A — Generator Agent

> **Wave 1**: launched in parallel with Step 1.5, Step 2.5, and Step 3A immediately after Step 2 completes — BEFORE the selector has picked its 2 reasons. The generator therefore does not know what Step 3A will choose; instead, it receives the FULL invalidation library and is told to generate reasons that go BEYOND the library categories.
>
> Duplicate filtering against Step 3A's actual selections happens after Wave 1 completes, in the orchestrator (see Step 3B "Filter rule"), before Wave 2 fires.

```
Agent(subagent_type="general-purpose", model="opus", prompt="
You are an Adversarial Invalidation Generator. Your job is to find ISSUE-SPECIFIC reasons
this issue might be invalid. You run IN PARALLEL with the generic invalidation check — do not
assume the issue has survived or failed generic checking.

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

**Wave 2**: launched in the SAME message as Step 3B's checkers — up to 4 agents in parallel (2 sonnet + 2 opus). Spawn checkers for the top 2 surviving (post-filter) adversarial reasons by confidence. Same prompt pattern as Step 3B, one agent per reason.

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

### 4C — Neutral Judge (symmetric trigger)

> Step 4C is the pipeline's final adversarial review. It is intentionally symmetric: it does NOT only fire when the issue looks invalid. It also fires when the result is ambiguous, so that "VALID" is never a default-to-pass outcome.

**Trigger conditions** (Step 4C fires if ANY of the following is true, AND Step 3C did NOT early-exit):

1. **HOLDS path**: at least one Step 4B checker returned `HOLDS`. Run 4C in `MODE = HOLDS_REVIEW` — assess whether the invalidation should result in INVALID or DOWNGRADE.
2. **CLOSE_CALL path (UNCERTAIN)**: no 4B checker returned HOLDS, but at least one returned `UNCERTAIN`. Run 4C in `MODE = CLOSE_CALL_REVIEW` — the checkers could not resolve at least one reason; the judge must decide whether the issue still holds despite the unresolved doubt.
3. **CLOSE_CALL path (HIGH-confidence FAIL)**: all 4B checkers returned `FAILS`, but at least one of those reasons was generated by Step 4A with `Confidence: HIGH`. Run 4C in `MODE = CLOSE_CALL_REVIEW` — a checker dismissing a HIGH-confidence generator reason is itself a signal worth a final review.

**Skip 4C only when** all three of: every 4B checker returned `FAILS`, no 4A reason had `Confidence: HIGH`, and Step 3A's advisory selections (rank 3-4) do not contradict the issue. In that case the issue is **confirmed VALID** and proceeds to Step 4D → Step 5 → OUTPUT.

```
Agent(subagent_type="general-purpose", model="opus", prompt="
You are the Neutral Judge. You make the FINAL impartial assessment of whether a security
issue is VALID, INVALID, or should be DOWNGRADED.

## Mode
{MODE — one of HOLDS_REVIEW or CLOSE_CALL_REVIEW}

- HOLDS_REVIEW: at least one adversarial checker concluded the issue should be invalidated
  or downgraded. Your job is to verify their reasoning and decide the verdict.
- CLOSE_CALL_REVIEW: no single checker reached HOLDS, but the verdict is not clean. Either
  a checker returned UNCERTAIN (could not resolve), or a HIGH-confidence generator reason
  was rejected by a checker. Your job is to decide, on the evidence, whether the issue
  genuinely holds OR whether the cumulative weight of partial invalidations justifies a
  DOWNGRADE or INVALID.

## Issue Report
{ISSUE_TEXT}

## Step 4B Checker Outcomes (full evidence + verdict for each adversarial reason)
{ALL_4B_RESULTS — verdict, evidence, explanation, severity impact for each reason}

## Step 3A Advisory Selections (rank 3-4, considered but not independently verified)
{STEP_3A_ADVISORY — for each: ID, title, why-this-applies. Use as context for what other
invalidation pathways were on the table. Do NOT treat as confirmed; they were not checked.}

## Protocol Documentation
{DOCS_SUMMARY}

## External Protocol Research (from Step 1.5)
{EXTERNAL_RESEARCH or "No external protocol research was conducted."}

## Out-of-Scope Notes (from Step 1)
{OUT_OF_SCOPE_NOTE or "All referenced files are in scope."}

## Instructions
1. Read the original issue report carefully and understand the claimed attack.
2. Read every 4B checker's evidence — both HOLDS and FAILS and UNCERTAIN. The pattern of
   UNCERTAIN responses tells you where the evidence is genuinely ambiguous.
3. Independently read the relevant source code.
4. Seek counterarguments: is there a way the attack still works despite the strongest
   invalidation reason?
5. Seek design-intent arguments: is the described behavior intentional? Check NatSpec,
   docs, and calling context.
6. Consider Step 3A's advisory selections — were any of them in fact correct, even though
   they were not directly checked?
7. Render your final verdict impartially. Do NOT bias toward VALID. Do NOT bias toward INVALID.
   Follow the evidence.

## ANTI-HALLUCINATION RULE (CRITICAL)
Numeric and behavioral claims must be grounded in EVIDENCE, not training-data assumptions.
The same rule applies to your final verdict: if your reasoning depends on a numeric claim
(gas cost, oracle heartbeat, token decimals, DEX liquidity, fee rate) or external protocol
behavior that you cannot cite from in-scope code, DOCS_SUMMARY, or External Protocol Research,
you must NOT use it as a load-bearing argument. Either find the evidence and cite it, or
do not rely on the claim. If the issue's verdict materially depends on an unverifiable
numeric claim made by the report itself, that is itself grounds for DOWNGRADE (the report
fails its own evidence burden).

## Balanced Standard
- INVALID: the invalidation reason(s) clearly hold AND there is no plausible way the issue
  manifests despite them. In CLOSE_CALL_REVIEW mode, INVALID requires that the cumulative
  weight of partial invalidations renders the issue effectively non-exploitable.
- DOWNGRADE: the issue is real but invalidation reasons or design constraints significantly
  limit severity or scope.
- VALID: invalidation reasons do not hold or are insufficient to negate the issue. In
  CLOSE_CALL_REVIEW mode, VALID requires that the unresolved doubts (UNCERTAIN verdicts)
  do not change the conclusion that the attack works.

Apply your best judgment. Do not default to VALID out of caution — a well-evidenced INVALID
is correct. Do not default to INVALID out of skepticism — a well-evidenced VALID is correct.

## Output Format
**Mode**: {HOLDS_REVIEW or CLOSE_CALL_REVIEW}
**Final Verdict**: VALID / INVALID / DOWNGRADE
**If DOWNGRADE, New Severity**: Critical / High / Medium / Low / Informational
**Justification**: {5-10 sentences, detailed reasoning. In CLOSE_CALL_REVIEW mode, explicitly
address what the UNCERTAIN verdicts or rejected HIGH-confidence reasons signaled and why
that did or did not change your conclusion.}
**Confidence**: HIGH / MEDIUM / LOW

SCOPE: Render your verdict and stop. Do not proceed to other pipeline steps.
")
```

### 4D — Verdict Aggregation (orchestrator inline)

> Step 4D aggregates verdicts from Steps 2, 2.5, and 4C. It does NOT compute severity directly — Step 5 (if it fires) and OUTPUT Section A own that.

After all Step 4 agents complete, gather:
1. **Judge verdict**: INVALID / DOWNGRADE / VALID from Step 4C (or VALID if 4C was skipped).
2. **Mitigation check** (if `MITIGATION_CHECK` has returned by now):
   - `TRADE_OFF` + HIGH confidence → update `MAX_SEVERITY = max(Low, existing MAX_SEVERITY)`. Log: `"Step 2.5: Trade-off detected — severity capped at Low. Downside: {downside_of_fix}"`
   - `TRADE_OFF` + MEDIUM confidence → log advisory note in pipeline trace only, no cap
   - `SOLVABLE` / `UNCERTAIN` / not returned → no change
3. Determine `PRELIM_VERDICT`:
   - If Step 4C says INVALID with HIGH confidence → `PRELIM_VERDICT = INVALID`
   - If Step 4C says INVALID with MEDIUM/LOW confidence → `PRELIM_VERDICT = DOWNGRADED` (treat as a low-conviction invalidation; convert to a downgrade for Step 5 to grade)
   - If Step 4C says DOWNGRADE → `PRELIM_VERDICT = DOWNGRADED` and remember `JUDGE_DOWNGRADE_TARGET` (the severity the judge proposed)
   - If Step 4C says VALID, or Step 4C was skipped → `PRELIM_VERDICT = VALID`
4. If `PRELIM_VERDICT = INVALID` → skip Step 5, jump straight to OUTPUT (severity is N/A for invalid issues).
5. Otherwise (VALID or DOWNGRADED) → proceed to **Step 5: Severity Calibrator**.

---

## STEP 5: Severity Calibrator (orchestrator-direct, post-verdict)

> Step 5 fires only when `PRELIM_VERDICT ∈ {VALID, DOWNGRADED}`. It runs once per issue, sequentially, after Step 4D. It produces an INDEPENDENT severity assessment from the verified attack path — it is not bound by the original claimed severity. This addresses the asymmetry where the previous pipeline could only downgrade severity, never correctly upgrade or recalibrate it.

**Why this exists**: An issue filed as Medium may, on inspection, allow direct theft of arbitrary user funds (Critical). An issue filed as Critical may, on inspection, only affect the attacker's own balance (Informational). The previous `min(claimed, MAX_SEVERITY, downgrade)` logic could not correct under-grading. Step 5 sets severity from the evidence, then OUTPUT Section A clamps it against the Step 2 / Step 2.5 caps.

**Build the calibrator input** from pipeline state:

```
ATTACK_PATH_SUMMARY = orchestrator-built summary, ≤200 words, covering:
  - The attack precondition (state required, role required, market conditions required)
  - The mechanism (what call sequence triggers the bug)
  - The direct effect (what state changes, what value moves)
  - The harmed party (whose funds, whose access, whose state)
  - Anything Step 4C concluded about the attack's boundaries (downgrade reason if any)
  - Anything Step 2.5 concluded about mitigation cost (trade-off vs solvable)
```

**Spawn agent**:

```
Agent(subagent_type="general-purpose", model="sonnet", prompt="
You are the Severity Calibrator. The pipeline has determined that a security issue is
real (verdict = VALID or DOWNGRADED). Your job is to assign severity INDEPENDENTLY of the
severity the report originally claimed, based purely on the verified attack path.

## Verified Attack Path
{ATTACK_PATH_SUMMARY}

## Original Issue Report (for reference, NOT for severity anchoring)
{ISSUE_TEXT}

## Pipeline-Derived Findings
- **Pipeline verdict**: {PRELIM_VERDICT}
- **Judge's downgrade target (if any)**: {JUDGE_DOWNGRADE_TARGET or 'None'}
- **MAX_SEVERITY caps applied (informational only — the orchestrator will re-clamp afterwards)**:
  {list of caps from Step 2 (trusted role) and Step 2.5 (trade-off) with their reasons}
- **Out-of-scope dependencies (if any)**: {OUT_OF_SCOPE_NOTE}

## Protocol Documentation
{DOCS_SUMMARY}

## Roles Summary
{ROLES_SUMMARY}

## Your Task
Score severity from the verified attack path using the rubric below. Do NOT anchor on the
report's claimed severity — many reports under-grade or over-grade. Anchor on what the
attack actually does.

## Severity Rubric

CRITICAL — direct, unconditional theft or loss of user funds, OR permanent freeze of
material funds, OR catastrophic protocol-level invariant break (e.g., infinite mint, total
collateral can be drained). Attack does not require unrealistic preconditions.

HIGH — theft or loss of user funds requires non-trivial but realistic preconditions
(e.g., specific market state, victim must perform a normal action, attacker needs modest
capital), OR temporary freeze of material funds, OR significant protocol-level invariant
break that propagates to user impact.

MEDIUM — material harm to a subset of users under specific conditions, OR loss bounded by
a parameter or rate limit, OR loss requires victim error / non-default usage, OR theft of
fees / yield (not principal) under realistic conditions, OR DoS of an important function
that has a viable workaround.

LOW — small economic harm (rounding, dust accumulation), OR griefing with high attacker
cost relative to victim cost, OR DoS of a non-critical function, OR informational-leak
issues with minor consequences.

INFORMATIONAL — no economic harm, OR self-harm only, OR purely cosmetic, OR a recommendation
without a concrete attack, OR the issue is a known design trade-off where any fix has
equivalent downside.

## ANTI-HALLUCINATION RULE (CRITICAL)
Numeric and behavioral claims must be grounded in EVIDENCE. If your severity grade depends
on a numeric claim (gas cost, oracle heartbeat, token decimals, DEX liquidity, fee rate)
or external protocol behavior that you cannot cite from in-scope code, DOCS_SUMMARY, or
the verified attack path, you must NOT use it as a load-bearing argument. Pick the more
conservative grade when an unverified numeric would otherwise push severity higher.

## Reasoning Discipline
- Identify the harmed party explicitly (which user, which role).
- Identify the attacker's required capital and required preconditions explicitly.
- If the attack only harms the attacker themselves → INFORMATIONAL regardless of the
  technical sophistication.
- If the attack requires a trusted admin to act maliciously → defer to MAX_SEVERITY cap;
  the orchestrator will re-apply Step 2's cap, but flag in your reasoning.
- If the attack is a design trade-off where any fix has equivalent downside → INFORMATIONAL
  or LOW; the orchestrator will re-apply Step 2.5's cap.

## Output Format
**Calibrated Severity**: Critical / High / Medium / Low / Informational
**Confidence**: HIGH / MEDIUM / LOW
**Justification**: {3-6 sentences. State the harmed party, the precondition, the mechanism,
and which rubric tier matches. If you are diverging from the original claimed severity by
more than one tier, explicitly say so and justify why.}
**Divergence from Claim**: {NONE / 1-tier / 2-tier / 3+-tier} — and direction (UPGRADED /
DOWNGRADED) if non-zero.

SCOPE: Calibrate severity from the verified attack path. Return and stop.
")
```

**Result variable**: store the agent result as `CALIBRATED_SEVERITY`, `CALIBRATION_CONFIDENCE`, `CALIBRATION_JUSTIFICATION`, `SEVERITY_DIVERGENCE`. Pass these into OUTPUT Section A.

**Result handling**:
- If the calibrator times out or fails → log "Step 5: calibrator failed, falling back to legacy severity logic" and proceed to OUTPUT Section A using the prior `min(claimed, MAX_SEVERITY, judge_downgrade)` logic as fallback.
- If `CALIBRATION_CONFIDENCE = LOW` AND `SEVERITY_DIVERGENCE ≥ 2-tier` → log a `SEVERITY_DIVERGENCE_WARNING` in the pipeline trace; the calibrated severity still applies, but the trace flags it for human review.

---

## OUTPUT

### A. Finalize Verdict and Severity

The verdict is already determined by Step 4D's `PRELIM_VERDICT`. The severity is determined primarily by Step 5's `CALIBRATED_SEVERITY`, then clamped against caps from Steps 2 and 2.5.

**Verdict**:
- `PRELIM_VERDICT = INVALID` → final verdict = `INVALID`, severity = `N/A`. Skip remaining severity logic.
- `PRELIM_VERDICT = DOWNGRADED` → final verdict = `DOWNGRADED`. Severity is set below.
- `PRELIM_VERDICT = VALID` → final verdict = `VALID`. Severity is set below.

**Severity** (only for non-INVALID verdicts):
1. Start with `final_severity = CALIBRATED_SEVERITY` from Step 5.
2. Apply caps (severity cannot exceed any active cap):
   - If Step 2 set `MAX_SEVERITY = Informational` (trusted-role cap) → `final_severity = min(final_severity, Informational)`
   - If Step 2.5 set `MAX_SEVERITY = Low` (trade-off cap) → `final_severity = min(final_severity, Low)`
3. If verdict is `DOWNGRADED` AND `JUDGE_DOWNGRADE_TARGET` is set AND `JUDGE_DOWNGRADE_TARGET` is stricter than `final_severity` → `final_severity = JUDGE_DOWNGRADE_TARGET`. (The judge can still pull severity down further than the calibrator if it found a specific bounding constraint the calibrator missed.)
4. Compute `SEVERITY_DIVERGENCE` for the trace: compare `final_severity` to `original_claimed_severity` (if any). Mark as `UPGRADED`, `DOWNGRADED`, or `CONFIRMED`.

**Fallback path** (Step 5 failed or was skipped due to error): use the legacy logic:
- Final severity = `min(original_claimed_severity, MAX_SEVERITY, JUDGE_DOWNGRADE_TARGET)`.
- If no severity was claimed → assign via Impact × Likelihood matrix:
  - Impact High + Likelihood High = Critical
  - Impact High + Likelihood Medium = High
  - Impact High + Likelihood Low = Medium
  - Impact Medium + Likelihood High = High
  - Impact Medium + Likelihood Medium = Medium
  - Impact Low + Any = Low
  - Informational = Informational
- Tag the trace with `Step 5: FALLBACK` so the human reviewer knows the calibrator did not run.

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
**Severity Divergence**: CONFIRMED / UPGRADED 1-tier / UPGRADED 2-tier / DOWNGRADED 1-tier / DOWNGRADED 2-tier / DOWNGRADED 3+-tier / N/A
**Pipeline Exit Point**: Step {1|2|3|4|5}
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
- **Step 1 (Initial Sweep)**: {PASSED / FAILED — reason. If applicable: PRIMARY out-of-scope file → INVALID, or SECONDARY out-of-scope dependency → flagged}
- **Step 2 (Privileged Roles)**: {SKIPPED / DOWNGRADED to Informational / NO_ISSUE}
- **Step 2.5 (Mitigation Check)**: {SOLVABLE / TRADE_OFF / UNCERTAIN / TIMEOUT — reason}
- **Step 3 (Generic Check)**: 4 reasons selected (rank 1-4), top 2 checked, {K} held, {J} confirmed; rank 3-4 passed to judge as advisory
- **Step 4 (Adversarial Check)**: {N} reasons generated, {M} checked, judge mode: {HOLDS_REVIEW / CLOSE_CALL_REVIEW / SKIPPED}, judge verdict: {verdict}
- **Step 5 (Severity Calibrator)**: {Calibrated severity, confidence, divergence direction, or FALLBACK / SKIPPED if INVALID}
- **Final Severity**: {severity} {(calibrated from X claimed; clamped by caps Y if applicable)}
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
| Step 2.5 agent times out or fails | Treat as UNCERTAIN, no severity impact |
| Step 2.5 returns TRADE_OFF with MEDIUM confidence | Advisory note in pipeline trace only; let Step 4C judge weigh it |
| All Step 4B checkers return FAILS, no HIGH-confidence 4A reason existed | Skip Step 4C, issue is confirmed VALID |
| Any Step 4B checker returns UNCERTAIN | Fire Step 4C in CLOSE_CALL_REVIEW mode |
| All Step 4B checkers FAIL but a HIGH-confidence 4A reason was rejected | Fire Step 4C in CLOSE_CALL_REVIEW mode |
| PRIMARY referenced file is out of scope | Step 1 EARLY EXIT: INVALID with "out of scope" reason |
| SECONDARY out-of-scope dependency only | Continue pipeline; inject `OUT_OF_SCOPE_NOTE` into 4A and 4C prompts |
| Step 5 calibrator times out or fails | Log "Step 5: FALLBACK"; use legacy `min(claimed, MAX_SEVERITY, judge_downgrade)` logic |
| Step 5 calibration diverges 2+ tiers from claimed with LOW confidence | Apply calibrated severity but tag trace `SEVERITY_DIVERGENCE_WARNING` for human review |
| Step 5 skipped because verdict = INVALID | Severity = N/A; final verdict stands as INVALID |
