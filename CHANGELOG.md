# Changelog

All notable changes to The Judge skill are documented in this file.

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
