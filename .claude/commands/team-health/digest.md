---
description: "Run a weekly pulse, save the report locally, and send a condensed summary to Slack. Designed for scheduled Friday morning automation."
argument-hint: "[--dry-run] [--no-slack] [--channel <channel-name>]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(mkdir -p .team-health/pulse-history), Bash(mkdir -p .team-health/digests)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`

## What This Command Does

`/team-health:digest` is a convenience wrapper that:
1. Runs the full pulse analysis (same as `/team-health:pulse`)
2. Saves the detailed pulse report to `.team-health/pulse-history/` (same as pulse)
3. Saves a manager-friendly digest to `.team-health/digests/`
4. Sends a condensed summary to the manager via Slack DM (unless `--no-slack` is passed)

This command is designed to be run on a schedule (every Friday morning) via the `schedule` skill, or manually at any time.

## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.
4. Parse $ARGUMENTS for flags:
   - `--dry-run`: generate the digest but do NOT send to Slack and do NOT write pulse history or baselines. Output only.
   - `--no-slack`: run pulse and save files locally, but skip the Slack send step.
   - `--channel <channel-name>`: send to a Slack channel instead of DM. Channel name without # prefix.

## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md - signal definitions, thresholds, two-signal rule
2. .claude/team-health/BASELINES.md - baseline computation algorithm
3. .claude/team-health/PRIVACY.md - output language rules, required disclaimer
4. .claude/team-health/COACHING.md - coaching frameworks (for digest coaching tip)

Follow these documents exactly.

## Phase A - Run Pulse Analysis

Execute the FULL pulse analysis as specified in pulse.md Phases A through C:
1. Read baselines.json (or initialize if absent)
2. Note available/unavailable sources
3. For each team member: collect signals from all active MCP sources
4. Compare to baselines, apply two-signal rule, determine flag status per person

Store results in memory. Do NOT write any files yet.

## Phase B - Generate Detailed Pulse Report

Generate the full pulse dashboard output as specified in pulse.md Phase D:
- Summary table with GREEN/YELLOW/RED per person
- Detail sections for flagged individuals
- Disclaimer

Store this output as FULL_PULSE_REPORT.

## Phase C - Generate Condensed Digest

From the pulse results, generate a condensed manager-friendly digest:

```
# Weekly Digest — [ISO week] ([today YYYY-MM-DD])

## Team Snapshot

[N] people scanned | Sources: [active sources]

| Status | Count |
|--------|-------|
| 🟢 GREEN | [N] |
| 🟡 YELLOW | [N] |
| 🔴 RED | [N] |
| ⏳ PENDING | [N] |

## Attention Needed

[If any YELLOW or RED members exist:]
[For each, one line:]
- **[Name]** ([STATUS]): [most significant triggering signal in plain English, e.g., "PR review count dropped 60% below baseline"]

[If no flags:]
✅ No flags this week. All team members within personal baselines.

## Quick Actions

[Generate 1-3 actionable items based on flags:]
[Examples:]
- Consider running `/team-health:prep [name]` before your next 1:1 with [flagged person]
- [Name] has been flagged for [N] consecutive weeks — may warrant a dedicated check-in
- Meeting load is high across [N] team members — consider reviewing recurring meetings

[If no flags, suggest:]
- Good week for career development conversations — run `/team-health:summary [name]` to review anyone's growth context
- Consider using `/team-health:log [name]` to capture any recent wins

## Coaching Tip of the Week

[Select ONE coaching framework from COACHING.md that is most relevant to this week's data. If flags exist, pick the framework matching the most common flag type. If no flags, rotate through frameworks sequentially based on ISO week number (week_number % 10 + 1 = framework number).]

**This week: [Framework Name]**
[2-3 sentence summary of the framework's core principle and one example opening question.]

---
*Digest generated [today]. Full details: .team-health/pulse-history/[ISO-week].md*
*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
```

Store this output as DIGEST_CONTENT.

## Phase D - File Writes

Skip this phase entirely if `--dry-run` was passed. Output DIGEST_CONTENT and stop.

### D1 - Write pulse history (same as pulse.md Phase E)

If NOT --dry-run:
1. Update baselines.json following pulse.md Phase E Step 1 exactly.
2. Run: Bash(mkdir -p .team-health/pulse-history)
3. Write pulse history snapshot to `.team-health/pulse-history/<ISO-week>.md` following pulse.md Phase E Step 2 format.

### D2 - Write digest file

Run: Bash(mkdir -p .team-health/digests)
Write DIGEST_CONTENT to `.team-health/digests/<ISO-week>-digest.md`

## Phase E - Slack Delivery

Skip this phase if `--no-slack` or `--dry-run` was passed.

### E1 - Determine delivery target

If `--channel <channel-name>` was passed:
- Use the Slack MCP to search for the channel by name.
- If found: deliver to that channel.
- If not found: warn "Channel '[channel-name]' not found. Digest saved locally but not sent to Slack." — stop Slack delivery.

If no --channel flag:
- Read the manager's `slack_user_id` from config.json.
- Wait — config.json stores the manager's name and github_username, but may not have their Slack user ID.
  - If config has a `manager.slack_user_id` field: use it.
  - If not: use the Slack MCP to search for the manager by name (config.manager.name). If found, use their Slack user ID.
  - If not found: warn "Could not determine your Slack user ID. Add `manager.slack_user_id` to config.json via `/team-health:setup --reset`, or use `--channel` flag. Digest saved locally." — stop Slack delivery.

### E2 - Format Slack message

Convert DIGEST_CONTENT to Slack-compatible markdown:
- Keep the structure but simplify formatting
- Remove file path references (not useful in Slack)
- Keep the disclaimer

### E3 - Send via Slack MCP

Use the Slack MCP send_message tool to deliver the formatted digest to the determined target (DM or channel).

If send fails: warn "Slack delivery failed: [error]. Digest saved locally at .team-health/digests/[ISO-week]-digest.md"

## Phase F - Confirmation

Output to the terminal:

```
Weekly digest complete — [ISO week]

📊 Pulse: [N] GREEN, [N] YELLOW, [N] RED, [N] PENDING
📁 Full report: .team-health/pulse-history/[ISO-week].md
📁 Digest: .team-health/digests/[ISO-week]-digest.md
[If Slack sent:] 💬 Sent to Slack: [DM to manager / #channel-name]
[If Slack skipped:] 💬 Slack delivery skipped (--no-slack or delivery issue)

Next digest: run again next Friday, or schedule with:
  /schedule create --name "team-health-digest" --interval "weekly" --day "friday" --time "09:00" --command "/team-health:digest"
```

---

## Scheduling Setup (for automated Friday runs)

To set up automated weekly digests, the manager should run:

```
/schedule create --name "team-health-digest" --interval "weekly" --day "friday" --time "09:00" --command "/team-health:digest"
```

This creates a scheduled task that runs the digest command every Friday at 9:00 AM. The schedule skill handles the automation; this command handles the content.

To verify the schedule:
```
/schedule list
```

To update the schedule:
```
/schedule update --name "team-health-digest" --time "08:00"
```

---

## Config Extension: Manager Slack ID

For Slack DM delivery, this command needs the manager's Slack user ID. The recommended approach is to add it to config.json during setup:

```json
{
  "manager": {
    "name": "Kamal Laungani",
    "github_username": "launganik",
    "timezone": "Europe/Berlin",
    "slack_user_id": "U0XXXXXXX"
  }
}
```

If `manager.slack_user_id` is absent, the command falls back to searching Slack by the manager's name. If that also fails, it skips Slack delivery and saves locally only.
