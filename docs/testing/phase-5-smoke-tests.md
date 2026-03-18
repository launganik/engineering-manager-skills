# Phase 5 - Skip-Level and Retro Prep: Manual Smoke Tests

**Phase: 5** - Skip-Level and Retro Prep
**Commands under test:** `/team-health:skip-level` and `/team-health:retro-prep`
**Document status:** Wave 0 validation contract (written before command implementation)
**Purpose:** This document is the verification anchor for Plan 05-01. The executor of the skip-level and retro-prep commands verifies their work against these scenarios.

---

## Prerequisites

Before running any scenario, confirm all of the following:

1. Phase 1 setup is complete - `.team-health/config.json` exists with `setup_complete: true`
2. Phase 3 pulse history exists - `.team-health/pulse-history/` contains at least 2-3 weekly snapshot files
3. Phase 2 people log exists for at least one person (for `--include-person` testing) - `.team-health/people/alice-chen.json` exists with entries populated
4. `.claude/team-health/SIGNALS.md`, `BASELINES.md`, and `PRIVACY.md` all exist
5. A live Claude Code session is open with the project loaded

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

**Pulse history fixtures:**

| File | Location | Contents |
|------|----------|----------|
| `2026-W09.md` | `.team-health/pulse-history/2026-W09.md` | Alice: GREEN, Bob: YELLOW (PR review lag) |
| `2026-W10.md` | `.team-health/pulse-history/2026-W10.md` | Alice: GREEN, Bob: YELLOW (PR review lag + commit days), Carol: PENDING |
| `2026-W11.md` | `.team-health/pulse-history/2026-W11.md` | Alice: YELLOW (commit days + review count), Bob: GREEN, Carol: PENDING |

**config.json skip-level state (default):**

```json
"last_skip_level": null
```

**Alice Chen's people log fixture (`.team-health/people/alice-chen.json`)** - same as Phase 4 canonical fixture, including:
- `entries`: 5 entries including concern, win, and career categories
- `open_commitments`: 1 open commitment ("Will introduce Alice to the platform team lead")
- `career_context.stated_goals`: `["Staff Engineer track"]`
- `career_context.last_promo_discussion`: `"2025-09-15"`

---

## Scenario 1 - SKIP-01: Skip-level produces 5-section brief

**Requirement:** SKIP-01

**Preconditions:**
- 3 pulse history files present (2026-W09, W10, W11)
- `sources.github = true`, all others false
- `config.json last_skip_level = null`

**Invocation:**
```
/team-health:skip-level
```

**Pass criteria:**
1. Output contains a header line with "Team Brief" and a date range (e.g., `# Team Brief - 2026-02-24 to 2026-03-17`)
2. Output contains a "Sources" or "Covering" line listing pulse history weeks and active MCP sources
3. Output contains **exactly these 5 section headers** in this order:
   - `## 1. Delivery Status`
   - `## 2. Risks and Watch Areas`
   - `## 3. People Themes`
   - `## 4. Asks / Escalations`
   - `## 5. Team Wins`
4. Each section has substantive content (not empty or placeholder-only)
5. Section 4 contains a placeholder prompt such as "[ Add escalation items here before sharing this brief ]" - Claude does NOT fabricate escalation items

**Fail indicators:**
- Fewer than 5 section headers in the output
- Section headers use different numbering or names than the 5 specified above
- Section 4 (Asks / Escalations) is pre-populated with invented escalation items
- Output is a flat narrative with no section structure

---

## Scenario 2 - SKIP-02: Lookback defaults to "since last skip-level" or 14-day fallback

**Requirement:** SKIP-02

**Preconditions:**
- `config.json last_skip_level = null` (first run)
- Pulse history files present for 2026-W09, W10, W11

**Part A - First run (14-day default):**

**Invocation:**
```
/team-health:skip-level
```

