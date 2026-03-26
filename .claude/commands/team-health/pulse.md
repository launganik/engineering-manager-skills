---
description: "Weekly team health pulse - scans all direct reports, scores signals against personal baselines, and produces a flagged team dashboard."
argument-hint: ""
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(mkdir -p .team-health/pulse-history), Bash(gh api *)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`

## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.

## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md - 11 signals, thresholds, two-signal rule, severity levels
2. .claude/team-health/BASELINES.md - baseline computation algorithm, first-run behavior, provenance fields
3. .claude/team-health/PRIVACY.md - output language rules, required disclaimer, degradation language

Follow these documents exactly. Do not improvise signal thresholds, flagging rules, or output language.

## Phase A - Setup

After reading reference docs:
1. Read .team-health/baselines.json using the Read tool.
   - If file does not exist or returns an error: treat as { "schema_version": "1", "last_updated": "", "people": {} }
   - Store the full structure in memory as CURRENT_BASELINES.
2. Read the sources map from config.json (already read in Pre-flight Check):
   - sources.github, sources.jira, sources.slack, sources.calendar
3. Note unavailable sources. For each source where the value is false, use PRIVACY.md Graceful Degradation Language to state which signals will be absent.
   Output at start of pulse run: "Sources active: [list true sources] | Unavailable: [list false sources] - [PRIVACY.md degradation phrasing]"
   If ALL sources are false: state all 4 are unavailable, list all 11 signals as absent, render the summary table with "no signals available" per person, include the disclaimer, then stop.

## Phase B - Per-person signal collection

For each team member in config.json team array (iterate in order):
  Record their: name, slug, github_username, jira_user_id, slack_user_id

  For each available source (skip the entire source block if sources.<name> is false):

  ### GitHub signals (if sources.github is true):
  Scope ALL GitHub queries to the github_org field from config.json. Do not query across all of GitHub.

  Check `sources.github_method` in config.json to determine how to fetch data:

  **If github_method is "cli" (preferred - uses `gh` CLI):**

  Use the `gh api` command via Bash to query GitHub's REST/Search API. All queries are scoped to the org.

  - **github_prs_merged_per_week:** Run:
    `gh api "search/issues?q=author:{github_username}+org:{github_org}+is:pr+is:merged+merged:>={7_days_ago_YYYY-MM-DD}&per_page=100" --jq '.total_count'`

  - **github_pr_review_count_per_week:** Run:
    `gh api "search/issues?q=reviewed-by:{github_username}+org:{github_org}+is:pr+updated:>={7_days_ago_YYYY-MM-DD}&per_page=100" --jq '.total_count'`

  - **github_pr_review_lag_days:** Run:
    `gh api "search/issues?q=author:{github_username}+org:{github_org}+is:pr+created:>={7_days_ago_YYYY-MM-DD}&per_page=100" --jq '.items[] | {number, created_at, repo: .repository_url}'`
    Then for each PR, fetch its reviews to find the first review date and compute the lag. If no PRs, use 0.
    Also check for absolute threshold: any open PR by this person with no reviews after 48h.

  - **github_commit_days_per_week:** Run:
    `gh api "search/commits?q=author:{github_username}+org:{github_org}+committer-date:>={7_days_ago_YYYY-MM-DD}&per_page=100" --jq '[.items[].commit.committer.date[:10]] | unique | length'`

  Date format for queries: YYYY-MM-DD. Compute 7_days_ago from the injected current date.

  **If github_method is "mcp" (fallback - uses GitHub MCP tools):**

  Query the GitHub MCP for this person's github_username over the past 7 days (current week):
  - github_prs_merged_per_week: count of PRs merged by this person in github_org this week
  - github_pr_review_count_per_week: count of PR reviews submitted by this person in github_org this week
  - github_pr_review_lag_days: average days from PR creation to first review for PRs where this person is the author (within github_org, this week). Use 0 if no PRs awaiting review.
  - github_commit_days_per_week: count of distinct calendar days with at least one commit by this person in github_org this week

  Use whichever commit-listing and PR-listing tools are available from the GitHub MCP - probe by namespace (any tool containing 'github' or starting with 'mcp_github'). Adapt tool names to what the manager has installed.

  **If github_method is absent or null:** treat as "cli" if `gh auth status` succeeds, otherwise treat as "mcp".

  ### Jira signals (if sources.jira is true):
  Query the Jira MCP for this person's jira_user_id:
  - jira_tickets_closed_per_week: count of tickets resolved/closed by this person this week
  - jira_tickets_in_progress_over_2_sprints: count of tickets assigned to this person that have been "in progress" for more than 2 sprint cycles (absolute threshold from SIGNALS.md - check window length before flagging as absolute threshold). IMPORTANT: Calculate time in progress from when the ticket transitioned into "In Progress" status (via changelog/history), NOT from the ticket creation date.
  - jira_blocked_tickets: count of tickets assigned to this person with a "blocked" or impediment status

  Scope to the project(s) associated with this team, using JQL where supported.

  ### Slack signals (if sources.slack is true):
  PRIVACY CONSTRAINT: Only use public channel data. Do not request or use DM content or private channel content. If the Slack MCP returns DM content, ignore it.
  Query for this person's slack_user_id:
  - slack_public_channel_participation_trend: relative participation trend this week vs. recent history (use a 0–1 normalized score or categorical: up/stable/down - note the unit so baseline comparison is consistent)
  - slack_response_latency_trend: trend in response latency in public channels (up/stable/down or normalized score)

  ### Calendar signals (if sources.calendar is true):
  Query **this person's own calendar** using their email as calendarId (e.g., `gcal_list_events(calendarId="firstname.lastname@forto.com")`). If the person has an `email` field in config.json, use that. Otherwise, derive from their name: lowercase `firstname.lastname@forto.com`. Only include events with 2+ real participants that were accepted or tentatively accepted.
  - calendar_meeting_load_pct: percentage of working hours in meetings this week
  - calendar_focus_time_blocks_per_week: count of uninterrupted blocks >=90 minutes this week

  Record all collected signal values as: { slug, metric_name, current_value, week_queried }

## Phase C - Baseline comparison and flag determination

For each person:
  1. Retrieve their entry from CURRENT_BASELINES.people[slug] (may not exist for new team members).
  2. For each metric with a collected current_value:
     a. Look up the metric's window in CURRENT_BASELINES (may not exist).
     b. Apply BASELINES.md first-run window-thinness check:
        - window does not exist OR len(window) == 0: label "(baseline pending - 0 weeks of data)"; do NOT compute deviation; do NOT flag
        - len(window) == 1 or 2: label "(baseline pending - N weeks of data)"; do NOT flag
        - len(window) >= 3: proceed to deviation computation
     c. If computing deviation:
        - deviation_stddev = (current_value - mean) / stddev
          NOTE: if stddev == 0 and current_value != mean: treat as extreme deviation (>2 stddev); if stddev == 0 and current_value == mean: treat as 0 deviation
        - For lag metrics (github_pr_review_lag_days, calendar_meeting_load_pct): flag on UPWARD deviation (deviation_stddev > 0 is concerning)
        - For all other metrics: flag on DOWNWARD deviation (deviation_stddev < 0 is concerning)
  3. Apply SIGNALS.md two-signal rule (verbatim - do not substitute):
     - RED: any single signal at absolute deviation_stddev >= 2 in the concerning direction
     - YELLOW: 2 or more signals each at deviation_stddev >= 1 in the concerning direction, AND no signal hits RED threshold
     - GREEN: everything else (including all baseline-pending states)
  4. Record per person: { slug, flag_status (GREEN/YELLOW/RED/PENDING), triggering_signals [], non_flagged_signals [] }

CRITICAL: Do NOT update baselines.json during this phase. Baseline updates happen in Phase E, after output is generated. Using this week's value in the baseline before comparing it would corrupt the deviation calculation.

## Phase D - Dashboard output

Render the following dashboard. Follow the format exactly.

### Summary table

## Team Health Pulse - [ISO week from injected date]

Sources active: [list sources where true] | Unavailable: [list sources where false] (or "None unavailable" if all active)

| Name | Status | Signals Checked |
|------|--------|-----------------|
[One row per team member. Status: GREEN / YELLOW / RED / PENDING]
[Signals Checked: "N/11 (source names)" where N is the count of signals queryable with active sources]
[PENDING rows: "(baseline pending - N weeks of data)"]

### Detail sections (flagged individuals only)

For each team member where flag_status is YELLOW or RED:

### [Full Name] - [STATUS] ([yellow: "watch" / red: "immediate attention"])

Signals triggering flag ([N] of [threshold] required):

[For each triggering signal, output exactly this format:]
- **[Human-readable metric name]:** [current_value] [unit] this week vs. 8-week avg [mean] [unit] ([delta:+/-N.N], [stddev_delta:+/-N.N] stddev)
  Source: [GitHub / Jira / Slack / Calendar] MCP

Other signals (within baseline):
[Brief one-line summary: "PRs merged: N (baseline avg: N.N) - within normal range"]

DO NOT render a detail section for GREEN or PENDING team members - table row only.
DO NOT use psychological language (see PRIVACY.md Rules 1 and 4). All observations are behavioral.
DO NOT compare one person to another. Every signal references only that person's own baseline.

### Disclaimer (mandatory - append after all output, verbatim)

---
*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*

## Phase E - State writes

Execute Phase E only after Phase D output is complete. Do not write state before output.

### Step 1: Update baselines.json

For each person × metric (where a current_value was collected this week):
  Apply BASELINES.md Updating a Baseline algorithm exactly:
  1. Append current_value to the window array (create empty array if metric is new)
  2. If len(window) > baseline_window_weeks (from config.json, default 8): remove first element
  3. mean = sum(window) / len(window)
  4. stddev = sqrt(sum((x - mean)^2 for x in window) / len(window))
     Special case: if len(window) == 1, stddev = 0
  5. last_value = current_value
  6. Set computed_to = today's date (from injected date)
  7. If this is the first entry for this person: set computed_from = today's date

Build the ENTIRE updated baselines.json in memory first.
Then write with a SINGLE Write tool call to .team-health/baselines.json.
The file must include schema_version, last_updated (today), and the complete people object.
DO NOT call Write multiple times (once per person) - this risks partial writes.

### Step 2: Write pulse history snapshot

First, run: Bash(mkdir -p .team-health/pulse-history)
Then write to: .team-health/pulse-history/<ISO-week>.md
(ISO week is the value injected at the top: !`date +%Y-W%V` - e.g., 2026-W11)

Contents:
# Team Pulse - [ISO week]

**Run date:** [today YYYY-MM-DD]
**Sources active:** [list sources where true, or "None"]
**Unavailable:** [list sources where false with PRIVACY.md degradation phrasing, or "None"]

## Summary

| Name | Status |
|------|--------|
[one row per team member]

## Flagged Members

[For each YELLOW/RED member: name, status, triggering signals with deltas in the same format as Phase D]

[If no flagged members: "No members flagged this week."]

---
*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
