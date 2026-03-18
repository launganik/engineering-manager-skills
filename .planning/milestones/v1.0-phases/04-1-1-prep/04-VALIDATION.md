---
phase: 4
slug: 1-1-prep
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-17
---

# Phase 4 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Manual only — pure markdown/JSON skill; no automated test runner |
| **Config file** | none |
| **Quick run command** | Manual smoke test per scenario in phase-4-smoke-tests.md |
| **Full suite command** | Run all scenarios in phase-4-smoke-tests.md |
| **Estimated runtime** | ~10-15 minutes manual |

---

## Sampling Rate

- **After every task commit:** Visually inspect generated file for structural correctness
- **After every plan wave:** Run Wave 0 smoke test scenarios applicable to completed tasks
- **Before `/gsd:verify-work`:** Full smoke test suite must pass
- **Max feedback latency:** N/A — manual verification

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 04-00-01 | 00 | 1 | PREP-01–10 | manual | n/a — Wave 0 smoke test doc | ❌ W0 | ⬜ pending |
| 04-01-01 | 01 | 2 | PREP-01,02,03,04,05,06,07,08,09,10 | manual | smoke-tests.md all scenarios | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `docs/testing/phase-4-smoke-tests.md` — manual smoke test scenarios for PREP-01 through PREP-10

*Wave 0 plan (04-00) creates this document before the command is implemented.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Prep sheet renders in ~2 minutes of reading | PREP-01 | Subjective timing; requires live invocation | Run `/team-health:prep <name>` with full setup; time reading the output |
| Signal flags use behavioral observation language only | PREP-09 | Language quality requires human judgment | Check all flagged signals for diagnostic language violations per PRIVACY.md |
| Talking points framed as questions | PREP-02 | Tone assessment requires human review | Verify each talking point ends in question form or implies a question |
| Graceful degradation states absent signals by name | PREP-10 | Requires simulated source unavailability | Set sources.github=false in config, run prep, verify degradation output |
| `prep_run` timestamp appended after output rendered | PREP-01 | State write ordering requires live execution | Check people log file after prep run; verify timestamp appended, not prepended |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < N/A (manual)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
