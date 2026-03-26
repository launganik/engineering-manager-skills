# Team Health - Signal Reference

Loaded by: /team-health:pulse, /team-health:prep
Purpose: Defines each tracked signal, its data source, how it is computed, and the threshold that triggers a flag.
Last updated: 2026-03-10

> When computing signal scores, follow this document exactly. Thresholds here are the defaults. They may be overridden in config.json (future feature - not yet implemented).

---

## Section 1: GitHub Signals

If `sources.github` is false in config.json, skip all signals in this section and note: "GitHub signals are unavailable - GitHub MCP is not configured. PR and commit data cannot be included."

---

### Signal: `github_prs_merged_per_week`

**Source:** GitHub MCP
**What it measures:** Count of pull requests merged by this person in the past 7 calendar days.
**How to compute it:**
1. Query the GitHub MCP for all PRs merged in the past 7 days where the author matches this person's `github_username` (from config.json team array).
2. Count the total number of merged PRs.
3. Compare to this person's 8-week rolling baseline (see BASELINES.md for computation instructions).

**Flag threshold:**
- Flag if `current_value < (baseline_mean - 2 * baseline_stddev)`
- Interpretation: A significant drop below their personal norm is worth surfacing.
- Do NOT flag based on absolute counts alone unless the two-signal rule is also triggered.

---

### Signal: `github_pr_review_count_per_week`

**Source:** GitHub MCP
**What it measures:** Count of pull requests where this person left at least one review comment in the past 7 calendar days.
**How to compute it:**
1. Query the GitHub MCP for all PR review events in the past 7 days where the reviewer matches this person's `github_username`.
2. Count distinct PRs (one count per PR, regardless of how many comments they left on that PR).
3. Compare to this person's 8-week rolling baseline.

**Flag threshold:**
- Flag if `current_value < (baseline_mean - 2 * baseline_stddev)`
- Interpretation: Review participation is a leading indicator of engagement. A drop may signal capacity pressure or disengagement.

---

### Signal: `github_pr_review_lag_days`

**Source:** GitHub MCP
**What it measures:** Average number of days between a PR being opened and this person leaving their first review comment, for PRs they reviewed in the past 14 calendar days.
**How to compute it:**
1. Query the GitHub MCP for PRs where this person submitted a review in the past 14 days.
2. For each such PR, compute `first_review_date - pr_opened_date` in fractional days.
3. Average across all qualifying PRs.
4. Compare to this person's 8-week rolling baseline.
5. Additionally, check for absolute-threshold violations (step 4b below).

**Flag threshold (baseline comparison):**
- Flag if `current_value > (baseline_mean + 2 * baseline_stddev)`
- Interpretation: Taking significantly longer to review PRs than their norm may indicate overload.

**Flag threshold (absolute - no baseline needed):**
- Also flag if any individual PR assigned to this person for review has been open for >48 hours with no review from them.
- This is an absolute threshold - applies even if they have no baseline history.

---

### Signal: `github_commit_days_per_week`

**Source:** GitHub MCP
**What it measures:** Count of distinct calendar days (Mon–Sun) in the past 7 days on which this person made at least one commit to any repository within the configured `github_org`.
**How to compute it:**
1. Query the GitHub MCP for all commits authored by this person's `github_username` in the past 7 days, scoped to the `github_org` from config.json.
2. Extract the unique set of calendar dates from the commit timestamps.
3. Count the number of distinct dates.
4. Compare to this person's 8-week rolling baseline.

**Flag threshold:**
- Flag if `current_value < (baseline_mean - 2 * baseline_stddev)`
- Interpretation: A drop in commit activity can indicate blockers, context-switching overload, or disengagement - but it is a weak signal alone.

---

## Section 2: Jira Signals

If `sources.jira` is false in config.json, skip all signals in this section and note: "Jira signals are unavailable - Jira MCP is not configured. Ticket velocity and blocker data cannot be included."

---

### Signal: `jira_tickets_closed_per_week`

**Source:** Jira MCP
**What it measures:** Count of Jira tickets transitioned to Done or Closed status by this person in the past 7 calendar days.
**How to compute it:**
1. Query the Jira MCP for tickets assigned to this person that transitioned to Done or Closed in the past 7 days. Match by `jira_user_id` from config.json.
2. Count the total number of such tickets.
3. Compare to this person's 8-week rolling baseline.

