# Architecture Patterns

**Domain:** Claude Code skill with multi-source MCP integration and persistent longitudinal state
**Researched:** 2026-03-09
**Confidence:** MEDIUM — Based on Claude Code documentation knowledge (training cutoff Aug 2025) and established patterns for MCP-integrated skills. External verification was blocked; flag for validation before phase build.

---

## Recommended Architecture

```
.claude/
  commands/
    team-health/
      prep.md          ← /team-health:prep <name>
      pulse.md         ← /team-health:pulse
      log.md           ← /team-health:log <name>
      skip-level.md    ← /team-health:skip-level
      retro-prep.md    ← /team-health:retro-prep

  team-health/          ← Reference docs (read by skills at runtime)
    SETUP.md            ← First-run onboarding guide
    SIGNALS.md          ← Signal taxonomy, scoring weights
    BASELINES.md        ← How to compute rolling stats inline
    PRIVACY.md          ← What to include/exclude per output type

.team-health/           ← Per-manager state (lives in user's project)
  config.json           ← Team roster, cadences, source prefs, setup flag
  people/
    <name>.json         ← Per-person structured people log
  baselines.json        ← 8-week rolling window snapshots per person per metric
  pulse-history/
    YYYY-MM-DD.json     ← Weekly pulse run outputs
  retro-history/
    YYYY-MM-DD.json     ← Retro prep run outputs
```

---

## Component Boundaries

### Layer 1: Skill Entry Points (`.claude/commands/team-health/*.md`)

These are the prompt files Claude Code loads when a slash command is invoked. Each file is a complete instruction set for one command — it defines what data to gather, how to compute, what to output, and what to write back.

| Component | Responsibility | Reads | Writes |
|-----------|---------------|-------|--------|
| `prep.md` | Build 1:1 prep sheet for one person | GitHub MCP, Jira MCP, Slack MCP, Calendar MCP, `people/<name>.json`, `baselines.json`, `SIGNALS.md` | Nothing (read-only analysis) |
| `pulse.md` | Build weekly team health dashboard | GitHub MCP, Jira MCP, Slack MCP, `baselines.json`, `config.json`, `SIGNALS.md`, `BASELINES.md` | `pulse-history/YYYY-MM-DD.json`, `baselines.json` (update rolling window) |
| `log.md` | Append to or query a person's log | `people/<name>.json` | `people/<name>.json` |
| `skip-level.md` | Build upward-facing team brief | GitHub MCP, Jira MCP, `pulse-history/` (last N), `config.json` | Nothing (read-only analysis, no people log) |
| `retro-prep.md` | Seed sprint retro agenda with real data | GitHub MCP, Jira MCP, `pulse-history/` (current sprint), `config.json` | `retro-history/YYYY-MM-DD.json` |

**Key rule:** Each skill file is self-contained. It includes the full instruction prompt — it does not `include` or `import` other skill files. Shared logic (signal taxonomy, baseline math) lives in reference docs the skill is instructed to read.

### Layer 2: Reference Documents (`.claude/team-health/*.md`)

Static markdown files that encode domain logic. Skills are instructed to read specific reference docs at the start of their run. They do not execute — they are knowledge.

| Component | Responsibility | Read By |
|-----------|---------------|---------|
| `SETUP.md` | First-run walkthrough: what to collect, how to write config.json, how to explain gaps | All skills (first-run check branch) |
| `SIGNALS.md` | Taxonomy of signals per source, scoring weights, what constitutes an "aligned flag" | `prep.md`, `pulse.md` |
| `BASELINES.md` | How to read `baselines.json`, compute rolling mean + std dev, update the window | `pulse.md` (writes), `prep.md` (reads for comparison) |
| `PRIVACY.md` | What to include vs. exclude per output type; skip-level exclusions; tone guide | All skills (output generation phase) |

### Layer 3: Persistent State (`.team-health/`)

Written and read by skills at runtime. This directory lives in the manager's project (not in the skill install), so people data stays with the manager.

| File | Schema Summary | Owner (Primary Writer) |
|------|---------------|----------------------|
| `config.json` | `{ team: [{name, role, start_date, 1on1_cadence}], sources: {github: bool, jira: bool, slack: bool, calendar: bool}, setup_complete: bool }` | `SETUP.md` first-run flow |
| `people/<name>.json` | `{ log: [{date, type, content, tags}], notes: string }` | `log.md` |
| `baselines.json` | `{ <name>: { <metric>: { window: [values], mean: float, stddev: float, last_updated: date } } }` | `pulse.md` |
| `pulse-history/YYYY-MM-DD.json` | Structured pulse output: per-person scores, flags, trend direction | `pulse.md` |
| `retro-history/YYYY-MM-DD.json` | Sprint retro agenda with seeded data items | `retro-prep.md` |

