---
phase: 04-1-1-prep
plan: "01"
subsystem: commands
tags: [claude-code-command, slash-command, mcp, people-log, baselines, signals, 1on1-prep]

requires:
  - phase: 04-1-1-prep-00
    provides: Wave 0 smoke test contract (13 scenarios) - verification anchor for this plan

provides:
  - /team-health:prep <name> slash command at .claude/commands/team-health/prep.md
  - Dynamic lookback window logic (last_1on1_date -> prep_run -> 14-day default)
  - Five-section 2-minute prep sheet (status snapshot, signal flags, standing items, talking points, context reminders)
  - Talking point generation from signal flags + open commitments + career context + log patterns
  - prep_run timestamp written to people log after output renders

affects:
  - Phase 05 (skip-level prep) - same Phase A-E structure applies
  - Phase 06 (retro prep) - same structure applies

tech-stack:
  added: []
  patterns:
    - "Phase A-E execution structure for prep commands (pre-flight → reference reads → data collection → output → state write)"
    - "Dynamic lookback window derived from people log fields (last_1on1_date -> prep_run -> 14-day default)"
    - "Output before state writes - prep sheet renders fully before prep_run timestamp is written"
    - "Consumer-only baseline pattern - prep reads baselines.json but never updates it (only pulse.md updates baselines)"
    - "Talking point generation with source labels (signal/commitment/career/pattern) and priority ordering"

key-files:
  created:
    - .claude/commands/team-health/prep.md
  modified: []

key-decisions:
  - "Lookback priority chain locked: last_1on1_date -> prep_run -> 14-day default; each independently testable in Scenarios 11-13"
  - "Prep does NOT update baselines.json - it is a consumer only; this prevents corrupting baseline history on non-pulse runs"
  - "Phase E writes both last_1on1_date and prep_run to people log - auto-records today as the new lookback anchor"
  - "Talking points use priority ordering: RED flags > overdue commitments >30d > YELLOW flags > career context >90d > log patterns > wins"

patterns-established:
  - "Phase A-E structure replicated from pulse.md with prep-specific adaptations (single person, dynamic lookback)"
  - "Pre-flight Check, Name Resolution, Required Reference Reads copied verbatim from existing commands"
  - "PRIVACY.md graceful degradation phrases used character-for-character for each unavailable source"

requirements-completed: [PREP-01, PREP-02, PREP-03, PREP-04, PREP-05, PREP-06, PREP-07, PREP-08, PREP-09, PREP-10]

duration: 1min
completed: 2026-03-17
---

# Phase 4 Plan 01: 1:1 Prep Command Summary

**`/team-health:prep <name>` command creating a 2-minute 5-section briefing from live MCP signals + people log data with dynamic lookback window and auto-recorded prep_run timestamp**

## Performance

- **Duration:** ~1 min
- **Started:** 2026-03-17T16:44:28Z
- **Completed:** 2026-03-17T16:46:22Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Created `.claude/commands/team-health/prep.md` (235 lines) - the sole Phase 4 deliverable
- Implemented all 5 execution phases (A: setup, B: people log + lookback derivation, C: MCP signal collection, D: 5-section prep sheet output, E: state write)
- Lookback window logic: reads `last_1on1_date` → falls back to `prep_run` → falls back to 14-day default, each step independently testable against Scenarios 11-13
- All 10 PREP requirements addressed; command structured to pass all 13 smoke test scenarios
- Verbatim PRIVACY.md degradation phrases for all 4 sources; verbatim disclaimer at end of output

## Task Commits

Each task was committed atomically:

1. **Task 1: Create /team-health:prep command file** - `b48a375` (feat)

**Plan metadata:** (added in this commit)

## Files Created/Modified

- `.claude/commands/team-health/prep.md` - /team-health:prep slash command with Phase A-E structure, dynamic lookback window, five-section prep sheet output, and prep_run state write

## Decisions Made

- Copied Pre-flight Check, Name Resolution, and Required Reference Reads verbatim from existing commands (pulse.md and log.md) to maintain exact consistency
- Phase C explicitly uses `lookback_start` from Phase B for all MCP queries, not the fixed weekly windows from SIGNALS.md
- Phase E writes both `last_1on1_date` and `prep_run` to people log - this means next invocation will use today as the anchor for both fields, ensuring no gap in lookback continuity
- Prep is consumer-only for baselines.json; updating it would corrupt the 8-week rolling window maintained by pulse.md

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required for the command file itself.

## Self-Check

- [x] `.claude/commands/team-health/prep.md` exists (235 lines - exceeds 150 line minimum)
- [x] `disable-model-invocation: true` present
- [x] 5 phases (A-E) present (`grep -c "^## Phase [A-E]"` returns 5)
- [x] Pre-flight Check, Name Resolution, Required Reference Reads present
- [x] All 5 output sections present (Status Snapshot, Signal Flags, Standing Items, Suggested Talking Points, Context Reminders)
- [x] `lookback_start`, `last_1on1_date`, `prep_run`, `14 days` all present
- [x] `baselines.json`, `SIGNALS.md`, `BASELINES.md`, `PRIVACY.md` all referenced
- [x] Verbatim disclaimer present
- [x] `open_commitments`, `career_context`, `github_org` all present
- [x] `Do NOT` appears 4 times (anti-patterns: don't re-derive slug, don't update baselines, don't truncate entries, DM privacy constraint)
- [x] No `date +%Y-W%V` (no ISO week injection needed)
- [x] Commit `b48a375` exists

## Self-Check: PASSED

## Next Phase Readiness

- Phase 4 Plan 01 complete - `/team-health:prep <name>` command ready for manual smoke testing against the 13 scenarios in `docs/testing/phase-4-smoke-tests.md`
- Smoke testing requires: `.team-health/config.json` (setup complete), people log fixtures for Alice Chen and Bob Smith, baselines.json with 8-week data, and at least one MCP source active (GitHub)
- Phase 5 (skip-level prep) can begin - it follows the same Phase A-E structure established here

---
*Phase: 04-1-1-prep*
*Completed: 2026-03-17*
