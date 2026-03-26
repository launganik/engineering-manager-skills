# team-health

A Claude Code skill for engineering managers that surfaces what a good manager with infinite attention span would already notice - without surveys, without surveillance, without extra process.

It pulls real signals from the tools your team already uses (GitHub, Jira, Slack, Calendar), compares them against each person's own personal 8-week rolling baseline, and gives you a private, thoughtful briefing before every 1:1, skip-level, and sprint retro.

**All data stays on your machine.** Nothing is sent to an external service.

---

## What it includes

Nine slash commands you run inside [Claude Code](https://claude.ai/code):

| Command | What it does |
|---------|-------------|
| `/team-health:setup` | One-time setup: collect team roster, detect available MCP sources |
| `/team-health:onboard <name>` | Guided context capture for a new direct report — career goals, strengths, growth areas, working style |
| `/team-health:log <name>` | Append notes to a direct report's people log, or query their history |
| `/team-health:summary <name>` | Read-only person profile — career goals, commitments, signal trends, log history, tenure |
| `/team-health:pulse` | Weekly team health scan — scores everyone against their personal baseline |
| `/team-health:prep <name>` | 2-minute 1:1 prep sheet with signals, standing items, talking points, and coaching hints |
| `/team-health:skip-level` | Upward-facing team brief for your own manager, privacy-gated by default |
| `/team-health:retro-prep` | Sprint retro agenda seeded with real data, attributed to work items not people |
| `/team-health:digest` | Weekly pulse + condensed Slack summary — designed for scheduled Friday automation |

---

## Prerequisites

**Required:**
- [Claude Code](https://claude.ai/code) installed and running in your terminal

**Optional (the more you have, the richer the signals):**

| MCP Source | Signals it enables |
|---|---|
| GitHub MCP | PR frequency, review participation, commit activity, review lag |
| Jira MCP | Ticket velocity, blocked tickets, sprint carry-over |
| Slack MCP | Channel participation trend, response latency (public channels only - no DM content) |
| Google Calendar MCP | Meeting load %, focus time blocks |
| Atlassian/Confluence MCP | Pages authored, comments, @-mentions — non-code contribution visibility |

The skill works without any MCP sources connected. GitHub-only is a fully supported configuration. Confluence signals are informational (they provide context but don't trigger flags).

---

## Installation

### Step 1 - Copy the skill files into your project

```bash
# Clone this repo into your project directory
git clone https://github.com/your-org/antigravity-skills .

# Or copy just the skill files into an existing project
cp -r /path/to/antigravity-skills/.claude ./
cp /path/to/antigravity-skills/SKILL.md ./
```

### Step 2 - Verify the files are in place

```bash
ls .claude/commands/team-health/
# setup.md  log.md  pulse.md  prep.md  skip-level.md  retro-prep.md
```

### Step 3 - Run first-time setup

Open Claude Code in your project directory:

```
/team-health:setup
```

The setup flow will:
1. Detect which MCP sources are available
2. Ask for your name, GitHub username, and org
3. Ask you to list your direct reports (names, roles, GitHub usernames)
4. Save everything to `.team-health/config.json`
5. Add `.team-health/` to `.gitignore` automatically

### Step 4 - Confirm it worked

```bash
cat .team-health/config.json
# Should show: "setup_complete": true and your team roster

grep ".team-health" .gitignore
# Should show: .team-health/
```

---

## Usage

### Onboard a new direct report

When someone joins your team (or when you want to backfill context for an existing report), run:

```
/team-health:onboard alice
```

This walks you through 5 sections: career goals, current strengths, growth areas, working style, and any commitments you've already made. Everything is saved to their people log and career context, making your first few 1:1s much richer.

---

### View a person summary (read-only)

```
/team-health:summary alice
```

Shows everything you know about a person in one view: profile, career goals, open commitments, signal trends over time, Confluence activity, and log history. This is read-only — it doesn't update any state, so you can look someone up at any time without side effects.

---

### Keep a people log

Run this any time you want to capture something about a direct report - feedback you gave, commitments you made, career conversations, wins, concerns:

```
/team-health:log alice
> gave her feedback on the auth PR design, really clean abstractions
```

Claude will infer the category (`feedback-given`) and structure the entry. Commitments are dual-tracked so they show up in prep sheets until you close them.

**Query the log** with natural language:

```
/team-health:log alice when did we last talk about promotion?
/team-health:log bob what commitments have I made this quarter?
```

---

### Run a weekly pulse

```
/team-health:pulse
```

Produces a team dashboard with a green/yellow/red row per person. Flags only fire when 2+ signals align OR a single signal exceeds 2 standard deviations from that person's own 8-week baseline - not the team average.

Flagged individuals get a detail section showing exactly which metric moved, the delta, and the source. Unflagged members get a table row only.

Every run writes a snapshot to `.team-health/pulse-history/YYYY-WNN.md` for use by skip-level and retro-prep later.

---

### Prep for a 1:1

```
/team-health:prep alice
```

Generates a 6-section prep sheet:

1. **Status Snapshot** - overall signal status since last 1:1
2. **Signal Flags** - any deviations from personal baseline (behavioral, not diagnostic)
2b. **Confluence Activity** - non-code contributions (pages authored, comments, mentions)
3. **Standing Items** - your open commitments and IC asks from the people log
4. **Suggested Talking Points** - 3-5 questions prioritized by urgency, each with an inline coaching hint that tells you *how* to raise the topic (not just what to raise)
5. **Context Reminders** - stated goals, last promo discussion date, recent log entries

The lookback window is dynamic: it starts from the date of your last 1:1 (tracked automatically), falling back to last prep run, then a 14-day default.

---

### Prepare for a skip-level

```
/team-health:skip-level
```

Generates an upward-facing team brief covering:
- Team delivery status
- Risks and themes (aggregated, never individual-attributed)
- Items requiring escalation (blank by default - you fill in)
- Team wins

**Individual names never appear in skip-level output by default.** Pulse observations are aggregated into themes: "two team members showing engagement signals worth monitoring" - not "Alice and Bob are at risk." Use `--include-person <name>` to explicitly include someone.

Supports `--weeks 2` or `--quarter` to adjust the lookback window.

---

### Seed a sprint retro

```
/team-health:retro-prep
```

Pulls sprint data from Jira (if connected) and pulse history to produce a retro agenda with:
- Sprint facts (velocity, carry-over count)
- Discussion seeds attributed to **work items, not people**: "the auth service PR went through 7 revision cycles - what happened?" not "Jordan had slow reviews"
- Blank space preserved for the team's own discussion

The command makes no state writes - it is a pure read-and-generate operation.

---

### Automated weekly digest

```
/team-health:digest
```

Runs the full pulse analysis, saves the detailed report locally, and sends a condensed summary to your Slack DM. Includes a team snapshot, attention items, quick actions, and a rotating coaching tip of the week.

**Flags:**
- `--dry-run` — generate the digest without saving files or sending to Slack
- `--no-slack` — save locally only, skip Slack delivery
- `--channel team-health` — post to a Slack channel instead of DM

**Schedule it for every Friday morning:**
```
/schedule create --name "team-health-digest" --interval "weekly" --day "friday" --time "09:00" --command "/team-health:digest"
```

---

## Where data lives

```
.team-health/              ← gitignored - stays on your machine only
  config.json              ← team roster, MCP sources, settings
  baselines.json           ← 8-week rolling baselines per person per signal
  people/
    alice-chen.json        ← longitudinal notes, commitments, career context
    bob-smith.json
  pulse-history/
    2026-W11.md            ← weekly pulse snapshots consumed by skip-level/retro
  digests/
    2026-W11-digest.md     ← condensed weekly digests (from /team-health:digest)

.claude/
  commands/team-health/    ← slash command files (safe to commit and share)
    setup.md               ← first-time setup
    onboard.md             ← guided context capture for new reports
    log.md                 ← people log append/query
    summary.md             ← read-only person profile
    pulse.md               ← weekly team health scan
    prep.md                ← 1:1 prep sheet with coaching hints
    skip-level.md          ← upward-facing team brief
    retro-prep.md          ← sprint retro agenda seeding
    digest.md              ← automated weekly pulse + Slack delivery
  team-health/             ← reference docs loaded by commands (safe to commit)
    SIGNALS.md             ← 14 signals (11 flag-eligible + 3 Confluence informational)
    BASELINES.md           ← baseline computation methodology
    PRIVACY.md             ← output language rules, degradation language
    COACHING.md            ← 10 situational coaching frameworks for inline hints
```

The `.team-health/` directory is gitignored automatically during setup. The `.claude/` skill files are safe to commit and share - they contain no personal data.

---

## Privacy design

- **Data is local only.** `.team-health/` is gitignored and never leaves your machine unless you explicitly remove the gitignore entry.
- **Slack: metadata only.** Channel participation counts and @-mention response latency in public channels. DM content is never accessed - this is not configurable and cannot be turned on.
- **Signals are not diagnoses.** Every output uses behavioral observations ("PR activity down 40% vs. 8-week baseline") not judgments ("seems disengaged"). PRIVACY.md is loaded by every command and its rules are structurally enforced.
- **Skip-level is anonymous by default.** Individual observations are aggregated into themes. Names require an explicit opt-in flag.
- **The test:** every output is designed to pass - "would I be comfortable if my direct report saw this?"

---

## Upgrading

Pull updated skill files over the existing ones. Your `.team-health/` data is unaffected.

```bash
cp -r /path/to/updated-antigravity-skills/.claude/commands/team-health/ .claude/commands/team-health/
cp -r /path/to/updated-antigravity-skills/.claude/team-health/ .claude/team-health/
```

## Uninstalling

```bash
rm -rf .claude/commands/team-health/ .claude/team-health/

# Optionally remove all local people data:
rm -rf .team-health/
```

---

## Typical weekly workflow

```
New team member joins:
  /team-health:onboard alice      ← guided context capture (career goals, strengths, style)

Friday morning (automated):
  /team-health:digest             ← pulse + Slack summary (schedule this!)

Monday morning (if not using digest):
  /team-health:pulse              ← scan the whole team, update baselines

Any time — quick lookup:
  /team-health:summary alice      ← read-only profile (no side effects)

Before each 1:1 (~30 seconds):
  /team-health:prep alice         ← prep sheet with coaching hints since last 1:1

After 1:1 (30 seconds):
  /team-health:log alice          ← log commitments, feedback, or notes

Before skip-level:
  /team-health:skip-level         ← team brief for your manager

Before sprint retro:
  /team-health:retro-prep         ← seed the agenda with real data
```