**Flag threshold:**
- Flag if `current_value < (baseline_mean - 2 * baseline_stddev)`
- Interpretation: A significant drop in ticket closure rate may indicate blocked work, scope creep, or disengagement.

---

### Signal: `jira_tickets_in_progress_over_2_sprints`

**Source:** Jira MCP
**What it measures:** Count of Jira tickets currently assigned to this person with status In Progress, where the ticket has been In Progress for more than 2 sprint cycles.
**How to compute it:**
1. Query the Jira MCP for all In Progress tickets assigned to this person.
2. Determine the sprint duration from `sprint_cadence_weeks` in config.json (default: 2 weeks). Two sprint cycles = `sprint_cadence_weeks * 2 * 7` days.
3. For each In Progress ticket, compute how many days it has been in the In Progress status.
4. Count tickets where `days_in_progress > (sprint_cadence_weeks * 2 * 7)`.

**Flag threshold:**
- Flag if `count >= 1` (absolute threshold - any stuck ticket is worth surfacing)
- No baseline comparison needed; persistent in-progress tickets are inherently worth flagging regardless of historical pattern.

---

### Signal: `jira_blocked_tickets`

**Source:** Jira MCP
**What it measures:** Count of Jira tickets assigned to this person currently in a Blocked status (or equivalent - check the Jira instance's status naming).
**How to compute it:**
1. Query the Jira MCP for all tickets assigned to this person with a status of Blocked (or Impediment, or equivalent blocking status used in this Jira instance).
2. Count the total.

**Flag threshold:**
- Flag if `count >= 1` (absolute threshold)
- No baseline comparison needed; blocked tickets require immediate manager attention regardless of historical pattern.

---

## Section 3: Slack Signals

If `sources.slack` is false in config.json, skip all signals in this section and note: "Slack signals are unavailable - Slack MCP is not configured. Participation metadata cannot be included."

> **Privacy note:** Slack signals use metadata only - channel participation counts and response latency in public channels. DM content is never accessed. This is not configurable; DM content access is permanently out of scope.

---

### Signal: `slack_channel_participation_per_week`

**Source:** Slack MCP
**What it measures:** Count of distinct public channels where this person posted at least one message in the past 7 calendar days.
**How to compute it:**
1. Query the Slack MCP for message events in the past 7 days attributed to this person's `slack_user_id` (from config.json).
2. Restrict to public channels only - do not include DMs, group DMs, or private channels.
3. Count the number of distinct channel IDs in the result.
4. Compare to this person's 8-week rolling baseline.

**Flag threshold:**
- Flag if `current_value < (baseline_mean - 1.5 * baseline_stddev)` [lower threshold than other signals]
- Interpretation: Channel participation is a softer signal than commit or PR activity. The 1.5 stddev threshold is intentionally more lenient; use this signal to support, not drive, a flag decision.

---

### Signal: `slack_response_latency_hours`

**Source:** Slack MCP
**What it measures:** Median number of hours between this person being @-mentioned in a public channel and their response, for the past 14 calendar days.
**How to compute it:**
1. Query the Slack MCP for @-mention events targeting this person's `slack_user_id` in public channels in the past 14 days.
2. For each @-mention, find the first message posted by this person in that same channel thread after the mention.
3. Compute `response_time = first_response_timestamp - mention_timestamp` in fractional hours.
4. Take the median across all qualifying @-mention events.
5. If fewer than 3 qualifying @-mention events exist, do not compute this signal - insufficient data.
6. Compare to this person's 8-week rolling baseline.

**Flag threshold:**
- Flag if `current_value > (baseline_mean + 2 * baseline_stddev)`
- Interpretation: Significantly slower responses than their norm may indicate overload, or reduced engagement with team communication.

---

## Section 4: Confluence Signals

If `sources.confluence` is false in config.json (or no Atlassian/Confluence MCP is available), skip all signals in this section and note: "Confluence signals are unavailable - Atlassian MCP is not configured. Documentation and knowledge-sharing activity cannot be included."

> **Note:** Confluence signals measure non-code contributions — design docs, RFCs, architecture reviews, onboarding guides, and cross-team knowledge sharing. These are valuable leading indicators of senior-level impact that do not appear in GitHub or Jira metrics.

---

### Signal: `confluence_pages_authored_per_month`

**Source:** Confluence MCP (via Atlassian MCP)
**What it measures:** Count of Confluence pages created or significantly edited by this person in the past 30 calendar days.
**How to compute it:**
1. Query the Atlassian MCP for Confluence pages where this person is the creator or last editor, modified in the past 30 days. Use their display name or Atlassian account ID (from config.json `jira_user_id` which is the same Atlassian account).
2. "Significantly edited" means: the person is listed as the creator, OR the person made edits that changed more than a trivial amount of content (if edit history is available). If edit granularity is not available, count any page where they are listed as creator or last modifier.
3. Count the total pages.
4. Compare to this person's 8-week rolling baseline (using 4-week observation windows averaged to weekly equivalent for baseline consistency).

**Flag threshold:**
- This signal does NOT trigger flags on its own. It is an informational signal only.
- It is surfaced in /team-health:summary and /team-health:prep as context, not as a flag input.
- Rationale: Documentation cadence is highly variable and task-dependent. A drop in page creation does not reliably indicate a problem.

---

### Signal: `confluence_comments_per_month`

**Source:** Confluence MCP (via Atlassian MCP)
**What it measures:** Count of comments this person left on Confluence pages in the past 30 calendar days.
**How to compute it:**
1. Query the Atlassian MCP for Confluence comments authored by this person in the past 30 days.
2. Count inline comments and footer comments separately if the API supports it; report the total.
3. Compare to this person's rolling baseline.

**Flag threshold:**
- Informational signal only — does not trigger flags.
- Surfaced in summary and prep for context on collaboration and review activity.

---

### Signal: `confluence_mentions_per_month`

**Source:** Confluence MCP (via Atlassian MCP)
**What it measures:** Count of Confluence pages published or updated in the past 30 calendar days where this person is @-mentioned in the body text.
**How to compute it:**
1. Query the Atlassian MCP using CQL: search for pages containing an @-mention of this person's display name or account ID, modified in the last 30 days.
2. Count distinct pages (not mention occurrences — one count per page regardless of how many times they're mentioned).
3. Compare to this person's rolling baseline.

**Flag threshold:**
- Informational signal only — does not trigger flags.
- High mention counts may indicate this person is a go-to expert or decision-maker for certain areas. Surface as positive context.

---

### Confluence Signal Aggregation for Prep and Summary

When surfacing Confluence data in /team-health:prep or /team-health:summary, aggregate the three Confluence signals into a single contextual block:

```
**Confluence activity (last 30 days):** [N] pages authored/edited, [M] comments, [P] pages mentioned in
Recent pages: [list up to 5 most recent page titles authored or edited]
```

If Confluence activity is notably higher than baseline while GitHub activity is lower:
- In prep: trigger the "Contribution Reframe" coaching hint from COACHING.md Framework 10.
- In summary: note in Growth Context that non-code contributions are significant.

---

## Section 5: Calendar Signals

If `sources.calendar` is false in config.json, skip all signals in this section and note: "Calendar signals are unavailable - Calendar MCP is not configured. Meeting load and 1:1 adherence data cannot be included."

---

### Signal: `calendar_meeting_load_percent`

**Source:** Calendar MCP
**What it measures:** Percentage of working hours (Monday–Friday, 9:00 AM–6:00 PM in this person's timezone, or the manager's timezone if the person's is not known) occupied by meetings in the past 7 calendar days.
**How to compute it:**
1. Query the Calendar MCP for events on the **primary calendar only** in the past 7 days where this person is a confirmed attendee. Exclude shared, subscribed, and delegated calendars - these inflate meeting load with events the person may not actually attend.
2. Calculate total available working minutes: 5 days × 9 hours × 60 minutes = 2,700 minutes.
3. Sum the duration of all meeting events that fall within the 9:00 AM–6:00 PM window (clip events that extend outside this window).
4. Compute `meeting_load_percent = (total_meeting_minutes / 2700) * 100`.

**Flag threshold:**
- Flag if `meeting_load_percent > 60` (absolute threshold - no baseline comparison)
- Interpretation: More than 60% of working hours in meetings is inherently unsustainable for an individual contributor. This threshold is fixed and does not require a personal baseline.

---

### Signal: `calendar_1on1_adherence`

**Source:** Calendar MCP
**What it measures:** Whether the manager's scheduled 1:1 with this person occurred within the expected window based on their configured `1on1_cadence_weeks`.
**How to compute it:**
1. From config.json, read this person's `1on1_cadence_weeks` (default: 1 week if not specified).
2. Query the Calendar MCP for events on the **primary calendar only** containing this person and the manager as attendees, with a title matching a 1:1 pattern (e.g., "1:1", "1on1", person's name).
3. Determine the expected 1:1 window: the meeting should have occurred within the last `1on1_cadence_weeks * 7` days.
4. Check if any matching event occurred within that window.

**Flag threshold:**
- Flag if no qualifying 1:1 event found in the expected window
- Interpretation: A missed 1:1 cycle is worth surfacing - it may indicate scheduling pressure or an unintentional lapse in manager commitment.

---

## Section 6: Two-Signal Rule

```
IMPORTANT - Flag Conservatively:

A person should only appear as flagged (yellow or red) if:
  (a) Two or more signals are below their individual thresholds, OR
  (b) One signal exceeds 2 standard deviations from their personal baseline

Do NOT flag a person based on:
  - One signal that is slightly below baseline (within 1 stddev)
  - Team-relative comparison (e.g., "lowest PRs on the team")
  - Absolute counts without baseline comparison (except for the explicit absolute thresholds listed above)

When in doubt, do not flag. A false negative (missed flag) is preferable to a false positive
(unfair flag) in every case.
```

**Severity levels:**
- **Yellow (watch):** Two or more signals below threshold, or one signal at 2–2.5 stddev from baseline
- **Red (act):** Three or more signals below threshold, or one signal at >2.5 stddev from baseline, or any combination of baseline deviation + absolute-threshold violation

**Baseline-only signals:** The following signals use deviation from personal baseline:
- `github_prs_merged_per_week` (low = bad)
- `github_pr_review_count_per_week` (low = bad)
- `github_commit_days_per_week` (low = bad)
- `jira_tickets_closed_per_week` (low = bad)
- `slack_channel_participation_per_week` (low = bad)
- `github_pr_review_lag_days` (high = bad)
- `slack_response_latency_hours` (high = bad)

**Absolute-threshold signals (no baseline needed):**
- `jira_tickets_in_progress_over_2_sprints` - flag if count >= 1
- `jira_blocked_tickets` - flag if count >= 1
- `calendar_meeting_load_percent` - flag if > 60%
- `calendar_1on1_adherence` - flag if 1:1 missed in expected window
- `github_pr_review_lag_days` - also flag if any single PR open >48h with no review

---

## Section 7: Signal Availability Matrix

| Signal | Source | Required MCP | Available without MCP? |
|--------|--------|--------------|------------------------|
| `github_prs_merged_per_week` | GitHub | github | No |
| `github_pr_review_count_per_week` | GitHub | github | No |
| `github_pr_review_lag_days` | GitHub | github | No |
| `github_commit_days_per_week` | GitHub | github | No |
| `jira_tickets_closed_per_week` | Jira | jira | No |
| `jira_tickets_in_progress_over_2_sprints` | Jira | jira | No |
| `jira_blocked_tickets` | Jira | jira | No |
| `slack_channel_participation_per_week` | Slack | slack | No |
| `slack_response_latency_hours` | Slack | slack | No |
| `calendar_meeting_load_percent` | Calendar | calendar | No |
| `calendar_1on1_adherence` | Calendar | calendar | No |
| `confluence_pages_authored_per_month` | Confluence | atlassian | No (informational only) |
| `confluence_comments_per_month` | Confluence | atlassian | No (informational only) |
| `confluence_mentions_per_month` | Confluence | atlassian | No (informational only) |

**Total signals:** 14 (11 flag-eligible + 3 informational Confluence signals)

**Minimum viable set (GitHub-only):** With only the GitHub MCP configured, the pulse command can compute the 4 GitHub signals and apply the two-signal rule across that subset. The output will note that 7 of 11 flag-eligible signals are unavailable. Confluence signals are informational and do not participate in flagging.

**Fallback behavior:** If a signal cannot be computed due to missing MCP, absent data, or insufficient baseline history, omit it from the output rather than showing a zero or placeholder. State which signals were omitted and why.
