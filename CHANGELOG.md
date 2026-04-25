# Changelog

All notable changes to Judge V1.1 are documented in this file.

---

## [1.1] - 2026-04-25

V1.1 adds two new pipeline stages on top of V0: a mitigation viability check that caps design trade-off issues at Low, and a post-verdict severity calibrator that re-grades from the verified attack path so under-graded findings can be upgraded, not only downgraded. The Step 4C judge now fires symmetrically (on HOLDS, UNCERTAIN, or rejected high-confidence reasons), and the anti-hallucination guard covers all numeric claims (gas, liquidity, decimals, oracles, fees), not just external protocol behavior. PRE-STEP auto-detects docs, scope, and roles across Solidity, Solana, and Move, and Step 1 rejects primary out-of-scope findings before the pipeline runs.

Per-version detail in entries 0.2 through 0.4 below.

---

## [0.4] - 2026-04-25

### Added
- Step 5: Severity Calibrator (sonnet, sequential, post-verdict). Runs on every VALID or DOWNGRADED outcome and computes severity independently from the verified attack path, using a structured rubric (Critical → Informational tiers anchored on harmed party, precondition realism, and effect type). Replaces the previous `min(claimed, MAX_SEVERITY, judge_downgrade)` logic, which could only ratchet severity downward and therefore could not correct under-graded reports. Step 2 (trusted role) and Step 2.5 (trade-off) caps still apply on top of the calibrated severity.
- Step 4C close-call mode: judge now fires symmetrically. In addition to the existing HOLDS path, the judge also fires when any 4B checker returns UNCERTAIN (no clean rejection of the issue) or when all 4B checkers FAIL but a HIGH-confidence 4A reason was rejected. Adds a new `MODE` field (HOLDS_REVIEW / CLOSE_CALL_REVIEW) to the judge prompt and reasoning.
- Step 3A advisory selections: selector now picks 4 ranked reasons (was 2). Top 2 are checked by 3B; ranks 3-4 are passed to the Step 4C judge as "considered alternatives" so the judge sees what other invalidation pathways were on the table even though they were not directly verified. The Step 4A duplicate filter compares against all 4 selections.
- Step 1 PRIMARY vs SECONDARY out-of-scope distinction: a primary reference (the file/function cited as the bug location) being out of scope now triggers an early INVALID exit. Secondary references (interaction points the in-scope code calls) continue the pipeline but inject an `OUT_OF_SCOPE_NOTE` into the 4A and 4C prompts so reasoning about them is held to the same evidence standard as external protocols.
- Severity divergence reporting: trace now includes a `Severity Divergence` field (CONFIRMED / UPGRADED N-tier / DOWNGRADED N-tier) and a `SEVERITY_DIVERGENCE_WARNING` tag when the calibrator diverges 2+ tiers from the claimed severity with LOW confidence, surfacing borderline calls for human review.

### Changed
- Anti-hallucination rule (3B, 4B, and now 4C and Step 5) broadened from "external protocol behavior only" to also cover numeric claims that cannot be derived from in-scope code or `DOCS_SUMMARY`: gas costs, real-world DEX liquidity, token decimals at specific deployments, oracle heartbeat / deviation thresholds, and fee rates. Verdicts depending on unverifiable numerics must return UNCERTAIN. Step 5 picks the more conservative grade when an unverified numeric would otherwise push severity higher; Step 4C may DOWNGRADE if the report itself relies on an unverifiable numeric.
- Pipeline overview diagram updated to include Step 4D (verdict aggregation) and Step 5, and to describe the Step 4C trigger as symmetric.
- Step 4D renamed from "Final Severity Assessment" to "Verdict Aggregation". Step 4D no longer computes severity; it determines `PRELIM_VERDICT` and routes to Step 5 (or to OUTPUT directly if INVALID).
- OUTPUT Section A rewritten: severity is `CALIBRATED_SEVERITY` clamped against active caps, with a fallback to legacy `min(...)` logic if Step 5 fails. Pipeline trace template adds Step 5 line and exit-point now ranges 1-5.
- CSV per-issue subagent prompt updated to include Step 2.5 (was missing from step list) and Step 5; Wave 1 reference now correctly mentions all four parallel agents.
- Worst-case agent count per issue documented as up to 8 (was 7), reflecting Step 5 addition.
- Invalidation library header updated from "picks the 2" to "picks the 4 (ranked); top 2 verified, bottom 2 advisory to judge".

### Fixed
- Step 1.5 wave-reference text now lists all four Wave 1 agents (1.5, 2.5, 3A, 4A); previously omitted 2.5.
- Step 3 section header count corrected: "1 selector + 2 checkers + 2 advisory" (was "1 selector + 2 checkers"), reflecting that ranks 3-4 now have a defined downstream consumer.

---

