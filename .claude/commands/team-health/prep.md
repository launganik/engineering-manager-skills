---
description: "Generate a 2-minute 1:1 prep sheet for a direct report: live signal snapshot, standing items from people log, and suggested talking points."
argument-hint: "<name>"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(gh api *)
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

1. Extract the person name from $ARGUMENTS. The name is everything before the first query keyword or question mark - if $ARGUMENTS is just a name, the entire value is the name.
2. Do NOT re-derive the slug from the argument text. Instead: read the `team` array from config.json (already read in Pre-flight Check), find the entry whose `name` field is a case-insensitive prefix match for the typed name. Use the `slug` field from that config entry. This guarantees slug consistency with setup.
3. Fuzzy prefix matching: "alice" matches "Alice Chen"; "alice chen" matches "Alice Chen". Match is case-insensitive. If multiple team members match (e.g. two people named "Alex"), list them and ask the manager to clarify.
4. If no match found: "No team member matching '[typed name]' found. Configured team: [list all names from config]. Did you mean one of these?" - stop.
5. File path: `.team-health/people/<slug>.json`

## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md - 14 signals (11 flag-eligible + 3 informational Confluence), thresholds, two-signal rule, severity levels
2. .claude/team-health/BASELINES.md - baseline computation algorithm, first-run behavior, provenance fields
3. .claude/team-health/PRIVACY.md - output language rules, required disclaimer, degradation language
4. .claude/team-health/COACHING.md - situational coaching frameworks for inline hints in talking points

Follow these documents exactly. Do not improvise signal thresholds, flagging rules, or output language.

## Phase A - Setup

After reading reference docs:
1. Read .team-health/baselines.json using the Read tool.
   - If file does not exist or returns an error: treat as { "schema_version": "1", "last_updated": "", "people": {} }
   - Store the full structure in memory as CURRENT_BASELINES.
2. Read the sources map from config.json (already read in Pre-flight Check):
   - sources.github, sources.jira, sources.slack, sources.calendar
3. Note unavailable sources. For each source where the value is false, use PRIVACY.md Graceful Degradation Language to state which signals will be absent.

## Phase B - People log read and lookback window derivation

1. Read `.team-health/people/<slug>.json` using the Read tool.
   - If file does not exist or Read returns error: treat as empty person with `entries: [], open_commitments: [], career_context: { stated_goals: [], last_promo_discussion: null, notes: "" }`. Note: "No people log file found for [Name]. Standing items and context reminders will be unavailable."
   - If file exists: parse into memory as PERSON_DATA.

2. Lookback window derivation (implement exactly):
   ```
   If PERSON_DATA.last_1on1_date exists and is valid YYYY-MM-DD:
     lookback_start = PERSON_DATA.last_1on1_date
     lookback_source = "since last 1:1 on [date]"
   Else if PERSON_DATA.prep_run exists and is valid YYYY-MM-DD:
     lookback_start = PERSON_DATA.prep_run
     lookback_source = "since last prep on [date]"
   Else:
     lookback_start = today - 14 days (compute from injected date)
     lookback_source = "14-day default (no prior 1:1 date recorded)"
   ```

3. Extract from people log:
   - `open_commitments`: filter PERSON_DATA.open_commitments for entries where status == "open"
   - `career_context`: read PERSON_DATA.career_context (stated_goals, last_promo_discussion, notes)
   - `log_patterns`: scan PERSON_DATA.entries since lookback_start. Count entries by category. If any category appears 3+ times within the last 6 weeks, flag as a recurring theme.
   - `recent_entries`: last 5 entries since lookback_start (newest first). If more than 5, take most recent 5.

## Phase C - MCP signal collection (single person, dynamic lookback window)

CRITICAL: Use `lookback_start` (derived in Phase B) as the date filter for ALL MCP queries - NOT fixed 7-day or 14-day windows from SIGNALS.md. SIGNALS.md defines WHAT to query; Phase B defines WHEN.

For each available source (skip entire source block if sources.<name> is false in config.json):

### GitHub signals (if sources.github is true):
- Scope ALL queries to github_org from config.json. Do NOT query across all of GitHub.

Check `sources.github_method` in config.json to determine how to fetch data:

**If github_method is "cli" or absent (preferred - uses `gh` CLI):**

Use the `gh api` command via Bash. Replace `{github_username}`, `{github_org}`, and `{lookback_start}` with actual values.

- **github_prs_merged:** Run:
  `gh api "search/issues?q=author:{github_username}+org:{github_org}+is:pr+is:merged+merged:>={lookback_start}&per_page=100" --jq '.total_count'`

- **github_pr_review_count:** Run:
  `gh api "search/issues?q=reviewed-by:{github_username}+org:{github_org}+is:pr+updated:>={lookback_start}&per_page=100" --jq '.total_count'`

