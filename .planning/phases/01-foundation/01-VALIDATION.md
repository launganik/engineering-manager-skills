---
phase: 1
slug: foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-10
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None — pure markdown/JSON skill with no runnable code |
| **Config file** | n/a |
| **Quick run command** | Manual: verify specific deliverable for each task (file exists, JSON valid, correct schema fields) |
| **Full suite command** | Manual: fresh project, no config → run each command → verify routing and output |
| **Estimated runtime** | ~10 minutes manual |

---

## Sampling Rate

- **After every task commit:** Manually verify the specific deliverable (file exists, JSON is valid, schema_version present where required)
- **After every plan wave:** Run full smoke test: fresh project, no config, run each command created so far, verify routing and output
- **Before `/gsd:verify-work`:** All smoke tests green; all reference docs exist and substantively complete

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 1-01-01 | 01 | 1 | SETUP-01 | smoke/manual | Run `/team-health:setup` without MCPs; verify all four report unavailable | ❌ Wave 0 | ⬜ pending |
| 1-01-02 | 01 | 1 | SETUP-01 | smoke/manual | Run `/team-health:setup` with GitHub MCP; verify sources.github = true in config.json | ❌ Wave 0 | ⬜ pending |
| 1-01-03 | 01 | 1 | SETUP-02 | smoke/manual | Run setup with 2 team members; verify config.json contains correct schema | ❌ Wave 0 | ⬜ pending |
| 1-01-04 | 01 | 1 | SETUP-03 | smoke/manual | Delete config.json; run `/team-health:prep alice`; verify routes to setup, not error | ❌ Wave 0 | ⬜ pending |
| 1-01-05 | 01 | 1 | SETUP-04 | schema review | Inspect generated config.json, people/<slug>.json, baselines.json for schema_version field | ❌ Wave 0 | ⬜ pending |
| 1-01-06 | 01 | 1 | SETUP-05 | smoke/manual | Run setup in a git repo; verify `.gitignore` contains `.team-health/` | ❌ Wave 0 | ⬜ pending |
| 1-02-01 | 02 | 1 | REF-01 | file check | Verify `.claude/team-health/SIGNALS.md` exists and contains signal definitions | ❌ Wave 0 | ⬜ pending |
| 1-02-02 | 02 | 1 | REF-02 | file check | Verify `.claude/team-health/BASELINES.md` exists and contains computation methodology | ❌ Wave 0 | ⬜ pending |
| 1-02-03 | 02 | 1 | REF-03 | file check | Verify `.claude/team-health/PRIVACY.md` exists and contains output language rules | ❌ Wave 0 | ⬜ pending |
| 1-02-04 | 02 | 1 | REF-04 | review | Human review of top-level SKILL.md for accuracy and completeness | ❌ Wave 0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `docs/testing/phase-1-smoke-tests.md` — step-by-step manual test procedure for all 10 scenarios above
- [ ] `.planning/phases/01-foundation/state-schemas.json` — canonical JSON schemas for config.json, people/<slug>.json, baselines.json

*No automated framework install needed — no runnable code in Phase 1.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| MCP detection reports availability correctly | SETUP-01 | Requires Claude Code runtime with/without MCP configured | Run `/team-health:setup` with and without GitHub MCP; check reported sources |
| Setup gate routes correctly | SETUP-03 | Requires Claude Code runtime to test routing behavior | Delete config.json; invoke any command; verify setup flow triggered |
| .team-health/ gitignored after setup | SETUP-05 | File system state check | Inspect .gitignore after running setup in a git repo |
| Reference docs are substantively complete | REF-01..03 | Content quality requires human review | Read each reference doc; verify coverage of all signals/thresholds/rules |
| SKILL.md accurate and complete | REF-04 | Human-facing doc quality | Read SKILL.md; verify install steps, MCP prereqs, command descriptions are correct |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 10 minutes per wave
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
