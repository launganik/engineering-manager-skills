---
phase: 01-foundation
plan: "03"
subsystem: documentation
tags: [skill, installation-guide, onboarding, mcp, team-health]

# Dependency graph
requires:
  - phase: 01-foundation-01
    provides: setup command file (.claude/commands/team-health/setup.md) that SKILL.md links to
  - phase: 01-foundation-02
    provides: SIGNALS.md and reference docs that SKILL.md describes
provides:
  - SKILL.md at project root - human-facing installation guide and command reference
  - Front-door document for EMs discovering the skill
  - MCP prerequisites table with install commands for all 4 sources
  - Complete command reference for all 6 planned commands with availability status
affects:
  - All future phases that add commands (must update SKILL.md command list)
  - Phase 2 (log command), Phase 3 (pulse), Phase 4 (prep), Phase 5 (skip-level, retro-prep)

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "SKILL.md as human-facing front door (not Claude Code command invocation file)"
    - "Phase status labels on future commands (Status: Phase N - not yet available)"
    - "Privacy-first framing: behavioral observations not diagnoses"

key-files:
  created:
    - SKILL.md
  modified: []

key-decisions:
  - "SKILL.md has no YAML front matter - it is a human document, not a Claude Code command file"
  - "All 6 planned commands documented upfront with phase availability labels so EMs know the roadmap before committing to install"
  - "Privacy section explicit and prominent: local-only storage, no DM content, behavioral observations not diagnoses"

patterns-established:
  - "Installation guide: numbered steps ending with verification commands (cat config.json, grep gitignore)"
  - "MCP table: 4 columns (Source, Signals Enabled, Install Command, Absent Signals)"
  - "Command docs: syntax, flags, example, requires, status per command"

requirements-completed:
  - REF-04

# Metrics
duration: 4min
completed: 2026-03-11
---

# Phase 1 Plan 03: SKILL.md Installation Guide Summary

**Self-contained EM installation guide covering prerequisites, numbered setup steps with verify commands, full 6-command reference with phase availability labels, MCP table for 4 sources, data layout diagram, and privacy design**

## Performance

- **Duration:** 4 min
- **Started:** 2026-03-11T09:50:15Z
- **Completed:** 2026-03-11T09:53:48Z
- **Tasks:** 1 of 1
- **Files modified:** 1

## Accomplishments

- SKILL.md created at project root with all 8 required sections (223 lines)
- Installation section is self-contained: an EM with no prior context can follow it to reach a working `/team-health:setup` invocation
- All 6 planned commands documented with syntax, examples, requirements, and phase availability labels (Phases 2–5 labeled "not yet available")
- MCP prerequisites table covers GitHub, Jira, Slack, and Google Calendar with install commands and absent-signal consequences
- Privacy design section explicitly covers local-only storage, no DM content, behavioral-not-diagnostic framing, skip-level aggregation

## Task Commits

1. **Task 1: Create top-level SKILL.md** - `f62274f` (feat)

**Plan metadata:** (to follow)

## Files Created/Modified

- `SKILL.md` - Human-facing installation guide and command reference for the team-health skill (223 lines)

## Decisions Made

- SKILL.md has no YAML front matter - it is a pure markdown human document, not a Claude Code command invocation file (unlike `.claude/commands/team-health/setup.md`)
- All 6 planned commands documented upfront with phase availability labels so EMs understand the full roadmap before installing, even though only `/team-health:setup` is available in Phase 1
- Privacy section made prominent and explicit because the skill handles sensitive people management data

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None - no external service configuration required.

## Next Phase Readiness

- SKILL.md is ready for Phase 2 - when `/team-health:log` is implemented, update its "Status: Phase 2 (not yet available)" label to remove the status note
- The command reference structure established here should be maintained for each new command added in subsequent phases
- No blockers for Phase 2 work

---
*Phase: 01-foundation*
*Completed: 2026-03-11*
