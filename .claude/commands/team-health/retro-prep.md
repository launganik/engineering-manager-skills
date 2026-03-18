---
description: "Generate a sprint retro agenda seeded with sprint facts. Attributed to work items, never individuals."
argument-hint: "[--sprint <sprint-id>]"
allowed-tools: Read, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(ls .team-health/pulse-history/)
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
1. .claude/team-health/SIGNALS.md - signal definitions and severity levels
2. .claude/team-health/PRIVACY.md - output language rules, work-item attribution rules, graceful degradation language

Follow these documents exactly. Do not improvise attribution rules or output language.

Note: .claude/team-health/BASELINES.md may be read optionally if context on signal computation is useful.

## Phase A - Sprint Window Determination

Parse $ARGUMENTS for the --sprint flag:

- If "--sprint <id>" is present AND sources.jira is true (from config.json):
  Use the Jira MCP to look up that sprint's start and end dates.
  State: "Retro scope: Sprint [id] ([start] to [end])"

- If "--sprint <id>" is present BUT sources.jira is false:
  Tell the user: "Jira MCP not configured - cannot look up sprint [id]. Run without --sprint flag to use pulse history window instead."
  Stop.

- If no --sprint flag is present AND sources.jira is true:
  Query the Jira MCP for the most recently completed sprint.
  Use that sprint's start and end dates as the sprint window.
  State: "Retro scope: [sprint name] ([start] to [end])"

- If no --sprint flag is present AND sources.jira is false:
  Read sprint_cadence_weeks from config.json (default: 2 if field is absent or null).
  The sprint window is: most recent sprint_cadence_weeks pulse history weeks.
  State: "Retro scope: last [N] weeks from pulse history (no Jira MCP - sprint boundary unavailable)"

## Phase B - Sprint Data Collection

### Pulse History (always - if files exist)

1. Run: Bash(ls .team-health/pulse-history/)
   - If the command errors or returns empty: note "No pulse history available." Continue with MCP data only if sources are active. If no MCPs and no history: tell the user the limitation, then stop.
2. Parse the listing. Files are named YYYY-WNN.md (e.g., 2026-W11.md).
3. Filter to files whose YYYY-WNN falls within the sprint window determined in Phase A.
4. Read each qualifying file using the Read tool.
5. Extract team-level flag summary from each week: "Week [ISO-week]: [N] flags"
6. Note the trend across qualifying weeks: improving (flags decreasing), stable, or degrading (flags increasing) signal health.

### GitHub Queries (if sources.github is true)

Scope ALL GitHub queries to github_org from config.json. Do not query across all of GitHub.

- PRs merged in the sprint window: count and list titles. Do NOT include author names in the output.
- PRs with >3 revision cycles (high review round-trips before merge): list titles only. No author names.
- PR review cycle time distribution for the sprint window: average and maximum. No per-person breakdowns.
- Reverted or hotfixed PRs in the sprint window: list titles. No author names.

CRITICAL: Do NOT query per-person breakdowns. All GitHub queries are team-scoped or work-item-scoped.

### Jira Queries (if sources.jira is true)

Scope to the project(s) associated with this team.

- Sprint velocity: count of tickets closed vs. count planned at sprint start.
- Carry-over count: tickets that were started but not closed within the sprint window.
- Blocked tickets at sprint end: count and ticket summaries. Do NOT include assignee names.
- Tickets in progress for more than 2 sprints: count only.

CRITICAL: Do NOT query per-person breakdowns. All Jira queries are team-scoped or work-item-scoped.

## Phase C - Attribution Check

Before rendering any output:

1. Note the team member names from the config.json team array.
2. Scan all data collected in Phase B for any occurrence of a team member name (first name, last name, or slug).
3. If a team member name is found in a seeded observation: rewrite that observation to attribute to the work item instead (PR title, ticket ID, or work area).

Example rewrite:
- Before (prohibited): "Alice had 5 revision cycles on the payment API PR"
- After (correct): "The payment API PR went through 5 revision cycles"

This check is mandatory. Every seeded discussion item must reference a work item, PR title, ticket ID, or work area - never a person name.

## Phase D - Output (Retro Agenda Format)

Render the following retro agenda. Follow the format exactly.

---

```
# Sprint Retro Agenda - [Sprint name or date range]

*Data sources: [active MCP sources used] and [N] pulse history snapshots*
*Note: All discussion seeds are attributed to work items, not individuals.*

---

## Sprint Facts

**Velocity:** [N tickets closed] / [M planned] - [carry-over count] carried over
**PRs:** [N merged], [M reverted or hotfixed], avg review cycle [X days]
**Blockers resolved:** [N] / [M remaining at sprint end]
```

If Jira is unavailable, replace velocity and ticket rows with:
"Ticket velocity unavailable - Jira MCP not configured."
Show only GitHub PR data and pulse history for Sprint Facts.

If GitHub is unavailable, replace PR rows with:
"PR data unavailable - GitHub MCP not configured."
Show only Jira data and pulse history.

If neither Jira nor GitHub is available:
"Sprint facts limited to pulse history - no GitHub or Jira MCP configured."
Show only pulse history summary.

---

```
## What Went Well - Seeds

*Team discusses. Add your own.*

- [Factual positive attributed to work item or area: e.g., "The [feature/service] shipped - N PRs merged cleanly"]
- [Delivery signal: e.g., "Sprint velocity was [X]% [above/at] baseline"]
- [Signal health: "Fewer team signal flags vs. previous sprint" - only if true from pulse history comparison]

**[ Team discussion space - what else went well? ]**

---

## What Was Hard - Seeds

*Team discusses. Add your own.*

- [Work-item attribution: e.g., "The [PR title / ticket ID] took [N] revision cycles - what drove the iterations?"]
- [Process observation: e.g., "[N] PRs had >3 revision cycles this sprint - what patterns do we see?"]
- [Blocker fact: e.g., "[N] tickets were blocked for >[timeframe]. What caused the blockage?"]

**[ Team discussion space - what else was hard? ]**

---

## Patterns Across Sprints

*Only populated if 2 or more pulse history weeks exist in the sprint window.*

- [Trend: e.g., "PR review cycle time has been [above/below] baseline for [N] consecutive weeks"]
- [Flag trend: e.g., "Team signal flags [increased/decreased/stable] over [N] weeks"]
```

If fewer than 2 qualifying pulse history weeks exist in the sprint window, output this line instead of trend items:
"Insufficient pulse history for cross-sprint patterns. Run /team-health:pulse weekly to build trend data."

```
**[ Team discussion space - what patterns do you see? ]**

---

## Action Items

**[ Team fills in during retro ]**

- [ ]
- [ ]
- [ ]

---

*Data seeds are derived from GitHub and Jira signals. All items are attributed to work, not people.*
```

---

**Key structural rules enforced in this command:**

1. Every data-seeded item references a work item (PR title, ticket ID, work area) - never a person name.
2. Every content section ends with a blank "Team discussion space" marker for the team to fill in during the retro.
3. The Action Items section contains ONLY blank checkboxes (3-5 of them). Do NOT pre-populate action items - the team generates actions during the retro.
4. No disclaimer is required (retro-prep is a process tool, not an individual health signal tool).
5. After generating seeds, run the Phase C attribution check: scan for team member names. If found, rewrite to attribute to work item instead.

**No Phase E - retro-prep makes zero state writes.**