### Layer 4: MCP Data Sources (External)

Skills call these via MCP tool invocations. They are not owned by the skill — they are queried at runtime.

| Source | What It Provides | Degradation Level |
|--------|-----------------|-------------------|
| GitHub MCP | PR activity, review turnaround, commit patterns, issue ownership | Core — usable solo |
| Jira MCP | Ticket velocity, blocked items, sprint completion, scope changes | Tier 2 — improves scoring |
| Slack MCP | Participation metadata (no DM content), response lag, channel activity | Tier 2 — sentiment proxy |
| Calendar MCP | 1:1 adherence, meeting load, focus time trends | Tier 3 — context enrichment |

---

## Data Flow

### `/team-health:prep <name>`

```
Invocation
  → Read config.json (check setup_complete; if false → branch to SETUP.md flow)
  → Read SIGNALS.md, PRIVACY.md
  → Read people/<name>.json (people log context)
  → Read baselines.json (personal baseline for <name>)
  → Query GitHub MCP: PR/commit activity last 14 days for <name>
  → Query Jira MCP (if available): ticket status for <name>
  → Query Slack MCP (if available): activity metadata for <name>
  → Query Calendar MCP (if available): upcoming 1:1, recent meeting load
  → Claude: compute signal scores vs. personal baselines inline
  → Claude: synthesize 1:1 prep sheet (talking points, flags, follow-ups)
  → Output: Markdown prep sheet to stdout (no file write)
```

### `/team-health:pulse`

```
Invocation
  → Read config.json
  → Read SIGNALS.md, BASELINES.md, PRIVACY.md
  → Read baselines.json (all team members)
  → For each team member:
      → Query GitHub MCP: activity metrics
      → Query Jira MCP (if available): velocity/block metrics
      → Query Slack MCP (if available): participation metrics
      → Claude: compute scores vs. personal 8-week baseline inline
      → Claude: flag outliers (>2 std devs) and trend directions
  → Claude: synthesize team dashboard + ranked concerns
  → Write pulse-history/YYYY-MM-DD.json
  → Update baselines.json (add this week's values, drop oldest if >8 weeks)
  → Output: Markdown dashboard to stdout
```

### `/team-health:log <name>`

```
Invocation + argument parsing (<name>, optional --query or --append)
  → Read people/<name>.json (or create if not exists)
  → If --append: Claude prompts for structured entry (type, content, tags)
  → If --query: Claude queries log entries and summarizes
  → Write people/<name>.json with new entry appended
  → Output: Confirmation or query result to stdout
```

### `/team-health:skip-level`

```
Invocation
  → Read config.json, PRIVACY.md
  → Read pulse-history/ (last 4 weeks)
  → Query GitHub MCP: team-level PR/delivery metrics
  → Query Jira MCP (if available): sprint delivery and blocked items
  → Claude: synthesize upward brief (team health, delivery trajectory, risks)
  → ENFORCE: no people log content, no individual flags by name unless positive
  → Output: Markdown brief to stdout (no file write)
```

### `/team-health:retro-prep`

```
Invocation
  → Read config.json
  → Read pulse-history/ (entries since last retro-history/ entry)
  → Query GitHub MCP: PR cycle time, review bottlenecks, incident markers in sprint
  → Query Jira MCP (if available): scope creep, blockers, velocity deltas
  → Claude: synthesize retro agenda items seeded with real data
  → Write retro-history/YYYY-MM-DD.json
  → Output: Markdown retro agenda to stdout
```

---

## Patterns to Follow

### Pattern 1: Setup Gate

**What:** Every skill checks `config.json: setup_complete` at the top of its prompt. If false, it branches to the setup flow before doing anything else.

**When:** First invocation by any new manager. Also when a new team member needs to be added.

**Implementation in skill prompt:**
```
Step 0: Read .team-health/config.json.
If setup_complete is false or the file doesn't exist,
follow the setup flow in .claude/team-health/SETUP.md
and create the config before proceeding.
```

**Why:** Without team roster and cadence data, all downstream queries are untargeted. Setup gate ensures the skill is always operating with known context.

### Pattern 2: MCP Availability Probe