- **github_pr_review_lag_days:** Run:
  `gh api "search/issues?q=author:{github_username}+org:{github_org}+is:pr+created:>={lookback_start}&per_page=100" --jq '.items[] | {number, created_at, repo: .repository_url}'`
  Then for each PR, fetch its reviews to find the first review date and compute the lag.
  Also check absolute threshold: any open PR by this person with no reviews after 48h.

- **github_commit_days:** Run:
  `gh api "search/commits?q=author:{github_username}+org:{github_org}+committer-date:>={lookback_start}&per_page=100" --jq '[.items[].commit.committer.date[:10]] | unique | length'`

**If github_method is "mcp" (fallback):**

Use whichever tools are available from GitHub MCP - probe by namespace (any tool containing 'github' or starting with 'mcp_github'). Adapt tool names to what is installed.
- `github_prs_merged`: count of PRs merged by this person since lookback_start
- `github_pr_review_count`: count of PRs reviewed since lookback_start
- `github_pr_review_lag_days`: average days from PR creation to first review. Also check absolute threshold: any PR open >48h with no review.
- `github_commit_days`: count of distinct calendar days with commits since lookback_start

### Jira signals (if sources.jira is true):
- `jira_tickets_closed`: count closed since lookback_start
- `jira_tickets_in_progress_over_2_sprints`: count (absolute threshold, no lookback needed). IMPORTANT: Calculate time in progress from when the ticket transitioned into "In Progress" status (via changelog/history), NOT from the ticket creation date.
- `jira_blocked_tickets`: count (absolute threshold, no lookback needed)

Scope to project(s) for this team using JQL where supported.

### Slack signals (if sources.slack is true):
PRIVACY CONSTRAINT: public channel metadata only. Do NOT request or use DM content.
- `slack_channel_participation`: count of distinct public channels with posts since lookback_start
- `slack_response_latency`: median @-mention response time in public channels since lookback_start

