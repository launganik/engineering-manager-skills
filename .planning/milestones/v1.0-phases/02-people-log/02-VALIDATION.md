---
phase: 2
slug: people-log
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-13
---

# Phase 2 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None - pure markdown/JSON skill, no runnable code |
| **Config file** | n/a |
| **Quick run command** | Manual: invoke `/team-health:log <name>` in Claude Code and verify output |
| **Full suite command** | Manual: run all 6 log scenarios in a project with Phase 1 setup complete |
| **Estimated runtime** | ~10 minutes (manual) |

---

## Sampling Rate

- **After every task commit:** Verify the specific deliverable (file exists, JSON structure matches schema, command front matter correct)
- **After every plan wave:** Run all 6 smoke scenarios in a fresh project with Phase 1 setup complete
- **Before `/gsd:verify-work`:** All 6 scenarios must pass
- **Max feedback latency:** Manual verification per task

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 02-01-01 | 01 | 1 | LOG-01 | smoke/manual | Run `/team-health:log alice`, type a note, verify `.team-health/people/alice-chen.json` exists with one entry | ❌ Wave 0 | ⬜ pending |
| 02-01-02 | 01 | 1 | LOG-02 | smoke/manual | Log one note per category type; inspect JSON for correct category strings | ❌ Wave 0 | ⬜ pending |
| 02-01-03 | 01 | 1 | LOG-03 | smoke/manual | Log "gave great code review feedback on the auth PR"; verify entry has date, inferred category, original content | ❌ Wave 0 | ⬜ pending |
| 02-01-04 | 01 | 1 | LOG-04 | smoke/manual | Add 6 entries to a person; run log again; verify only last 5 are shown in context display | ❌ Wave 0 | ⬜ pending |
| 02-01-05 | 01 | 1 | LOG-05 | smoke/manual | Log 3 entries including a career note; run `/team-health:log alice when did I last discuss promotion?`; verify date and content | ❌ Wave 0 | ⬜ pending |
| 02-01-06 | 01 | 1 | LOG-06 | smoke/manual | Run setup, do NOT log anything; run `/team-health:log bob` where bob has no existing log file; verify file created cleanly | ❌ Wave 0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

Wave 0 for this phase creates the smoke test scenarios doc:

- [ ] `docs/testing/phase-2-smoke-tests.md` - manual test procedure for LOG-01 through LOG-06

*All verification is manual for this phase - pure markdown/JSON skill has no automated test runner.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `/team-health:log alice` appends entry to alice-chen.json | LOG-01 | Claude Code skill - no unit test runner | Run command in live Claude Code session, inspect .team-health/people/ |
| Category inference from free-form text | LOG-02, LOG-03 | Requires Claude inference to test | Log notes in each category domain, verify category field in JSON |
| Last 3–5 entries shown as context | LOG-04 | Requires reading Claude's output | Add >5 entries, run log again, check display |
| Natural language query accuracy | LOG-05 | Requires Claude's query reasoning | Log career note, query for it by question, verify correct answer |
| Graceful first-time creation | LOG-06 | File system + Claude output | New person with no log, run log command, verify clean creation |
| Commitment dual-write (entries + open_commitments) | LOG-06 | JSON structure inspection | Log a commitment note, verify both arrays updated |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < manual verification per task
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