**What:** Before querying any MCP source, the skill instruction tests for tool availability and communicates gaps clearly.

**When:** Every run. MCP connections can change between sessions.

**Implementation approach:**

The skill prompt instructs Claude to attempt a minimal probe call per source (e.g., list repos for GitHub, list projects for Jira). If the tool call fails or the tool is not listed, Claude notes it as unavailable and adjusts output accordingly.

```
Data gathering:
- GitHub: list repos via mcp_github_list_repos. If unavailable, note:
  "GitHub MCP not connected — PR/commit signals omitted."
- Jira: list projects via mcp_jira_list_projects. If unavailable, note:
  "Jira MCP not connected — velocity/blocker signals omitted."
- Slack: test connection via mcp_slack_list_channels. If unavailable, note:
  "Slack MCP not connected — participation signals omitted."
- Calendar: test via mcp_calendar_list_events. If unavailable, note:
  "Calendar MCP not connected — meeting load signals omitted."
```

**Degradation tiers:**

| Available Sources | Capability |
|-------------------|-----------|
| GitHub only | Delivery/contribution signals, no soft signals |
| GitHub + Jira | Full delivery picture, no soft signals |
| GitHub + Jira + Slack | Delivery + early soft signal detection |
| All four | Full capability including 1:1 adherence and meeting load |

**Confidence floor:** With only GitHub, `/pulse` should note that baseline scores are delivery-only and soft signal categories are unavailable. Do not fabricate scores for missing sources.

### Pattern 3: Inline Baseline Computation

**What:** Claude reads the raw rolling window from `baselines.json` and computes mean, standard deviation, and z-score inline during a pulse run. No external scripts.

**When:** Every `/team-health:pulse` run.

**Implementation — BASELINES.md instructs Claude to:**

1. Read `baselines.json` for each person.
2. For each metric (e.g., `pr_merge_rate`, `review_turnaround_hours`), extract the `window` array (up to 8 weekly values).
3. Compute rolling mean: `sum(window) / len(window)`.
4. Compute std dev: `sqrt(sum((x - mean)^2 for x in window) / len(window))`.
5. Compute z-score for this week's value: `(current - mean) / stddev` (handle stddev = 0 gracefully — flag as "insufficient history" not as an alert).
6. Flag if `|z| > 2.0` with direction (high/low) and metric name.
7. After all scoring, append current week's values to each person's window arrays and drop oldest entry if `len(window) > 8`.
8. Write updated `baselines.json`.

**Confidence note:** Claude's inline arithmetic is reliable for simple statistics over small arrays (8 values per metric per person). This is a deliberate constraint — keep metric counts per person under ~10 to avoid compounding arithmetic errors in long sessions.

### Pattern 4: Privacy Enforcement Layer

**What:** `PRIVACY.md` defines a mandatory output filter. Every skill is instructed to apply it before generating final output.

**When:** All output generation phases.

**Rules encoded in PRIVACY.md:**

- `/skip-level`: Never include individual people log entries. Never name individuals with negative flags. Aggregate only.
- All outputs: No DM content from Slack, ever.
- All outputs: Phrase as signals not diagnoses ("PR review rate dropped 40% vs. baseline" not "seems disengaged").
- Multi-signal threshold: Do not flag a person unless 2+ aligned signals OR 1 signal at >2 std devs from their personal baseline.
- Tone test: "Would I be comfortable if this direct report saw this?" before including any individual observation.

### Pattern 5: Reference Doc Injection

**What:** Skill prompts explicitly instruct Claude to read specific reference docs at the start of execution. This is how shared logic is reused without skill files calling each other.

**Why not inline the logic in each skill file:** Reference docs can be updated independently without touching all skill files. SIGNALS.md scoring weights can evolve. BASELINES.md math instructions can be refined. Skills inherit the update automatically.

**Implementation:** Each skill prompt begins with a "Read these files" block:
```
Before doing anything else, read:
- .claude/team-health/SIGNALS.md
- .claude/team-health/BASELINES.md (if this skill writes baselines)
- .claude/team-health/PRIVACY.md
- .team-health/config.json
```

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Shared State Mutation Without Schema

**What:** Multiple skills write to the same state files without a consistent schema.

**Why bad:** `baselines.json` evolves as new metrics are added. If `pulse.md` writes a different schema than `prep.md` reads, state becomes corrupted across runs.

**Instead:** Define the canonical schema for each state file in `BASELINES.md` / reference docs. All write operations follow the schema. When the schema changes, a migration note goes into the reference doc.

