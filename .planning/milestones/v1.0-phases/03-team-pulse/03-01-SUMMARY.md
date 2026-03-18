---
phase: 03-team-pulse
plan: "01"
subsystem: team-health
tags: [claude-commands, pulse, baselines, signals, mcp, slash-command]

# Dependency graph
requires:
  - phase: 03-team-pulse-00
    provides: smoke test contract (phase-3-smoke-tests.md) written before implementation
  - phase: 01-foundation
    provides: state-schemas.json (config.json + baselines.json schemas), setup.md and log.md command patterns
provides:
  - /team-health:pulse slash command - full weekly team health scan pipeline
  - Per-person baseline comparison with two-signal flagging rule
  - Graceful degradation for absent MCP sources
  - Atomic baselines.json update with provenance fields
  - pulse-history/YYYY-WNN.md snapshot per run
affects: [03-team-pulse future plans, any commands that read baselines.json or pulse-history]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - Pre-flight Check boilerplate (read config, stop if not setup_complete)
    - Required Reference Reads before analysis (SIGNALS.md, BASELINES.md, PRIVACY.md)
    - disable-model-invocation: true on all state-writing commands
    - Date injection via !`date +%Y-%m-%d` and !`date +%Y-W%V`
    - Phase-gated execution (A→B→C→D→E): setup, collect, compare, output, write state
    - Atomic single-write pattern for baselines.json (build full object in memory, one Write call)
    - mkdir -p guard before pulse-history write

key-files:
  created:
    - .claude/commands/team-health/pulse.md
  modified: []

key-decisions:
  - "Output before state writes: Phase D (dashboard) completes before Phase E (baselines.json + snapshot) to avoid corrupting deviation calculations with the current week's value"
  - "Atomic baselines.json write: entire file built in memory then written in a single Write tool call - prevents partial-write data corruption"
  - "Flag status PENDING distinct from GREEN: baseline-pending states never get a flag color, shown as PENDING with weeks-of-data label"
  - "Two-signal rule faithfully transcribed: RED on single signal >=2 stddev; YELLOW on 2+ signals each >=1 stddev; stddev==0 edge case handled"

patterns-established:
  - "Phase-gated pipeline pattern: Pre-flight → Reference Reads → Setup (A) → Collect (B) → Compare (C) → Output (D) → Write State (E)"
  - "Lag signal directionality: github_pr_review_lag_days and calendar_meeting_load_pct flag on upward deviation; all other signals flag on downward deviation"
  - "Window-thinness guard: 0 weeks = PENDING, 1-2 weeks = PENDING (no flag), 3+ weeks = compute deviation"

requirements-completed: [PULSE-01, PULSE-02, PULSE-03, PULSE-04, PULSE-05, PULSE-06, PULSE-07, PULSE-08, PULSE-09, PULSE-10]

# Metrics
duration: 1min
completed: 2026-03-17
---

# Phase 3 Plan 01: Team Pulse Summary

**Slash command /team-health:pulse implementing full weekly scan pipeline - per-person MCP signal collection, personal baseline comparison with two-signal flagging rule, flagged-only detail sections, and atomic state writes for baselines.json and pulse-history snapshots**

## Performance

- **Duration:** ~2 min
- **Started:** 2026-03-17T15:38:34Z
- **Completed:** 2026-03-17T15:40:14Z
- **Tasks:** 1 of 1
- **Files modified:** 1

## Accomplishments

- Complete /team-health:pulse command file covering all 10 PULSE requirements (PULSE-01 through PULSE-10)
- Faithful implementation of two-signal flagging rule: RED on single >=2 stddev signal, YELLOW on 2+ signals each >=1 stddev
- Phase D output strictly before Phase E state writes - prevents baseline corruption by keeping current week's value out of comparison window
- Graceful degradation for all 4 MCP sources with PRIVACY.md-anchored language

## Task Commits

Each task was committed atomically:

1. **Task 1: Write .claude/commands/team-health/pulse.md** - `929129b` (feat)

**Plan metadata:** (final commit, see below)

## Files Created/Modified

- `.claude/commands/team-health/pulse.md` - Complete /team-health:pulse slash command: Pre-flight Check, Required Reference Reads, Phase A (setup/degradation), Phase B (per-person signal collection for all 4 MCP sources), Phase C (baseline comparison + two-signal flagging), Phase D (dashboard + flagged detail sections + verbatim disclaimer), Phase E (atomic baselines.json write + pulse-history snapshot)

## Decisions Made

- Output before state writes: Phase D (dashboard) completes before Phase E (baselines.json + snapshot) to avoid corrupting deviation calculations with the current week's value
- Atomic baselines.json write: entire file built in memory then written in a single Write tool call - prevents partial-write data corruption
- Flag status PENDING distinct from GREEN: baseline-pending states never get a flag color, shown as PENDING with weeks-of-data label
- Two-signal rule faithfully transcribed: RED on single signal >=2 stddev; YELLOW on 2+ signals each >=1 stddev; stddev==0 edge case handled

## Deviations from Plan

None - plan executed exactly as written. The pulse.md command was built section by section following the exact instructions in the plan, including verbatim copy of the front matter, Pre-flight Check boilerplate, and disclaimer text.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required for this plan. The command itself will prompt users to run /team-health:setup before proceeding.

## Next Phase Readiness

- /team-health:pulse command is ready for Phase 3 manual smoke tests (docs/testing/phase-3-smoke-tests.md)
- All 10 scenarios can be run in a live Claude Code session with Phase 1 setup complete and GitHub MCP configured
- Phase 3 is complete as of this plan - remaining phases (04, 05) can proceed

---
*Phase: 03-team-pulse*
*Completed: 2026-03-17*
