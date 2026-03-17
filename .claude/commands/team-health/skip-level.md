---
description: "Generate an upward-facing team brief for your skip-level meeting. People log content excluded by default."
argument-hint: "[--weeks N | --quarter] [--include-person <name>]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(ls .team-health/pulse-history/)
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

## Privacy Gate

CRITICAL — read this before any data collection:

Do NOT read any files under .team-health/people/ during this command.
People log content is excluded from skip-level output by default.

Exception: if $ARGUMENTS contains "--include-person <name>":
  1. Resolve <name> to slug using config.json team array (case-insensitive prefix match, same as prep.md)
  2. Read .team-health/people/<slug>.json
  3. Include that person's log themes in the People Themes section, labeled as "(opt-in content)"
  4. All other individuals remain unnamed in the output

Default behavior (no --include-person flag):
  - All pulse aggregations use counts only: "N team members", never individual names
  - No people log content in any section
  - This is enforced structurally — skip the Read call entirely for people files

## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md — 11 signals, thresholds, two-signal rule, severity levels
2. .claude/team-health/BASELINES.md — baseline computation algorithm, first-run behavior, provenance fields
3. .claude/team-health/PRIVACY.md — output language rules, required disclaimer, degradation language

Follow these documents exactly. Do not improvise signal thresholds, flagging rules, or output language.

## Phase A — Argument Parsing and Lookback Window

After reading reference docs, parse $ARGUMENTS and determine the lookback window:

1. Check $ARGUMENTS for flags:
   - If "--quarter" is present: lookback_start = first day of the current calendar quarter (e.g., if today is 2026-03-17, lookback_start = 2026-01-01)
   - If "--weeks N" is present (N is a positive integer): lookback_start = today minus (N × 7 days)
   - Otherwise (default): read config.json.last_skip_level
     - If last_skip_level is a valid YYYY-MM-DD date: lookback_start = that date, source = "since last skip-level on [date]"
     - If last_skip_level is null: lookback_start = today minus 14 days (2-week default), source = "default — no prior skip-level recorded"

2. State the determined window: "Covering [lookback_start] to [today] ([N weeks], [source])"

3. Also parse "--include-person <name>" if present:
   - If found: extract the name argument (everything after "--include-person" up to the next flag or end of $ARGUMENTS)
   - Store as INCLUDE_PERSON_NAME (or null if flag not present)
   - Privacy Gate rules above govern whether to read the people file

## Phase B — Pulse History Discovery and Loading

1. Run: Bash(ls .team-health/pulse-history/)
   - If the command errors or the directory is empty: note "No pulse history found for this period." Continue below.
     - If no MCPs are active either: tell the user "No pulse history and no MCP sources configured — cannot generate a meaningful brief." Stop.
     - Otherwise: note the limitation and continue with available MCP data.

2. Parse the listing. Files are named YYYY-WNN.md (e.g., 2026-W11.md).

3. Convert lookback_start to its ISO week. Filter the listed files: keep only files where YYYY-WNN >= lookback_start's ISO week and YYYY-WNN <= today's ISO week.

4. Read each qualifying file using the Read tool (in order, oldest to newest).

5. Store contents in memory as PULSE_HISTORY array (ordered oldest to newest).

## Phase C — Aggregation (team-level only by default)

From pulse history files, extract team-level aggregate data:

**Flag aggregation (CRITICAL — anonymized counts only):**
- When reading pulse history files, extract team-level aggregate counts only. Do NOT carry forward individual names from the "Flagged Members" sections into the brief output. Aggregate as: "N team members were flagged in week X."
- Count total flagged members per week (extract from "Flagged Members" sections — count rows, discard names)
- Identify signal types mentioned in flags (PR review lag, commit days, meeting load, etc.)
- Track flag trend across the lookback window: improving (fewer flags in recent weeks), stable, or degrading (more flags in recent weeks)

**Delivery aggregation:**
- Extract aggregate PR counts and ticket counts mentioned in summary or "Sprint Facts" sections (team totals, not per-person)
- Identify any particularly high-output or low-output periods relative to adjacent weeks in the dataset

**opt-in content (if --include-person was passed):**
- After confirming --include-person flag in $ARGUMENTS, resolve the name to a slug:
  1. Read the `team` array from config.json (already in memory from Pre-flight Check)
  2. Find the entry whose `name` field is a case-insensitive prefix match for INCLUDE_PERSON_NAME
  3. Use the `slug` field from that entry
  4. If multiple matches: list them and stop; if no match: "No team member matching '[name]' found" and stop
