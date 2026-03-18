---
phase: 05-skip-level-and-retro-prep
plan: "00"
subsystem: testing

tags: [smoke-tests, skip-level, retro-prep, privacy-gate, pulse-history, wave-0]

requires:
  - phase: 04-1-1-prep
    provides: "phase-4-smoke-tests.md structure and canonical test fixture format"
  - phase: 03-team-pulse
    provides: "phase-3-smoke-tests.md structure and pulse-history format"

provides:
  - "Wave 0 verification contract for all 9 Phase 5 requirements"
  - "13 manual smoke test scenarios for /team-health:skip-level and /team-health:retro-prep"
  - "Canonical pulse history test fixtures (W09, W10, W11 with realistic flag patterns)"

affects:
  - 05-01-skip-level-and-retro-prep

tech-stack:
  added: []
  patterns:
    - "Wave 0 smoke test contract: verification criteria written before command implementation"
    - "Pulse history fixture pattern: 3 snapshot files with realistic signal flags across team members"

key-files:
  created:
    - docs/testing/phase-5-smoke-tests.md
  modified: []

key-decisions:
  - "Wave 0 smoke test contract written before skip-level and retro-prep implementation — same verification-first pattern as Phases 3 and 4"
  - "Asks/Escalations section (Section 4) uses placeholder text only — Claude never pre-populates escalation items; EM fills in manually"
  - "Retro-prep sprint scope fallback: when Jira unavailable, use most recent sprint_cadence_weeks pulse history weeks"
  - "Scenario 5 combines SKIP-03 and SKIP-04 to test opt-in jointly — Alice named with label, all others anonymous"

patterns-established:
  - "Privacy gate scenario pattern: test exclusion by default (Scenario 4), then opt-in (Scenario 5)"
  - "Two-part scenario structure for state write verification: run once (verify default), run again (verify updated anchor)"

requirements-completed:
  - SKIP-01
  - SKIP-02
  - SKIP-03
  - SKIP-04
  - SKIP-05
  - RETRO-01
  - RETRO-02
  - RETRO-03
  - RETRO-04

duration: 2min
completed: 2026-03-17
---

# Phase 5 Plan 00: Skip-Level and Retro Prep — Wave 0 Smoke Tests Summary

**13-scenario manual test contract covering all 9 Phase 5 requirements with pulse history fixtures for privacy gate and aggregation verification**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-17T17:30:43Z
- **Completed:** 2026-03-17T17:33:11Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Created `docs/testing/phase-5-smoke-tests.md` with 13 scenarios covering all 9 SKIP/RETRO requirements plus 2 edge cases
- Established canonical pulse history fixtures (W09, W10, W11) with realistic flag patterns for Alice (W11 YELLOW) and Bob (W09-W10 YELLOW)
- Defined explicit privacy gate verification scenarios (SKIP-03 + SKIP-04) with concrete search-and-verify instructions
- Documented retro-prep work-item attribution check with explicit "search for team member names" pass/fail test
- Followed exact structure of phase-3-smoke-tests.md and phase-4-smoke-tests.md

## Task Commits

Each task was committed atomically:

1. **Task 1: Create phase-5-smoke-tests.md with all 9 requirement scenarios** - `89a46ed` (feat)

**Plan metadata:** (see final commit below)

## Files Created/Modified

- `docs/testing/phase-5-smoke-tests.md` — 13 manual smoke test scenarios for `/team-health:skip-level` and `/team-health:retro-prep`; Wave 0 validation contract for all Phase 5 requirements

## Decisions Made

- **Asks/Escalations section placeholder:** Section 4 of the skip-level brief uses a placeholder only (`[ Add escalation items here before sharing this brief ]`). The plan's RESEARCH.md recommendation was to keep this manual. Scenario 1 tests this explicitly — pre-populated escalation items are a FAIL indicator.
- **Retro-prep sprint scope fallback:** When `sources.jira = false`, Scenario 13 verifies the fallback to most recent `sprint_cadence_weeks` pulse history weeks (derived from RESEARCH.md Open Question 4).
- **Two-part lookback scenario:** Scenario 2 tests SKIP-02 as two sequential sub-runs to verify both the 14-day default (Part A) and the "since last skip-level" behavior (Part B) — the state write in Part A creates the fixture for Part B.

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- `docs/testing/phase-5-smoke-tests.md` is the verification anchor for Plan 05-01 (command implementation)
- All 9 SKIP/RETRO requirements have at least one testable scenario with clear pass/fail criteria
- Canonical fixtures are defined in detail sufficient for any tester to set up without ambiguity
- Edge case scenarios (12 and 13) cover the most common failure modes identified in RESEARCH.md

---
*Phase: 05-skip-level-and-retro-prep*
*Completed: 2026-03-17*
