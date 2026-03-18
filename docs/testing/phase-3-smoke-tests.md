# Phase 3 - Team Pulse: Manual Smoke Tests

**Phase:** 3 - Team Pulse
**Command under test:** `/team-health:pulse`
**Document status:** Wave 0 validation contract (written before command implementation)
**Purpose:** This document is the verification anchor for Plan 03-01. The executor of the pulse command verifies their work against these scenarios.

---

## Prerequisites

Before running any scenario, confirm all of the following:

1. Phase 1 setup is complete - `.team-health/config.json` exists with `setup_complete: true`
2. At least two team members are configured in `config.json`
3. `.claude/team-health/SIGNALS.md`, `BASELINES.md`, and `PRIVACY.md` all exist
4. A live Claude Code session is open with the project loaded

### Canonical Test Fixtures

All scenarios use these fixtures unless overridden in the scenario preconditions:

**Team members:**

| Name | Slug | GitHub username | Baseline status |
|------|------|-----------------|-----------------|
| Alice Chen | `alice-chen` | `achen` | 8 weeks of data present |
| Bob Smith | `bob-smith` | `bsmith` | 8 weeks of data present |
| Carol Davis | `carol-davis` | `cdavis` | NEW - no baseline entry |

**Source configuration (default for most scenarios):**

```json
"sources": { "github": true, "jira": false, "slack": false, "calendar": false }
```

**Baseline window:** 8 weeks (`baseline_window_weeks: 8`)

---

## Scenario 1 - PULSE-01: Command scans all configured team members

**Requirement:** PULSE-01

**Precondition:**
- `config.json` has 3 team members: Alice Chen, Bob Smith, Carol Davis
- `.team-health/baselines.json` exists with entries for `alice-chen` and `bob-smith`
- No entry exists in `baselines.json` for `carol-davis`
- `sources.github = true`, all others false

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Output includes a summary table listing all 3 team members: Alice Chen, Bob Smith, and Carol Davis
2. No team member is silently skipped or omitted from the output
3. Carol Davis's row in the summary table shows "(baseline pending - 0 weeks of data)" instead of a green/yellow/red status indicator
4. Alice and Bob show status indicators (GREEN, YELLOW, or RED) based on their actual baseline comparison

**Fail indicators:**
- Only 2 rows in the summary table
- Carol Davis does not appear at all
- Carol Davis shows a GREEN/YELLOW/RED status without having any baseline history

---

## Scenario 2 - PULSE-02: Scoring uses personal baseline, not team averages

**Requirement:** PULSE-02

**Precondition:**
- Alice Chen has a strong baseline: `github_pr_review_count_per_week` mean = 4.0/week, stddev = 0.5 (window: [4, 4, 5, 4, 4, 3, 4, 4])
- Bob Smith has a weaker baseline: `github_pr_review_count_per_week` mean = 1.0/week, stddev = 0.5 (window: [1, 1, 1, 0, 1, 1, 1, 2])
- This week, BOTH Alice and Bob reviewed exactly 2 PRs

**Expected calculation:**
- Alice: current=2, mean=4.0, stddev=0.5 → deviation = (2−4.0)/0.5 = −4.0 stddev → FLAGGED (exceeds 2 stddev threshold)
- Bob: current=2, mean=1.0, stddev=0.5 → deviation = (2−1.0)/0.5 = +2.0 stddev → ABOVE baseline, NOT flagged

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Alice is flagged (YELLOW or RED) due to the PR review count drop relative to her own 4.0/week baseline
2. Bob is NOT flagged (GREEN) despite reviewing the same absolute number of PRs as Alice
3. Alice's detail section references "8-week avg 4.0/week" or similar language specific to her baseline
4. Bob's row references his own "8-week avg 1.0/week" (or similar) - NOT Alice's values
5. Output contains NO phrase like "team average", "compared to team", "team benchmark", or "team norm"
6. All comparisons are phrased as "[person] vs. their own baseline" - never "[person A] vs. [person B]"

