# Phase 5: Skip-Level and Retro Prep - Research

**Researched:** 2026-03-17
**Domain:** Claude Code command authoring, pulse history aggregation, privacy gate design, sprint retro agenda generation from structured data
**Confidence:** HIGH - all foundations locked in prior phases; Phase 5 is entirely a command authoring and output-format design exercise

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SKIP-01 | `/team-health:skip-level` generates upward-facing brief: team delivery status, risks, people themes (not individuals), escalation asks, team wins | Pulse history files (`pulse-history/YYYY-WNN.md`) provide the raw team-level data; aggregation rules defined in this research |
| SKIP-02 | Default timeframe is "since last skip-level" (tracked in config); supports `--weeks 2` / `--quarter` overrides | `config.json` already has `last_skip_level` field (null by default per state-schemas.json); command reads and updates it |
| SKIP-03 | People log content is explicitly excluded from skip-level output by default - does not read `.team-health/people/` without `--include-person <name>` flag | Hard gate: command instructions explicitly forbid reading people files unless flag present; same pattern as Slack DM gate in PRIVACY.md |
| SKIP-04 | Individual pulse flags aggregated into themes: "two team members showing engagement indicators worth monitoring" - never named without explicit EM opt-in | Aggregation language pattern defined in this research; PRIVACY.md Rules 1-4 apply |
| SKIP-05 | Output passes "would I be comfortable if my entire team saw this?" test by design | Structural constraint enforced in command instructions; explicit compliance check step before output |
| RETRO-01 | `/team-health:retro-prep` analyzes most recent sprint and produces retro agenda seeded with real data | Jira MCP queries scoped to sprint; GitHub MCP for PR patterns; pulse history for signal trends |
| RETRO-02 | Output includes: sprint facts (velocity, carry-over), what went well (seeded), what was hard (seeded), patterns across sprints | Multi-section output format defined in this research |
| RETRO-03 | Discussion seeds attributed to work items, not individuals: "the auth service PR went through 7 revision cycles" not "Jordan had slow reviews" | Work-item attribution pattern; explicit prohibition on individual attribution |
| RETRO-04 | Output preserves blank space for team conversation - data seeds the agenda, does not replace it | Structural output convention: every seeded item followed by blank discussion space |
</phase_requirements>

---

## Summary

Phase 5 delivers two commands: `/team-health:skip-level` and `/team-health:retro-prep`. Both are consumers of existing state - they read pulse history snapshots, MCP data, and config but write no new state files (except updating `last_skip_level` in config.json after a skip-level run). All infrastructure was established in Phases 1-4.

The critical design challenge in this phase is not data access but privacy gate enforcement. The skip-level command must be structurally incapable of surfacing individual person details unless the manager explicitly opts in with `--include-person`. This means the command's instruction set must treat people files as off-limits by default and aggregate all pulse-derived content at the team level only. The "would my entire team be comfortable seeing this?" test is a structural constraint, not a post-hoc check.

The retro-prep command faces a parallel constraint: discussion seeds must be attributed to work items (PRs, tickets) never to individuals. The output format must include literal blank space for the team to fill in during the retro itself - Claude provides the bones of the agenda, not the full agenda.

Both commands follow established Phase 3/4 structural patterns: Pre-flight Check, Required Reference Reads, phased execution (data collection → output → optional state write), `disable-model-invocation: true`, and graceful degradation when MCP sources are absent.

**Primary recommendation:** Two command files - `skip-level.md` and `retro-prep.md` in `.claude/commands/team-health/`. Both are read-heavy consumers of pulse history and MCP signals. Neither updates baselines. `skip-level.md` updates `config.json` to record `last_skip_level` date after output is rendered.

---

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| `.claude/commands/team-health/skip-level.md` | n/a | Slash command for `/team-health:skip-level` | Established Phase 1 colon-namespace pattern in `.claude/commands/team-health/` |
| `.claude/commands/team-health/retro-prep.md` | n/a | Slash command for `/team-health:retro-prep` | Same pattern |
| `.claude/team-health/PRIVACY.md` | n/a | Output language rules - Rules 1-4, graceful degradation | Required Reference Read for both commands; PRIVACY.md already anticipates these commands (header lists skip-level and retro-prep) |
| `.team-health/pulse-history/YYYY-WNN.md` | n/a | Per-week team snapshots written by `/team-health:pulse` | Primary data source for skip-level aggregation; established in Phase 3 |
| `config.json` field `last_skip_level` | n/a | Tracks "since last skip-level" anchor date | Already in state-schemas.json (locked Phase 1): `"last_skip_level": null` |
| `Read` tool | n/a | Load config, pulse history files, baselines, reference docs | Pre-approved in `allowed-tools`; main read mechanism |
| `Write` tool | n/a | Update `config.json` with `last_skip_level` after skip-level run | Skip-level only; retro-prep makes no state writes |
| `Bash(date +%Y-%m-%d)` | n/a | Inject today's date for config.json update | Date injection pattern established in Phase 1 |
| `Bash(date +%Y-W%V)` | n/a | Identify current ISO week and find pulse history files | Established in Phase 3 |
| `Bash(ls .team-health/pulse-history/)` | n/a | Enumerate available pulse history files | Required to determine which weeks fall in the lookback window |

