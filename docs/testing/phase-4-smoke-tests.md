# Phase 4 - 1:1 Prep: Manual Smoke Tests

**Phase: 4** - 1:1 Prep
**Command under test:** `/team-health:prep <name>`
**Document status:** Wave 0 validation contract (written before command implementation)
**Purpose:** This document is the verification anchor for Plan 04-01. The executor of the prep command verifies their work against these scenarios.

---

## Prerequisites

Before running any scenario, confirm all of the following:

1. Phase 1 setup is complete - `.team-health/config.json` exists with `setup_complete: true`
2. Phase 2 people log exists for at least one person with entries, `open_commitments`, and `career_context` populated
3. Phase 3 baselines exist - `.team-health/baselines.json` exists with baseline data for at least one person
4. At least one MCP source is configured (`github: true` by default in fixtures)
5. A live Claude Code session is open with the project loaded

### Canonical Test Fixtures

All scenarios use these fixtures unless overridden in the scenario preconditions:

**Team members:**

| Name | Slug | Baseline status | People log status |
|------|------|-----------------|-------------------|
| Alice Chen | `alice-chen` | 8 weeks of data present | Full data (entries + open commitments + career context) |
| Bob Smith | `bob-smith` | 8 weeks of data present | Partial (entries, no open commitments) |
| Carol Davis | `carol-davis` | NEW - no baseline entry | No people log file |

**Source configuration (default for most scenarios):**

```json
"sources": { "github": true, "jira": false, "slack": false, "calendar": false }
```

**Alice Chen's people log fixture (`.team-health/people/alice-chen.json`):**

```json
{
  "schema_version": "1",
  "name": "Alice Chen",
  "slug": "alice-chen",
  "last_updated": "2026-03-17",
  "last_1on1_date": "2026-03-10",
  "entries": [
    {
      "id": "2026-03-12-001",
      "date": "2026-03-12",
      "category": "concern",
      "content": "Alice mentioned she's been swamped with review requests from other teams",
      "created_at": "2026-03-12T10:00:00Z"
    },
    {
      "id": "2026-03-08-001",
      "date": "2026-03-08",
      "category": "concern",
      "content": "Alice seemed quieter than usual in team sync - may be stretched thin",
      "created_at": "2026-03-08T14:00:00Z"
    },
    {
      "id": "2026-03-03-001",
      "date": "2026-03-03",
      "category": "concern",
      "content": "Alice flagged that she's blocked on the platform team API access",
      "created_at": "2026-03-03T09:00:00Z"
    },
    {
      "id": "2026-02-28-001",
      "date": "2026-02-28",
      "category": "win",
      "content": "Alice shipped the auth refactor ahead of schedule",
      "created_at": "2026-02-28T16:00:00Z"
    },
    {
      "id": "2026-02-20-001",
      "date": "2026-02-20",
      "category": "career",
      "content": "Alice confirmed she's targeting Staff Engineer track - wants to lead a project by Q3",
      "created_at": "2026-02-20T11:00:00Z"
    }
  ],
  "open_commitments": [
    {
      "id": "2026-03-05-001",
      "date": "2026-03-05",
      "content": "Will introduce Alice to the platform team lead",
      "status": "open"
    }
  ],
  "career_context": {
    "stated_goals": ["Staff Engineer track"],
    "last_promo_discussion": "2025-09-15",
    "notes": ""
  }
}
```

**Bob Smith's people log fixture (`.team-health/people/bob-smith.json`):**

```json
{
  "schema_version": "1",
  "name": "Bob Smith",
  "slug": "bob-smith",
  "last_updated": "2026-03-12",
  "prep_run": "2026-03-12",
  "entries": [
    {
      "id": "2026-03-10-001",
      "date": "2026-03-10",
      "category": "note",
      "content": "Bob asked about the on-call rotation schedule",
      "created_at": "2026-03-10T09:00:00Z"
    },
    {
      "id": "2026-03-05-001",
      "date": "2026-03-05",
      "category": "win",
      "content": "Bob's PR for the payments API was merged after a thorough review cycle",
      "created_at": "2026-03-05T15:00:00Z"
    }
  ],
  "open_commitments": [],
  "career_context": {
    "stated_goals": [],
    "last_promo_discussion": null,
    "notes": ""
  }
}
```

