---
phase: 03-team-pulse
verified: 2026-03-17T00:00:00Z
status: human_needed
score: 8/8 must-haves verified (automated checks)
human_verification:
  - test: "Run all 10 smoke test scenarios in a live Claude Code session"
    expected: "All 10 scenarios in docs/testing/phase-3-smoke-tests.md pass without deviation"
    why_human: "pulse.md is a Claude Code slash command — it runs as a live LLM prompt, not executable code. Correctness of signal collection, flagging logic, and output formatting can only be confirmed by invoking the command against real (or fixture-injected) state files."
  - test: "Scenario 4 Sub-test A: Bob has 1 signal at -1.5 stddev — verify GREEN, no detail section"
    expected: "Bob shows GREEN. No detail section rendered."
    why_human: "The conservative two-signal rule false-negative path must be confirmed by actually running the command with manually edited baselines.json."
  - test: "Scenario 7: Disclaimer verbatim check"
    expected: "Last block of output is exactly: 'These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.' — no paraphrase."
    why_human: "LLM compliance with verbatim disclaimer requires live invocation. The command file includes the text correctly but only runtime behavior confirms it is not paraphrased."
  - test: "Scenario 9: Verify macOS date +%Y-W%V produces valid ISO week filename"
    expected: "pulse-history/2026-WNN.md created with correct ISO week number; second run following week creates new file."
    why_human: "macOS BSD date %V flag compatibility is a known risk flagged in both the smoke test doc and the plan. Cannot verify without running on target machine."
  - test: "Scenario 10: Graceful degradation phrasing matches PRIVACY.md verbatim"
    expected: "All 4 sources named as unavailable; phrasing matches PRIVACY.md Graceful Degradation Language section exactly."
    why_human: "Phrasing compliance requires live invocation and character-by-character comparison against PRIVACY.md."
---

# Phase 3: Team Pulse Verification Report

**Phase Goal:** Managers can run a weekly team health scan that scores each person against their personal 8-week baseline, flags anomalies conservatively, and stores baselines and pulse history for downstream commands

**Verified:** 2026-03-17
**Status:** human_needed (all automated checks passed; 5 items require live invocation)
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|---------|
| 1 | Manager can run /team-health:pulse and receive a dashboard with a per-person green/yellow/red summary table | ? HUMAN | Command exists and specifies summary table format verbatim in Phase D; runtime behavior requires live test |
| 2 | Flagged individuals have detail sections with specific metric name, delta, stddev delta, and MCP source — unflagged persons have table row only | ? HUMAN | Phase C/D logic is fully implemented and correct; format template matches PULSE-06 spec exactly; requires live run to confirm no paraphrase |
| 3 | Flags fire only when 2+ signals align OR a single signal exceeds 2 standard deviations from the person's own baseline | ? HUMAN | Phase C implements the rule verbatim from SIGNALS.md with stddev==0 edge case handled; requires live test of all 3 sub-test branches |
| 4 | After each run, baselines.json is updated with computed_from, computed_to, source, and schema_version provenance fields | ? HUMAN | Phase E Step 1 implements provenance fields and atomic write explicitly; first-run (Carol) behavior implemented; requires live run to confirm file is actually written |
| 5 | A pulse-history/YYYY-WNN.md snapshot file is written after each run | ? HUMAN | Phase E Step 2 implements mkdir -p guard and ISO week filename; macOS date compatibility risk flagged; requires live run |
| 6 | Command degrades gracefully when MCP sources are absent, naming which signals are unavailable | ? HUMAN | Phase A implements all-sources-false handler; names all 4 sources; delegates phrasing to PRIVACY.md; requires live run to confirm phrasing compliance |
| 7 | The required disclaimer appears verbatim at the end of every pulse output | ? HUMAN | Disclaimer text present verbatim in Phase D and Phase E pulse-history snapshot; requires live run to confirm no LLM paraphrase |
| 8 | Running the command before setup is complete routes to the setup flow instead of erroring | ✓ VERIFIED | Pre-flight Check block present and complete: reads config.json, checks setup_complete, emits exact message "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources." and stops |