### Supporting

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| GitHub MCP | varies | Query PR-level data for retro-prep (PR cycle counts, review patterns) | retro-prep when `sources.github` is true |
| Jira MCP | varies | Query sprint velocity, carry-over tickets for retro-prep | retro-prep when `sources.jira` is true |
| `disable-model-invocation: true` | n/a | Prevent auto-execution | Required for all commands; established pattern |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Reading pulse-history/ .md files for skip-level data | Querying MCPs live for skip-level | Pulse history files are already aggregated summaries scoped to the team - they avoid re-querying all MCPs and respect the privacy gate naturally (no individual detail sections unless explicitly in the snapshot). Use pulse history for skip-level. |
| Listing pulse-history/ with Bash | Hardcoding week range math | Bash `ls` is already in the `allowed-tools` pattern; week range computation is error-prone; listing files and filtering by date is more robust. |
| Separate "include-person" command flag | Always anonymizing | SKIP-03 is a hard requirement - people log content must be off by default. A flag is the right mechanism; it forces explicit EM intent. |

**No new dependencies.** Both commands use only existing Claude tools and MCP sources already configured.

---

## Architecture Patterns

### Recommended Project Structure

```
.claude/
  commands/
    team-health/
      setup.md          # Phase 1
      log.md            # Phase 2
      pulse.md          # Phase 3
      prep.md           # Phase 4
      skip-level.md     # Phase 5 - NEW
      retro-prep.md     # Phase 5 - NEW

  team-health/
    SIGNALS.md          # Read by skip-level + retro-prep
    PRIVACY.md          # Read by skip-level + retro-prep (mandatory)
    BASELINES.md        # Read by retro-prep for context (optional)

.team-health/           # Per-manager state (gitignored)
  config.json           # last_skip_level field read + written by skip-level
  baselines.json        # Read by retro-prep for signal trend context
  pulse-history/
    2026-W08.md         # Read by skip-level to aggregate team themes
    2026-W09.md
    2026-W10.md
    2026-W11.md         # Written by pulse.md (Phase 3 deliverable)
  people/
    alice-chen.json     # NEVER read by skip-level or retro-prep by default
```

### Pattern 1: Command File Front Matter

Both commands follow the established front matter pattern from Phases 3 and 4.

```yaml
---
description: "Generate an upward-facing team brief for your skip-level meeting. Aggregates team pulse data; excludes individual people log content by default."
argument-hint: "[--weeks N | --quarter] [--include-person <name>]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(ls .team-health/pulse-history/)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`
```

```yaml
---
description: "Generate a sprint retro agenda seeded with sprint facts: velocity, carry-over, PR patterns. Attributed to work items, never individuals."
argument-hint: "[--sprint <sprint-id>]"
allowed-tools: Read, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(ls .team-health/pulse-history/)
disable-model-invocation: true
---
```

Note: `retro-prep.md` does not need the `Write` tool - it produces output only, makes no state writes.

### Pattern 2: Pre-flight Check and Argument Parsing (established boilerplate)

Both commands copy the Pre-flight Check verbatim from existing commands:

```
## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop.
3. If setup_complete is true, continue.
```

After the Pre-flight Check, both commands parse `$ARGUMENTS` for flags before proceeding.

### Pattern 3: Skip-Level Lookback Window (SKIP-02)

```
## Lookback Window Derivation (skip-level only)

Parse $ARGUMENTS:
- If "--quarter" present: lookback_start = first day of current calendar quarter
- If "--weeks N" present (N is a positive integer): lookback_start = today - (N * 7 days)
- Otherwise (default): read config.json.last_skip_level
  - If last_skip_level is a valid YYYY-MM-DD date: lookback_start = last_skip_level
  - If last_skip_level is null: lookback_start = today - 14 days (2-week default)
  - State which source determined the window: "Covering [N] weeks since [date] ([source])"
```

The `last_skip_level` field is already in the locked config.json schema (`state-schemas.json`, Phase 1). No schema change needed.

### Pattern 4: Pulse History File Discovery and Filtering

```
## Pulse History Discovery

