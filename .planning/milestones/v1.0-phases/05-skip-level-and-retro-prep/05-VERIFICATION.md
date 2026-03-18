---
phase: 05-skip-level-and-retro-prep
verified: 2026-03-17T00:00:00Z
status: human_needed
score: 12/12 must-haves verified
human_verification:
  - test: "Run /team-health:skip-level with no --include-person flag; inspect output for any individual names or people log content"
    expected: "Zero individual names in Sections 2-3; no people log content anywhere in output"
    why_human: "Privacy gate is structural (no Read call to .team-health/people/ by default) but only live execution against a seeded fixture confirms Claude honors the gate during output generation"
  - test: "Run /team-health:skip-level --include-person Alice; verify (opt-in content) label appears in People Themes and Bob/Carol remain unnamed"
    expected: "Alice's log themes labeled '(opt-in content)'; Bob and Carol referenced only by count language throughout"
    why_human: "Label placement and exclusion of non-opted-in members cannot be confirmed without live output"
  - test: "Run /team-health:skip-level twice in sequence; verify config.json last_skip_level is set after first run and used as lookback anchor in second run"
    expected: "Part A: last_skip_level written to config.json; Part B: output references 'since last skip-level on [date]'"
    why_human: "Phase E state write and subsequent lookback window derivation require a live session with a real config.json"
  - test: "Run /team-health:retro-prep with sources.github=true, sources.jira=false; inspect Sprint Facts section"
    expected: "Sprint Facts shows degradation language for Jira data; GitHub PR data populates PR sections; sprint window attributed to pulse history weeks"
    why_human: "Degradation language rendering and source-conditional section content require live MCP state"
  - test: "Inspect Action Items section of retro-prep output; verify only blank checkboxes, no pre-populated items"
    expected: "Three blank '- [ ]' checkboxes, no text after them"
    why_human: "Output content depends on live sprint data; only human can confirm Claude does not generate action item text"
---

# Phase 5: Skip-Level and Retro Prep Verification Report

**Phase Goal:** Managers can generate an upward-facing team brief for their own manager and a data-seeded sprint retro agenda - both consuming pulse history with explicit privacy gates that prevent people log content from surfacing without opt-in
**Verified:** 2026-03-17
**Status:** human_needed - all automated checks passed; 5 behavioral tests require live execution
**Re-verification:** No - initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Every Phase 5 requirement (SKIP-01 through SKIP-05, RETRO-01 through RETRO-04) has at least one smoke test scenario | VERIFIED | 13 scenarios in phase-5-smoke-tests.md; all 9 req IDs each have 3+ matches in the doc |
| 2 | Each scenario has preconditions, steps, and expected results specific enough for a human tester to execute | VERIFIED | All 13 scenarios follow "Preconditions / Invocation / Pass criteria / Fail indicators" structure; concrete fixture paths specified |
| 3 | Smoke test doc follows the same structure as phase-3-smoke-tests.md and phase-4-smoke-tests.md | VERIFIED | Header block, Prerequisites, Canonical Test Fixtures, numbered scenarios, Traceability table - same pattern |
| 4 | Manager can run /team-health:skip-level and receive a 5-section team brief | VERIFIED | Phase D output format in skip-level.md specifies exactly 5 numbered sections: Delivery Status, Risks and Watch Areas, People Themes, Asks/Escalations, Team Wins |
| 5 | People log content never appears in output unless --include-person is explicitly passed | VERIFIED (structure) | Privacy Gate section explicitly blocks Read to .team-health/people/ by default; requires human confirmation of runtime behavior |
| 6 | Individual names never appear in pulse aggregations without --include-person opt-in | VERIFIED (structure) | Phase C aggregation rules prohibit carrying forward names from Flagged Members sections; Compliance Check step scans for violations; requires human confirmation |
| 7 | Lookback window defaults to since last skip-level (or 14-day fallback) and supports --weeks and --quarter overrides | VERIFIED | Phase A documents priority chain: --quarter > --weeks N > last_skip_level > 14-day fallback; all three branches specified |
| 8 | config.json last_skip_level is updated after output is rendered | VERIFIED (structure) | Phase E writes last_skip_level after Phase D output; order enforced in command text; requires live execution to confirm |
| 9 | Manager can run /team-health:retro-prep and receive a retro agenda seeded with sprint facts | VERIFIED | Phase D output format specifies Sprint Facts + 4 content sections; Jira/GitHub collection in Phase B |
| 10 | All discussion seeds are attributed to work items (PRs, tickets), never to individuals | VERIFIED (structure) | Phase C Attribution Check explicitly rewrites person names to work-item references before output; requires human confirmation of runtime behavior |
| 11 | Every content section has blank discussion space for the team | VERIFIED | "[ Team discussion space - ... ]" markers present in What Went Well, What Was Hard, and Patterns Across Sprints sections |
| 12 | Action items section is structurally blank (empty checkboxes only) | VERIFIED | Action Items section contains only 3 blank "- [ ]" checkboxes; command instructions prohibit pre-populating items |