**Pass criteria (Part A):**
1. Output states the lookback window covers approximately 14 days (2-week default)
2. Output uses language like "Covering: [14-day period] (default - no prior skip-level recorded)" or equivalent
3. After the output is fully rendered, open `.team-health/config.json` - `last_skip_level` is now set to today's date (`YYYY-MM-DD`)
4. `last_updated` in config.json is also set to today's date

**Part B - Second run (since last skip-level):**

**Invocation** (run immediately after Part A):
```
/team-health:skip-level
```

**Pass criteria (Part B):**
1. Lookback now references "since last skip-level on [today's date]" (the value just written in Part A)
2. Output clearly states the anchor date that determined the lookback window

**Fail indicators:**
- First run does not update `last_skip_level` in config.json
- Second run still uses 14-day default instead of the stored `last_skip_level`
- config.json is corrupted after the write (missing fields, truncated)
- Lookback anchor date is not disclosed in the output

---

## Scenario 3 - SKIP-02 override: --weeks flag

**Requirement:** SKIP-02

**Preconditions:**
- `config.json last_skip_level` is set to a recent date (e.g., one week ago)

**Invocation:**
```
/team-health:skip-level --weeks 4
```

**Pass criteria:**
1. Output states the lookback covers 4 weeks (28 days) from today
2. Output does NOT reference "since last skip-level" - the `--weeks 4` flag explicitly overrides the stored anchor
3. The lookback period covers 4 full weeks of pulse history (W08 through W11 if all exist)
4. config.json `last_skip_level` is still updated to today's date after the run (flag overrides the input window, not the output write)

**Fail indicators:**
- Output uses "since last skip-level" window instead of the 4-week override
- Output covers fewer than 4 weeks despite `--weeks 4` being passed
- `last_skip_level` is not updated in config.json after the `--weeks` run

---

## Scenario 4 - SKIP-03: People log excluded by default

**Requirement:** SKIP-03

**Preconditions:**
- Alice Chen has people log entries with commitments and career context (use canonical Alice fixture)
- Alice's people log contains these specific phrases: "Staff Engineer track", "introduced Alice to the platform team lead", "auth refactor ahead of schedule"
- `config.json last_skip_level = null`

**Invocation:**
```
/team-health:skip-level
```

**Pass criteria:**
1. Output contains **zero** references to any of the following:
   - "Staff Engineer track"
   - "platform team lead"
   - "auth refactor"
   - Any content that appears in alice-chen.json entries
2. No section in the output includes individual commitment details, career goals, or private log context
3. The brief reads as a purely team-level factual summary

**How to verify:** After generating the output, search it for any text that appears verbatim in `.team-health/people/alice-chen.json`. Any match is a FAIL.

**Fail indicators:**
- Any career goal, commitment text, or people log entry content appears in the output
- Output references Alice's career aspirations, open commitments, or log concerns

---

## Scenario 5 - SKIP-03 + SKIP-04: --include-person opt-in

**Requirements:** SKIP-03, SKIP-04

**Preconditions:**
- Alice Chen has people log entries (use canonical Alice fixture)
- Pulse history (2026-W11) has Alice flagged YELLOW (commit days + review count)
- Bob and Carol are also in the team roster

**Invocation:**
```
/team-health:skip-level --include-person Alice
```

**Pass criteria:**
1. Alice's people log themes appear in the People Themes section (Section 3)
2. Alice's log themes are clearly labeled as "(opt-in content)" or equivalent
3. Alice may be named in the People Themes section for the opt-in content block
4. Bob and Carol remain **unnamed** throughout the output - referenced only by count language (e.g., "N team members")
5. No other team member's people log content appears in the output

