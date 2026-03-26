---
description: "Read-only person profile: career goals, open commitments, signal trends, log history, and tenure — no state writes."
argument-hint: "<name>"
allowed-tools: Read, Bash(date +%Y-%m-%d), Bash(ls .team-health/pulse-history/)
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
2. Do NOT re-derive the slug from the argument text. Instead: read the `team` array from config.json (already read in Pre-flight Check), find the entry whose `name` field is a case-insensitive prefix match for the typed name. Use the `slug` field from that config entry. This guarantees slug consistency with setup.
3. Fuzzy prefix matching: "derese" matches "Derese Getachew"; "derese g" matches "Derese Getachew". Match is case-insensitive. If multiple team members match (e.g. two people named "Alex"), list them and ask the manager to clarify.
4. If no match found: "No team member matching '[typed name]' found. Configured team: [list all names from config]. Did you mean one of these?" - stop.
5. Store the resolved full name, slug, and the entire team entry object from config.json.

## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md - signal definitions (needed for trend context)
2. .claude/team-health/BASELINES.md - baseline computation methodology
3. .claude/team-health/PRIVACY.md - output language rules, required disclaimer
4. .claude/team-health/COACHING.md - coaching frameworks (needed for the growth context section)

Follow these documents exactly. Do not improvise output language.

## Phase A - Data Collection (read-only)

Collect all available data for this person. Do NOT write to any files during this command.

### A1 - Config profile

From the config.json team entry (already in memory), extract:
- name, role, github_username, jira_user_id, slack_user_id
- 1on1_cadence_weeks
- start_date (if present; compute tenure as days/months/years from start_date to today)

### A2 - People log

Read `.team-health/people/<slug>.json` using the Read tool.

If file does not exist or Read returns error:
- Note: "No people log found for [Name]. Use /team-health:log [name] to start logging, or /team-health:onboard [name] for guided context capture."
- Set PERSON_DATA to empty structure: { entries: [], open_commitments: [], career_context: { stated_goals: [], last_promo_discussion: null, notes: "" } }

If file exists, parse into memory as PERSON_DATA. Extract:
- **All entries** (full history)
- **Open commitments** where status == "open"
- **Career context**: stated_goals, last_promo_discussion, notes
- **Entry statistics**: total count, count by category, date of first entry, date of most recent entry
- **Recurring themes**: any category appearing 3+ times in the last 6 weeks
- **Last 1:1 date**: from PERSON_DATA.last_1on1_date (if present)
- **Last prep run**: from PERSON_DATA.prep_run (if present)

### A3 - Baselines and signal trends

Read `.team-health/baselines.json` using the Read tool.

If file does not exist or person has no entry: note "No baseline data for [Name]. Run /team-health:pulse to begin building signal history."

If person has an entry, extract:
- For each metric in their `metrics` object:
  - Current window values (the array)
  - mean, stddev, last_value
  - Compute trend direction: compare the average of the last 3 values to the average of the first 3 values in the window
    - If last_3_avg > first_3_avg + 0.5*stddev: "trending up"
    - If last_3_avg < first_3_avg - 0.5*stddev: "trending down"
    - Otherwise: "stable"
  - Window length (weeks of data)
- Provenance fields: computed_from, computed_to, source array, window_weeks

### A4 - Pulse history (person-specific flags)

Run: Bash(ls .team-health/pulse-history/)

If pulse history exists:
1. Read the most recent 4 pulse history files (newest first).
2. For each file, search for this person's name in the "Flagged Members" section.
3. Record: which weeks they were flagged, what status (YELLOW/RED), and which signals triggered the flag.
4. Build a flag history: "Flagged in W12 (YELLOW: PR review lag), W10 (RED: commit days + PR reviews). Not flagged in W13, W11."

If no pulse history: note "No pulse history available."

### A5 - Confluence activity (if sources include confluence)