**Score:** 12/12 truths structurally verified; 5 require human execution to confirm runtime behavior

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `docs/testing/phase-5-smoke-tests.md` | Manual smoke test scenarios for all Phase 5 requirements | VERIFIED | 487 lines; 13 scenarios; all 9 req IDs present with 3+ matches each; Wave 0 validation contract header confirmed |
| `.claude/commands/team-health/skip-level.md` | Skip-level command file with Privacy Gate | VERIFIED | 206 lines; all acceptance criteria patterns confirmed present (see detail below) |
| `.claude/commands/team-health/retro-prep.md` | Retro prep command file | VERIFIED | 199 lines; all acceptance criteria patterns confirmed present; Write absent from allowed-tools |
| `SKILL.md` | Updated installation guide with all Phase 5 commands available | VERIFIED | 213 lines; 0 "not yet available" labels; skip-level.md and retro-prep.md in ls output; Privacy Design section preserved |

### skip-level.md Pattern Verification

All 17 acceptance criteria patterns confirmed via grep:

| Pattern | Count |
|---------|-------|
| Privacy Gate | 2 |
| Do NOT read any files under .team-health/people/ | 1 |
| include-person | 13 |
| Pre-flight Check | 3 |
| SIGNALS.md | 1 |
| BASELINES.md | 1 |
| PRIVACY.md | 3 |
| Delivery Status | 1 |
| Risks and Watch Areas | 1 |
| People Themes | 2 |
| Asks / Escalations | 1 |
| Team Wins | 1 |
| last_skip_level | 5 |
| Phase E | 2 |
| N team members | 4 |
| opt-in content | 4 |
| Compliance Check | 2 |

### retro-prep.md Pattern Verification

All acceptance criteria patterns confirmed via grep:

| Pattern | Count |
|---------|-------|
| Pre-flight Check | 1 |
| PRIVACY.md | 1 |
| Sprint Facts | 2 |
| What Went Well | 1 |
| What Was Hard | 1 |
| Patterns Across Sprints | 1 |
| Action Items | 2 |
| Team discussion space | 4 |
| Team fills in during retro | 1 |
| attributed to work items, not individuals | 2 |
| sprint_cadence_weeks | 2 |
| Write absent from allowed-tools | CONFIRMED - allowed-tools line: Read, Bash(date), Bash(ls) only |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| skip-level.md | .team-health/config.json | Read in Pre-flight Check; Write in Phase E | WIRED | 7 references to config.json; Pre-flight Check reads it; Phase E writes it with all fields |
| skip-level.md | .team-health/pulse-history/ | Bash ls + Read in Phase B | WIRED | 2 references to pulse-history; Phase B runs Bash(ls .team-health/pulse-history/) and reads qualifying files |
| skip-level.md | .claude/team-health/PRIVACY.md | Required Reference Reads | WIRED | PRIVACY.md referenced in Required Reference Reads section (line 43); Compliance Check cites PRIVACY.md Rules 1 and 3 |
| retro-prep.md | .team-health/pulse-history/ | Bash ls + Read in Phase B | WIRED | 2 references to pulse-history; Phase B runs Bash(ls .team-health/pulse-history/) and reads qualifying files |
| retro-prep.md | .claude/team-health/PRIVACY.md | Required Reference Reads | WIRED | PRIVACY.md in Required Reference Reads section (line 24) |
| SKILL.md | .claude/commands/team-health/skip-level.md | Documentation reference | WIRED | skip-level.md appears in installation ls output and in /team-health:skip-level command section |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|------------|-------------|--------|----------|
| SKIP-01 | 05-00, 05-01 | /team-health:skip-level generates 5-section upward-facing brief | SATISFIED | Phase D output format in skip-level.md has exactly 5 numbered sections; Scenario 1 in smoke tests verifies header names |
| SKIP-02 | 05-00, 05-01 | Default timeframe "since last skip-level"; supports --weeks / --quarter overrides | SATISFIED | Phase A priority chain documents all 3 override paths; Phase E writes last_skip_level after output; Scenarios 2-3 verify both paths |
| SKIP-03 | 05-00, 05-01 | People log excluded by default; only unlocked by --include-person flag | SATISFIED | Privacy Gate section prohibits reading .team-health/people/ without --include-person; Scenarios 4-5 verify both default and opt-in |
| SKIP-04 | 05-00, 05-01 | Individual pulse flags aggregated into anonymous themes | SATISFIED | Phase C aggregation rule: "Do NOT carry forward individual names"; Compliance Check enforces; Scenarios 5-6 verify |
| SKIP-05 | 05-00, 05-01 | Output passes "would my team be comfortable seeing this?" test | SATISFIED | Compliance Check scans for individual names, diagnostic language, team-relative scoring before finalizing; Scenario 7 verifies |
| RETRO-01 | 05-00, 05-02 | /team-health:retro-prep produces retro agenda seeded with sprint data | SATISFIED | Phase B collects GitHub/Jira/pulse data; Phase D renders Sprint Facts + 4 sections; Scenario 8 verifies |
| RETRO-02 | 05-00, 05-02 | Output includes sprint facts, went well, was hard, patterns sections | SATISFIED | Phase D output format has all 4 content sections plus Action Items in specified order; Scenario 9 verifies |
| RETRO-03 | 05-00, 05-02 | Discussion seeds attributed to work items, not individuals | SATISFIED | Phase C Attribution Check rewrites person names to work-item references; Scenario 10 verifies search-and-find test |
| RETRO-04 | 05-00, 05-02 | Blank discussion space preserved; Action Items blank | SATISFIED | "Team discussion space" markers in 3 content sections; Action Items has only blank checkboxes; Scenario 11 verifies |

