# Phase 2 — People Log: Manual Smoke Tests

**Phase:** 2 — People Log
**Command under test:** `/team-health:log`
**Last updated:** 2026-03-13

---

## Prerequisites

Before running any scenario:

1. Phase 1 setup complete — `.team-health/config.json` exists with `setup_complete: true`
2. At least two team members configured with the following canonical fixtures:
   - **Alice Chen** / slug `alice-chen`
   - **Bob Smith** / slug `bob-smith`
3. All tests run in a live Claude Code session (not simulated)
4. Schema reference: `.planning/phases/01-foundation/state-schemas.json`

---

## Scenario 1 — LOG-01: Append entry creates file and adds entry

**Requirement:** LOG-01
**Precondition:** `.team-health/people/alice-chen.json` does NOT exist

**Setup:**
```bash
rm -f .team-health/people/alice-chen.json
```

**Invocation:**
1. Run `/team-health:log Alice`
2. When prompted, type: `gave strong feedback on the auth PR`

**Pass criteria:**
- [ ] `.team-health/people/alice-chen.json` exists after the command completes
- [ ] `entries` array has exactly 1 entry
- [ ] Entry contains all required fields: `id`, `date`, `category`, `content`, `created_at`
- [ ] `date` value equals today's date in `YYYY-MM-DD` format
- [ ] `schema_version` is `"1"`

**Fail signal:** File does not exist, `entries` is empty, or any required field is missing.

---

## Scenario 2 — LOG-02: All 7 category values written correctly

**Requirement:** LOG-02
**Precondition:** `alice-chen.json` exists (from Scenario 1 or freshly created)

**Invocation:** Run 7 separate `/team-health:log alice` calls, one per category. Use these notes as inputs:

| Input note | Expected category |
|------------|-------------------|
| `gave strong feedback on Alice's design doc` | `feedback-given` |
| `Alice received positive feedback from the principal engineer` | `feedback-received` |
| `Alice mentioned she wants to move toward staff engineer` | `career` |
| `I promised to introduce Alice to the platform lead by end of month` | `commitment` |
| `Alice seems stressed and less engaged this week` | `concern` |
| `Alice shipped the auth refactor ahead of schedule — great win` | `win` |
| `Alice asked about the on-call rotation schedule` | `note` |

**Pass criteria (check after each invocation):**
- [ ] `category` field in the new entry exactly matches the expected value from the table above
- [ ] Valid category values are: `feedback-given`, `feedback-received`, `career`, `commitment`, `concern`, `win`, `note`
- [ ] No category value outside the 7 valid values appears in the JSON

**Fail signal:** Category field is `null`, absent, or contains a value not in the valid set.

---

## Scenario 3 — LOG-03: Free-form note is structured with date, category, content

**Requirement:** LOG-03
**Precondition:** `alice-chen.json` exists

**Invocation:**
1. Run `/team-health:log alice`
2. Type: `gave great code review feedback on the auth refactor PR — clean API design`

**Pass criteria:**
- [ ] New entry has `date` = today's date (`YYYY-MM-DD`)
- [ ] `category` = `"feedback-given"` (inferred from "gave")
- [ ] `content` contains the original note text — not paraphrased or summarised away
- [ ] `created_at` is a valid ISO 8601 timestamp (e.g. `2026-03-13T14:32:00Z`)

**Fail signal:** Category is wrong, content is rewritten rather than preserved, or `created_at` is missing or malformed.

---

## Scenario 4 — LOG-04: Last 3–5 entries shown as context before prompting

**Requirement:** LOG-04
**Precondition:** `alice-chen.json` exists with 6 or more entries

**Setup:** If fewer than 6 entries exist, log additional notes until there are at least 6.

**Invocation:**
1. Run `/team-health:log alice` (do not type a note — just open the log)

**Pass criteria:**
- [ ] Claude's response displays exactly 5 recent entries before asking for a new note
- [ ] Entries are shown in `[YYYY-MM-DD] · [category] — [content]` format
- [ ] Entries 1 through N-5 (the oldest entries) are NOT shown
- [ ] After showing recent entries, Claude asks: "What would you like to log?" or equivalent

**Fail signal:** Fewer than 5 or more than 5 entries shown, or oldest entries appear in context display.

---

## Scenario 5 — LOG-05: Natural language query returns accurate answer

**Requirement:** LOG-05
**Precondition:** `alice-chen.json` exists with at least 3 entries, including one with `category: "career"` and the word "promotion" in the content

**Setup:** If no career/promotion entry exists, log one first:
1. Run `/team-health:log alice`
2. Type: `Alice brought up her promotion timeline — aiming for staff by Q4`

**Invocation:**
```
/team-health:log alice when did I last discuss promotion?
```

**Pass criteria:**
- [ ] Claude responds with the date and content of the career entry referencing promotion
- [ ] Claude does NOT prompt for a new log entry (this is a query, not a log action)
- [ ] If relevant, Claude references data from both `entries` and `career_context`

**Fail signal:** Claude prompts for a new note instead of answering the query, or returns the wrong date/entry.

---

## Scenario 6 — LOG-06: First-time log creation is graceful

**Requirement:** LOG-06
**Precondition:** `.team-health/people/bob-smith.json` does NOT exist (Bob is in config but has never been logged)

**Setup:**
```bash
rm -f .team-health/people/bob-smith.json
```

**Invocation:**
1. Run `/team-health:log Bob`

**Pass criteria:**
- [ ] Claude says "Starting a new log for Bob Smith" or an equivalent acknowledgement that this is a first log
- [ ] `.team-health/people/bob-smith.json` is created
- [ ] File has `schema_version: "1"`
- [ ] File has `entries: []` (empty array)
- [ ] File has `open_commitments: []` (empty array)
- [ ] File has a `career_context` object (may be empty/default values)
- [ ] No error message is shown
- [ ] Claude then prompts for a note

**Fail signal:** File not created, error shown, or Claude does not acknowledge first-time creation.

---

## Bonus Scenario — Commitment dual-write (validates LOG-02 integrity)

**Requirement:** LOG-02 (commitment category dual-write)
**Precondition:** `alice-chen.json` exists

**Invocation:**
1. Run `/team-health:log alice`
2. Type: `I promised to connect Alice with the platform lead by end of month`

**Pass criteria:**
- [ ] A new entry appears in `entries` array with `category: "commitment"`
- [ ] The same commitment also appears in `open_commitments` array
- [ ] The `open_commitments` entry has `status: "open"`
- [ ] Both array entries reference the same content

**Fail signal:** Entry appears only in `entries` but not in `open_commitments`, or `status` is not `"open"`.

---

## Notes

- All tests are manual — invoke commands in a live Claude Code session with Phase 1 setup complete
- People log files are stored at `.team-health/people/<slug>.json`
- Schema reference: `.planning/phases/01-foundation/state-schemas.json`
- Canonical test fixtures: Alice Chen (`alice-chen`), Bob Smith (`bob-smith`)
- After each scenario, inspect the JSON file directly to verify structural correctness