**Fail indicators:**
- Both Alice and Bob show GREEN (missed Alice's significant drop)
- Both Alice and Bob show the same status despite opposite deviations from their personal baselines
- Output mentions "team average" or compares one person's metrics to another's

---

## Scenario 3 - PULSE-03: Correct signals tracked per available source

**Requirement:** PULSE-03

**Precondition:**
- `sources.github = true`, `sources.jira = false`, `sources.slack = false`, `sources.calendar = false`
- Alice and Bob both have 8-week baselines for all 4 GitHub signals

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Output includes a "Sources active: GitHub" header line (or equivalent)
2. Output explicitly names Jira, Slack, and Calendar as unavailable - they must not be silently absent
3. The 4 GitHub signals are checked for each team member:
   - PR merges per week (`github_prs_merged_per_week`)
   - PR review count per week (`github_pr_review_count_per_week`)
   - PR review lag in days (`github_pr_review_lag_days`)
   - Commit days per week (`github_commit_days_per_week`)
4. Output does NOT claim to have checked Jira, Slack, or Calendar signals
5. The 7 signals from Jira/Slack/Calendar appear in the output as unavailable/not checked - not silently omitted

**Fail indicators:**
- No mention of which sources are active or unavailable
- Fewer than 4 GitHub signals appear to be checked
- Output claims to have checked Jira/Slack/Calendar signals when those sources are disabled
- Unavailable sources are simply not mentioned (silent omission)

---

## Scenario 4 - PULSE-04: Conservative two-signal flagging rule

**Requirement:** PULSE-04

This scenario requires three sub-tests run back-to-back with different `baselines.json` states. Modify `baselines.json` manually before each sub-test to inject the required baseline data for Bob Smith.

---

### Sub-test A - Should NOT flag (one signal at moderate deviation)

**Setup:** Modify Bob's baselines.json entry so that:
- `github_pr_review_count_per_week`: current week = 2, mean = 3.2, stddev = 0.8 → deviation = −1.5 stddev
- All other signals for Bob are within normal range (within ±1 stddev)

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Bob shows GREEN in the summary table
2. No detail section appears for Bob below the summary table
3. Reasoning: 1 signal at −1.5 stddev does NOT meet the two-signal rule threshold (requires 2+ signals OR 1 signal at ≥2 stddev)

**Fail indicators:**
- Bob shows YELLOW or RED
- A detail section appears for Bob

---

### Sub-test B - Should flag YELLOW (two signals at ≥1 stddev)

**Setup:** Modify Bob's baselines.json entry so that:
- `github_pr_review_count_per_week`: current week = 2, mean = 4.0, stddev = 1.0 → deviation = −2.0 stddev
- `github_commit_days_per_week`: current week = 2, mean = 4.0, stddev = 1.5 → deviation = −1.3 stddev
- All other signals for Bob are within normal range

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Bob shows YELLOW in the summary table
2. A detail section appears for Bob listing both triggering signals
3. The detail section includes the delta for each triggering signal (absolute and stddev units)
4. Reasoning: 2 signals at ≥1 stddev each → meets the two-signal rule → YELLOW flag

**Fail indicators:**
- Bob shows GREEN (false negative - missed two-signal pattern)
- Bob shows RED when only YELLOW criteria are met
- Detail section missing or shows only one of the two triggering signals

---

### Sub-test C - Should flag RED (one signal at ≥2 stddev)

**Setup:** Modify Bob's baselines.json entry so that:
- `github_pr_review_count_per_week`: current week = 0, mean = 4.2, stddev = 0.7 → deviation = (0−4.2)/0.7 = −6.0 stddev
- All other signals for Bob are within normal range

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Bob shows RED in the summary table
2. A detail section appears for Bob with the single triggering signal
3. The detail section states: metric name, current value (0 reviews), baseline avg (4.2/week), absolute delta (−4.2), stddev delta (−6.0 stddev)
4. Reasoning: 1 signal at ≥2 stddev → meets the single-signal severity threshold → RED flag

**Fail indicators:**
- Bob shows GREEN (false negative)
- Bob shows YELLOW when RED is warranted (≥2 stddev single signal = RED, not YELLOW)
- Detail section missing or omits the stddev delta

---

## Scenario 5 - PULSE-05: Summary table includes all members; detail sections for flagged only

**Requirement:** PULSE-05

**Precondition:**
- Alice Chen: GREEN (all signals within normal range)
- Bob Smith: YELLOW (2 signals at ≥1 stddev, as in Sub-test B of Scenario 4)
- Carol Davis: no baseline data (0 weeks)

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. The summary table has exactly 3 rows - one for each team member
2. Bob Smith has a detail section below the summary table
3. Alice Chen does NOT have a detail section (GREEN members have table row only)
4. Carol Davis does NOT have a detail section (baseline-pending members have table row only, with pending label)
5. The order in the output: summary table first, then detail sections for flagged members only

**Fail indicators:**
- Summary table has fewer than 3 rows
- Alice has a detail section when she is GREEN
- Carol has a detail section when she has no baseline data
- Bob's detail section is missing when he is YELLOW

---

## Scenario 6 - PULSE-06: Flagged signal output states metric, delta, and source

**Requirement:** PULSE-06

**Precondition:**
- Bob Smith is YELLOW, flagged specifically on PR review count
- Bob's baselines.json entry: `github_pr_review_count_per_week` window = [4, 4, 3, 4, 4, 4, 3, 4], mean = 3.8, stddev = 0.4
- This week Bob reviewed 0 PRs → deviation = (0−3.8)/0.4 = −9.5 stddev

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:** Bob's detail section must contain ALL of the following elements:

1. **Metric name:** "PR review count" (or `github_pr_review_count_per_week` - human-readable form required)
2. **Current value:** "0 reviews this week" (explicit zero, not omitted)
3. **Baseline average:** "8-week avg 3.8/week" (or equivalent - must reference the 8-week window)
4. **Absolute delta:** "−3.8" (negative delta from baseline mean)
5. **Standard deviation delta:** "−9.5 stddev" (or similar - stddev units must be present)
6. **Source attribution:** "GitHub MCP" (identifies which data source provided this signal)

Output must NOT:
- Use phrases like "engagement indicators" or "activity patterns" (opaque)
- Omit any of the 6 required elements listed above
- Show only a composite score without naming the specific metric

**Example of passing output:**
```
- **PR review count:** 0 reviews this week vs. 8-week avg 3.8/week (−3.8, −9.5 stddev)
  Source: GitHub MCP
```

**Fail indicators:**
- Any of the 6 elements (metric, current value, baseline avg, absolute delta, stddev delta, source) is missing
- Language like "engagement indicators" or "below normal" without specifics
- Delta shown without units or without both absolute and stddev forms

---

## Scenario 7 - PULSE-07: Disclaimer appears verbatim at end of every output

**Requirement:** PULSE-07

**Precondition:** Use any valid pulse run state - Scenario 1's setup (3 team members, GitHub only, Carol baseline-pending) is sufficient.

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. The LAST paragraph (or final block of text) of the pulse output is EXACTLY:

   > These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.

2. No paraphrase is accepted - the text must match character-for-character (punctuation included)
3. The disclaimer appears even when all team members are GREEN
4. The disclaimer appears after any detail sections, not before the summary table

**How to verify:** Copy the last paragraph of the output and compare character-by-character to the required text. Even minor rewordings (e.g., "These are signals" or "always talk to your team") are a FAIL.

**Fail indicators:**
- Last paragraph differs from the required text by even one word
- Disclaimer is present but not at the end of the output (e.g., appears in the middle)
- Disclaimer is absent when all team members are GREEN
- Disclaimer is omitted entirely

---

## Scenario 8 - PULSE-08: baselines.json written with provenance fields after each run

**Requirement:** PULSE-08

**Precondition:**
- `.team-health/baselines.json` exists with Alice's baseline data (8-week window, all 4 GitHub signals)
- Carol Davis has no entry in baselines.json (fresh team member)
- `sources.github = true`, all others false

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:** After the run, open `.team-health/baselines.json` and verify:

**Top-level fields:**
- `schema_version`: `"1"` (string, not integer)
- `last_updated`: today's date in `YYYY-MM-DD` format

**Alice's entry (`people.alice-chen`):**
- `computed_from`: a date string (YYYY-MM-DD) representing the start of her baseline window
- `computed_to`: today's run date (YYYY-MM-DD)
- `source`: an array listing active sources (e.g., `["github"]`)
- `window_weeks`: `8`

**Per-metric sub-objects (for Alice, at minimum `github_pr_review_count_per_week`):**
- `window`: array of up to 8 numeric values
- `mean`: numeric (recomputed after appending this week's value)
- `stddev`: numeric
- `last_value`: this week's observed value

**Carol's entry (`people.carol-davis`) - first-run behavior:**
- Entry is CREATED in baselines.json (Carol did not exist before)
- `window`: array with exactly 1 element (this week's value)
- `mean`: equals `window[0]` (the single observed value)
- `stddev`: `0` (special case for single-element window)
- `computed_to`: today's date
- `computed_from`: today's date (start of window)

**Fail indicators:**
- `schema_version` missing from top-level
- `computed_from` or `computed_to` missing from any person's entry
- `source` field missing (should be an array of source names)
- Carol's entry not created after her first pulse run
- Carol shows `stddev` ≠ 0 on first run
- `computed_to` does not match today's run date

---

## Scenario 9 - PULSE-09: Pulse history snapshot written to pulse-history/

**Requirement:** PULSE-09

**Precondition:**
- `.team-health/pulse-history/` directory does NOT exist (simulate fresh install by deleting it if present)
- Alice and Bob have 8-week baselines; Carol is baseline-pending
- `sources.github = true`, all others false

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. The directory `.team-health/pulse-history/` is created
2. A file `.team-health/pulse-history/<YYYY-WNN>.md` exists, where `YYYY-WNN` is the current ISO week (e.g., `2026-W11`)
3. The history file contains:
   - Run date (today's YYYY-MM-DD)
   - Sources active and unavailable
   - Summary table with all team members and their status
   - Detail section for any flagged member (YELLOW or RED)
   - The verbatim disclaimer at the end
4. Running pulse again the FOLLOWING week creates a NEW file (`2026-W12.md`) - it does NOT overwrite the previous week's file

**How to verify week-over-week:** After running once, manually verify the filename matches today's ISO week. Then note the filename and confirm the next week's run produces a different filename.

**macOS date compatibility note:** Before running this scenario, verify that `date +%Y-W%V` produces a valid ISO week number in your terminal. On macOS (BSD `date`), `%V` may not be supported - if `date +%Y-W%V` returns an error or unexpected output (e.g., `%V` literal), the pulse command will use a fallback format. Note the actual filename format used in your test result. If the fallback format is used, document it (e.g., `2026-W11-fallback`) and verify it is consistent across runs.

**Fail indicators:**
- `.team-health/pulse-history/` not created on first run
- History file not created, or created at wrong path
- History file created but missing run date, sources, summary table, or disclaimer
- Second run in same week overwrites the existing history file with new content
- Second run in the following week overwrites the previous week's file (no accumulation)

---

## Scenario 10 - PULSE-10: Graceful degradation names all unavailable sources

**Requirement:** PULSE-10

**Precondition:**
- Set `config.json` sources to ALL false: `{ "github": false, "jira": false, "slack": false, "calendar": false }`
- Alice and Bob have existing baseline data; Carol is baseline-pending

**Invocation:**
```
/team-health:pulse
```

**Pass criteria:**
1. Output does NOT error, crash, or produce a stack trace
2. Output explicitly names ALL 4 unavailable sources: GitHub, Jira, Slack, Calendar
3. Each unavailable source uses the degradation phrasing from `.claude/team-health/PRIVACY.md` - not invented or paraphrased language
4. Summary table still renders - showing "no signals available" or "baseline pending" (or equivalent) for each person since no source data was collected
5. The verbatim disclaimer appears at the end:
   > These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.

**How to verify phrasing:** Open `.claude/team-health/PRIVACY.md` and locate the "Graceful Degradation Language" section. Compare the unavailability phrasing in the pulse output character-by-character to the PRIVACY.md text. Minor variations are a FAIL.

**Fail indicators:**
- Command errors or crashes when no sources are configured
- One or more sources are not mentioned in the output (silent omission)
- Degradation phrasing does not match PRIVACY.md language
- Summary table is absent (no output at all, or only an error message)
- Disclaimer missing from the end of output

---

## Footer

**Testing environment:** All tests are manual - invoke `/team-health:pulse` in a live Claude Code session with Phase 1 and Phase 2 setup complete.

**State files location:** `.team-health/` directory (gitignored - this directory is never committed)

**Schema reference:** `.planning/phases/01-foundation/state-schemas.json` - locked canonical schemas for `config.json`, `baselines.json`, and `people/<slug>.json`

**Reference documents:**
- `.claude/team-health/SIGNALS.md` - 11 signals, thresholds, two-signal rule, severity levels
- `.claude/team-health/BASELINES.md` - rolling baseline computation algorithm, first-run behavior, provenance fields
- `.claude/team-health/PRIVACY.md` - output language rules, required disclaimer, graceful degradation language

**macOS date compatibility note:** Before running Scenario 9, verify `date +%Y-W%V` produces correct ISO week numbers in your terminal. BSD `date` (macOS) differs from GNU `date` (Linux). If `%V` is unsupported, the pulse command's history filename will use a fallback format - note this in your test result and confirm the format is consistent across weekly runs.

**Traceability:**

| Scenario | Requirement | Key assertion |
|----------|-------------|---------------|
| 1 | PULSE-01 | All 3 team members appear in output; Carol shows baseline-pending |
| 2 | PULSE-02 | Personal baseline used; no team-average comparisons |
| 3 | PULSE-03 | 4 GitHub signals checked; 7 others named as unavailable |
| 4A | PULSE-04 | 1 signal at 1.5 stddev → GREEN (no flag) |
| 4B | PULSE-04 | 2 signals at ≥1 stddev → YELLOW (two-signal rule) |
| 4C | PULSE-04 | 1 signal at ≥2 stddev → RED (single high-deviation rule) |
| 5 | PULSE-05 | Detail sections only for flagged members; table has all members |
| 6 | PULSE-06 | Flagged signal output: metric + current value + baseline avg + delta + stddev + source |
| 7 | PULSE-07 | Verbatim disclaimer at end of every output, even all-GREEN run |
| 8 | PULSE-08 | baselines.json has schema_version, computed_from, computed_to, source, window |
| 9 | PULSE-09 | pulse-history/YYYY-WNN.md created; accumulates per week, no overwrites |
| 10 | PULSE-10 | All 4 sources named as unavailable; PRIVACY.md phrasing; no crash |