All 9 requirements accounted for. No orphaned requirements found - all SKIP/RETRO IDs are claimed by plans 05-00, 05-01, and 05-02.

---

## Anti-Patterns Found

No anti-patterns detected. Grep scans for TODO, FIXME, XXX, HACK, PLACEHOLDER, "coming soon", `return null`, `return {}` across all three primary artifacts returned zero matches.

---

## Human Verification Required

All automated structural checks passed. The following items require a live Claude Code session with Phases 1-4 setup complete and pulse history fixtures populated (W09, W10, W11 as defined in phase-5-smoke-tests.md Canonical Test Fixtures).

### 1. Privacy Gate - Default Exclusion (Scenario 4)

**Test:** Run `/team-health:skip-level` with no `--include-person` flag. Alice Chen has a people log with entries containing "Staff Engineer track", "platform team lead", and "auth refactor".
**Expected:** Zero occurrences of any of those phrases, or any content from alice-chen.json, appear in the output.
**Why human:** The Privacy Gate is structurally correct - the command instructs Claude not to call Read on .team-health/people/ - but only live execution confirms Claude honors that instruction during output generation.

### 2. Privacy Gate - Opt-in Labeling (Scenario 5)

**Test:** Run `/team-health:skip-level --include-person Alice`. Bob and Carol have pulse history entries.
**Expected:** Alice's log themes appear in People Themes labeled "(opt-in content)". Bob and Carol are referenced only by count language throughout the full output.
**Why human:** Correct exclusion of non-opted-in members and correct labeling of opt-in content can only be verified from live output.

### 3. State Write Discipline (Scenario 2)

**Test:** Run `/team-health:skip-level` twice sequentially. Check config.json between runs.
**Expected:** After first run, `last_skip_level` is set to today's date. Second run states "since last skip-level on [that date]" as the lookback anchor.
**Why human:** Phase E write order (after Phase D output) and the subsequent lookback derivation in Part B require a live session with a real config.json.

### 4. Retro-prep Jira Degradation (Scenario 13)

**Test:** Run `/team-health:retro-prep` with `sources.jira=false` and `sources.github=true`. Two pulse history weeks present.
**Expected:** Sprint Facts shows degradation language for velocity/carry-over ("Ticket velocity unavailable - Jira MCP not configured"). Sprint window stated as "last N weeks from pulse history". GitHub PR data populates PR sections.
**Why human:** Source-conditional rendering and degradation language require live MCP state to confirm.

### 5. Action Items Blank (Scenario 11)

**Test:** Run `/team-health:retro-prep` with standard fixtures. Examine Action Items section in full output.
**Expected:** Section contains exactly 3 blank `- [ ]` checkboxes and no pre-populated text.
**Why human:** Whether Claude generates action item text during live sprint data synthesis cannot be confirmed statically - the command prohibits it structurally but live confirmation is needed.

---

## Commit Verification

All 4 commits documented in summaries confirmed present in git log:

| Commit | Plan | Description |
|--------|------|-------------|
| `89a46ed` | 05-00 | feat: add phase-5 smoke test contract for skip-level and retro-prep |
| `e0dd78a` | 05-01 | feat: implement /team-health:skip-level command |
| `0c9ba44` | 05-02 | feat: create retro-prep.md command file |
| `149865d` | 05-02 | feat: update SKILL.md to mark all commands as available |

---

## Summary

Phase 5 goal is structurally achieved. All 4 artifacts exist, are substantive (206, 199, 487, and 213 lines respectively), and are correctly wired to their dependencies. All 9 requirements (SKIP-01 through SKIP-05, RETRO-01 through RETRO-04) have explicit coverage in the smoke test contract and in the command implementations.

The privacy gates for both skip-level (Privacy Gate section + Compliance Check) and retro-prep (Phase C Attribution Check) are implemented as named, explicit steps - not assumptions. The SKILL.md installation guide reflects all 6 commands as available with zero "not yet available" labels remaining.

5 behavioral tests require a live Claude Code session to confirm that the structural prohibitions (privacy gate, aggregation rules, state write order, degradation language, action item prohibition) hold during actual runtime execution.

---

_Verified: 2026-03-17_
_Verifier: Claude (gsd-verifier)_
