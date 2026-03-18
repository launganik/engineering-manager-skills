---
phase: 05-skip-level-and-retro-prep
plan: 01
subsystem: commands
tags: [claude-code, skip-level, team-health, privacy-gate, pulse-history]

# Dependency graph
requires:
  - phase: 05-00
    provides: phase-5 smoke test contract (verification scenarios 1-7 for skip-level)
  - phase: 03-team-pulse
    provides: pulse-history snapshot files consumed by skip-level aggregation
  - phase: 01-foundation
    provides: config.json schema with last_skip_level field, PRIVACY.md, SIGNALS.md, BASELINES.md
provides:
  - /team-health:skip-level slash command with privacy gate and 5-section team brief output
affects:
  - phase-05-plan-02 (retro-prep command - parallel deliverable in same phase)
  - SKILL.md update (flip skip-level availability label from "Phase 5" to "Available")

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Privacy Gate: structural prohibition on reading .team-health/people/ by default, unlocked only by --include-person flag"
    - "Anonymized aggregation: pulse flag names stripped, replaced with count language (N team members)"
    - "Compliance Check step: post-generation scan for individual names, diagnostic language, team-relative scoring before finalizing output"
    - "Lookback priority chain: --weeks N / --quarter override → last_skip_level anchor → 14-day default"
    - "Output-before-state-write discipline: Phase D renders full brief before Phase E writes last_skip_level to config.json"
    - "Atomic config.json write: complete object rebuilt in memory with all fields before single Write call"

key-files:
  created:
    - .claude/commands/team-health/skip-level.md
  modified: []

key-decisions:
  - "Asks/Escalations section (Section 4) uses placeholder text only - Claude never pre-populates escalation items; EM fills in manually before sharing"
  - "Compliance Check is an explicit named step after all 5 sections are generated - enforces SKIP-05 structurally, not as an assumption"
  - "Phase E writes both last_skip_level and last_updated atomically; uses the config.json already in memory from Pre-flight Check to avoid partial-write data loss"

patterns-established:
  - "Privacy Gate pattern: place before Required Reference Reads to make the prohibition load before any data collection begins"
  - "opt-in content labeling: named subsection with '(opt-in content)' suffix so the brief self-documents that individual content was an explicit manager choice"

requirements-completed: [SKIP-01, SKIP-02, SKIP-03, SKIP-04, SKIP-05]

# Metrics
duration: 3min
completed: 2026-03-17
---

# Phase 5 Plan 01: Skip-Level Command Summary

**`/team-health:skip-level` command with structural privacy gate, anonymized pulse aggregation, 5-section team brief, and atomic last_skip_level state write**

## Performance

- **Duration:** 3 min
- **Started:** 2026-03-17T17:35:20Z
- **Completed:** 2026-03-17T17:38:30Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments

- Created skip-level.md command file (206 lines) following the established Phase 3/4 structural pattern
- Privacy Gate section structurally blocks .team-health/people/ reads by default - enforces SKIP-03 at the instruction level before any data collection begins
- Anonymized aggregation rules prevent individual names from pulse history "Flagged Members" sections from leaking into the brief output (SKIP-04)
- 5-section output format (Delivery Status, Risks and Watch Areas, People Themes, Asks/Escalations, Team Wins) satisfies SKIP-01
- Lookback window priority chain (--weeks / --quarter override → last_skip_level → 14-day fallback) satisfies SKIP-02
- Compliance Check step after output generation enforces SKIP-05 by scanning for individual names, diagnostic language, and team-relative scoring
- Phase E atomically writes last_skip_level to config.json after brief is rendered, satisfying the state-write discipline from Phases 2-4

## Task Commits

Each task was committed atomically:

1. **Task 1: Create skip-level.md command file** - `e0dd78a` (feat)

**Plan metadata:** (docs commit - in progress)

## Files Created/Modified

- `.claude/commands/team-health/skip-level.md` - Skip-level slash command: pre-flight check, privacy gate, required reference reads, Phase A-E execution pattern, 5-section output, compliance check, atomic config.json write

## Decisions Made

- Compliance Check is placed after Phase D output generation (not inline during each section) to give a complete view of the entire brief before scanning for violations - this catches cross-section issues a per-section check would miss
- opt-in content from --include-person is appended as a labeled subsection within Section 3 (People Themes) rather than a separate section - keeps the brief at exactly 5 sections as required by SKIP-01
- Privacy Gate placed before Required Reference Reads to establish the prohibition before any instructions to read data appear in the command

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- skip-level.md is complete and committed; satisfies SKIP-01 through SKIP-05
- Phase 05-02 can now implement retro-prep.md as the parallel deliverable
- SKILL.md needs a small update to flip skip-level from "Phase 5 - not yet available" to "Available" (noted in Research open question 5; can be bundled with 05-02)

---
*Phase: 05-skip-level-and-retro-prep*
*Completed: 2026-03-17*
