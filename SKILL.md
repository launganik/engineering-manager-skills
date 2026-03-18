# team-health

Help for engineering managers who want to be more human - remembering what their people care about, noticing when someone might be struggling, keeping commitments.

This skill surfaces what a good manager with infinite attention span would already notice.

---

## What This Does

team-health synthesizes signals from tools your team already uses - GitHub, Jira, Slack, and Google Calendar - into situational intelligence for engineering managers. It tracks behavioral signals (PR activity, ticket velocity, meeting load, participation patterns) against each person's own personal 8-week rolling baseline, so you see deviations from their norm rather than comparisons to the rest of the team.

The skill does not diagnose, score, or judge your direct reports. It surfaces observations - not conclusions. Every output is framed as a behavioral pattern for a manager to interpret, not a verdict. This tool is designed for engineering managers who want a thoughtful, private prep aid, not automated pipelines or HR reporting.

All data stays local on your machine. Nothing is sent to an external service.

---

## Prerequisites

### Required

- **Claude Code** ([code.claude.com](https://code.claude.com)) installed and active in your terminal

### Optional MCP Sources

The skill works without any MCP sources - you will just have fewer signals. GitHub is the most valuable source for most engineering teams.

| MCP Source | Signals Enabled | Install Command | Absent Signals |
|---|---|---|---|
| GitHub MCP | PR frequency, review participation, commit activity, PR review lag | `claude mcp add github ...` | All PR/commit signals unavailable |
| Jira MCP | Ticket velocity, blocked tickets, sprint carry-over | `claude mcp add jira ...` | Jira signals unavailable |
| Slack MCP | Channel participation trend, response latency (metadata only, no DM content) | `claude mcp add slack ...` | Slack signals unavailable |
| Google Calendar MCP | Meeting load %, 1:1 adherence | `claude mcp add google-calendar ...` | Calendar signals unavailable |

---

## Installation

1. Copy the skill files into your project directory:

   ```bash
   # Option A: Clone this repo into your project
   git clone https://github.com/your-org/antigravity-skills .

   # Option B: Copy just the skill files
   cp -r /path/to/antigravity-skills/.claude ./
   cp /path/to/antigravity-skills/SKILL.md ./
   ```

2. Verify the command files are present:

   ```bash
   ls .claude/commands/team-health/
   # Should show: setup.md log.md pulse.md prep.md skip-level.md retro-prep.md
   ```

3. Open Claude Code in your project directory and run first-time setup:

   ```
   /team-health:setup
   ```

   The setup flow will ask for your team roster and detect which MCP sources are available.

4. Verify setup completed:

   ```bash
   cat .team-health/config.json
   # Should show: setup_complete: true and your team roster
   ```

5. Verify gitignore protection:

   ```bash
   grep ".team-health" .gitignore
   # Should show: .team-health/
   ```

---

## Commands

### `/team-health:setup`

First-time setup. Collects team roster, 1:1 cadences, and detects available MCP sources. Must be run before any other command.

**Syntax:** `/team-health:setup`

**Flags:** `--reset` to reconfigure an existing setup

**Requires:** Nothing (runs first)

**Example:**
```
/team-health:setup
```

---

### `/team-health:log <name>`

Append a note to a direct report's people log. Structures free-form input into dated, categorized entries.

**Syntax:** `/team-health:log alice` or `/team-health:log "Alice Chen"`

**Example:**
```
/team-health:log alice
> gave feedback on the auth PR, really clean API design
```

**Requires:** Setup complete. Creates `people/<slug>.json` on first use.

---

### `/team-health:pulse`

Weekly team health scan. Scores each person against their personal 8-week baseline, flags anomalies conservatively, and stores baselines for future runs.

**Syntax:** `/team-health:pulse`

**Output:** Summary table (green/yellow/red per person) + detail sections for flagged individuals

**Requires:** Setup complete. GitHub MCP strongly recommended.

---

### `/team-health:prep <name>`

Generate a 2-minute prep sheet for an upcoming 1:1. Synthesizes live tool signals against personal baselines and surfaces standing items from the people log.

**Syntax:** `/team-health:prep alice`

**Output:** Status snapshot, signal flags, standing items, talking points, context reminders

**Requires:** Setup complete. Pulse run at least once recommended.

---

### `/team-health:skip-level`

Generate an upward-facing team brief. Aggregates team-level themes - individual names do not appear without explicit opt-in.

**Syntax:** `/team-health:skip-level`

**Flags:** `--weeks 2`, `--quarter`, `--include-person <name>`

**Requires:** Setup complete. Pulse history.

---

### `/team-health:retro-prep`

Generate a data-seeded sprint retro agenda. Discussion seeds are attributed to work items, never to individuals.

**Syntax:** `/team-health:retro-prep`

**Requires:** Setup complete. Jira MCP recommended.

---

## Where Data Lives

```
.team-health/              # Gitignored - stays on your machine
  config.json              # Team roster, MCP sources, settings
  baselines.json           # 8-week rolling baselines per person per signal
  people/                  # One file per direct report
    alice-chen.json        # Longitudinal notes, commitments, career context
  pulse-history/           # Weekly pulse snapshots
    2026-W11.md

.claude/
  commands/team-health/    # Slash command files (safe to commit)
  team-health/             # Reference docs loaded by commands (safe to commit)
    SIGNALS.md
    BASELINES.md
    PRIVACY.md
```

The `.team-health/` directory is added to `.gitignore` automatically during setup. Skill files under `.claude/` are safe to commit and share with your team.

---

## Privacy Design

All people data is stored locally in `.team-health/` and gitignored by default - nothing leaves your machine unless you explicitly commit it, which the gitignore prevents. Slack signals use channel participation metadata only: message counts, distinct channel activity, and @-mention response latency in public channels. DM content is never accessed and this is not configurable - DM content access is permanently out of scope. All outputs use behavioral observations ("PR activity dropped 40% below their 8-week baseline") rather than diagnoses or judgments. The `/team-health:skip-level` command aggregates team-level themes without attributing observations to individuals unless you explicitly include a person by name. The goal throughout is a briefing tool that passes the test: would you be comfortable if your direct report saw this output?

---

## Upgrading / Uninstalling

### Upgrading

Pull or copy new skill files over `.claude/commands/team-health/` and `.claude/team-health/`. Your `.team-health/` data is unaffected - it lives in your project directory and is not part of the skill install.

```bash
cp -r /path/to/updated-antigravity-skills/.claude/commands/team-health/ .claude/commands/team-health/
cp -r /path/to/updated-antigravity-skills/.claude/team-health/ .claude/team-health/
```

### Uninstalling

```bash
rm -rf .claude/commands/team-health/ .claude/team-health/
```

Optionally delete `.team-health/` to remove all local people data:

```bash
rm -rf .team-health/
```