### Calendar signals (if sources.calendar is true):
Query **this person's own calendar** using their email as calendarId (e.g., `gcal_list_events(calendarId="firstname.lastname@forto.com")`). If the person has an `email` field in config.json, use that. Otherwise, derive from their name: lowercase `firstname.lastname@forto.com`. Only include events with 2+ real participants that were accepted or tentatively accepted.
- `calendar_meeting_load_pct`: meeting percentage since lookback_start
- `calendar_1on1_adherence`: was 1:1 held within expected cadence window? (query the **manager's primary calendar** for this signal, since it measures the manager-IC meeting)

### Confluence signals (if sources.confluence is true, or if Atlassian/Confluence MCP tools are available):
These are informational signals — they do NOT participate in flag determination.
- `confluence_pages_authored_per_month`: count of pages created/edited by this person in the last 30 days (use their Atlassian account ID, same as jira_user_id from config)
- `confluence_comments_per_month`: count of comments left on Confluence pages in the last 30 days
- `confluence_mentions_per_month`: count of pages where this person is @-mentioned, modified in the last 30 days
- Also collect: up to 5 most recent page titles authored or edited (titles only, no content)

Store Confluence data separately from flag-eligible signals. It will be rendered in a dedicated block in the prep output.

For each collected flag-eligible signal, compare to baseline from CURRENT_BASELINES using BASELINES.md deviation algorithm:
1. Look up person's baseline entry: CURRENT_BASELINES.people[slug].metrics[metric_name]
2. If no entry or window length < 3: label "(baseline pending - N weeks of data)", do NOT flag
3. If window >= 3: compute deviation = (current_value - mean) / stddev
   - If stddev == 0 and current_value != mean: extreme deviation (flag)
   - If stddev == 0 and current_value == mean: deviation = 0 (no flag)
4. Apply direction from SIGNALS.md: low=bad for most metrics, high=bad for lag/latency metrics
5. Check absolute thresholds for applicable signals

Apply SIGNALS.md two-signal rule to determine flag_status:
- RED: any single signal at absolute deviation >= 2 stddev in concerning direction
- YELLOW: 2+ signals each >= 1 stddev in concerning direction, none hits RED
- GREEN: everything else (including all baseline-pending)

CRITICAL: Prep does NOT update baselines.json. Prep is a consumer of baselines, not a producer. Only pulse.md updates baselines.

## Phase D - Prep sheet output

Render the five-section prep sheet. Output ALL of this before any state writes.

```
# 1:1 Prep: [Full Name] - [today YYYY-MM-DD]
*Lookback: [lookback_start] to today ([N days] - [lookback_source])*

---

## 1. Status Snapshot

[1-3 sentences: current signal status.
If signals flagged: name the flags factually. E.g., "2 GitHub signals flagged (YELLOW) since [lookback_start]. 4/11 signals checked (GitHub only)."
If no signals flagged: "No signal anomalies since [lookback_start]. [N] signals checked ([active source names])."
If all baseline-pending: "Insufficient baseline history - raw values shown, no flags applied."
If all sources unavailable: "No MCP sources configured - signal analysis unavailable. Prep based on people log only."]

Available signals: [active sources] | Unavailable: [inactive sources with PRIVACY.md degradation phrasing]

---

## 2. Signal Flags

[If any signals flagged, for each flag use this detail format:]
- **[Human-readable metric name]:** [current_value] [unit] vs. 8-week avg [mean] [unit] ([delta:+/-N.N], [stddev_delta:+/-N.N] stddev)
  Source: [GitHub / Jira / Slack / Calendar] MCP

[If no flags:]
*No flags. All available signals within personal baseline.*

[If signals unavailable, list each with PRIVACY.md degradation language:]
- GitHub signals are unavailable - GitHub MCP is not configured. PR and commit data cannot be included.
- Jira signals are unavailable - Jira MCP is not configured. Ticket velocity and blocked-ticket data cannot be included.
- Slack signals are unavailable - Slack MCP is not configured. Channel participation and response latency cannot be included.
- Calendar signals are unavailable - Calendar MCP is not configured. Meeting load and 1:1 adherence cannot be checked.

---

## 2b. Confluence Activity (Informational)

[If Confluence signals were collected:]
**Confluence activity (last 30 days):** [N] pages authored/edited, [M] comments, [P] pages mentioned in
Recent pages:
- [Page title 1]
- [Page title 2]
- [up to 5 titles]

[If Confluence activity is notably higher than baseline while GitHub activity is lower than baseline:]
*Note: Non-code contributions appear elevated while code metrics are lower — this may reflect design, documentation, or coordination work. See talking points for context.*

[If Confluence MCP not available:]
*Confluence activity unavailable — Atlassian MCP not configured.*

[If Confluence MCP available but no activity found:]
*No Confluence activity found in the last 30 days.*

---

## 3. Standing Items from People Log

[If open commitments exist:]
**Open commitments (you owe them):**
- [date] - [content] *(open since [N days])*

[Cap at 5 open commitments. If more than 5: show oldest 3 + newest 2, with "and N others" note.]

[If no open commitments:]
*No open commitments in people log.*

[If no people log file:]
*No people log entries recorded. Use /team-health:log [name] to start logging.*

---

## 4. Suggested Talking Points

[Generate 3-5 talking points. Each is a QUESTION the manager can ask.]
[Label each with source type: (signal), (commitment), (career), (pattern)]
[Priority order: (1) RED flags, (2) overdue commitments >30 days, (3) YELLOW flags, (4) career context overdue >90 days since last promo discussion, (5) log patterns (same concern 3+ times in 6 weeks), (6) wins worth acknowledging]

[COACHING HINTS: After each talking point, append an inline coaching hint from COACHING.md. Use the Framework Selection table in COACHING.md to determine which framework applies. Customize the hint with actual data from this person's signals and people log. Maximum one hint per talking point. If no framework clearly applies, omit the hint.]

1. [Question] *(source: [type] - [brief context])*
   💡 *Coaching: [1-2 sentence hint from the matching COACHING.md framework, with placeholders replaced by actual data]*
2. [Question] *(source: [type] - [brief context])*
   💡 *Coaching: [hint]*
3. [Question] *(source: [type] - [brief context])*
   💡 *Coaching: [hint]*

[COMPLIANCE CHECK: After generating talking points and coaching hints, review each against PRIVACY.md Rules 1 and 4. Rewrite any point or hint containing diagnostic or psychological language. Every point must be a behavioral observation framed as a question. Coaching hints must suggest HOW to ask, not WHAT to conclude.]

[If fewer than 3 data sources have content: generate from available sources. Minimum 3 points even if all come from one source.]

---

## 5. Context Reminders

**Goals:** [stated_goals list, or "None recorded"]
**Last promo discussion:** [date, or "Not recorded"] [if recorded: "(N months ago)"]
**Notes:** [career_context.notes, or "-"]

**Recent log entries:**
[Up to 5 entries since lookback_start, newest first:]
- [YYYY-MM-DD] · [category] - [content]

[If no entries since lookback: "No entries since [lookback_start]."]
[If no people log: "No people log entries recorded."]

---

*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
```

## Phase E - State write

Execute ONLY after Phase D output is complete.

1. Take PERSON_DATA loaded in Phase B (already in memory).
   - If no person file existed: create the base structure:
     ```json
     {
       "schema_version": "1",
       "name": "[full name from config]",
       "slug": "[slug]",
       "created": "[today]",
       "last_updated": "[today]",
       "last_1on1_date": "[today]",
       "prep_run": "[today]",
       "entries": [],
       "open_commitments": [],
       "career_context": {
         "stated_goals": [],
         "last_promo_discussion": null,
         "notes": ""
       }
     }
     ```
2. Set: `person.last_1on1_date = [today's date]`
3. Set: `person.prep_run = [today's date]`
4. Set: `person.last_updated = [today's date]`
5. Write the COMPLETE updated person object to `.team-health/people/<slug>.json` using the Write tool. Do NOT truncate entries, open_commitments, or career_context.
6. Confirm: "Prep sheet generated for [Name]. Next prep will look back from today."