1. Run: Bash(ls .team-health/pulse-history/)
   - If directory listing fails or returns empty: note "No pulse history found for this period."
     Proceed with MCP data only (if available). If no MCPs and no history: state limitation and stop.
2. Parse the listing - each file is named YYYY-WNN.md (e.g., 2026-W11.md)
3. Filter to files where the week falls within [lookback_start, today]:
   - Convert lookback_start to ISO week using the injected date
   - Keep only files whose YYYY-WNN falls at or after lookback_start's ISO week
4. Read each qualifying pulse history file using the Read tool
5. Store contents in memory for aggregation
```

### Pattern 5: Privacy Gate - People Log Exclusion (SKIP-03, SKIP-04)

This is the most important architectural constraint in Phase 5. The command instructions must explicitly prohibit reading people files by default.

```
## Privacy Gate - People Log

CRITICAL: Do NOT read any files in .team-health/people/ during this command unless the
--include-person <name> flag is explicitly present in $ARGUMENTS.

Default behavior (no flag):
- Do not reference any individual team member by name in the output
- Do not include any content from people log files
- Aggregate all pulse signals at team level: "N of M team members showed [signal] this period"

With --include-person <name>:
- Resolve <name> to a slug using the team array in config.json (same fuzzy match as log.md)
- Read that person's people log file only
- Include that person's log themes in the People section, clearly labeled as opt-in content
- All other individuals remain anonymized
```

The PRIVACY.md document is already configured for this - its four rules apply to all output. The skip-level command must enforce Rule 3 (No Team-Relative Scoring) and ensure it also never names an individual unless explicitly opted in.

### Pattern 6: Skip-Level Output Format (SKIP-01, SKIP-04, SKIP-05)

The output is an upward-facing brief - designed to be shared with the manager's own manager. Five sections:

```markdown
# Team Brief - [lookback period]

*Covering: [lookback_start] to [today] ([N weeks])*
*Sources: [active pulse history weeks] | MCP: [active sources]*

---

## 1. Delivery Status

[Team-level velocity: sprint facts, PRs merged, tickets closed - aggregated across team, not per person]
[E.g., "The team merged 23 PRs and closed 41 tickets across 3 sprints. Velocity was stable in weeks 1-2, dropped ~20% in week 3."]

---

## 2. Risks and Watch Areas

[Team-level aggregate flags from pulse history]
[If N people showed signals: "N team members showed [signal type] signals during this period - worth monitoring as a group trend."]
[Never: "Jordan showed..." unless --include-person Jordan was passed]
[If no flags: "No team-level risk signals in this period."]

---

## 3. People Themes

