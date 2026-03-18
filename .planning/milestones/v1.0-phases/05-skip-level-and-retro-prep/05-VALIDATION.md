---
phase: 5
slug: skip-level-and-retro-prep
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-17
---

# Phase 5 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual only - pure markdown/JSON skill; no automated test runner |
| **Config file** | none |
| **Quick run command** | Manual smoke test per scenario in phase-5-smoke-tests.md |
| **Full suite command** | Run all scenarios in phase-5-smoke-tests.md |
| **Estimated runtime** | ~15-20 minutes manual |

---

## Sampling Rate

- **After every task commit:** Visually inspect generated file for structural correctness (section headers, placeholder structure, privacy gate language)
- **After every plan wave:** Run applicable Wave 0 smoke test scenarios
- **Before `/gsd:verify-work`:** Full smoke test suite must pass
- **Max feedback latency:** N/A - manual verification

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 05-00-01 | 00 | 1 | SKIP-01–05, RETRO-01–04 | manual | n/a - Wave 0 smoke test doc | ❌ W0 | ⬜ pending |
| 05-01-01 | 01 | 2 | SKIP-01,02,03,04,05 | manual | smoke-tests.md scenarios 1-6 | ❌ W0 | ⬜ pending |
| 05-01-02 | 01 | 2 | RETRO-01,02,03,04 | manual | smoke-tests.md scenarios 7-10 | ❌ W0 | ⬜ pending |
| 05-01-03 | 01 | 2 | REF-04 | manual | grep "skip-level\|retro-prep" .claude/team-health/SKILL.md | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `docs/testing/phase-5-smoke-tests.md` - manual smoke test scenarios for SKIP-01–05 and RETRO-01–04

*Wave 0 plan (05-00) creates this document before commands are implemented.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Privacy gate blocks people log by default | SKIP-03 | Runtime instruction enforcement, not code | Run `/team-health:skip-level`; verify no `.team-health/people/` content appears in output |
| Individual names absent from pulse aggregations | SKIP-04 | Language compliance requires human judgment | Check skip-level output; confirm flagged persons described as counts only ("N team members") |
| Output passes "team comfort" test | SKIP-05 | Subjective standard requires human review | Read output as if you were a team member seeing it |
| Retro seeds attributed to work items, not people | RETRO-03 | Language quality requires human judgment | Verify all "What Was Hard" entries cite PR/ticket IDs, not names |
| Blank discussion space preserved | RETRO-04 | Structural convention requires live inspection | Confirm each retro section has labeled discussion space and empty action items |
| `last_skip_level` updated in config.json | SKIP-02 | State write ordering requires live execution | Check config.json after run; verify timestamp updated |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < N/A (manual)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
