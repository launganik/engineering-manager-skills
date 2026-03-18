---
phase: 3
slug: team-pulse
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-03-13
---

# Phase 3 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | None - pure markdown/JSON command file, no runnable code |
| **Config file** | n/a |
| **Quick run command** | Manual: invoke `/team-health:pulse` in Claude Code with GitHub MCP configured |
| **Full suite command** | Manual: run all 10 PULSE scenarios in a project with Phase 1 setup complete and at least one MCP source available |
| **Estimated runtime** | ~15–30 minutes (manual testing) |

---

## Sampling Rate

- **After every task commit:** Verify the specific deliverable (file exists, correct front matter, required sections present)
- **After every plan wave:** Run all 10 smoke scenarios in a fresh project with Phase 1 setup complete
- **Before `/gsd:verify-work`:** Full suite must be green
- **Max feedback latency:** N/A (manual tests; verify after each plan)

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|-----------|-------------------|-------------|--------|
| 3-00-01 | 00 | 0 | PULSE-01..10 | manual | Verify `docs/testing/phase-3-smoke-tests.md` exists with all 10 scenarios | ❌ W0 | ⬜ pending |
| 3-01-01 | 01 | 1 | PULSE-01 | manual | Run `/team-health:pulse`; verify output covers all team members | ❌ W0 | ⬜ pending |
| 3-01-02 | 01 | 1 | PULSE-02 | manual | Verify no team-relative comparisons; all flags reference personal baseline | ❌ W0 | ⬜ pending |
| 3-01-03 | 01 | 1 | PULSE-03 | manual | Run with GitHub only; verify 4 GitHub signals, 7 others marked unavailable | ❌ W0 | ⬜ pending |
| 3-01-04 | 01 | 1 | PULSE-04 | manual | Test 1-signal-at-1.5-stddev (no flag); test 2-signals-at-2+stddev (yellow flag) | ❌ W0 | ⬜ pending |
| 3-01-05 | 01 | 1 | PULSE-05 | manual | Verify table has all members; detail section exists only for flagged | ❌ W0 | ⬜ pending |
| 3-01-06 | 01 | 1 | PULSE-06 | manual | Inspect flagged output: metric name + current value + delta + stddev delta + source | ❌ W0 | ⬜ pending |
| 3-01-07 | 01 | 1 | PULSE-07 | manual | Verify last paragraph is verbatim disclaimer from PRIVACY.md | ❌ W0 | ⬜ pending |
| 3-01-08 | 01 | 1 | PULSE-08 | manual | Inspect `baselines.json` after run: computed_from, computed_to, source, schema_version present | ❌ W0 | ⬜ pending |
| 3-01-09 | 01 | 1 | PULSE-09 | manual | Verify `.team-health/pulse-history/YYYY-WNN.md` exists after run | ❌ W0 | ⬜ pending |
| 3-01-10 | 01 | 1 | PULSE-10 | manual | Run with no MCP sources; verify output names all 4 unavailable with PRIVACY.md degradation phrasing | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `docs/testing/phase-3-smoke-tests.md` - step-by-step manual test procedure for all 10 PULSE scenarios
- [ ] Verify `date +%Y-W%V` works on target OS (macOS) - add to Wave 0 checklist
- [ ] Verify `Bash(mkdir -p .team-health/pulse-history)` is supported in `allowed-tools` Bash scoping

*No automated framework - no runnable code; all tests are manual CLI invocations.*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `/team-health:pulse` scans all team members | PULSE-01 | Requires live MCP + Claude Code runtime | Run command; verify output mentions every person in config.json team array |
| Personal baseline deviation (not team average) | PULSE-02 | Runtime behavior of LLM following instructions | Verify all flags reference "vs. 8-week avg" for that person, not a team metric |
| Two-signal conservative flagging | PULSE-04 | Requires injected test scenario with known baselines | Inject 1-signal-at-1.5-stddev → no flag; 2-signals-at-1-stddev → no flag; 2-signals-at-2+stddev → yellow |
| Disclaimer appears verbatim | PULSE-07 | Output content verification | Verify final paragraph is exactly: "These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions." |
| baselines.json provenance fields | PULSE-08 | File inspection after run | Open `.team-health/baselines.json`; confirm `computed_from`, `computed_to`, `source`, `schema_version` for each person |
| Graceful degradation naming | PULSE-10 | Run with all sources disabled | Set all sources to false; verify output names GitHub, Jira, Slack, Calendar as unavailable with PRIVACY.md phrasing |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30min (manual tests)
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