**Fail indicators:**
- Alice's log content does not appear in the output despite `--include-person Alice` being passed
- Bob or Carol are named in any section (outside the opt-in block)
- opt-in labeling is absent (Alice's content mixed in without indicating it was explicitly included)
- Both Bob and Carol's log content appears (the flag should only unlock Alice)

---

## Scenario 6 - SKIP-04: Pulse flags aggregated anonymously

**Requirement:** SKIP-04

**Preconditions:**
- Pulse history has Bob flagged YELLOW in 2026-W09 and 2026-W10 (PR review lag, commit days)
- Pulse history has Alice flagged YELLOW in 2026-W11 (commit days + review count)
- Running without any `--include-person` flag

**Invocation:**
```
/team-health:skip-level
```

**Pass criteria:**
1. The Risks and Watch Areas section (Section 2) and People Themes section (Section 3) use count language only:
   - Acceptable: "2 team members showed engagement indicator signals in recent weeks"
   - Acceptable: "1 team member was flagged for 2 consecutive weeks"
2. The output **never** contains the phrases "Bob showed...", "Alice showed...", "Bob Smith", or "Alice Chen" in the context of signal flags
3. Signal aggregations are expressed as counts and trends, not as individual attributions

**How to verify:** Search the output for each team member's first name and last name. Any occurrence in a risk, flag, or people-theme context (without `--include-person` having been passed) is a FAIL.

**Fail indicators:**
- Any team member named in a risk or flag section
- "Bob" or "Alice" appears anywhere in Sections 2-3 without `--include-person`
- Count language absent (flags described without quantification)

---

## Scenario 7 - SKIP-05: "Would my team be comfortable seeing this?" compliance

**Requirement:** SKIP-05

**Preconditions:**
- Use the output from Scenario 1 or Scenario 6 (no `--include-person` flag)

**Steps:**
1. Read the full skip-level output from a standard run
2. Search for each of the following prohibited items:
   - Individual team member names (without opt-in)
   - Diagnostic language: "burned out", "struggling", "disengaged", "checked out", "overwhelmed", "anxious", "stressed", "burnout"
   - Team-relative scoring: "bottom", "worst", "weakest", "least productive", "below the team"
   - People log content (career goals, commitments, private notes)

**Pass criteria:**
1. Zero occurrences of any prohibited word or phrase
2. All signal references use metric-and-count language ("PR review count was down across the team")
3. All people-theme language uses aggregate counts ("N team members")
4. The output reads as a factual team status brief that any team member could read without concern about privacy or surveillance

**Fail indicators:**
- Any single prohibited word appears anywhere in the output
- Output reads like an individual performance review rather than a team-level brief
- People log content surfaces without an opt-in flag

---

## Scenario 8 - RETRO-01: Retro agenda generated from sprint data

**Requirement:** RETRO-01

**Preconditions:**
- `sources.github = true`, `sources.jira = false`, all others false
- Pulse history exists for 2026-W10 and 2026-W11 (at least 2 weeks)

**Invocation:**
```
/team-health:retro-prep
```

**Pass criteria:**
1. Output contains the header line `# Sprint Retro Agenda` with a sprint identifier or date range
2. Output contains a data sources line noting which sources were used (e.g., "Data sources: GitHub MCP, pulse history weeks W10-W11")
3. Output contains a **Sprint Facts** section with at least one of: velocity, PR count, carry-over count, or avg review cycle time
4. Output does NOT error when Jira is disabled - it proceeds with GitHub and pulse history data
5. Output includes the line: "Sprint window estimated from last [N] pulse history weeks (no Jira MCP configured)" or equivalent degradation language for the sprint scope

**Fail indicators:**
- Command errors or crashes when `sources.jira = false`
- Sprint Facts section is empty or absent
- No data sources line in the output
- Output claims to have Jira sprint data when `sources.jira = false`

---

## Scenario 9 - RETRO-02: All 4 content sections present

**Requirement:** RETRO-02

**Preconditions:**
- Use the same setup as Scenario 8 (GitHub only, 2 pulse history weeks)

**Invocation:**
```
/team-health:retro-prep
```

**Pass criteria:**
1. Section `## Sprint Facts` is present and contains at least one data point
2. Section `## What Went Well - Seeds` is present and contains at least one seeded item
3. Section `## What Was Hard - Seeds` is present and contains at least one seeded item
4. Section `## Patterns Across Sprints` is present (may state "Insufficient history for cross-sprint patterns" if only 1 qualifying week, but must be present as a section)
5. Section `## Action Items` is present (blank - team fills in during retro)
6. All 5 sections appear in the output in the order listed above

**Fail indicators:**
- Any of the 5 sections is absent from the output
- Sections appear in a different order than specified
- The "What Went Well" or "What Was Hard" sections have zero seeded items (entirely empty)

---

## Scenario 10 - RETRO-03: Seeds attributed to work items, not individuals

**Requirement:** RETRO-03

**Preconditions:**
- `sources.github = true`, pulse history exists for 2+ weeks
- GitHub has PR data including at least one PR with multiple revision cycles

**Invocation:**
```
/team-health:retro-prep
```

**Steps:**
1. Run the command and capture the output
2. Read the `## What Went Well - Seeds` section
3. Read the `## What Was Hard - Seeds` section
4. Search both sections for team member names: "Alice", "Bob", "Carol", "Alice Chen", "Bob Smith", "Carol Davis"

**Pass criteria:**
1. Every seeded item in "What Went Well" references a work item, PR title, ticket, or work area - NOT a person:
   - Acceptable: "Auth service refactor shipped - 22 PRs merged cleanly"
   - Acceptable: "Sprint velocity was 15% above 8-week baseline"
   - Prohibited: "Alice shipped the auth refactor"
2. Every seeded item in "What Was Hard" references a work item, process, or metric - NOT a person:
   - Acceptable: "The payments API PR went through 7 revision cycles - what happened?"
   - Prohibited: "Bob had 7 revision cycles on his PR"
3. Zero occurrences of any team member name in the seeded items of either section

**Fail indicators:**
- Any team member name appears in a seeded discussion item
- Observations are attributed to individuals ("Jordan's PRs", "Alice's reviews")
- Seeds are purely abstract with no work-item references

---

## Scenario 11 - RETRO-04: Blank discussion space and empty action items

**Requirement:** RETRO-04

**Preconditions:**
- Use the output from Scenario 8 or 9 (any standard retro-prep run)

**Steps:**
1. Read the full retro-prep output
2. Examine each of the content sections: "What Went Well", "What Was Hard", "Patterns Across Sprints"
3. Examine the "Action Items" section

**Pass criteria:**
1. Each content section (`## What Went Well - Seeds`, `## What Was Hard - Seeds`, `## Patterns Across Sprints`) contains a visible "Team discussion space" marker, such as:
   - `**[ Team discussion space - what else went well? ]**`
   - `**[ Team discussion space - what else was hard? ]**`
   - `**[ Team discussion space - what patterns do you see? ]**`
2. The `## Action Items` section contains **only** blank checkboxes (3-5 of them):
   - `- [ ]`
   - `- [ ]`
   - `- [ ]`
3. The Action Items section has **zero** pre-populated items - Claude must not write any action item text
4. The blank discussion space markers appear at the end of each section (after seeded items), not replacing them

**Fail indicators:**
- Any content section is missing its "Team discussion space" marker
- Action Items section has pre-written items (e.g., "Alice should improve review turnaround")
- Action Items section is entirely absent (not just blank)
- Blank checkboxes have content after them

---

## Scenario 12 - Edge case: No pulse history available

**Preconditions:**
- `.team-health/pulse-history/` directory is empty or does not exist
- `sources.github = true`, all others false

**Invocation:**
```
/team-health:skip-level
```

**Pass criteria:**
1. Command does **not** error - output is generated without a stack trace or failure message
2. Output states "No pulse history found for this period" or equivalent
3. If GitHub MCP is active, the brief continues with available MCP data and notes the limitation
4. If no MCPs are active either, the command states the limitation and stops gracefully (no error)
5. Output does not produce malformed sections or partial output

**Fail indicators:**
- Command crashes or errors on missing pulse-history directory
- Output is blank with no explanatory message
- Command silently produces a brief as if data existed (fabricated data)

---

## Scenario 13 - Edge case: Retro-prep without Jira

**Preconditions:**
- `sources.jira = false`, `sources.github = true`
- Pulse history exists for 2026-W10 and 2026-W11

**Invocation:**
```
/team-health:retro-prep
```

**Pass criteria:**
1. Output states that the sprint window was estimated from pulse history (not Jira): "Sprint window estimated from last [N] pulse history weeks (no Jira MCP configured)" or equivalent
2. GitHub data populates PR-related content in Sprint Facts and seed sections
3. Sprint Facts section shows **degradation language** for Jira-specific data:
   - Velocity (from Jira tickets closed) - either omitted with explanation or stated as "unavailable - Jira MCP not configured"
   - Carry-over count - same: unavailable or estimated from pulse flags
4. The output is coherent and useful despite missing Jira data
5. No Jira ticket data is fabricated when `sources.jira = false`

**Fail indicators:**
- Command errors when `sources.jira = false`
- Jira unavailability is silently omitted (no mention of the limitation)
- Jira-specific data (velocity, carry-over) appears in the output without `sources.jira = true`
- Sprint window defaults to 8 weeks instead of the configured sprint cadence

---

## Footer

**Testing environment:** All tests are manual - invoke `/team-health:skip-level` or `/team-health:retro-prep` in a live Claude Code session with Phases 1-4 setup complete and pulse history populated.

**State files location:** `.team-health/` directory (gitignored - this directory is never committed)

**Schema reference:** `.planning/phases/01-foundation/state-schemas.json` - locked canonical schemas for `config.json`, `baselines.json`, and `people/<slug>.json`

**Reference documents:**
- `.claude/team-health/SIGNALS.md` - 11 signals, thresholds, two-signal rule, severity levels
- `.claude/team-health/BASELINES.md` - rolling baseline computation algorithm, first-run behavior, provenance fields
- `.claude/team-health/PRIVACY.md` - output language rules, required disclaimer, graceful degradation language

**Skip-level lookback window logic:** `--weeks N` / `--quarter` override → `last_skip_level` anchor → 14-day default (in priority order). See 05-RESEARCH.md for full derivation.

**Retro sprint scope logic:** `--sprint <id>` → Jira most recent sprint → most recent `sprint_cadence_weeks` pulse history weeks (in priority order). See 05-RESEARCH.md for fallback details.

**Traceability:**

| Scenario | Requirement | Key assertion |
|----------|-------------|---------------|
| 1 | SKIP-01 | 5-section brief generated; Team Brief header + date range; Section 4 placeholder only |
| 2 | SKIP-02 | First run: 14-day default, last_skip_level written. Second run: "since last skip-level" |
| 3 | SKIP-02 | --weeks 4 overrides stored anchor; still updates last_skip_level |
| 4 | SKIP-03 | People log content absent from output without --include-person |
| 5 | SKIP-03, SKIP-04 | --include-person Alice shows Alice's log themes labeled "(opt-in content)"; Bob and Carol unnamed |
| 6 | SKIP-04 | All pulse flags aggregated anonymously - count language only; no individual names |
| 7 | SKIP-05 | Zero prohibited words; no individual names; no people log content; team-comfort test |
| 8 | RETRO-01 | Retro agenda generated from sprint data; sprint window stated; graceful without Jira |
| 9 | RETRO-02 | All 4 content sections + Action Items present in output |
| 10 | RETRO-03 | Zero individual names in seed items; all attributed to work items |
| 11 | RETRO-04 | Blank discussion space in every section; Action Items blank checkboxes only |
| 12 | - | No pulse history → no crash; limitation stated; continues gracefully |
| 13 | - | Jira disabled → sprint scope estimated from pulse history; Jira data degradation language |