**Note:** Carol Davis has no people log file (`.team-health/people/carol-davis.json` does not exist) and no entry in `baselines.json`.

---

## Scenario 1 - PREP-01: Prep sheet generated and scannable

**Requirement:** PREP-01

**Precondition:**
- Alice Chen configured in `config.json`
- `alice-chen.json` people log exists with the canonical fixture data above
- `baselines.json` has 8-week baseline data for Alice
- `sources.github = true`, all others false

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Output contains the header `1:1 Prep: Alice Chen`
2. Output contains a lookback line referencing the period start date (e.g., "since last 1:1 on 2026-03-10")
3. Output is structured with exactly 5 numbered section headers
4. Total output fits within approximately 60 lines (the "2-minute scan" test - not a hard character count, but the overall length should feel like a briefing, not a data dump)
5. After the output is fully rendered, the people log file `.team-health/people/alice-chen.json` has a `prep_run` field set to today's date (`YYYY-MM-DD`)

**Fail indicators:**
- Output is a raw data dump without section structure
- `prep_run` is not written to the people log file after the run
- Header is missing or uses wrong name/date format
- Output exceeds ~100 lines (too long to scan in 2 minutes)

---

## Scenario 2 - PREP-02: All five output sections present

**Requirement:** PREP-02

**Precondition:**
- Alice Chen with full data (people log entries, open commitments, career context, baselines, GitHub source active)
- Use canonical Alice fixture

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Section "1. Status Snapshot" is present (or equivalent numbered header)
2. Section "2. Signal Flags" is present (or equivalent numbered header)
3. Section "3. Standing Items from People Log" is present (or equivalent numbered header)
4. Section "4. Suggested Talking Points" is present (or equivalent numbered header)
5. Section "5. Context Reminders" is present (or equivalent numbered header)
6. The disclaimer line appears verbatim at the end:

   > These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.

**Fail indicators:**
- Any of the 5 sections is absent
- Disclaimer is paraphrased, absent, or not at the end of the output

---

## Scenario 3 - PREP-03: GitHub signals in prep output

**Requirement:** PREP-03

**Precondition:**
- Alice Chen, `sources.github = true`
- `baselines.json` has 8-week baseline data for Alice covering all 4 GitHub signals
- Use canonical Alice fixture

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Section 2 (Signal Flags) mentions at least one of: PRs merged, PR review count, PR review lag, commit days
2. If any GitHub signal deviates significantly from Alice's baseline (using the two-signal rule thresholds from SIGNALS.md), the output shows the delta and the stddev units (e.g., "−2.3 stddev")
3. Section 1 (Status Snapshot) references GitHub activity in some form (e.g., "GitHub: [signal summary]")
4. The lookback window used for GitHub signal queries matches the period from `last_1on1_date` to today (not a fixed "past 7 days" window)

**Fail indicators:**
- No GitHub signals mentioned anywhere in the output
- Signal deviations shown without stddev units
- Lookback window is "past 7 days" instead of "since last 1:1"
- Output claims to check GitHub when `sources.github = false`

---

## Scenario 4 - PREP-04: Jira signals when available, degradation when not

**Requirement:** PREP-04

This scenario requires two sub-tests.

---

### Sub-test A - Jira source disabled (default fixture)

**Precondition:** `sources.jira = false` (default in canonical fixtures)

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Output contains the exact degradation phrase: "Jira signals are unavailable - Jira MCP is not configured. Ticket velocity and blocked-ticket data cannot be included."
2. The phrase matches PRIVACY.md Graceful Degradation Language character-for-character
3. No Jira ticket data is fabricated or inferred

**Fail indicators:**
- Jira unavailability is silently omitted
- Degradation phrase is paraphrased (e.g., "Jira is not set up")
- Ticket data appears in output despite `jira = false`

---

### Sub-test B - Jira source enabled

