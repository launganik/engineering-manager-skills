---
phase: 01-foundation
plan: "01"
subsystem: skills
tags: [claude-code, slash-commands, mcp-detection, gitignore, team-health]

# Dependency graph
requires:
  - phase: 01-00
    provides: research confirming .claude/commands/ colon-namespace, front matter keys, MCP probe patterns
provides:
  - /team-health:setup slash command (first-run setup flow for engineering managers)
  - .gitignore entry protecting .team-health/ people data from VCS
affects: [01-02, 01-03, 01-04, 01-05, 02-prep, 03-pulse]

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Colon-namespace slash commands via .claude/commands/<namespace>/<command>.md"
    - "disable-model-invocation: true on state-writing commands to prevent auto-invocation"
    - "Pre-flight Check pattern: every command reads config.json and gates on setup_complete"
    - "Implementation-agnostic MCP probe: check tool namespace not specific tool name"
    - "Slug derivation: lowercase + hyphens + strip special chars from full name"

key-files:
  created:
    - .claude/commands/team-health/setup.md
    - .gitignore
  modified: []

key-decisions:
  - "Use .claude/commands/team-health/ (legacy format) not .claude/skills/ - colon namespace (/team-health:setup) only available via commands subdirectory path"
  - "MCP probe is namespace-based not tool-name-based - robust to different community MCP server implementations"
  - "Static .gitignore entry at install time plus dynamic check in setup command - defense in depth for people data protection"

patterns-established:
  - "Pre-flight Check pattern: Read .team-health/config.json; if missing or setup_complete !== true, tell user to run /team-health:setup and stop"
  - "All state-writing commands use disable-model-invocation: true to prevent unsolicited execution"
  - "config.json schema_version field present for future migration capability"

requirements-completed: [SETUP-01, SETUP-02, SETUP-03, SETUP-04, SETUP-05]

# Metrics
duration: 8min
completed: 2026-03-10
---

# Phase 1 Plan 01: /team-health:setup command and .gitignore protection

**First-run setup slash command that probes MCP availability, collects team roster, writes config.json with canonical schema_version "1", and gitignores .team-health/ - the entry point every other command depends on.**

## Performance

- **Duration:** ~8 min
- **Started:** 2026-03-10T07:51:43Z
- **Completed:** 2026-03-10T07:59:00Z
- **Tasks:** 2 of 2
- **Files modified:** 2

## Accomplishments

- Created `.claude/commands/team-health/setup.md` with complete 6-step setup flow: already-configured guard, namespace-based MCP probe, manager + team roster collection, gitignore check, config.json write, and confirmation summary
- Established the canonical Pre-flight Check pattern that all subsequent commands copy verbatim
- Created `.gitignore` with `.team-health/` entry - static protection that exists before the user runs any command

## Task Commits

Each task was committed atomically:

1. **Task 1: Create /team-health:setup command file** - `d63a517` (feat)
2. **Task 2: Ensure .gitignore protects .team-health/** - `b7ca002` (feat)

**Plan metadata:** (docs commit - see below)

## Files Created/Modified

- `.claude/commands/team-health/setup.md` - Complete first-run setup slash command with YAML front matter, 6-step flow, canonical config.json schema write, and Pre-flight Check pattern
- `.gitignore` - New file with `# team-health` section and `.team-health/` entry for people data VCS protection

## Decisions Made

- Used `.claude/commands/team-health/` (legacy path) instead of `.claude/skills/team-health-setup/SKILL.md` because the colon namespace `/team-health:setup` is only available via the commands subdirectory structure - the skills format produces `/team-health-setup` (hyphen), which conflicts with the project's established naming convention.
- MCP probe instructions check by tool namespace ("any tool containing 'github'") rather than specific tool names - this makes the probe robust to different community GitHub/Jira/Slack/Calendar MCP implementations which may expose tools under different names.
- Added both static `.gitignore` entry (installed with the skill) and dynamic check in the setup command (run at setup time) for defense-in-depth protection of people data.

## Deviations from Plan

None - plan executed exactly as written.

## Issues Encountered

None. The research phase had already resolved the key question about colon-namespace vs. hyphen-namespace (`.claude/commands/` vs `.claude/skills/`), so execution was direct.

## User Setup Required

None - no external service configuration required for this plan. The setup.md command itself guides users through MCP configuration when they run `/team-health:setup`.

## Next Phase Readiness

- Foundation entry point is complete. Every subsequent command can now copy the Pre-flight Check pattern from setup.md.
- Plan 02 (reference documents: SIGNALS.md, BASELINES.md, PRIVACY.md) can proceed immediately.
- The canonical config.json schema (schema_version "1") is established and locked - future phases must write and read against this exact shape.
- Concern carried forward: `allowed-tools` YAML key and exact front matter syntax should be verified against live Claude Code if anything breaks at command invocation time.

## Self-Check: PASSED

- FOUND: `.claude/commands/team-health/setup.md`
- FOUND: `.gitignore`
- FOUND: `.planning/phases/01-foundation/01-01-SUMMARY.md`
- FOUND commit: `d63a517` (Task 1)
- FOUND commit: `b7ca002` (Task 2)

---
*Phase: 01-foundation*
*Completed: 2026-03-10*
