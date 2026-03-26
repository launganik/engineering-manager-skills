---
description: "Guided context capture for a new or existing direct report. Walks through career goals, working style, growth areas, and manager commitments to seed a rich people log."
argument-hint: "<name>"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-%m-%dT%H:%M:%SZ)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`

## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.

## Name Resolution

1. Extract the person name from $ARGUMENTS.
2. Do NOT re-derive the slug from the argument text. Instead: read the `team` array from config.json (already read in Pre-flight Check), find the entry whose `name` field is a case-insensitive prefix match for the typed name. Use the `slug` field from that config entry.
3. Fuzzy prefix matching: "derese" matches "Derese Getachew". Match is case-insensitive. If multiple team members match, list them and ask the manager to clarify.
4. If no match found: "No team member matching '[typed name]' found. Configured team: [list all names from config]. Did you mean one of these?" - stop.
5. Store the resolved full name, slug, role, and start_date from config.

## Existing File Check

Read `.team-health/people/<slug>.json` using the Read tool.

If file exists AND has entries (entries array length > 0):
- Output:
  ```
  [Name] already has a people log with [N] entries (first entry: [date]).
  You can still run onboard to update their career context and capture new goals.
  Existing entries and commitments will be preserved.
  ```
- Load PERSON_DATA from the file. Continue to Phase A.

If file exists but entries is empty:
- Load PERSON_DATA from the file. Continue to Phase A.

If file does not exist:
- Initialize PERSON_DATA:
  ```json
  {
    "schema_version": "1",
    "name": "[full name from config]",
    "slug": "[slug]",
    "created": "[today YYYY-MM-DD]",
    "last_updated": "[today YYYY-MM-DD]",
    "entries": [],
    "open_commitments": [],
    "career_context": {
      "stated_goals": [],
      "last_promo_discussion": null,
      "notes": ""
    }
  }
  ```
- Continue to Phase A.

## Required Reference Read

Read .claude/team-health/COACHING.md using the Read tool. This will help frame the questions appropriately and link the person's goals to coaching frameworks during the summary output.

## Phase A - Guided Interview (5 sections, asked one at a time)

Present each section as a conversational prompt. Wait for the manager's response after each section before proceeding to the next. The manager can type "skip" for any section to move on.

### Section 1: Career Goals

Ask:
```
Let's capture context for [Name] ([Role]).

1. CAREER GOALS
What are [Name]'s stated career goals? Where do they want to grow?
(Examples: "wants to move into tech lead role", "interested in system design ownership", "exploring management track", "deepening backend expertise")

Type your answer, or "skip" to move on.
```

Wait for response. Store as GOALS_INPUT.

### Section 2: Current Strengths

Ask:
```
2. CURRENT STRENGTHS
What does [Name] do well? What are they known for on the team?
(Examples: "strong code reviewer, catches edge cases others miss", "great at unblocking others", "excellent written communication")

Type your answer, or "skip" to move on.
```

Wait for response. Store as STRENGTHS_INPUT.

### Section 3: Growth Areas

Ask:
```
3. GROWTH AREAS
What areas would benefit from development? What have you given feedback on?
(Examples: "could improve estimation accuracy", "tends to work in isolation — could pair more", "needs more visibility with stakeholders")

Type your answer, or "skip" to move on.
```

Wait for response. Store as GROWTH_INPUT.

### Section 4: Working Style & Preferences

Ask:
```
4. WORKING STYLE
Anything about how [Name] prefers to work, communicate, or receive feedback?
(Examples: "prefers async communication", "likes detailed written feedback over verbal", "needs time to process before responding", "does best work in morning focus blocks")

Type your answer, or "skip" to move on.
```

Wait for response. Store as STYLE_INPUT.

### Section 5: Manager Commitments

Ask:
```
5. COMMITMENTS
Any commitments you've already made to [Name]? Things you've promised to follow up on?
(Examples: "promised to connect them with the platform team lead", "committed to discussing promotion timeline in Q2", "said I'd review their design doc by Friday")

Type your answer, or "skip" to move on.
```

Wait for response. Store as COMMITMENTS_INPUT.

## Phase B - Confirmation

After all 5 sections, present a summary for confirmation:

```
Here's what I'll save for [Name]:

Career Goals: [GOALS_INPUT or "skipped"]
Strengths: [STRENGTHS_INPUT or "skipped"]
Growth Areas: [GROWTH_INPUT or "skipped"]
Working Style: [STYLE_INPUT or "skipped"]
Commitments: [COMMITMENTS_INPUT or "skipped"]

Save this? (yes / edit [section number] / cancel)
```

- If "yes": proceed to Phase C.
- If "edit N": re-ask that specific section, then re-confirm.
- If "cancel": "Onboard cancelled. No data saved." - stop.

## Phase C - Write to People Log

Get the current ISO timestamp: call Bash(date +%Y-%m-%dT%H:%M:%SZ)
Get today's date from the injected date at the top.

### C1 - Update career_context

If GOALS_INPUT is not "skip":
- Parse the goals into a list. Split on commas, semicolons, or line breaks. Trim each item.
- Set PERSON_DATA.career_context.stated_goals = [parsed list]
- If any goal contains "promotion" or "promo": set PERSON_DATA.career_context.last_promo_discussion = today's date

If STRENGTHS_INPUT or GROWTH_INPUT or STYLE_INPUT is not "skip":
- Build a notes string from the non-skipped sections:
  ```
  **Strengths:** [STRENGTHS_INPUT]
  **Growth areas:** [GROWTH_INPUT]
  **Working style:** [STYLE_INPUT]
  ```
  Only include sections that were not skipped.
- If PERSON_DATA.career_context.notes already has content, APPEND to it with a separator:
  ```
  [existing notes]

  --- Onboard update [today YYYY-MM-DD] ---
  [new notes string]
  ```
- If career_context.notes is empty: set it to the new notes string.

### C2 - Create log entries

For each non-skipped section, create a log entry. Compute entry IDs using the same logic as log.md:
- Count existing entries for today's date. ID format: <YYYY-MM-DD>-<NNN> where NNN is (count + 1) zero-padded to 3 digits.
- Increment the counter for each entry created.

Entry mapping:

| Section | Category | Content |
|---------|----------|---------|
| Career Goals | career | "Onboard: Career goals — [GOALS_INPUT]" |
| Current Strengths | note | "Onboard: Strengths — [STRENGTHS_INPUT]" |
| Growth Areas | note | "Onboard: Growth areas — [GROWTH_INPUT]" |
| Working Style | note | "Onboard: Working style — [STYLE_INPUT]" |
| Commitments | commitment | "Onboard: [COMMITMENTS_INPUT]" |

Each entry object:
```json
{
  "id": "<computed id>",
  "date": "<today YYYY-MM-DD>",
  "category": "<from table>",
  "content": "<from table>",
  "tags": ["onboard"],
  "created_at": "<ISO timestamp from Bash call>"
}
```

Append all entries to PERSON_DATA.entries.

### C3 - Create commitment entries

If COMMITMENTS_INPUT is not "skip":
- Parse into individual commitments (split on commas, semicolons, or line breaks if multiple).
- For each commitment, append to PERSON_DATA.open_commitments:
  ```json
  {
    "id": "<same id as the matching entry>",
    "date": "<today YYYY-MM-DD>",
    "content": "<individual commitment text>",
    "status": "open"
  }
  ```

### C4 - Update metadata and write

1. Set PERSON_DATA.last_updated = today's date.
2. Write the COMPLETE updated PERSON_DATA to `.team-health/people/<slug>.json` using the Write tool.
   IMPORTANT: Write the complete file - do not truncate or omit existing entries.

### C5 - Update config.json start_date if provided

If the manager mentioned a start date during any section AND the config.json team entry for this person has `start_date: null`:
- Read config.json (already in memory).
- Update the team entry's start_date field.
- Write the complete updated config.json.

## Phase D - Confirmation

Output:
```
Onboard complete for [Name].

Saved:
- [N] career goal(s) captured
- [M] log entries created (categories: [list categories used])
- [P] commitment(s) tracked
- Career context updated

Next steps:
- /team-health:summary [name] — view their full profile
- /team-health:prep [name] — generate a 1:1 prep sheet
- /team-health:log [name] — add notes after your next conversation
```
