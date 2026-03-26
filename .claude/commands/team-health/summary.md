---
description: "Read-only person profile: career goals, open commitments, signal trends, log history, and tenure — no state writes."
argument-hint: "<name> [from:\"DD-Mon-YYYY\"]"
allowed-tools: Read, Bash(date +%Y-%m-%d), Bash(ls .team-health/pulse-history/), Bash(gh api *)
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

## Argument Parsing

Parse $ARGUMENTS for:
1. **Person name:** everything before `from:` (or the entire string if no `from:` is present). Trim whitespace.
2. **from date (optional):** if `from:"<date>"` or `from:<date>` is present, parse the date. Accepted formats: `DD-Mon-YYYY` (e.g., `01-Mar-2026`), `YYYY-MM-DD` (e.g., `2026-03-01`), or `DD/MM/YYYY`. Convert to YYYY-MM-DD and store as `FROM_DATE`.
3. If no `from:` parameter is present, set `FROM_DATE` to null (default lookback windows apply: 7 days for GitHub, 30 days for Confluence).
4. If `FROM_DATE` is set, it overrides both the GitHub lookback (replaces "7 days ago") and the Confluence lookback (replaces "30 days"). The `from:` date becomes the start of the query window for all live signal queries.

## Name Resolution

1. Extract the person name from the parsed arguments (the part before `from:`, or the full $ARGUMENTS if no `from:` was found).
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

### A5 - Live GitHub signals (if sources include github)

If `sources.github` is true in config.json, fetch GitHub data for this person so that GitHub signals always appear in the Signal Trends section -- even if baselines.json has no GitHub history.

Check `sources.github_method` in config.json. If "cli" or absent, use `gh api` commands. If "mcp", use GitHub MCP tools.

**Using gh CLI (preferred):**

Determine the lookback start date:
- If `FROM_DATE` is set: use `FROM_DATE` as `lookback_start`.
- If `FROM_DATE` is null: compute `lookback_start` as 7 days before the injected current date (YYYY-MM-DD).

- **github_prs_merged_per_week:** Run:
  `gh api "search/issues?q=author:{github_username}+org:{github_org}+is:pr+is:merged+merged:>={lookback_start}&per_page=100" --jq '.total_count'`

- **github_pr_review_count_per_week:** Run:
  `gh api "search/issues?q=reviewed-by:{github_username}+org:{github_org}+is:pr+updated:>={lookback_start}&per_page=100" --jq '.total_count'`

- **github_commit_days_per_week:** Run:
  `gh api "search/commits?q=author:{github_username}+org:{github_org}+committer-date:>={lookback_start}&per_page=100" --jq '[.items[].commit.committer.date[:10]] | unique | length'`

Store results as live GitHub signals. These will be rendered in the Signal Trends table alongside baseline data. If baselines.json also has GitHub metrics for this person, include both the live current value and the baseline avg/trend. If baselines.json has no GitHub metrics, show the live value with "no baseline" noted.

If `FROM_DATE` was used, label the GitHub signals in the output as "since [FROM_DATE]" instead of "this week".

If `sources.github` is false: note in output that GitHub signals are unavailable.

### A6 - Confluence activity (if sources include confluence)

If the config.json sources object contains `confluence: true` (or if Atlassian/Confluence MCP tools are available):

Determine the Confluence lookback:
- If `FROM_DATE` is set: compute the number of days between `FROM_DATE` and today. Use that as the CQL lookback (e.g., `now("-25d")` if FROM_DATE is 25 days ago).
- If `FROM_DATE` is null: use the default 30-day lookback (`now("-30d")`).

1. Query for pages authored/edited by this person (by Atlassian account ID from `jira_user_id`, using CQL: `contributor = "<jira_user_id>" AND type = page AND lastModified >= now("-<N>d")`).
2. Query for comments created by this person (CQL: `creator = "<jira_user_id>" AND type = comment AND created >= now("-<N>d")`).
3. Query for pages where this person is @-mentioned (CQL: `type = page AND text ~ "<display_name>" AND lastModified >= now("-<N>d")`).
4. Aggregate: "N pages authored/edited, M comments, P pages mentioned in"
5. List up to 5 most recent page titles (no content, titles only).

These will be rendered in the Signal Trends section alongside other metrics. If `FROM_DATE` was used, label Confluence signals as "since [FROM_DATE]" instead of "(30d)".

If Confluence MCP is not available: note "Confluence signals unavailable - Atlassian MCP not configured."

## Phase B - Output (5-section profile)

Render the following profile. Output ALL of this. Make NO state writes afterward.

```
# Person Summary: [Full Name]
*Generated [today YYYY-MM-DD] — read-only view, no state changes*
[If FROM_DATE is set: "*Signal window: [FROM_DATE] to [today]*"]

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

Render a unified signal trends table that always includes GitHub and Confluence data when the source is available, alongside any baseline metrics from baselines.json.

| Signal | Current | 8-Wk Avg | Trend | Window |
|--------|---------|-----------|-------|--------|
[Baseline metrics first: one row per metric from baselines.json with data. Current = last_value, 8-Wk Avg = mean (+/-stddev), Trend = trending up/stable/trending down, Window = N weeks]
[GitHub live signals from A5: one row per GitHub metric collected. Current = live value from this week's query. If baselines.json also has this metric, show 8-Wk Avg and Trend from baseline. If no baseline exists for this metric, show "no baseline" in 8-Wk Avg and Trend columns.]
[Confluence signals from A6: render as rows in the table:]
- Confluence pages authored/edited (30d) | [N] | [baseline or "no baseline"] | [trend or "-"] | 30d
- Confluence comments (30d) | [M] | [baseline or "no baseline"] | [trend or "-"] | 30d
- Confluence mentions (30d) | [P] | [baseline or "no baseline"] | [trend or "-"] | 30d

[If a source is unavailable (sources.github = false, sources.confluence = false), note below the table:]
[e.g., "GitHub signals unavailable - GitHub MCP/CLI not configured. PR and commit data not included."]

[If no baseline data exists AND no live signals could be fetched:]
*No signal data available. Run /team-health:pulse weekly to build trend data.*

[After the table, if Confluence data exists from A6, list recent pages:]
**Recent Confluence pages:**
[list up to 5 titles with space name and date]

[After Confluence pages (or after the table if no Confluence data), render the flag history:]
**Recent pulse flags:**
[List from Phase A4, e.g.:]
- W13: GREEN (no flags)
- W12: YELLOW - PR review lag elevated
- W11: GREEN (no flags)
- W10: RED - commit days and PR review count both dropped

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
