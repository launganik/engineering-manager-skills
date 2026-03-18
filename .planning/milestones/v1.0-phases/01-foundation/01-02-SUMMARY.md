---
phase: 01-foundation
plan: "02"
status: complete
completed: 2026-03-10
---

# Plan 01-02 Summary - Signal/Baseline/Privacy Reference Docs

## One-Liner

Authored three shared reference documents encoding all domain logic (signals, baseline computation, output language rules) that every future team-health command depends on.

## What Shipped

### Key Files Created

- `.claude/team-health/SIGNALS.md` - 274 lines. Defines all 11 signals across 4 sources (GitHub, Jira, Slack, Calendar) with exact computation methods, thresholds, the two-signal rule, and a signal availability matrix. Written as executable instructions Claude follows at runtime.
- `.claude/team-health/BASELINES.md` - 234 lines. Self-contained rolling 8-week baseline computation guide with step-by-step update instructions, worked numerical examples (mean, stddev), first-run guidance, and provenance field requirements.
- `.claude/team-health/PRIVACY.md` - 121 lines. Four mandatory output language rules with prohibited/approved examples for each, required disclaimer verbatim, and graceful degradation phrasing patterns.

### Requirements Covered

- REF-01 ✓ - SIGNALS.md
- REF-02 ✓ - BASELINES.md
- REF-03 ✓ - PRIVACY.md

## Decisions Made

- Domain logic lives in reference docs, not command files - commands load these at runtime so thresholds/rules evolve without rewriting every command
- Two-signal rule encoded in SIGNALS.md: flag only when 2+ signals below threshold OR 1 signal >2 stddev from personal baseline
- Personal baselines only - never team-relative comparisons (Rule 3 in PRIVACY.md)

## Issues / Deviations

PRIVACY.md write was initially blocked by a permission prompt mid-execution; written directly by orchestrator and committed separately (same content, same commit standard).

## Self-Check

- [x] SIGNALS.md: 274 lines (≥80 required), Two-Signal Rule present, 2-stddev threshold present
- [x] BASELINES.md: 234 lines (≥60 required), sqrt/stddev computation present
- [x] PRIVACY.md: 121 lines (≥50 required), required disclaimer verbatim present
- [x] All three committed to git
- [x] REF-01, REF-02, REF-03 marked complete in REQUIREMENTS.md