[Patterns derived from pulse flags - anonymized]
[E.g., "Two team members had elevated meeting load for 2+ consecutive weeks. One team member has had blocked tickets for >1 sprint."]
[With --include-person: add a labeled section for that person's log themes]

[Prohibited: individual names, people log content, psychological labels, team-relative scoring]

---

## 4. Asks / Escalations

[Items requiring the manager's manager's attention]
[These come from the manager typing them, not from signals - Claude should prompt: "Any specific asks or escalation items to include?"]
[If $ARGUMENTS contains no ask text: "No escalations listed - consider adding before sharing."]

---

## 5. Team Wins

[Positive delivery signals from pulse history - PRs merged, sprints completed, notable velocity]
[Frame positively and factually: "Team shipped the auth service refactor (22 PRs merged in W09)"]

---

*This brief reflects behavioral data signals. People log content is excluded by default.*
```

**"Would my team be comfortable seeing this?" compliance check:** After generating all five sections, before finalizing output, the command must run an internal compliance check: scan for individual names not covered by --include-person, scan for diagnostic language (PRIVACY.md Rule 1), scan for team-relative scoring (PRIVACY.md Rule 3). If any violation found: rewrite the offending sentence before output.

### Pattern 7: State Write - Update last_skip_level (SKIP-02)

After output is fully rendered:

```
## Phase E - State Write (skip-level only)

Execute ONLY after output is complete.

1. Take the config.json already in memory.
2. Set config.last_skip_level = today's date (YYYY-MM-DD from injected date)
3. Set config.last_updated = today's date
4. Write the COMPLETE updated config.json using the Write tool.
   Do NOT truncate or omit any fields. Write the full object.
5. Confirm: "Brief generated. Next skip-level will cover from today forward."
```

This follows the same output-before-state-write discipline established in Phases 2-4.

### Pattern 8: Retro-Prep Output Format (RETRO-01 through RETRO-04)

The retro-prep command generates a meeting agenda, not a report. The key constraint is RETRO-03 (work-item attribution, not individual attribution) and RETRO-04 (blank space preserved for team conversation).

```markdown
# Sprint Retro Agenda - [Sprint N] / [date range]

*Data sources: [active Jira/GitHub MCPs and pulse history weeks used]*
*Note: All discussion seeds are attributed to work items, not individuals.*

---

## Sprint Facts

**Velocity:** [N tickets closed] / [M in progress at start] - [carry-over count] carried over
**PRs:** [N merged], [M reverted or hotfixed], avg review cycle time [X days]
**Blockers resolved:** [N] / [M] remaining open at sprint end

---

## What Went Well - Seeds

*Team discusses. Add your own.*

- [Factual positive: "Auth service refactor shipped - N PRs merged cleanly, avg review lag [X days]"]
- [Positive delivery signal: "Sprint velocity was [X]% above the 8-week team baseline"]
- [Optional: if pulse history shows fewer flags this sprint than prior - "Fewer team signal flags vs. previous 2 sprints"]

**[ Team discussion space - what else went well? ]**

---

## What Was Hard - Seeds

*Team discusses. Add your own.*

- [Work-item attribution: "The [ticket/PR title] took 3 sprint cycles - what blocked it?"]
- [Process observation: "7 PRs had >3 revision cycles this sprint - what patterns do we see?"]
- [Blocker fact: "[N] tickets were blocked for >1 sprint. What caused the blockage? What would unblock faster?"]

**[ Team discussion space - what else was hard? ]**

---

## Patterns Across Sprints

*[Only populated if 2+ weeks of pulse history in the lookback window]*

- [Trend: "PR review lag has been above baseline for [N] consecutive sprints - worth discussing as a team process"]
- [Velocity trend: "Team velocity has been [stable/trending up/trending down] for [N] sprints"]

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

**Key structural rules for retro-prep:**
- Every data-seeded item is a question or observation about a work item, never a statement about a person
- Every section has a blank discussion space labeled for the team
- Action items section is entirely blank - Claude does not pre-fill actions
- No disclaimer required (no individual health signals; retro-prep is a process tool)

### Pattern 9: Retro-Prep MCP Data Collection

Unlike pulse (which queries all 11 signals per person), retro-prep queries at the sprint level - aggregated, work-item-focused.

```
## Phase B - Sprint Data Collection (retro-prep)

Determine sprint window:
- If $ARGUMENTS contains "--sprint <id>": use Jira MCP to look up that sprint's dates
- Otherwise: determine most recent completed sprint from Jira (if sources.jira is true)
  OR use the most recent 2-week window from pulse history (if Jira unavailable)

GitHub queries (if sources.github is true):
- PRs merged in sprint window - count and list titles (no author names in the agenda output)
- PRs with >3 revision cycles (high comment/review count before merge) - list titles
- PR review cycle time distribution - avg, max (no per-person breakdowns)
- Reverted or hotfixed PRs in sprint window - list titles

Jira queries (if sources.jira is true):
- Sprint velocity: tickets closed vs. planned
- Carry-over count: tickets started in this sprint but not closed
- Blocked tickets at sprint end: count and ticket summaries (not assignees)
- Tickets in progress >2 sprints: count

Pulse history (always, if files exist):
- Read pulse-history files for weeks within the sprint window
- Extract team-level flag summary from each week: "Week X: N flags"
- Note trend: improving, stable, or degrading signal health

Do NOT query per-person breakdowns for retro-prep. All queries are team-scoped or work-item-scoped.
```

### Anti-Patterns to Avoid

- **Reading people files by default in skip-level:** The privacy gate in SKIP-03 is a hard structural constraint. The command instructions must say explicitly: "Do NOT read .team-health/people/ unless --include-person is present."
- **Naming individuals in skip-level aggregations:** "Two team members" not "Jordan and Alex" - unless --include-person was passed for those individuals.
- **Attributing retro observations to people:** "The auth service PR went through 7 revision cycles" - correct. "Jordan's PRs had slow reviews" - prohibited.
- **Filling in the action items section:** Retro-prep generates the agenda seeds; the team generates the actions. The Action Items section must be structurally blank.
- **Updating baselines.json in skip-level or retro-prep:** These commands are read-only consumers of baselines. Only pulse.md updates baselines.json.
- **Writing skip-level state before rendering output:** Same discipline as all prior commands - render output first, write `last_skip_level` to config.json after.
- **Using `context: fork`:** skip-level writes config.json. Do not fork - the write must persist in the main session.
- **Querying MCP sources not in config.json sources map:** Apply the same `sources.<name>` gate as all prior commands.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Sprint window computation | Custom date math | Jira MCP sprint lookup or 2-week default from pulse history | Sprint dates are in Jira; fallback to pulse history ISO weeks is more reliable than calendar math |
| ISO week-to-date conversion | Custom week number math | `Bash(date +%Y-W%V)` + `Bash(ls .team-health/pulse-history/)` | List files, match against ISO week prefix - simpler and correct |
| Privacy compliance check | Post-processing filter | Explicit compliance check step in command instructions (scan for individual names before output) | The command prompt is the enforcement mechanism; no code exists to write a filter |
| Velocity trend computation | Statistical library | Claude reads 4-8 pulse history files and derives trend inline | Same pattern as baseline arithmetic - fits inline for ≤8 weeks of history |
| Skip-level summary writing | Template library | Claude authors prose summary from aggregated pulse signals | This is exactly Claude's strength; templates would be lower quality |

**Key insight:** Both commands are about synthesis and output formatting, not about new data infrastructure. The pulse history files are the pre-built aggregation layer for skip-level. Jira's sprint API is the authoritative sprint-scope layer for retro-prep. Neither command needs to build new pipelines.

---

## Common Pitfalls

### Pitfall 1: Individual Names Leaking into Skip-Level Output

**What goes wrong:** Skip-level output contains "Jordan showed engagement flags in week 11" because Claude read the pulse history detail section which includes names.

**Why it happens:** Pulse history files include names in the flagged members section. The command reads those files and may inadvertently include names in the aggregation.

**How to avoid:** The skip-level command instructions must state: "When reading pulse history files, extract team-level aggregate counts only. Do NOT carry forward individual names from the 'Flagged Members' sections into the brief output. Aggregate as: 'N team members were flagged in week X.'"

**Warning signs:** SKIP-04 test fails - output contains a direct report's name without --include-person being passed.

### Pitfall 2: last_skip_level Not Updated After Run

**What goes wrong:** Manager runs skip-level twice in one week; the second run uses the same stale "since last skip-level" anchor because the state write was omitted.

**Why it happens:** Phase E (state write to config.json) was not included in the command, or the command was written without the Write tool in `allowed-tools`.

**How to avoid:** `allowed-tools` must include `Write`. Phase E is mandatory for skip-level. The command instructions must end with the config.json update step.

**Warning signs:** Running `/team-health:skip-level` twice returns the same lookback window the second time.

### Pitfall 3: Retro Seeds Attributing Observations to People

**What goes wrong:** Retro agenda says "Alice's PRs averaged 4 revision cycles" instead of "PRs in the auth-service area averaged 4 revision cycles."

**Why it happens:** The retro-prep command collected per-person PR data and included the person's name when surfacing the finding.

**How to avoid:** The retro-prep command instructions must state: "All observations in What Was Hard and What Went Well sections must be attributed to work items (PRs, tickets, or work areas) - not to individuals. Do not use person names in retro seeds."

**Warning signs:** RETRO-03 test fails - agenda contains a team member's name in a seeded discussion item.

### Pitfall 4: Retro Agenda Fills in Action Items

**What goes wrong:** The action items section has pre-populated items like "Alice should improve PR review response time" - both naming an individual and filling in what should be team-generated.

**Why it happens:** Claude infers what actions should be taken from the data and fills them in automatically.

**How to avoid:** The retro-prep command instructions must explicitly state: "The Action Items section must be left blank (3-5 blank checkbox rows only). The team fills in actions during the retro. Do not pre-populate action items."

**Warning signs:** RETRO-04 test fails - action items section has content instead of blank checkboxes.

### Pitfall 5: No Pulse History Available

**What goes wrong:** skip-level or retro-prep is run before any pulse history exists (Phases 3-4 outputs not yet generated), and the command errors or produces empty output.

**Why it happens:** The commands depend on `pulse-history/` files that may not exist.

**How to avoid:** Both commands must handle the "no pulse history" case gracefully. Skip-level: "No pulse history found for this period. Brief based on MCP data only." Retro-prep: "No pulse history found. Retro seeds based on GitHub and Jira data only." The commands continue with available data.

**Warning signs:** Command fails with a file-not-found error on the first run after installation.

### Pitfall 6: config.json Overwritten Partially

**What goes wrong:** skip-level reads config.json, updates `last_skip_level`, but writes only that field - losing the full team array, sources map, and manager block.

**Why it happens:** Partial-write pattern: reading config.json then writing only the changed field.

**How to avoid:** Same atomic-write discipline as baselines.json in Phase 3 - build the full config.json in memory (all existing fields intact), set `last_skip_level` and `last_updated`, write the complete object in a single Write call.

**Warning signs:** After running skip-level, the config.json is missing the `team` array or `sources` map.

### Pitfall 7: Retro-Prep Without Jira Has No Sprint Scope

**What goes wrong:** When `sources.jira` is false, the retro-prep command has no sprint boundary to scope data to, and queries GitHub for "all time" or produces confusingly scoped output.

**Why it happens:** Sprint boundaries come from Jira. Without Jira, no natural sprint scope exists.

**How to avoid:** When `sources.jira` is false, the retro-prep command uses the most recent 1-2 pulse history weeks as the sprint window. State this clearly: "Jira MCP not configured - retro scope based on most recent 2 pulse history weeks (sprint boundary unavailable)."

**Warning signs:** Retro-prep output covers 8 weeks of data when sources.jira is false.

---

## Code Examples

Verified patterns from prior phase implementations:

### Command Front Matter - skip-level.md

```yaml
---
description: "Generate an upward-facing team brief for your skip-level meeting. People log content excluded by default."
argument-hint: "[--weeks N | --quarter] [--include-person <name>]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(ls .team-health/pulse-history/)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`
```

Source: Pattern from `.claude/commands/team-health/pulse.md` (Phase 3 reference implementation)

### Command Front Matter - retro-prep.md

```yaml
---
description: "Generate a sprint retro agenda seeded with sprint facts. Attributed to work items, never individuals."
argument-hint: "[--sprint <sprint-id>]"
allowed-tools: Read, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(ls .team-health/pulse-history/)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`
```

Note: No `Write` in allowed-tools - retro-prep makes no state writes.

### Pre-flight Check (verbatim - copy from existing commands)

```
## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.
```

Source: Established in Phase 1; used verbatim in all Phase 2, 3, 4 commands.

### Privacy Gate Instruction Block (skip-level.md)

```
## Privacy Gate

CRITICAL - read this before any data collection:

Do NOT read any files under .team-health/people/ during this command.
People log content is excluded from skip-level output by default.

Exception: if $ARGUMENTS contains "--include-person <name>":
  1. Resolve <name> to slug using config.json team array (fuzzy prefix match, case-insensitive)
  2. Read .team-health/people/<slug>.json
  3. Include that person's log themes in the People Themes section, labeled as "(opt-in content)"
  4. All other individuals remain unnamed in the output

Default behavior (no --include-person flag):
  - All pulse aggregations use counts only: "N team members", never individual names
  - No people log content in any section
  - This is enforced structurally - skip the Read call entirely
```

### Aggregation Language - Anonymized Pulse Flags

```
When reading pulse-history files and aggregating flags:

Input (from pulse history file):
## Flagged Members
- Alice Chen (YELLOW): PR review count down 2.1 stddev
- Bob Smith (YELLOW): commit days down 1.8 stddev

Correct aggregation output:
"2 team members showed engagement indicator signals during this period."

Prohibited output:
"Alice Chen and Bob Smith were flagged for engagement signals."

With --include-person Alice:
"2 team members showed engagement indicator signals. Alice Chen (opt-in): PR review count was down 2.1 stddev from her personal baseline."
```

### config.json Update After Skip-Level Run (atomic write pattern)

```
## Phase E - Update last_skip_level

After output is fully rendered:
1. Take config.json already in memory from Pre-flight Check.
2. Set: config.last_skip_level = [today's date from injected date]
3. Set: config.last_updated = [today's date]
4. Write the COMPLETE config.json object using the Write tool.
   Include ALL existing fields: schema_version, setup_complete, manager, team,
   sources, baseline_window_weeks, github_org, sprint_cadence_weeks, last_skip_level, last_updated.
   Do NOT omit any field. Build the full object in memory, write once.
5. Confirm: "Brief generated. Next skip-level will cover from today forward."
```

Source: Atomic write pattern from `baselines.json` update in pulse.md (Phase 3); config.json atomic write from setup.md (Phase 1).

### Pulse History File Discovery Pattern

```
## Load Pulse History

1. Run: Bash(ls .team-health/pulse-history/)
   - If error or empty: note "No pulse history available for this period."
     Continue with MCP data only if sources are active.
     If no MCPs and no history: tell the user, stop.
2. Parse the listing. Files are named: YYYY-WNN.md
3. Compare each file's YYYY-WNN to the lookback window:
   - Keep files where YYYY-WNN >= lookback_start ISO week
4. Read each kept file using the Read tool.
5. Store in memory as PULSE_HISTORY array (ordered oldest to newest).
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Skip-level update as free-form narrative | Structured five-section brief driven by pulse signal aggregates | Phase 5 design | Reproducible format; privacy-gated by default; upward-facing not peer-facing |
| Retro prep: manager draws from memory | Data-seeded retro agenda from Jira/GitHub sprint facts | Phase 5 design | Seeds are specific and work-item-attributed; blank space preserved for team |
| Individual pulse flags surfaced to manager's manager | Anonymized team-level themes ("N team members") | Phase 5 design | Prevents people signals from propagating up without EM intent |
| last_skip_level tracked manually | Stored in config.json, automatically updated after each run | Phase 5 design | "Since last skip-level" lookback works automatically |

**No deprecated patterns to migrate.** Phase 5 is a net-new capability; all prior phase patterns continue unchanged.

---

## Open Questions

1. **Should skip-level also query live MCP data, or rely entirely on pulse history?**
   - What we know: Pulse history snapshots are already team-level aggregations with provenance. They contain all signals available at pulse run time. For the lookback window, they cover weeks that have been run.
   - What's unclear: If the manager ran pulse irregularly (skipped a week), the pulse history will have gaps. Should skip-level fill gaps with live MCP queries?
   - Recommendation: Use pulse history as the primary source. For the current week (if no pulse history yet), offer to query live MCP data. State gaps explicitly: "Note: no pulse history found for weeks [X, Y] - data for those weeks is unavailable." Do not silently fill gaps.

2. **Where do the "Asks / Escalations" section contents come from in skip-level?**
   - What we know: SKIP-01 lists "asks requiring escalation" as a section in the brief. These cannot be derived from signal data - they require manager intent.
   - What's unclear: Does the command prompt the manager interactively for asks? Or does the EM pass them as $ARGUMENTS?
   - Recommendation: The command should include a placeholder section labeled "Asks / Escalations" with: "[ Add escalation items here before sharing this brief ]". The EM fills it in manually after generation. This is simpler than interactive prompting and avoids state management complexity. The planner should make this call explicit in the plan.

3. **Should retro-prep update any state after running?**
   - What we know: RETRO-01 through RETRO-04 say nothing about state writes. The retro agenda is a one-shot output.
   - What's unclear: Should the date of the last retro-prep run be stored anywhere (analogous to `last_skip_level`)?
   - Recommendation: No. Retro-prep makes no state writes. The output is a standalone document. There is no "since last retro" lookback - retro-prep always targets the most recent completed sprint.

4. **How does retro-prep determine "most recent completed sprint" without Jira?**
   - What we know: Without Jira, there is no API source for sprint boundaries. `sprint_cadence_weeks` in config.json (default: 2) defines the expected sprint length.
   - What's unclear: The current sprint start date is not stored anywhere in state.
   - Recommendation: When sources.jira is false, retro-prep uses the most recent N pulse history weeks where N = `sprint_cadence_weeks` from config.json. State this explicitly in the output: "Sprint window estimated from last [N] pulse history weeks (no Jira MCP configured)."

5. **Does SKILL.md need to be updated to reflect the two new commands?**
   - What we know: Phase 1 deliverable included SKILL.md with all 6 commands documented with phase availability labels. The Phase 5 commands were listed as "Phase 5 - not yet available."
   - What's unclear: Whether updating SKILL.md is part of Phase 5 plans or a separate task.
   - Recommendation: Include a SKILL.md update task in Phase 5 plans to flip the availability labels for skip-level and retro-prep from "Phase 5" to "Available." This is a small but important installation experience improvement.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None - pure markdown/JSON command files, no runnable code |
| Config file | n/a |
| Quick run command | Manual: invoke `/team-health:skip-level` or `/team-health:retro-prep` in Claude Code with Phase 3 pulse history in place |
| Full suite command | Manual: run all 9 SKIP/RETRO scenarios in a project with Phases 1-4 complete and pulse history populated |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SKIP-01 | Brief covers 5 sections: delivery status, risks, people themes, escalations, wins | smoke/manual | Run command; verify output has 5 section headers | ❌ Wave 0 |
| SKIP-02 | Lookback defaults to "since last skip-level"; overrides (`--weeks 2`) work | smoke/manual | Run with no flags (verify uses last_skip_level or 14-day default); run with `--weeks 4` (verify 4-week window) | ❌ Wave 0 |
| SKIP-03 | People log content not read unless --include-person passed | smoke/manual | Run without flag; verify output has zero individual names. Run with `--include-person Alice`; verify Alice's log themes appear labeled as opt-in | ❌ Wave 0 |
| SKIP-04 | Pulse flags aggregated anonymously: "N team members" not individual names | smoke/manual | Set up pulse history with 2 flagged members; run skip-level; verify no individual names in output without --include-person | ❌ Wave 0 |
| SKIP-05 | Output passes team-visibility test | smoke/manual | Read output; manually verify: no individual names (without opt-in), no diagnostic labels, no team-relative scoring | ❌ Wave 0 |
| RETRO-01 | Retro agenda generated from most recent sprint data | smoke/manual | Run command; verify output has sprint facts section with velocity and carry-over data | ❌ Wave 0 |
| RETRO-02 | Output has sprint facts, what went well, what was hard, cross-sprint patterns | smoke/manual | Run command; verify all 4 content sections present | ❌ Wave 0 |
| RETRO-03 | Seeds attributed to work items not individuals | smoke/manual | Inspect "What Was Hard" section; verify no team member names in seeded items | ❌ Wave 0 |
| RETRO-04 | Blank discussion space in every section; action items blank | smoke/manual | Verify each section has a visible "Team discussion space" marker; verify Action Items section has only blank checkboxes | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** Verify file exists, correct front matter, required sections present in output format
- **Per wave merge:** Run all 9 scenarios in a fresh project with Phase 3 pulse history populated
- **Phase gate:** All 9 scenarios pass before marking Phase 5 complete

### Wave 0 Gaps

- [ ] `docs/testing/phase-5-smoke-tests.md` - manual test procedure for all 9 SKIP/RETRO scenarios above (Wave 0 plan creates this first, same as Phases 3 and 4)
- [ ] `.claude/commands/team-health/skip-level.md` - primary deliverable (Wave 1 plan)
- [ ] `.claude/commands/team-health/retro-prep.md` - primary deliverable (Wave 1 plan)
- [ ] Verify `Bash(ls .team-health/pulse-history/)` is pre-approvable in `allowed-tools` (same pattern as `Bash(mkdir -p .team-health/pulse-history)` in pulse.md - should work)
- [ ] Update `SKILL.md` - flip phase-5 commands from "Phase 5 - not yet available" to "Available" (small task, can be bundled with Wave 1)

*(No automated test framework needed - no runnable code; all tests are manual)*

---

## Sources

### Primary (HIGH confidence)

- `.planning/phases/01-foundation/state-schemas.json` - confirms `last_skip_level: null` field already in locked config.json schema; no schema change needed for SKIP-02
- `.claude/team-health/PRIVACY.md` - Rules 1-4 fully apply to skip-level and retro-prep; document header explicitly lists these commands as consumers
- `.claude/team-health/SIGNALS.md` - signal definitions and two-signal rule; referenced by both commands
- `.claude/commands/team-health/pulse.md` - pulse history snapshot format locked; Phase 5 commands consume this format
- `.planning/phases/03-team-pulse/03-RESEARCH.md` - pulse history format, atomic write patterns, probe-by-namespace MCP pattern
- `.planning/phases/04-1-1-prep/04-RESEARCH.md` - Phase A→E structure, people log integration patterns, privacy gate patterns
- `.planning/REQUIREMENTS.md` - SKIP-01 through SKIP-05, RETRO-01 through RETRO-04 requirement text (authoritative)
- `.planning/ROADMAP.md` - Phase 5 success criteria (confirmed four conditions)

### Secondary (MEDIUM confidence)

- STATE.md decisions - confirms privacy-first principle and skip-level design intent; confirms skip-level excludes people log by default
- PROJECT.md constraints - "skip-level output excludes people log content unless explicitly chosen"; "would I be comfortable if my direct report saw this?" test
- ROADMAP.md Phase 5 description - "explicit privacy gates that prevent people log content from surfacing without opt-in"

### Tertiary (LOW confidence - verify at implementation)

- MCP tool names for Jira sprint queries - community implementations vary; probe by namespace per Phase 1 pattern; exact sprint-query API may differ
- `Bash(ls .team-health/pulse-history/)` pre-approval in allowed-tools - same general pattern as `Bash(mkdir -p ...)` in pulse.md; should work but verify in Wave 0

---

## Metadata

**Confidence breakdown:**
- Command file format and front matter: HIGH - direct from Phase 1/3/4 implementations, unchanged pattern
- State schemas (config.json last_skip_level field): HIGH - locked in state-schemas.json Phase 1; no new schema needed
- Privacy gate design: HIGH - requirements are explicit; PRIVACY.md + REQUIREMENTS.md are authoritative sources
- Skip-level output format: HIGH-MEDIUM - five-section structure derived directly from SKIP-01 requirements text; section names and content rules are logical
- Retro-prep output format: HIGH-MEDIUM - four-section structure derived from RETRO-01 through RETRO-04; blank-space convention is explicit in RETRO-04
- MCP tool names for sprint queries: MEDIUM - Jira MCP community packages vary; probe by namespace handles this
- Pulse history file discovery via `ls`: MEDIUM - pattern works on macOS/bash; same considerations as `date +%Y-W%V` from Phase 3

**Research date:** 2026-03-17
**Valid until:** 2026-06-17 (stable foundations; locked schemas and reference docs will not change without schema_version bump; MCP landscape may evolve - re-verify Jira sprint query tooling at implementation)
