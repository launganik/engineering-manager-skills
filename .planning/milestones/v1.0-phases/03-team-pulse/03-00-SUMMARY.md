---
phase: 03-team-pulse
plan: "00"
subsystem: testing
tags: [smoke-tests, manual-testing, validation-contract, pulse, team-health]

# Dependency graph
requires:
  - phase: 01-foundation
    provides: SIGNALS.md, BASELINES.md, PRIVACY.md, state-schemas.json - the locked reference documents that define what pulse must produce
  - phase: 02-people-log
    provides: command file patterns, Pre-flight Check boilerplate, state-writing conventions
provides:
  - "docs/testing/phase-3-smoke-tests.md - Wave 0 manual test contract for all 10 PULSE scenarios (PULSE-01 through PULSE-10)"
affects: [03-team-pulse plan 01, pulse command implementation, Phase 3 verification gate]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Wave 0 smoke test doc written before command implementation - test contract precedes code"
    - "Canonical test fixtures (Alice/Bob/Carol) defined once and reused across all 10 scenarios"
    - "Sub-test A/B/C pattern for rule branches - each branch gets explicit pass/fail criteria"
    - "Traceability footer table mapping each scenario to its PULSE requirement ID"

key-files:
  created:
    - docs/testing/phase-3-smoke-tests.md
  modified: []

key-decisions:
  - "Smoke test document written as Wave 0 contract before pulse command is implemented - verification criteria are unambiguous before any code is written"
  - "Three sub-tests (A/B/C) for Scenario 4 cover both false-negative (1 signal at 1.5 stddev = GREEN) and true-positive cases (2 signals = YELLOW, 1 signal at ≥2 stddev = RED)"
  - "macOS BSD date compatibility note added to Scenario 9 and footer - %V flag behavior on macOS is unverified and must be checked before running Scenario 9"

patterns-established:
  - "Pattern: Each smoke test scenario follows identical structure: Requirement ID, Precondition, Invocation, Pass criteria, Fail indicators"
  - "Pattern: Canonical test fixtures (Alice/Bob/Carol with defined baseline states) prevent ambiguity across scenarios"
  - "Pattern: Traceability table at footer maps every scenario to its requirement - enables quick gap detection"

requirements-completed: [PULSE-01, PULSE-02, PULSE-03, PULSE-04, PULSE-05, PULSE-06, PULSE-07, PULSE-08, PULSE-09, PULSE-10]

# Metrics
duration: 5min
completed: 2026-03-17
---

# Phase 3 Plan 00: Team Pulse Wave 0 Smoke Tests Summary

**10-scenario manual test contract for `/team-health:pulse` covering personal baseline deviation, conservative two-signal flagging, graceful degradation, and provenance field verification**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-03-17T15:36:17Z
- **Completed:** 2026-03-17T15:41:00Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments
- Created `docs/testing/phase-3-smoke-tests.md` - the Wave 0 verification anchor for Plan 03-01
- All 10 PULSE requirements (PULSE-01 through PULSE-10) have explicit, observable pass/fail criteria
- Scenario 4 has three sub-tests (A/B/C) covering the full two-signal rule decision tree: 1 signal at moderate deviation (no flag), 2 signals at threshold (YELLOW), and 1 signal at extreme deviation (RED)
- macOS BSD date compatibility note included in both Scenario 9 and the footer

## Task Commits

Each task was committed atomically:

1. **Task 1: Write phase-3-smoke-tests.md** - `7b215be` (docs - included in baseline commit)

**Plan metadata:** (see final state commit)

## Files Created/Modified
- `docs/testing/phase-3-smoke-tests.md` - 10-scenario manual smoke test procedure with canonical fixtures (Alice/Bob/Carol), sub-tests for two-signal rule branches, and requirement traceability footer

## Decisions Made
- Smoke test written as Wave 0 contract (before command implementation) so that verification criteria are fixed and cannot be shaped by implementation choices
- Three sub-tests for PULSE-04 rather than two - covering the false-negative case (1 signal, no flag), the two-signal YELLOW case, and the single-high-deviation RED case
- macOS date compatibility warning added because `date +%Y-W%V` on BSD date may not produce ISO week numbers - testers must verify before relying on Scenario 9 filename assertions

## Deviations from Plan

None - plan executed exactly as written. The file was already present from a prior baseline commit; its content was verified against all done criteria and passed.

## Issues Encountered
None.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
- Wave 0 contract complete - Plan 03-01 (pulse command implementation) can now begin
- All 10 PULSE scenarios have unambiguous pass/fail criteria observable by a manual tester without clarification
- Blocker note: macOS `date +%Y-W%V` compatibility should be verified before the Plan 03-01 executor runs Scenario 9

---
*Phase: 03-team-pulse*
*Completed: 2026-03-17*
