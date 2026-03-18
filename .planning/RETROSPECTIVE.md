# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 - MVP

**Shipped:** 2026-03-18
**Phases:** 5 | **Plans:** 13 | **Timeline:** 8 days (2026-03-09 → 2026-03-17)

### What Was Built
- `/team-health:setup` - first-run setup with MCP capability detection, team roster, gitignored state dir
- `/team-health:log` - longitudinal people log with append, query, category inference, commitment dual-write
- `/team-health:pulse` - weekly team health scan with personal 8-week baseline scoring, two-signal flagging, pulse history
- `/team-health:prep` - 2-minute 1:1 prep sheet synthesizing MCP signals, baselines, and people log context
- `/team-health:skip-level` - privacy-gated upward-facing team brief consuming pulse history
- `/team-health:retro-prep` - sprint retro agenda with work-item attribution enforcement

### What Worked
- **Wave 0 smoke test contracts per phase**: Writing the test scenarios before implementation fixed verification criteria upfront - each phase shipped with a clear manual test contract
- **Dependency-ordered phase structure**: Foundation → People Log → Team Pulse → 1:1 Prep → Skip-Level/Retro was a strict dependency chain that prevented broken intermediate states
- **Privacy via structure not convention**: Encoding privacy rules into PRIVACY.md as a reference doc consumed by commands made it structural rather than an assumption
- **Atomic state writes**: Building full JSON objects in memory before writing prevented partial-write corruption (used in baselines.json and config.json updates)

### What Was Inefficient
- **STATE.md drift**: STATE.md's progress counters and "Current focus" were stale for most of the milestone (showed Phase 1 even after Phase 5 shipped) - state tracking could be more automated
- **MILESTONES.md one-liner extraction**: gsd-tools `summary-extract --fields one_liner` returned null for all summaries; accomplishments had to be written manually from memory
- **MCP viability uncertainty carried as blockers**: Jira MCP, Google Calendar MCP, and Slack metadata scope were recorded as blockers but never resolved during the milestone - they're deferred to when users actually encounter them

### Patterns Established
- **Wave 0 always first**: Every phase started with a smoke test document before any implementation - this became a firm convention by Phase 3
- **Consumer-only pattern for prep**: Commands that read baselines never write them (prep.md is consumer-only, pulse.md is producer) - preserves 8-week rolling window integrity
- **Attribution check as an explicit named phase step**: Privacy-sensitive commands (skip-level, retro-prep) enforce their attribution rules as an explicit `Phase C: [Check Name]` step in the command, not as assumed behavior
- **`.claude/commands/team-health/` path for colon-namespace**: Using `.claude/commands/` (not `.claude/skills/`) produces the `:` namespace required for `team-health:command` invocation

### Key Lessons
1. **Encode constraints structurally**: Privacy, signal philosophy, tone - all of these became reference docs (PRIVACY.md, SIGNALS.md) consumed inline, not left as prose guidance
2. **Wave 0 verification contracts before implementation**: Locking the test scenarios first meant implementation had a clear acceptance bar - no ambiguity about "did this phase pass?"
3. **Degrade by naming gaps, not silently**: Every command was designed to name which signals are absent ("Jira not connected - velocity signals unavailable") rather than silently omit sections

### Cost Observations
- Model mix: balanced profile (opus planner / sonnet executor)
- Sessions: ~10 estimated across 8 days
- Notable: Phase 3 (Pulse) had the most complex command with the two-signal flagging rule and baseline computation - took the most planning iterations

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Phases | Plans | Key Change |
|-----------|--------|-------|------------|
| v1.0 | 5 | 13 | Established Wave 0 pattern, dependency-ordered phases, privacy-by-structure |

### Top Lessons (Verified Across Milestones)

1. Encode constraints as structural artifacts (reference docs) rather than assumptions
2. Wave 0 smoke test contracts before implementation eliminate ambiguity at verification time