**Precondition:** `sources.jira = true`, Jira MCP is active and reachable

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Section 2 (Signal Flags) mentions at least one of: closed tickets count, blocked tickets, or in-progress tickets over 2 sprints
2. No Jira unavailability message appears
3. Signal data is scoped to the lookback window (since last 1:1), not a fixed sprint window

**Fail indicators:**
- Jira data absent despite `jira = true` and MCP active
- Blocked tickets or in-progress-over-2-sprints tickets not surfaced when present

---

## Scenario 5 - PREP-05: Calendar signals when available, degradation when not

**Requirement:** PREP-05

This scenario requires two sub-tests.

---

### Sub-test A - Calendar source disabled (default fixture)

**Precondition:** `sources.calendar = false` (default in canonical fixtures)

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Output contains the exact degradation phrase: "Calendar signals are unavailable - Calendar MCP is not configured. Meeting load and 1:1 adherence cannot be checked."
2. Phrase matches PRIVACY.md Graceful Degradation Language character-for-character
3. No meeting load or 1:1 adherence data is fabricated

**Fail indicators:**
- Calendar unavailability silently omitted
- Degradation phrase paraphrased or absent

---

### Sub-test B - Calendar source enabled

**Precondition:** `sources.calendar = true`, Calendar MCP active

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Section 2 mentions meeting load percentage or 1:1 adherence status
2. No Calendar unavailability message appears
3. Meeting load percentage is stated with a number (e.g., "Meeting load: 58% this period")

**Fail indicators:**
- Calendar data absent despite `calendar = true` and MCP active
- Meeting load shown without a numeric value

---

## Scenario 6 - PREP-06: Slack metadata only (no DM content)

**Requirement:** PREP-06

This scenario requires two sub-tests.

---

### Sub-test A - Slack source disabled (default fixture)

**Precondition:** `sources.slack = false` (default in canonical fixtures)

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Output contains the exact degradation phrase: "Slack signals are unavailable - Slack MCP is not configured. Channel participation and response latency cannot be included."
2. Phrase matches PRIVACY.md Graceful Degradation Language character-for-character
3. No Slack activity data is fabricated

**Fail indicators:**
- Slack unavailability silently omitted
- Degradation phrase paraphrased or absent

---

### Sub-test B - Slack source enabled

**Precondition:** `sources.slack = true`, Slack MCP active

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Output contains NO references to DMs, private channels, or message content (Rule 2 from PRIVACY.md)
2. Only public channel participation counts and @-mention latency appear in the output
3. Section 2 references at least one Slack metadata signal (channel participation or response latency)

**How to verify:** Search the entire output for the words "DM", "direct message", "private channel", "message content". Any occurrence is a FAIL.

**Fail indicators:**
- DM or private channel content surfaced
- Slack signals absent despite `slack = true` and MCP active
- Output contains any of the prohibited phrases above

---

## Scenario 7 - PREP-07: Open commitments surfaced

**Requirement:** PREP-07