If the config.json sources object contains `confluence: true` (or if Atlassian/Confluence MCP tools are available):
1. Query for pages authored by this person in the last 30 days (by display name or account ID).
2. Query for pages where this person commented in the last 30 days.
3. Query for pages where this person is @-mentioned in the last 30 days.
4. Aggregate: "N pages authored, M pages commented on, P pages mentioned in"
5. List up to 5 most recent page titles (no content, titles only).

If Confluence MCP is not available: note "Confluence signals unavailable - Atlassian MCP not configured."

## Phase B - Output (5-section profile)

Render the following profile. Output ALL of this. Make NO state writes afterward.

```
# Person Summary: [Full Name]
*Generated [today YYYY-MM-DD] — read-only view, no state changes*

---

## 1. Profile

**Name:** [Full Name]
**Role:** [Role from config]
**Tenure:** [computed from start_date, or "Start date not recorded" if null]
**GitHub:** @[github_username] | **Jira:** [jira_user_id or "not configured"] | **Slack:** [slack_user_id or "not configured"]
**1:1 cadence:** every [N] week(s) | **Last 1:1:** [date from people log, or "not recorded"]
**Last prep run:** [date, or "never"]

---

## 2. Career Context

**Stated goals:**
[Bulleted list of stated_goals, or "None recorded. Use /team-health:onboard [name] or /team-health:log [name] to capture goals."]

**Last promotion discussion:** [date and "(N months ago)", or "Not recorded"]

**Manager notes:** [career_context.notes, or "—"]

**Growth context:**
[If career_context has stated_goals: reference COACHING.md to identify which coaching framework is most relevant to their current goals. Output 1-2 sentences: "Based on their goal of [X], consider the [framework name] approach from COACHING.md — [one-line summary of what it suggests]."]
[If no goals recorded: "No career goals recorded — consider using /team-health:onboard [name] to capture growth context."]

---

## 3. Open Commitments

[If open commitments exist, list each:]
- [date] — [content] *(open [N days] — [urgency label: "current" if <14 days, "aging" if 14-30 days, "overdue" if >30 days])*

[Cap at 10. If more than 10: show oldest 5 + newest 5, with "and N others" note.]

[If no open commitments:]
*No open commitments recorded.*

**Commitment health:** [X] open, [Y] total ever logged. [If any overdue: "⚠ [N] commitment(s) overdue (>30 days). Consider addressing in next 1:1."]

---

## 4. Signal Trends (8-Week Baseline)

[If baseline data exists, render a table:]

| Signal | Current | 8-Wk Avg | Trend | Window |
|--------|---------|-----------|-------|--------|
[One row per metric with data. Current = last_value, 8-Wk Avg = mean (±stddev), Trend = trending up/stable/trending down, Window = N weeks]

[After the table, render the flag history:]
**Recent pulse flags:**
[List from Phase A4, e.g.:]
- W13: GREEN (no flags)
- W12: YELLOW — PR review lag elevated
- W11: GREEN (no flags)
- W10: RED — commit days and PR review count both dropped

[If no baseline data:]
*No signal baseline established. Run /team-health:pulse weekly to build trend data.*

[If Confluence data exists from A5:]
**Confluence activity (last 30 days):** [N] pages authored, [M] commented on, [P] mentioned in
Recent pages: [list up to 5 titles]

---

## 5. Log History

**Log statistics:** [total entries] entries since [first entry date]. Categories: [breakdown by category with counts, e.g., "feedback-given: 8, career: 3, win: 5, concern: 2, note: 12"]

**Recurring themes:** [themes from A2, or "No recurring themes identified."]

**Recent entries (last 10):**
[Up to 10 entries, newest first:]
- [YYYY-MM-DD] · [category] — [content]

[If no entries:]
*No log entries recorded. Use /team-health:log [name] to start logging.*

---

*This is a read-only summary. No state was modified. Use /team-health:prep [name] to generate a 1:1 prep sheet (writes last_1on1_date).*
*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
```

## IMPORTANT: No Phase E

This command makes ZERO state writes. It does not update:
- last_1on1_date
- prep_run
- last_updated
- baselines.json
- config.json
- Any people file

It is a pure read-and-render operation. This is by design — managers should be able to look up a person's profile at any time without side effects.
