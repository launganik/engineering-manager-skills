---
phase: 05-skip-level-and-retro-prep
plan: 02
subsystem: team-health
tags: [claude-commands, retro-prep, sprint-retrospective, privacy, work-item-attribution]

# Dependency graph
requires:
  - phase: 05-00
    provides: Phase 5 smoke test contract (scenarios 8-13 for retro-prep)
  - phase: 05-01
    provides: skip-level command (structural template for retro-prep front matter pattern)
  - phase: 03-team-pulse
    provides: pulse history format consumed by retro-prep for cross-sprint patterns
provides:
  - .claude/commands/team-health/retro-prep.md — sprint retro agenda command with work-item attribution enforcement
  - Updated SKILL.md with all Phase 5 commands marked available (no "not yet available" labels)
affects:
  - End users installing the skill (SKILL.md is the installation guide)
  - Phase 5 completion verification (all planned commands now available)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "retro-prep: read-only command (no Write in allowed-tools) that produces meeting agenda output only"
    - "Sprint window fallback: Jira sprint lookup when available, sprint_cadence_weeks pulse history weeks when not"
    - "Work-item attribution enforcement via explicit Phase C attribution check before output"
    - "SKILL.md availability model: remove phase labels rather than add 'Available' text"

key-files:
  created:
    - .claude/commands/team-health/retro-prep.md
  modified:
    - SKILL.md

key-decisions:
  - "retro-prep makes zero state writes — no Write in allowed-tools, no Phase E"
  - "Sprint window fallback: when Jira unavailable, use most recent sprint_cadence_weeks pulse history weeks"
  - "Attribution check (Phase C) is an explicit named step before output — enforces RETRO-03 structurally"
  - "Action Items section has only blank checkboxes — team fills in during the retro, Claude never pre-populates"
  - "SKILL.md availability: remove all 'Phase N (not yet available)' labels rather than replace with 'Available' tag"

patterns-established:
  - "Read-only command pattern: omit Write from allowed-tools to structurally prevent state writes"
  - "Attribution check step: explicit Phase C scan for team member names before rendering output"

requirements-completed: [RETRO-01, RETRO-02, RETRO-03, RETRO-04]

# Metrics
duration: 3min
completed: 2026-03-17
---

# Phase 5 Plan 02: Retro Prep Command and SKILL.md Update Summary

**Sprint retro agenda command with Jira/GitHub sprint data, work-item attribution enforcement via explicit Phase C check, and SKILL.md updated with all 6 commands available**

## Performance

- **Duration:** ~3 min
- **Started:** 2026-03-17T17:40:00Z
- **Completed:** 2026-03-17T17:42:24Z
- **Tasks:** 2
- **Files modified:** 2

## Accomplishments

- Created `.claude/commands/team-health/retro-prep.md` (199 lines) with full 5-section retro agenda: Sprint Facts, What Went Well, What Was Hard, Patterns Across Sprints, Action Items
- Implemented work-item attribution enforcement as explicit Phase C step — scans all generated seeds for team member names before output and rewrites to work-item references
- Structured sprint window determination: Jira sprint lookup (with --sprint flag or auto most-recent) with pulse history fallback using sprint_cadence_weeks when Jira unavailable
- Removed all 5 "Phase N (not yet available)" status labels from SKILL.md and updated Installation ls output to show all 6 command files

## Task Commits

Each task was committed atomically:

1. **Task 1: Create retro-prep.md command file** - `0c9ba44` (feat)
2. **Task 2: Update SKILL.md to mark Phase 5 commands as available** - `149865d` (feat)

## Files Created/Modified

- `.claude/commands/team-health/retro-prep.md` — Sprint retro agenda command with Pre-flight Check, Required Reference Reads (SIGNALS.md, PRIVACY.md), Phase A (sprint window), Phase B (sprint data collection from Jira/GitHub/pulse history), Phase C (attribution check), Phase D (agenda output). No state writes.
- `SKILL.md` — Removed "not yet available" labels from log, pulse, prep, skip-level, and retro-prep commands; updated Installation ls output to show all 6 command files.

## Decisions Made

- retro-prep makes zero state writes: Write is absent from allowed-tools, and no Phase E exists. The retro agenda is a one-shot output — there is no "since last retro" anchor to track.
- Sprint window fallback: when `sources.jira` is false, use most recent `sprint_cadence_weeks` pulse history weeks (from config.json, default 2). State this explicitly in output.
- Phase C attribution check is a named, explicit step (not an implicit assumption): after data collection, before output, scan for team member names and rewrite to work-item attribution. This enforces RETRO-03 structurally.
- Action Items section is structurally blank: the command instructions prohibit pre-populating action items. The team fills in actions during the retro.
- SKILL.md update approach: remove the status lines entirely (matching how setup.md has no status line) rather than replacing with an "Available" label.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

Phase 5 is complete. All 6 planned commands are now implemented and available:
- `/team-health:setup` (Phase 1)
- `/team-health:log` (Phase 2)
- `/team-health:pulse` (Phase 3)
- `/team-health:prep` (Phase 4)
- `/team-health:skip-level` (Phase 5)
- `/team-health:retro-prep` (Phase 5)

Validation against smoke test scenarios 8-13 (phase-5-smoke-tests.md) requires a live Claude Code session with Phases 1-4 setup complete and pulse history populated. The command structure satisfies the structural criteria verifiable without a live session.

---
*Phase: 05-skip-level-and-retro-prep*
*Completed: 2026-03-17*