**Automated score:** 8/8 must-haves have complete, substantive implementations. All 8 are gated on human (live invocation) verification. 1/8 is fully verifiable without live run (Pre-flight Check routing).

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `.claude/commands/team-health/pulse.md` | /team-health:pulse command — full pulse scan pipeline | ✓ VERIFIED | 190 lines; all 5 phases (A-E) present; front matter complete; date injection present |
| `docs/testing/phase-3-smoke-tests.md` | Manual test procedure for all 10 PULSE scenarios | ✓ VERIFIED | 463 lines; all 10 PULSE IDs present; 3 sub-tests for Scenario 4; traceability table at footer |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| `pulse.md` | `.team-health/baselines.json` | Read tool reads existing baselines (Phase A); Write tool writes full updated file atomically (Phase E) | ✓ WIRED | Lines 32, 160: explicit Read + single atomic Write instructions; CRITICAL note on ordering (Phase D before Phase E) present |
| `pulse.md` | `.team-health/pulse-history/<ISO-week>.md` | Write tool after baselines.json update; mkdir -p creates directory on first run | ✓ WIRED | Lines 166-167: `Bash(mkdir -p .team-health/pulse-history)` guard + Write instruction explicit |
| `pulse.md` | `.claude/team-health/SIGNALS.md + BASELINES.md + PRIVACY.md` | Read tool loads all three reference docs before any analysis | ✓ WIRED | Lines 23-25: Required Reference Reads section lists all three in order; follow-reference instructions on lines 27, 85, 94, 149 |
| `pulse.md` | `.team-health/config.json` | Pre-flight Check reads config.json; sources map gates each MCP section; team array drives person loop | ✓ WIRED | Lines 14, 35, 43, 49, 151: config.json referenced in Pre-flight, Phase A sources map, Phase B iteration, and Phase E window calculation |
| `docs/testing/phase-3-smoke-tests.md` | `.claude/commands/team-health/pulse.md` | Smoke test scenarios define observable behaviors the pulse command must produce | ✓ WIRED | smoke-tests.md references `/team-health:pulse` as command under test (line 4) and in all 10 invocation blocks |

---

## Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|---------|
| PULSE-01 | 03-00, 03-01 | Scans all configured direct reports | ✓ SATISFIED | Phase B: `For each team member in config.json team array (iterate in order)`; Scenario 1 covers all 3 members including new member (Carol) |
| PULSE-02 | 03-00, 03-01 | Per-person scoring uses personal 8-week baseline, not team averages | ✓ SATISFIED | Phase C: compares against `CURRENT_BASELINES.people[slug]`; Phase D line 135: `DO NOT compare one person to another`; Scenario 2 explicitly tests Alice vs. Bob differential |
| PULSE-03 | 03-00, 03-01 | Correct signals tracked per available source | ✓ SATISFIED | Phase B: all 4 source blocks with exact signal names; Phase A: names unavailable sources; 4 GitHub + 3 Jira + 2 Slack + 2 Calendar = 11 signals |
| PULSE-04 | 03-00, 03-01 | Conservative two-signal flagging rule | ✓ SATISFIED | Phase C lines 94-97: verbatim RED/YELLOW/GREEN rules; stddev==0 edge case line 91; lag directionality line 92-93; Scenario 4 A/B/C sub-tests |
| PULSE-05 | 03-00, 03-01 | Summary table all members; detail sections flagged only | ✓ SATISFIED | Phase D: table row for every member; detail section only when `flag_status is YELLOW or RED`; line 133: `DO NOT render a detail section for GREEN or PENDING` |
| PULSE-06 | 03-00, 03-01 | Flagged signal output states metric, delta, source | ✓ SATISFIED | Phase D lines 126-128: exact format template with human-readable metric, current value, 8-week avg, delta, stddev delta, Source: [MCP name]; Scenario 6 verifies all 6 elements |
| PULSE-07 | 03-00, 03-01 | Verbatim disclaimer at end of every output | ✓ SATISFIED | Phase D line 140: verbatim disclaimer; Phase E line 190: same verbatim disclaimer in pulse-history snapshot; Scenario 7 verifies character-level match |
| PULSE-08 | 03-00, 03-01 | baselines.json written with provenance fields | ✓ SATISFIED | Phase E lines 156-161: computed_to, computed_from, schema_version, last_updated, source array all specified; first-run behavior for Carol explicit |
| PULSE-09 | 03-00, 03-01 | Pulse history snapshots written to pulse-history/YYYY-WNN.md | ✓ SATISFIED | Phase E lines 164-190: mkdir -p guard + ISO week filename + full snapshot structure; macOS compatibility risk documented in smoke tests |
| PULSE-10 | 03-00, 03-01 | Graceful degradation names unavailable sources | ✓ SATISFIED | Phase A lines 37-39: all-sources-false handler; names all 4; delegates to PRIVACY.md phrasing; renders summary table + disclaimer before stopping |

