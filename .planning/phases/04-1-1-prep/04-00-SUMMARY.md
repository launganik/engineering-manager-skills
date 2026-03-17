---
phase: 04-1-1-prep
plan: "00"
subsystem: testing

tags: [smoke-tests, verification-contract, team-health, prep, 1on1]

requires:
  - phase: 03-team-pulse
    provides: "smoke test format and pattern — Scenarios numbered per requirement, canonical fixtures, sub-tests A/B/C structure"
  - phase: 02-people-log
    provides: "people log schema (entries, open_commitments, career_context) used in fixture definitions"
  - phase: 01-foundation
    provides: "SIGNALS.md, PRIVACY.md graceful degradation language, BASELINES.md — all referenced verbatim in pass criteria"

provides:
  - "Wave 0 verification contract for Plan 04-01 (prep command implementation)"
  - "13 numbered scenarios: PREP-01 through PREP-10 plus 3 edge cases (first use, prep_run lookback, last_1on1_date lookback)"
  - "Canonical test fixtures: Alice (full data), Bob (partial), Carol (new — no file)"
  - "Enumerated prohibited diagnosis words for Scenario 9 (PREP-09)"
  - "Verbatim PRIVACY.md degradation phrases quoted per scenario (4A, 5A, 6A, 10)"

affects:
  - "04-1-1-prep Plan 04-01 (prep command executor verifies against this document)"

tech-stack:
  added: []
  patterns:
    - "Wave 0 contract pattern: smoke test document written before command implementation — all prior phases follow this"
    - "Sub-test A/B pattern for source-enabled/disabled scenarios (borrowed from Phase 3 Scenario 4)"
    - "Edge case triad: first use / prep_run fallback / last_1on1_date primary — covers full lookback priority chain"

key-files:
  created:
    - docs/testing/phase-4-smoke-tests.md
  modified: []

key-decisions:
  - "Scenario 11-13 cover the lookback priority chain in order (first-use → prep_run → last_1on1_date) so each step is independently testable"
  - "Canonical Alice fixture includes 3 concern entries in 6 weeks to enable Scenario 9 log pattern detection without additional setup"
  - "Scenarios 4/5/6 use A/B sub-test structure to test both the degradation path and the happy path for each source"

patterns-established:
  - "Pattern 1: Acceptance criteria verified inline (grep commands) before committing document"
  - "Pattern 2: Prohibited words enumerated explicitly in no-diagnosis scenario rather than referencing PRIVACY.md by pointer"

requirements-completed:
  - PREP-01
  - PREP-02
  - PREP-03
  - PREP-04
  - PREP-05
  - PREP-06
  - PREP-07
  - PREP-08
  - PREP-09
  - PREP-10

duration: 2min
completed: 2026-03-17
---

# Phase 4 Plan 00: 1:1 Prep Smoke Tests Summary

**Wave 0 verification contract for `/team-health:prep` — 13 scenarios covering PREP-01 through PREP-10 plus first-use, prep_run lookback, and last_1on1_date lookback edge cases**

## Performance

- **Duration:** 2 min
- **Started:** 2026-03-17T16:38:58Z
- **Completed:** 2026-03-17T16:41:00Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Created `docs/testing/phase-4-smoke-tests.md` with 13 scenarios following the exact format of `phase-3-smoke-tests.md`
- All 10 PREP requirement IDs (PREP-01 through PREP-10) appear mapped to individual scenarios
- Canonical test fixtures defined: Alice (full data with `last_1on1_date`, open commitments, career context, 3 concern entries), Bob (partial — `prep_run` only, no open commitments), Carol (new — no people log file)
- Verbatim PRIVACY.md graceful degradation phrases quoted in Scenarios 4A, 5A, 6A, and 10 to enable character-for-character verification
- Lookback priority chain (first-use → `prep_run` → `last_1on1_date`) covered by Scenarios 11, 12, 13 in order

## Task Commits

1. **Task 1: Create Phase 4 smoke test document** - `ab84274` (feat)

**Plan metadata:** [pending — final metadata commit]

## Files Created/Modified

- `docs/testing/phase-4-smoke-tests.md` — Wave 0 verification contract: 13 scenarios, canonical fixtures, traceability table

## Decisions Made

- Scenarios 11–13 were ordered to test the lookback chain from simplest (no data) to most specific (last_1on1_date takes priority over prep_run). This order makes each scenario independently testable without overlap.
- Alice's canonical fixture includes 3 `concern` entries in the past 6 weeks so Scenario 9 (PREP-09 no-diagnosis check) can trigger the log pattern detection path without extra setup instructions.
- The `**Phase: 4**` frontmatter line was formatted to match the literal substring `Phase: 4` required by the plan's acceptance criteria grep.

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

Minor: the plan's acceptance criterion `grep "Phase: 4"` required the literal substring `Phase: 4` in the document. The initial draft used `**Phase:** 4` (following Markdown bold convention), which would fail the grep. Fixed by reformatting to `**Phase: 4**` — cosmetically equivalent in rendered Markdown and passes the grep check.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness

- Wave 0 verification contract complete
- Plan 04-01 executor has a full set of scenarios to verify the `/team-health:prep` command against before and after implementation
- No blockers for Plan 04-01

---
*Phase: 04-1-1-prep*
*Completed: 2026-03-17*

## Self-Check: PASSED

- FOUND: `docs/testing/phase-4-smoke-tests.md`
- FOUND: `.planning/phases/04-1-1-prep/04-00-SUMMARY.md`
- FOUND: commit `ab84274` — feat(04-1-1-prep-00): add Phase 4 smoke test document