## [0.3] - 2026-04-12

### Added
- **PRE-STEP Auto-Detection** — Automatically scans the working directory for documentation files (WHITEPAPER.md, SPEC.md, DESIGN.md, docs/, README.md, PDFs), scope files (scope.txt, scope.md, foundry.toml/hardhat source dirs, standard contract directories), and role/trust files before prompting the user. Supports Solidity, Solana (Anchor), and Move (Aptos/Sui) projects.
- **Per-field user confirmation with override** — Each auto-detected value (docs, scope, roles) is presented to the user for confirmation with the option to use a different file, provide free text, or skip. Auto-detection is never silently accepted.
- **Session context reuse** — If `.validation_context/` already exists from a prior run, the user is offered three options: reuse as-is, selectively update specific fields, or start fresh. Eliminates re-entering the same context across multiple `/judge` invocations in the same working directory.
- **Multi-language scope detection** — Auto-detect phase recognizes `.sol` (Solidity), `.rs` + `Anchor.toml` (Solana), and `.move` + `Move.toml` (Aptos/Sui) projects and adjusts scope resolution accordingly.

### Changed
- PRE-STEP user interaction reduced from 3-6 rounds (worst case) to 1-3 confirmations when auto-detection succeeds
- Scope resolution logic extracted into its own subsection, shared between auto and manual flows
- Manual input flow preserved as fallback for fields not auto-detected or when user chooses "Set everything manually"

---

## [0.2] - 2026-04-09

### Added
- **Step 2.5: Mitigation Viability Check** — New background agent (sonnet) that runs in Wave 1 alongside Steps 1.5, 3A, and 4A. Evaluates whether a reported issue has a clean fix or represents an inherent design trade-off where any mitigation introduces equivalent downsides. Trade-off issues with HIGH confidence are capped at Low severity.
- Step 2.5 result consumption in Step 4D and OUTPUT Section A
- Step 2.5 line in pipeline trace template
- Edge case handling for Step 2.5 timeout and MEDIUM confidence results

### Changed
- Pipeline overview diagram updated to show Step 2.5 in Wave 1
- Wave 1 now launches 4 parallel agents (was 3)
- CSV subagent prompt updated to include Step 2.5 in pipeline sequence
- **Step 2 trusted role early exit**: downgrade severity changed from Low to Informational. Pure "trusted role can rug" issues now exit as INVALID or Informational (was INVALID or Low)
- **Step 3B checkers downgraded from opus to sonnet** — generic invalidation checking is a focused code verification task, sonnet is sufficient
- **Checker count reduced from 3+3 to 2+2** — Step 3A now selects 2 reasons (was 3), Step 3B spawns 2 checkers (was 3), Step 4B spawns up to 2 checkers (was 3). Wave 2 drops from 6 agents to 4. Step 3C early exit threshold changed from 2-of-3 majority to 2-of-2 unanimous. Total worst-case agents per issue: 7 (was 11)

### Fixed
- **Stale "select 3" in 3A inner prompt** — agent prompt text now says "select 2" matching the outer instructions
- **Stale "picks the 5" in invalidation-library.md header** — updated to "picks the 2"
- **Misleading 4A prompt** — removed "survived generic invalidation checking" (4A runs in parallel with 3A, not after it)
- **Wave 1 launch text** — now references all 4 agents including Step 2.5
- **Step 3C overrule logic** — made explicit that with 2 checkers, any overrule breaks unanimity (no early exit)
- **Duplicated severity logic** — Step 4D now references OUTPUT Section A as canonical; no longer duplicates the severity matrix
- **Missing 4B FAILS/UNCERTAIN handling** — added explicit "skip 4C, issue is VALID" when no 4B checker returns HOLDS
- **Outdated model description** — header line now accurately lists which models are used where
- **Step 2.5 missing from project file** — all Step 2.5 content (section, diagram, pipeline trace, edge cases, 4D/OUTPUT consumption) now present in the project-local SKILL.md

---

## [0.1] - Pre-2026-04-09

Initial release of The Judge validation pipeline.

- Mode selection (single issue / CSV batch)
- Pre-step context collection (docs, roles, scope, external integrations)
- Step 1: Initial sweep with early exit
- Step 1.5: External protocol research (Wave 1, parallel)
- Step 2: Privileged roles check with early exit
- Step 3A: Generic invalidation selector (sonnet)
- Step 3B: 3 parallel generic invalidation checkers (opus)
- Step 3C: Orchestrator confirmation with early exit
- Step 4A: Issue-specific adversarial generator (opus)
- Step 4B: 3 parallel adversarial checkers (opus)
- Step 4C: Neutral judge (opus)
- Step 4D: Final severity assessment
- Output: console (single) and file (CSV batch) modes
- Batch notes and external research caching for CSV mode
- Invalidation library reference