**No orphaned requirements.** All 10 PULSE IDs appear in both Plan 03-00 and Plan 03-01 requirements lists and are covered in REQUIREMENTS.md exactly as Phase 3.

---

## Anti-Patterns Found

| File | Pattern | Severity | Impact |
|------|---------|----------|--------|
| None found | — | — | — |

No TODO, FIXME, placeholder comments, empty implementations, or stub handlers found in either deliverable.

---

## Human Verification Required

### 1. Full 10-Scenario Smoke Test Suite

**Test:** In a live Claude Code session with Phase 1 setup complete and GitHub MCP configured, run all 10 scenarios from `docs/testing/phase-3-smoke-tests.md` in order, manually setting up `baselines.json` fixture state per each scenario's preconditions.

**Expected:** Each scenario's pass criteria are met without deviation.

**Why human:** `pulse.md` is a Claude Code slash command (a prompt file). Its correctness is the runtime behavior of the LLM following the instructions — not statically verifiable. The file structure, all phases, logic, and text are confirmed correct; live execution is the only way to confirm the LLM follows the instructions faithfully.

### 2. Conservative Flagging False-Negative Path (Scenario 4A)

**Test:** Set Bob's `baselines.json` to have 1 signal at -1.5 stddev, all others within range. Run `/team-health:pulse`.

**Expected:** Bob shows GREEN. No detail section appears for Bob.

**Why human:** The two-signal rule false-negative (do NOT flag) branch is the most critical behavioral check — over-flagging would erode manager trust. Cannot confirm without live run against fixture data.

### 3. Verbatim Disclaimer Check (Scenario 7)

**Test:** Run `/team-health:pulse` with an all-GREEN team state. Copy the last block of output.

**Expected:** Last block is exactly: "These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions." — character-for-character.

**Why human:** LLMs can paraphrase even when instructed not to. The disclaimer text is embedded verbatim in Phase D, but live invocation is required to confirm it is emitted without modification.

### 4. macOS Date Compatibility — pulse-history Filename (Scenario 9)

**Test:** Before running Scenario 9, run `date +%Y-W%V` in terminal. Then run `/team-health:pulse` on a fresh install (no pulse-history directory).

**Expected:** `date +%Y-W%V` produces a valid ISO week string (e.g., `2026-W11`). The pulse-history directory is created. A file named `2026-WNN.md` (matching that week) is created inside it.

**Why human:** macOS BSD `date` may not support `%V`. This is a documented risk in the smoke tests and plan. Cannot verify without running on the actual macOS target machine.

### 5. Graceful Degradation Phrasing Match (Scenario 10)

**Test:** Set all 4 sources to `false` in `config.json`. Run `/team-health:pulse`. Open `.claude/team-health/PRIVACY.md` and locate the "Graceful Degradation Language" section. Compare the output phrasing character-by-character.

**Expected:** Phrasing matches PRIVACY.md exactly for each of the 4 unavailable sources. Summary table renders. Disclaimer present at end.

**Why human:** Phrasing compliance requires cross-referencing live output against the PRIVACY.md source. Character-level verification is not possible without running the command.

---

## Gaps Summary

No gaps. All automated checks passed:

- Both primary deliverables exist and are substantive (190-line pulse command, 463-line smoke test document)
- All 4 key links are wired and explicit
- All 10 PULSE requirements are implemented in pulse.md and have explicit test coverage in the smoke test document
- No anti-patterns, placeholders, or stubs found
- Front matter is complete and correct (`disable-model-invocation: true`, all required `allowed-tools`, date injection)
- The two-signal flagging logic is verbatim-faithful including the stddev==0 edge case and lag-signal directionality inversion
- Atomic baselines.json write is enforced with an explicit `DO NOT call Write multiple times` constraint
- Output-before-state ordering is enforced (Phase D before Phase E) with an explicit CRITICAL note

Phase goal is structurally achieved. Human verification of live command execution is required before marking Phase 3 complete.

---

_Verified: 2026-03-17_
_Verifier: Claude (gsd-verifier)_