### Anti-Pattern 2: Fabricating Scores for Missing Sources

**What:** Generating a score for a signal category when the MCP providing it is not connected.

**Why bad:** False precision erodes trust. A manager seeing "Slack participation: 7/10" when Slack is not connected will make decisions on made-up data.

**Instead:** Explicitly mark categories as "unavailable — [Source] MCP not connected." Omit them from rankings and trend lines.

### Anti-Pattern 3: Putting Shared Logic Inline in Each Skill

**What:** Copy-pasting the signal scoring rubric or baseline math into each `.claude/commands/team-health/*.md` file.

**Why bad:** Changes require touching five files. Skills diverge over time as each gets partial updates.

**Instead:** Logic lives in reference docs (`.claude/team-health/`). Skills call out to them with explicit read instructions.

### Anti-Pattern 4: People Log Surfacing in Skip-Level

**What:** Including observations from `people/<name>.json` in the `/skip-level` output.

**Why bad:** People logs contain the manager's private assessments of individuals. Skip-level output goes upward — surfacing individual assessments violates the trust model.

**Instead:** `skip-level.md` is explicitly forbidden from reading `people/` directory. It reads only `pulse-history/` (aggregates) and MCP delivery data.

### Anti-Pattern 5: Single Metric Flagging

**What:** Flagging a person as a concern because one metric moved.

**Why bad:** Week-to-week variance in any single metric is normal. False alerts cause alert fatigue and erode signal trust.

**Instead:** Enforce the multi-signal threshold: flag only when 2+ signals align OR 1 signal is >2 std devs from personal baseline (not team average).

---

## Build Order

The architecture has hard dependencies that dictate phase ordering:

```
1. Reference Docs (.claude/team-health/)
   ↓ Required by all skills
2. State Schema + Config
   (.team-health/config.json shape, people/<name>.json shape, baselines.json shape)
   ↓ Required before any skill can write or read state
3. First-Run Setup Flow (SETUP.md + setup gate in all skills)
   ↓ Required before any skill can operate with real team data
4. /team-health:log (people/<name>.json writer)
   ↓ People log populates context read by /prep
5. /team-health:pulse (baselines.json writer, pulse-history writer)
   ↓ Baselines must exist for /prep to compare against
       Pulse history must exist for /skip-level and /retro-prep
6. /team-health:prep (reads people log + baselines, no writes)
7. /team-health:skip-level (reads pulse history, no people log)
8. /team-health:retro-prep (reads pulse history + sprint MCPs)
```

**Phase implications:**

| Phase Topic | Must Exist Before | Reason |
|-------------|-------------------|--------|
| Reference docs | Everything | All skills read them |
| State schemas | All skill prompts | Writes/reads need schema contract |
| Setup gate + config | All skills | Skills cannot target queries without roster |
| `/log` command | `/prep` | People log provides 1:1 context |
| `/pulse` command | `/skip-level`, `/retro-prep` | Both read pulse history |
| MCP probe pattern | All MCP-dependent skills | Required for graceful degradation |

---

## Scalability Considerations

This is a local-state, single-user skill. Scalability here means graceful behavior as team and history grow.

| Concern | At 3 team members | At 15 team members | Notes |
|---------|------------------|--------------------|-------|
| Baseline computation | Trivial inline | Still manageable (~10 metrics × 15 people = 150 values) | Keep metric count under 10/person |
| Pulse run duration | Fast | Slower — 15 MCP queries per source | Consider grouping MCP calls where APIs support batch |
| People log size | Small | `people/<name>.json` can grow large over months | Recommend log truncation strategy after 6 months |
| Pulse history size | Small | Daily/weekly JSON files accumulate | Skip-level/retro reads last N, not all history |
| Context window | Fine | Long pulse runs with 15 people may approach limits | Pulse output should be structured/compact, not verbose |

---

## Sources

- PROJECT.md: authoritative project requirements and constraints (HIGH confidence)
- Claude Code custom commands architecture: based on training knowledge of `.claude/commands/` structure and prompt file conventions (MEDIUM confidence — verify against current Claude Code docs)
- MCP tool availability detection: pattern derived from how Claude handles missing tool namespaces at runtime (MEDIUM confidence — verify probe call behavior in current Claude Code version)
- Rolling statistics inline computation: established practice for small-N stats without external dependencies (HIGH confidence — arithmetic is well-defined)
- Privacy/output filtering pattern: derived from project constraints in PROJECT.md (HIGH confidence)