**Precondition:**
- Alice Chen, canonical fixture
- `open_commitments` array contains the entry: `{ id: "2026-03-05-001", date: "2026-03-05", content: "Will introduce Alice to the platform team lead", status: "open" }`

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Section 3 (Standing Items from People Log) contains an "Open commitments" sub-header or label
2. The commitment content "Will introduce Alice to the platform team lead" appears in the output
3. The commitment date (2026-03-05) is shown
4. A days-open count is shown (e.g., "12 days open" - calculated from 2026-03-05 to today's date)

**Fail indicators:**
- Open commitments not surfaced in Section 3
- Commitment content omitted or paraphrased
- No date or days-open count shown
- Commitment with `status: "open"` silently skipped

---

## Scenario 8 - PREP-08: Career context surfaced

**Requirement:** PREP-08

**Precondition:**
- Alice Chen, canonical fixture
- `career_context.stated_goals = ["Staff Engineer track"]`
- `career_context.last_promo_discussion = "2025-09-15"`

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Section 5 (Context Reminders) contains "Goals:" followed by "Staff Engineer track" (or equivalent label with the goal text)
2. Section 5 contains "Last promo discussion:" referencing the date 2025-09-15 AND a months-ago calculation (e.g., "~6 months ago")
3. Section 5 shows at least the most recent 2-3 people log entries as standing context

**Fail indicators:**
- Career goals absent from Section 5
- `last_promo_discussion` date not surfaced
- Months-ago calculation missing (raw date shown without contextualizing elapsed time)
- People log entries not surfaced in Section 5

---

## Scenario 9 - PREP-09: No diagnosis language

**Requirement:** PREP-09

**Precondition:**
- Alice Chen, canonical fixture
- Alice has 3 `concern` entries in the past 6 weeks (present in canonical fixture above)
- Additionally, engineer a signal deviation: modify `baselines.json` so that Alice's `github_pr_review_count_per_week` shows: current week = 0, mean = 3.8, stddev = 0.4 → deviation = −9.5 stddev (triggers a flag)

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. The entire output contains NONE of the following words or phrases:
   - "burned out"
   - "disengaged"
   - "struggling"
   - "checked out"
   - "overwhelmed"
   - "depressed"
   - "anxious"
   - "stressed"
   - "unmotivated"
   - "burnout"
2. All flagged signals in Section 2 state specific metrics with numbers (e.g., "PR review count: 0 this period vs. 8-week avg 3.8/week")
3. All items in Section 4 (Suggested Talking Points) are phrased as questions (not statements), in the manager's voice

**How to verify Rule 1:** Search the entire output for each prohibited word. Even one occurrence is a FAIL.
**How to verify Rule 3:** Read Section 4 out loud. Each talking point should sound like a question you would ask, not a conclusion you have drawn.

**Fail indicators:**
- Any prohibited word appears anywhere in the output
- Flagged signals use opaque language like "below normal" without specifics
- Talking points are stated as facts ("Alice is overloaded") instead of questions ("How is the review load feeling right now?")

---

## Scenario 10 - PREP-10: Graceful degradation with all sources disabled

**Requirement:** PREP-10

**Precondition:**
- Set `config.json` sources to ALL false: `{ "github": false, "jira": false, "slack": false, "calendar": false }`
- Alice Chen people log exists with canonical fixture data (entries, open commitments, career context)

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. All 4 unavailable sources are named using the exact PRIVACY.md degradation phrasing:
   - "GitHub signals are unavailable - GitHub MCP is not configured. PR and commit data cannot be included."
   - "Jira signals are unavailable - Jira MCP is not configured. Ticket velocity and blocked-ticket data cannot be included."
   - "Slack signals are unavailable - Slack MCP is not configured. Channel participation and response latency cannot be included."
   - "Calendar signals are unavailable - Calendar MCP is not configured. Meeting load and 1:1 adherence cannot be checked."
2. Section 2 (Signal Flags) states "No signals available - all MCP sources are unconfigured" or equivalent
3. Sections 3–5 still render from people log data only: open commitments appear in Section 3, career context in Section 5, log entries in Section 3/5
4. Section 4 (Suggested Talking Points) generates 3–5 talking points sourced from people log data alone (commitments, career context, log patterns) - not from MCP signals
5. The verbatim disclaimer appears at the end:

   > These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.

**How to verify degradation phrasing:** Open `.claude/team-health/PRIVACY.md` and locate the "Graceful Degradation Language" section. Compare each unavailability statement character-by-character.

**Fail indicators:**
- Any source is silently omitted (not named as unavailable)
- Degradation phrases do not match PRIVACY.md language
- Sections 3–5 are empty when people log data exists
- Disclaimer is absent
- Command errors or crashes when all sources are false

---

## Scenario 11 - First use: no people log file

**Precondition:**
- Carol Davis is in `config.json` (slug `carol-davis`)
- `.team-health/people/carol-davis.json` does NOT exist
- No entry for `carol-davis` in `baselines.json`
- `sources.github = true`, others false

**Invocation:**
```
/team-health:prep Carol
```

**Pass criteria:**
1. Command does not error - output is generated without a stack trace or failure message
2. Section 3 shows "No people log entries recorded" (or equivalent)
3. Section 5 shows "No people log entries recorded" (or equivalent)
4. Lookback defaults to 14 days (no `last_1on1_date` and no `prep_run` to read from)
5. The header includes "(14-day default - no prior 1:1 date recorded)" or equivalent language indicating the fallback window was used

**Fail indicators:**
- Command errors because no people log file exists
- Output defaults to 7 days or any period other than 14 days
- Fallback window not disclosed in the header
- Sections 3 and 5 are blank rather than showing an explicit "no data" message

---

## Scenario 12 - Lookback from prep_run (no last_1on1_date)

**Precondition:**
- Bob Smith, canonical fixture
- `bob-smith.json` has `prep_run: "2026-03-12"` but NO `last_1on1_date` field
- `sources.github = true`, others false

**Invocation:**
```
/team-health:prep Bob
```

**Pass criteria:**
1. Lookback header shows "since last prep on 2026-03-12" (or equivalent language referencing the `prep_run` date)
2. Signal queries (GitHub, etc.) cover the period from 2026-03-12 to today - not a fixed 7-day or 14-day window

**Fail indicators:**
- Lookback falls back to 14-day default when `prep_run` is present and should be used
- Header does not disclose which anchor date was used for the lookback window

---

## Scenario 13 - Lookback from last_1on1_date

**Precondition:**
- Alice Chen, canonical fixture
- `alice-chen.json` has `last_1on1_date: "2026-03-10"`
- `sources.github = true`, others false

**Invocation:**
```
/team-health:prep Alice
```

**Pass criteria:**
1. Lookback header shows "since last 1:1 on 2026-03-10" (or equivalent - "since 2026-03-10" is acceptable)
2. Signal queries cover the period from 2026-03-10 to today - `last_1on1_date` takes priority over `prep_run` when both are present

**Fail indicators:**
- Lookback uses `prep_run` or a default window instead of `last_1on1_date`
- Date in the header does not match `last_1on1_date`

---

## Footer

**Testing environment:** All tests are manual - invoke `/team-health:prep <name>` in a live Claude Code session with Phase 1, 2, and 3 setup complete.

**State files location:** `.team-health/` directory (gitignored - this directory is never committed)

**Schema reference:** `.planning/phases/01-foundation/state-schemas.json` - locked canonical schemas for `config.json`, `baselines.json`, and `people/<slug>.json`

**Reference documents:**
- `.claude/team-health/SIGNALS.md` - 11 signals, thresholds, two-signal rule, severity levels
- `.claude/team-health/BASELINES.md` - rolling baseline computation algorithm, first-run behavior, provenance fields
- `.claude/team-health/PRIVACY.md` - output language rules, required disclaimer, graceful degradation language

**Lookback window logic:** `last_1on1_date` → `prep_run` → 14-day default (in priority order). See CONTEXT.md and RESEARCH.md for full derivation notes.

**Traceability:**

| Scenario | Requirement | Key assertion |
|----------|-------------|---------------|
| 1 | PREP-01 | Prep sheet generated; prep_run written to people log after output |
| 2 | PREP-02 | All 5 sections present; verbatim disclaimer at end |
| 3 | PREP-03 | GitHub signals checked over lookback window; deviations with stddev units |
| 4A | PREP-04 | Jira unavailable → exact PRIVACY.md degradation phrase |
| 4B | PREP-04 | Jira active → ticket data in Section 2 |
| 5A | PREP-05 | Calendar unavailable → exact PRIVACY.md degradation phrase |
| 5B | PREP-05 | Calendar active → meeting load % in Section 2 |
| 6A | PREP-06 | Slack unavailable → exact PRIVACY.md degradation phrase |
| 6B | PREP-06 | Slack active → public metadata only; no DM/private content |
| 7 | PREP-07 | Open commitments with content, date, and days-open count in Section 3 |
| 8 | PREP-08 | Career goals + last promo discussion date + months-ago in Section 5 |
| 9 | PREP-09 | Zero prohibited diagnosis words; talking points are questions |
| 10 | PREP-10 | All 4 sources named as unavailable; Sections 3–5 render from people log |
| 11 | - | First use: no people log → no error; 14-day default disclosed in header |
| 12 | - | Lookback from prep_run when no last_1on1_date |
| 13 | - | Lookback from last_1on1_date takes priority over prep_run |