- Read .team-health/people/<slug>.json
- Extract log themes: recurring categories (any category appearing 3+ times in recent entries), open_commitments (open status only), career_context.stated_goals (list only, no raw notes)
- Do NOT extract raw entry content or verbatim log text — themes only

## Phase D — Output (5-section brief)

Render this exact format. Follow section numbering and header names exactly.

---

# Team Brief — [lookback_start] to [today]

*Covering: [lookback_start] to [today] ([N weeks])*
*Sources: [count of qualifying pulse history files] pulse snapshot(s) | MCP: [list of active sources from config, or "none configured"]*

---

## 1. Delivery Status

[Team-level velocity from pulse history: aggregate PR counts, ticket counts — totals across the team, not per person]
[Describe trend: stable, improving, declining, and over what period]
[If no pulse history: "Delivery data unavailable — no pulse history found for this period."]
[If no pulse history and no MCPs: this section will be empty — state the limitation]

---

## 2. Risks and Watch Areas

[Team-level aggregate flags from pulse history]
[Use count language exclusively: "N team members showed [signal type] signals during this period — worth monitoring as a group trend."]
[State the signal type by name (e.g., "PR review lag", "commit activity", "meeting load") — never the person's name]
[NEVER use individual names here unless --include-person was passed for that specific person AND this section explicitly requires it — risks sections use count language regardless]
[If no flags in any week of the lookback window: "No team-level risk signals in this period."]

---

## 3. People Themes

[Patterns derived from pulse flags — anonymized by default]
[Use count language: "Two team members had elevated PR review lag for 2+ consecutive weeks." or "One team member has had blocked tickets for more than 1 sprint."]
[NEVER name individuals here unless --include-person opt-in applies]
[If --include-person was passed: append a labeled subsection after the anonymized count summary:]

### [Full Name from config] (opt-in content)

[Log themes for the named person — recurring categories from people log, open commitments summary, career goals list. No raw log entry content. No verbatim notes.]
[Label this subsection clearly as "(opt-in content)" so the brief clearly indicates this was an explicit choice]

[If no pulse flags and no --include-person: "No recurring signal themes in this period." or "No people log content included — run with --include-person <name> to include a team member's log themes."]

---

## 4. Asks / Escalations

[ Add escalation items here before sharing this brief ]

*This section is for items requiring your manager's attention. Edit the brief to add your asks before the meeting.*

---

## 5. Team Wins

[Positive delivery signals from pulse history — PRs merged, tickets closed, velocity stable or above trend]
[Frame factually: "Team shipped [X] PRs in [week range]" or "Team velocity was stable across all [N] weeks in this period"]
[If no positive delivery data found: "No specific delivery highlights identified in pulse data for this period."]

---

*This brief reflects behavioral data signals. People log content is excluded by default.*

---

## Compliance Check

After generating all 5 sections above, scan the complete output before finalizing:

1. **Individual names check:** Search for any team member name not explicitly covered by --include-person opt-in. If found: rewrite the sentence to use count language ("N team members").

2. **Diagnostic language check (PRIVACY.md Rule 1):** Search for any of these prohibited phrases: "burned out", "burnout", "struggling", "disengaged", "checked out", "overwhelmed", "anxious", "stressed", "seems to be", "appears to be". If found: rewrite as a behavioral observation (e.g., "commit activity dropped" instead of "seems disengaged").

3. **Team-relative scoring check (PRIVACY.md Rule 3):** Search for: "bottom", "worst", "weakest", "least", "compared to", "below the team", "below average". If found: remove the comparative language.

4. **People log leak check:** If --include-person was NOT passed, confirm that zero content from any .team-health/people/ file appears in the output. Any match is a violation — remove it.

If any violation is found: rewrite the offending sentence before finalizing output.

## Phase E — State Write

Execute Phase E ONLY after Phase D output and the Compliance Check are complete. Do not write state before output.

1. Take the config.json already in memory from the Pre-flight Check.
2. Set config.last_skip_level = today's date (YYYY-MM-DD from the injected date at the top of this command)
3. Set config.last_updated = today's date (same date)
4. Build the COMPLETE updated config.json in memory — include ALL existing fields:
   - schema_version
   - setup_complete
   - manager (full object)
   - team (full array — do NOT omit any team members or fields)
   - sources (full object with all 4 keys)
   - baseline_window_weeks
   - github_org
   - sprint_cadence_weeks
   - last_skip_level (now set to today)
   - last_updated (now set to today)
   Do NOT omit any field. Do NOT write only the changed fields.
5. Write the complete updated config.json using a SINGLE Write tool call to .team-health/config.json.
6. Confirm: "Brief generated. Next skip-level will cover from today forward."
