# Phase 3: Team Pulse - Research

**Researched:** 2026-03-13
**Domain:** Claude Code command authoring, MCP tool invocation, rolling statistical baseline computation, JSON state read/write, dashboard output formatting
**Confidence:** HIGH for architecture and patterns (all from project-internal locked sources); MEDIUM for MCP-specific tool names (community implementations vary)

---

## Summary

Phase 3 delivers `/team-health:pulse` — the weekly team health scan that reads signal data from available MCP sources, computes deviations against each person's personal 8-week rolling baseline, applies conservative flagging rules, and writes updated baselines and pulse history to disk.

Everything this phase needs is already specified. Phase 1 locked: the `baselines.json` schema (with `computed_from`, `computed_to`, `source`, `window_weeks`, and per-metric `window`/`mean`/`stddev`/`last_value` fields), the `config.json` schema (with `sources` map and `team` roster), the full signal taxonomy (11 signals across 4 MCPs — `SIGNALS.md`), the baseline computation algorithm (step-by-step in `BASELINES.md` including first-run behavior, worked examples, and provenance fields), and the output language rules (`PRIVACY.md` including the mandatory disclaimer). Phase 2 established the Pre-flight Check boilerplate and command file conventions.

Phase 3 is primarily a command file authoring task — `.claude/commands/team-health/pulse.md`. The command orchestrates: (1) read reference docs, (2) read config, (3) for each team member × each available source, query MCP tools, (4) compare to baselines, (5) apply two-signal rule, (6) render dashboard with green/yellow/red table plus detail sections for flagged individuals, (7) update and write `baselines.json`, (8) write pulse history snapshot. No new infrastructure is needed.

The critical open questions from STATE.md — Jira MCP package name/maintenance, Calendar MCP viability, Slack MCP metadata-only scope — are addressed in this research. The pulse command must probe by namespace, not tool name, because multiple community implementations exist with different naming conventions.

**Primary recommendation:** One command file (`.claude/commands/team-health/pulse.md`) handles the entire pulse flow. It reads all three reference docs first, then processes each MCP source independently, applies the pre-defined flagging logic verbatim from `SIGNALS.md`/`BASELINES.md`, renders a structured dashboard, and writes two output files (`baselines.json` and `pulse-history/YYYY-WNN.md`). `disable-model-invocation: true` is mandatory.

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PULSE-01 | `/team-health:pulse` scans all configured direct reports and produces a team-level health dashboard | Command iterates over `config.json` `team` array; queries each available MCP source per person; final output is a team dashboard |
| PULSE-02 | Per-person signal scoring uses personal 8-week rolling baseline (not team averages) — flags changes, not absolute values | `BASELINES.md` deviation algorithm (already locked): `deviation = (current - mean) / stddev`; personal per-slug entries in `baselines.json` |
| PULSE-03 | Signals tracked: commit/PR frequency, review participation, Jira velocity, meeting density (if Calendar), Slack activity shift (if Slack) | All 11 signals fully defined in `SIGNALS.md` — GitHub (4), Jira (3), Slack (2), Calendar (2); command reads `SIGNALS.md` before querying |
| PULSE-04 | Flags are conservative: fires only when 2+ signals align OR a single signal exceeds 2 standard deviations from personal baseline | Two-Signal Rule in `SIGNALS.md` Section 5 (already locked); severity levels (yellow/red) defined there; command follows verbatim |
| PULSE-05 | Dashboard output includes summary table (green/yellow/red per person) and per-person detail sections for flagged individuals only | Output format pattern defined in this research; unflagged individuals show only the table row, no detail section |
| PULSE-06 | Every flagged signal states the specific metric, delta, and source: "PR review lag: +3.2 days vs. 8-week avg" — never opaque scores | Output language pattern: metric name + observed value + delta from mean + source; examples in this research |
| PULSE-07 | Disclaimer included in every pulse output: "These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions." | `PRIVACY.md` Required Disclaimer section — verbatim text locked; command loads `PRIVACY.md` and includes disclaimer at end of output |
| PULSE-08 | Computed baselines stored in `.team-health/baselines.json` with `computed_from`, `computed_to`, `source`, and `schema_version` provenance fields after each pulse run | `baselines.json` schema locked in Phase 1 `state-schemas.json`; provenance fields defined in `BASELINES.md` — command writes entire file using Write tool |
| PULSE-09 | Pulse history snapshots written to `.team-health/pulse-history/YYYY-WNN.md` | New file per pulse run; ISO week number format; command writes snapshot after baseline update |
| PULSE-10 | Pulse degrades gracefully: operates on available MCP sources, names which signals are unavailable and why | `config.json` `sources` map controls which sections to skip; `PRIVACY.md` Graceful Degradation Language section provides exact phrasing |
</phase_requirements>

---

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| `.claude/commands/team-health/pulse.md` | n/a | Slash command file for `/team-health:pulse` | Established in Phase 1: colon-namespace via `.claude/commands/team-health/` subdirectory |
| `.claude/team-health/SIGNALS.md` | n/a | Signal taxonomy, thresholds, two-signal rule | Phase 1 deliverable — pulse command reads this at runtime before any scoring |
| `.claude/team-health/BASELINES.md` | n/a | Rolling baseline computation algorithm | Phase 1 deliverable — pulse command reads this at runtime before any baseline update |
| `.claude/team-health/PRIVACY.md` | n/a | Output language rules, required disclaimer | Phase 1 deliverable — pulse command reads this at runtime before generating output |
| `baselines.json` schema (`schema_version: "1"`) | n/a | Persisted per-person per-metric rolling windows | Locked in Phase 1 `state-schemas.json` — must not deviate |
| `Read` tool | n/a | Load config, baselines, reference docs, people files | Pre-approved in `allowed-tools`; main read mechanism |
| `Write` tool | n/a | Write updated `baselines.json` and pulse history snapshot | Pre-approved in `allowed-tools`; only write mechanism |
| `Bash(date +%Y-%m-%d)` | n/a | Inject today's date for provenance fields and history filename | Date injection pattern established in Phase 1 |
| MCP tools (GitHub, Jira, Slack, Calendar) | varies by implementation | Query signal data per person | Controlled by `config.json` `sources` map; probe by namespace |

### Supporting

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| `!``date +%Y-%m-%d``` pre-invocation bash | n/a | Inject current date at invocation time | Include as first line of command body — critical for provenance field accuracy |
| `disable-model-invocation: true` | n/a | Prevent Claude from auto-running the pulse command | Required for all state-writing commands; established in Phase 1 |
| `sources` map in `config.json` | n/a | Gate each MCP section — read before any signal queries | `config.json` `sources.github`, `sources.jira`, `sources.slack`, `sources.calendar` |

### MCP Tool Name Reality

**Key finding (MEDIUM confidence — community implementations vary):**

There is no single canonical MCP tool name set. Multiple Jira, Slack, GitHub, and Calendar MCP server packages exist, each with different naming conventions. The pulse command must probe by namespace, not by specific tool name. This was established as a pattern in Phase 1 (`setup.md` probes by namespace).

**GitHub MCP (`github/github-mcp-server` — official, 77 tools):**
- `list_pull_requests` — list PRs for a repo
- `get_pull_request_reviews` — reviews on a PR
- `list_commits` / `search_commits` — commit history
- `search_pull_requests` — search across PRs
- Toolsets available: `pull_requests`, `repos`, `issues`, `actions` (configurable via `--toolsets` flag)
- Source: [GitHub Blog - Practical guide to GitHub MCP server](https://github.blog/ai-and-ml/generative-ai/a-practical-guide-on-how-to-use-the-github-mcp-server/)

**Jira MCP (multiple community packages — no official Atlassian MCP as of March 2026):**
- `@aashari/mcp-server-atlassian-jira` (actively maintained, v3.3.0): tools `jira_ls_issues`, `jira_get_issue`, `jira_ls_projects` — supports JQL queries
- `@teolin/mcp-jira` (v3.3.6, updated within days of this research): actively maintained
- `@xuandev/atlassian-mcp` (51 tools, supports sprints and boards): includes sprint/board tools needed for `jira_tickets_in_progress_over_2_sprints`
- **Recommended:** Prefer `@aashari/mcp-server-atlassian-jira` or `@xuandev/atlassian-mcp` — both actively maintained, both support JQL which enables the sprint-based queries SIGNALS.md requires
- Source: npm search results (March 2026)

**Slack MCP (`modelcontextprotocol/server-slack` — reference implementation, or `korotovsky/slack-mcp-server`):**
- Tools include: `get_channel_history`, `list_channels`, `list_users` (naming varies by implementation)
- **Privacy constraint**: Slack MCP must be configured to return public channel metadata only; DM access is permanently out of scope per `PRIVACY.md` Rule 2
- The Slack MCP implementation in use must be verified to not return DM content — this is a configuration concern for the manager, not a code concern for the pulse command
- Source: [Slack MCP Server docs](https://docs.slack.dev/ai/slack-mcp-server/), community search

**Calendar MCP (uncertain — STATE.md blocker confirmed):**
- No official Google Calendar MCP as of March 2026
- Community options: thin wrappers around Google Calendar API exist but maintenance is poor
- **Recommendation:** Treat Calendar as optional/deferred. Pulse command should check `sources.calendar` and skip gracefully with PRIVACY.md degradation language if false. Do not block Phase 3 delivery on Calendar MCP — `sources.calendar` will be `false` for most users.

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Single `pulse.md` command | Separate `pulse-collect.md` + `pulse-render.md` | Single file avoids the "skills can't call each other" constraint. Claude orchestrates the full pipeline in one command session. |
| Inline Claude baseline arithmetic | Python script for mean/stddev | PROJECT.md explicitly rules out Python. Claude computes over ≤8 values × ≤15 people × ≤10 signals = ≤1,200 data points inline. |
| Writing baselines.json last | Writing baselines.json before rendering output | Write after rendering — if output generation fails mid-way, baselines are not partially updated. Baselines are written as a single atomic write at the end. |

**No package installation needed.** This command uses only Claude's built-in tools plus whatever MCP servers the manager has configured.

---

## Architecture Patterns

### Recommended Project Structure

```
.claude/
  commands/
    team-health/
      setup.md          # Phase 1 — exists
      log.md            # Phase 2 — exists
      pulse.md          # Phase 3 — this phase creates this file

  team-health/
    SIGNALS.md          # Phase 1 — read by pulse.md at runtime
    BASELINES.md        # Phase 1 — read by pulse.md at runtime
    PRIVACY.md          # Phase 1 — read by pulse.md at runtime

.team-health/           # Per-manager state (gitignored)
  config.json           # Read by pulse: sources map, team roster, baseline_window_weeks
  baselines.json        # Read and written by pulse: rolling window per person × metric
  pulse-history/
    2026-W11.md         # Written by pulse after each run (ISO week number)
    2026-W12.md
  people/
    alice-chen.json     # Written by log (Phase 2); pulse does NOT write people files
```

### Pattern 1: Command File Structure for `/team-health:pulse`

**Front matter:**
```yaml
---
description: "Weekly team health pulse — scans all direct reports, computes signal deviations from personal baselines, and produces a flagged dashboard."
argument-hint: ""
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`
```

**Why `Bash(date +%Y-W%V)` in allowed-tools:** The pulse history snapshot filename uses ISO week number format (`YYYY-WNN.md`). Pre-approving this specific bash call avoids an approval dialog.

**Why `disable-model-invocation: true`:** This command writes two state files (`baselines.json` and a pulse history snapshot). Mandatory for all state-writing commands per Phase 1 convention.

### Pattern 2: Reference Doc Load Order

The command must read all three reference documents before doing any analysis. This separates evolving domain logic from command structure.

```
## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md — 11 signals, thresholds, two-signal rule, severity levels
2. .claude/team-health/BASELINES.md — baseline computation algorithm, first-run behavior, provenance fields
3. .claude/team-health/PRIVACY.md — output language rules, required disclaimer, degradation language

Follow these documents exactly. Do not improvise signal thresholds or output language.
```

### Pattern 3: Execution Sequence (the full pulse loop)

```
## Execution Sequence

Phase A — Setup
1. Pre-flight Check (same boilerplate as log.md — read config.json, check setup_complete)
2. Read reference docs (Step in Pattern 2 above)
3. Read .team-health/baselines.json (or treat as empty if file doesn't exist)
4. Read config.json sources map: note which of github/jira/slack/calendar are available
5. State which sources are unavailable using PRIVACY.md Graceful Degradation Language

Phase B — Per-person signal collection (loop over config.json team array)
For each person:
  For each available source (skip if sources.<name> is false):
    Query the appropriate MCP tools to get current week's signal values
    Record: person slug, metric name, current value, date range queried

Phase C — Per-person baseline comparison and flag determination
For each person × metric:
  1. Look up existing baseline window in baselines.json
  2. Apply BASELINES.md Comparing to Baseline algorithm
  3. Apply SIGNALS.md two-signal rule and severity levels
  4. Record: flag status (green/yellow/red), triggering signals with deltas

Phase D — Output generation
1. Render summary table (green/yellow/red per person)
2. For each flagged person: render detail section with specific signals + deltas
3. Unflagged persons: table row only, no detail section
4. Include PRIVACY.md required disclaimer at end (verbatim)

Phase E — State writes
1. Update baselines.json: for each person × metric, apply BASELINES.md Updating a Baseline algorithm
   Write the entire file using Write tool
2. Write pulse history snapshot to .team-health/pulse-history/<ISO-week>.md
```

**Order discipline:** Always write state last (Phase E after Phase D). If output generation fails, baselines are not partially updated. A failed pulse run should leave baselines unchanged.

### Pattern 4: Dashboard Output Format

The output format is not in any existing reference doc — it needs to be specified in the command prompt itself.

**Summary table (all team members):**
```markdown
## Team Health Pulse — 2026-W11

Sources active: GitHub, Jira | Unavailable: Slack, Calendar

| Name | Status | Signals Checked |
|------|--------|-----------------|
| Alice Chen | GREEN | 7/11 (GitHub + Jira) |
| Bob Smith | YELLOW | 7/11 (GitHub + Jira) |
| Carol Davis | GREEN | 7/11 (GitHub + Jira) |
```

**Detail section (flagged individuals only):**
```markdown
### Bob Smith — YELLOW (watch)

Signals triggering flag (2 of 2 required):

- **PR review count:** 0 reviews this week vs. 8-week avg 3.8/week (−3.8, −5.4 stddev)
  Source: GitHub MCP
- **Commit days:** 1 day this week vs. 8-week avg 3.9 days (−2.9, −2.3 stddev)
  Source: GitHub MCP

Other signals (within baseline):
- PRs merged: 1 (baseline avg: 1.9) — within normal range
- Jira tickets closed: 2 (baseline avg: 2.1) — within normal range
```

**Format rules:**
- For each flagged signal: metric name, current value, baseline avg, absolute delta, stddev delta, source
- For absolute-threshold flags (stuck tickets, blocked, meeting load): state count and threshold
- For unflagged signals in a detail section: brief "within normal range" summary
- Do NOT show signal detail for green team members — table row only

### Pattern 5: Pulse History Snapshot Format

Written to `.team-health/pulse-history/YYYY-WNN.md` after each run.

```markdown
# Team Pulse — 2026-W11

**Run date:** 2026-03-13
**Sources:** GitHub, Jira
**Unavailable:** Slack (not configured), Calendar (not configured)

## Summary

| Name | Status |
|------|--------|
| Alice Chen | GREEN |
| Bob Smith | YELLOW |
| Carol Davis | GREEN |

## Flagged: Bob Smith (YELLOW)

- PR review count: 0 vs. baseline 3.8 (−5.4 stddev)
- Commit days: 1 vs. baseline 3.9 (−2.3 stddev)

---
*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
```

**Why a separate history file:** Phase 5 (Skip-Level, Retro Prep) consumes `pulse-history/` files. They must be readable, summarizable, and auditable independently of `baselines.json`.

### Pattern 6: Baseline First-Run and Window Thinness

Per `BASELINES.md` First-Run Baseline section (locked):

- **0 weeks of history:** Person not in `baselines.json` — initialize window with current value, do not flag, show "(baseline pending — 0 weeks of data)"
- **1–2 weeks of history:** Window exists but too thin — show raw value with "(baseline pending — N weeks of data)", do not flag
- **3–7 weeks of history:** Begin computing deviations and applying thresholds
- **8 weeks:** Full rolling window in effect

The command must check `len(window)` before applying any flag logic.

### Pattern 7: ISO Week Number for History Filenames

`date +%Y-W%V` produces ISO 8601 week numbers: `2026-W11`. This is unambiguous, sortable, and human-readable. The planner should include `Bash(date +%Y-W%V)` in `allowed-tools` to pre-approve this call.

### Anti-Patterns to Avoid

- **Writing baselines before output:** Baseline writes happen in Phase E, after output is generated. If the write fails, re-running the command will re-compute correctly.
- **Querying MCP tools if sources.<name> is false in config.json:** The `sources` map is the gating mechanism. Never attempt GitHub MCP calls when `sources.github` is false.
- **Team-relative comparisons:** PRIVACY.md Rule 3 prohibits this. All comparisons are against the person's own baseline. The command prompt must state this explicitly.
- **Recomputing baselines before writing:** The update algorithm in BASELINES.md is the only correct path. Do not compute mean/stddev from scratch from raw MCP data — use the window update algorithm.
- **Using `context: fork`:** This command writes two state files. Forked subagents may not persist writes correctly. Run inline.
- **Assuming specific MCP tool names:** Probe by namespace. For GitHub: "any tool containing 'github' or starting with 'mcp_github'". For Jira: "any tool containing 'jira'". The exact tool names depend on which community package the manager installed.
- **Psychological language in output:** PRIVACY.md prohibits this — behavioral observations only. The command prompt must reference PRIVACY.md Rules 1 and 4 explicitly.
- **Omitting the disclaimer:** PRIVACY.md Required Disclaimer must appear verbatim at end of every pulse output. The command instructions must include the exact text: "These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions."

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Rolling mean/stddev | Python script, external service | Claude inline arithmetic over baselines.json window array per BASELINES.md | PROJECT.md explicitly rules out Python. Claude computes over ≤1,200 data points inline. BASELINES.md provides worked examples. |
| Statistical flagging logic | Custom threshold engine | SIGNALS.md Section 5 two-signal rule, verbatim | Already defined in Phase 1. The command reads SIGNALS.md and applies the rule — does not need to re-derive it. |
| ISO week number | Custom date math | `date +%Y-W%V` bash command | Standard POSIX date format. Pre-approve in allowed-tools. |
| MCP result aggregation | Custom data pipeline | Claude iterates over team array, queries each MCP inline | Data volume: ≤15 people × ≤11 signals = ≤165 queries per pulse run. Manageable inline with no pipeline. |
| Dashboard formatting | Markdown template library | Plain markdown in command prompt instructions | Output is markdown by convention across all Claude Code commands. No library needed. |
| Pulse history indexing | Database, search index | One .md file per week in pulse-history/ | Phase 5 reads these files directly. Human-inspectable, no query infrastructure needed. |
| Graceful degradation messaging | Feature flag system | `sources` map in config.json + PRIVACY.md degradation phrasing | Already defined in Phase 1. The command checks `sources` and uses the exact phrases from PRIVACY.md. |

**Key insight:** Phase 3 has zero new dependencies. Every behavior — signal definitions, baseline math, output rules, flag thresholds, degradation language — was specified in Phase 1 reference documents. The pulse command is an orchestration file that reads those references and applies them. The main authoring work is structuring the command flow clearly enough that Claude executes it correctly every time.

---

## Common Pitfalls

### Pitfall 1: MCP Query Scope Too Broad

**What goes wrong:** The GitHub MCP query for PR reviews returns reviews across all of GitHub (not just the configured `github_org`). Alice might have reviewed 50 open source PRs this week, masking a drop in team PR participation.

**Why it happens:** The command instructions say "query PRs reviewed by this person" without scoping to the configured org.

**How to avoid:** Every GitHub MCP query must scope to `github_org` from config.json. The command instructions must state: "Scope all GitHub queries to the `github_org` field in config.json."

**Warning signs:** PR/commit counts are much higher than expected; baseline windows show implausibly high values.

### Pitfall 2: Baseline Update Order (Race Condition)

**What goes wrong:** The command updates Alice's baseline window during her signal collection, then uses the newly updated window when computing her flag status. This inflates or deflates her deviation score incorrectly.

**Why it happens:** Mixing collection, comparison, and update into one loop.

**How to avoid:** The execution sequence is strict — Phase B (collect), Phase C (compare against existing baseline), Phase D (output), Phase E (update baseline). Never update a baseline before using it for flag determination in the same run.

**Warning signs:** A person is flagged this week with a deviation score that includes this week's value in the baseline mean.

### Pitfall 3: First-Run False Flags

**What goes wrong:** A new team member (no baseline history) gets flagged because their first week of data is compared to a zero-window baseline.

**Why it happens:** The first-run check (`len(window) < 3`) was omitted from the command instructions.

**How to avoid:** The command must explicitly check window length before applying flag logic. Per BASELINES.md: no flags if `len(window) < 3`; show "(baseline pending — N weeks of data)" label instead.

**Warning signs:** A person on their first or second pulse run shows a red flag.

### Pitfall 4: baselines.json Partial Write

**What goes wrong:** The Write tool fails after writing some but not all person entries. Subsequent pulse runs read a partially updated baselines.json and produce wrong baselines.

**Why it happens:** The Write tool is called multiple times (once per person) rather than once with the full updated structure.

**How to avoid:** Build the entire updated `baselines.json` in memory first, then write it once with a single Write tool call. The command instructions must say: "Update all person entries in memory first, then write the complete baselines.json file using a single Write tool call."

**Warning signs:** Some people's baselines have newer `computed_to` dates than others after a single pulse run.

### Pitfall 5: Opaque Flag Language

**What goes wrong:** Output says "Alice is flagged — engagement indicators detected" instead of "Alice: PR review count: 0 this week vs. 8-week avg 3.8/week (−5.4 stddev)".

**Why it happens:** The command prompt does not specify the exact output format for flagged signals.

**How to avoid:** Pattern 4 above (Dashboard Output Format) specifies the exact format for each flagged signal: `metric name: current value vs. baseline avg (delta, stddev delta). Source: [MCP name]`. The command instructions must include this format verbatim.

**Warning signs:** PULSE-06 smoke test fails — flagged signal output does not include specific metric name, delta, and source.

### Pitfall 6: Pulse History File Accumulation

**What goes wrong:** The pulse-history/ directory doesn't exist on first run, and the Write tool fails when trying to write `pulse-history/2026-W11.md`.

**Why it happens:** The Write tool requires the parent directory to exist before writing a file in a subdirectory.

**How to avoid:** The command must create the directory if it doesn't exist. Use `Bash(mkdir -p .team-health/pulse-history)` before writing the history snapshot. Add `Bash(mkdir -p .team-health/pulse-history)` to `allowed-tools` in the front matter.

**Warning signs:** PULSE-09 smoke test fails — pulse history file not created.

### Pitfall 7: Slack MCP Privacy Scope

**What goes wrong:** The Slack MCP implementation the manager installed returns DM content or private channel messages, violating PRIVACY.md Rule 2.

**Why it happens:** Different Slack MCP implementations have different default scopes. Some expose full message content.

**How to avoid:** The pulse command instructions must state: "When querying Slack MCP: only use public channel data. Do not request or use DM content or private channel content. If the Slack MCP returns DM content, ignore it and use public channel data only." Additionally, SKILL.md installation notes should document this requirement for manager awareness.

**Warning signs:** Slack signal output references content from a direct message or private channel.

---

## Code Examples

Verified patterns from Phase 1 and Phase 2 implementations:

### Command Front Matter

```yaml
---
description: "Weekly team health pulse — scans all direct reports, scores signals against personal baselines, and produces a flagged team dashboard."
argument-hint: ""
allowed-tools: Read, Write, Bash(date +%Y-%m-%d), Bash(date +%Y-W%V), Bash(mkdir -p .team-health/pulse-history)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
Current ISO week: !`date +%Y-W%V`
```

Source: Phase 1 RESEARCH.md Pattern 1 (command file structure); Phase 2 RESEARCH.md Pattern 1

### Pre-flight Check (verbatim boilerplate — copy exactly from log.md)

```
## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.
```

Source: Phase 1 RESEARCH.md Pattern 2; confirmed in setup.md and log.md

### Flag Output Format (PULSE-06)

```
For each flagged signal, output exactly:
- **[metric_name]:** [current_value] [unit] this week vs. 8-week avg [mean] [unit] ([delta:+/-N.N], [stddev_delta:+/-N.N] stddev)
  Source: [MCP source name] MCP

Examples:
- **PR review count:** 0 reviews this week vs. 8-week avg 3.8/week (−3.8, −5.4 stddev)
  Source: GitHub MCP
- **PR review lag:** 4.2 days vs. 8-week avg 1.1 days (+3.1, +2.6 stddev)
  Source: GitHub MCP
- **Jira tickets closed:** 0 tickets vs. 8-week avg 2.9/week (−2.9, −4.8 stddev)
  Source: Jira MCP
```

### Baseline Update (from BASELINES.md — verbatim reference)

```
After generating the dashboard (Phase D), update baselines.json:

For each person × metric:
1. Append current week's value to the window array
2. If len(window) > baseline_window_weeks: remove first element
3. Recompute mean = sum(window) / len(window)
4. Recompute stddev = sqrt(sum((x - mean)^2 for x in window) / len(window))
   Special case: if len(window) == 1, stddev = 0
5. Set last_value to the new current value
6. Set computed_to to today's date

Build the full updated baselines.json in memory, then write with a single Write tool call.
```

Source: `.claude/team-health/BASELINES.md` — Updating a Baseline section

### Pulse History Snapshot

```
## Write Pulse History Snapshot

After writing baselines.json, write the pulse history snapshot:

File path: .team-health/pulse-history/<ISO-week>.md
  (ISO week is the value injected by !`date +%Y-W%V` — e.g., 2026-W11)

Contents:
# Team Pulse — [ISO week]

**Run date:** [today YYYY-MM-DD]
**Sources active:** [list sources where sources.<name> is true]
**Unavailable:** [list sources where sources.<name> is false, with PRIVACY.md degradation language]

## Summary

| Name | Status |
|------|--------|
[one row per team member]

## Flagged Members

[For each yellow/red member: name, status, triggering signals with deltas]

---
*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
```

### baselines.json Schema (locked — Phase 1)

```json
{
  "schema_version": "1",
  "last_updated": "2026-03-13",
  "people": {
    "alice-chen": {
      "computed_from": "2025-12-01",
      "computed_to": "2026-03-13",
      "source": ["github", "jira"],
      "window_weeks": 8,
      "metrics": {
        "github_prs_merged_per_week": {
          "window": [3, 1, 2, 2, 3, 2, 2, 1],
          "mean": 2.0,
          "stddev": 0.71,
          "last_value": 1
        }
      }
    }
  }
}
```

Source: `.planning/phases/01-foundation/state-schemas.json` (locked)

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Team-average comparison ("bottom 20%") | Personal baseline deviation only | Phase 1 decision (PRIVACY.md) | Each person compared to themselves — flags changes, not absolute values |
| Opaque composite scores | Per-signal factual statements with specific deltas | Phase 1 decision (PRIVACY.md, SIGNALS.md) | Manager sees exactly what triggered a flag and why |
| Flag on any single signal dip | Two-signal rule: 2+ signals OR 1 signal at 2+ stddev | Phase 1 decision (SIGNALS.md Section 5) | Conservative flagging reduces false positives |
| Static baselines set at onboarding | 8-week rolling window updated each pulse run | Phase 1 decision (BASELINES.md) | Baselines adapt to seasonal patterns and role changes automatically |
| Python/script for statistics | Claude inline arithmetic | Phase 1 decision (PROJECT.md) | Zero runtime dependencies; works offline; no install friction |

**MCP landscape evolution (March 2026):**
- Official GitHub MCP (`github/github-mcp-server`) is now available with 77 tools and configurable toolsets — HIGH confidence this is the standard for GitHub
- No official Atlassian/Jira MCP — multiple community packages with different naming conventions; the pulse command must probe by namespace
- Slack's official MCP server exists (`docs.slack.dev/ai/slack-mcp-server`) but scoping to public-only requires configuration verification
- No mature Calendar MCP — treat `sources.calendar = false` as the default expected state for v1 users

---

## Open Questions

1. **Which GitHub MCP tool name to use for "commits authored by user in past 7 days scoped to github_org"?**
   - What we know: Official GitHub MCP has 77 tools including `list_commits`; toolsets can be restricted; tool naming uses camelCase in the blog (`listPullRequests`) vs. snake_case in some docs
   - What's unclear: Whether a single tool can scope commits by org AND author AND date range, or whether multiple calls are needed
   - Recommendation: Command instructions should say "Query the GitHub MCP for commits by this person's github_username within the past 7 days in the github_org from config.json, using whichever commit-listing tool is available." The probe-by-namespace approach handles naming variation.

2. **Does `date +%Y-W%V` produce ISO 8601 week numbers on macOS?**
   - What we know: `%V` is ISO 8601 week number (01–53); `%G` is ISO 8601 year — but on macOS/BSD `date` differs from GNU `date`
   - What's unclear: macOS `date` may not support `%V` — test required
   - Recommendation: Use `date +%Y` combined with `date +%W` as a fallback, or instruct Claude to compute the week number from the current date inline. Add to Wave 0 checklist.

3. **Should the pulse command read people log files during the run?**
   - What we know: PULSE-01 through PULSE-10 say nothing about people log integration; that is Phase 4 (prep)
   - What's unclear: Whether surfacing a recent "concern" log entry would improve pulse output
   - Recommendation: No — keep Phase 3 scoped to MCP signals and baselines only. People log integration is Phase 4. Mixing concerns will make the command harder to test and maintain.

4. **What happens when baselines.json doesn't exist on first pulse run?**
   - What we know: BASELINES.md First-Run Baseline section handles this: initialize window with current value, `mean = current_value`, `stddev = 0`, no flagging
   - What's unclear: The command instructions need to handle the case where baselines.json doesn't exist at all (Read returns an error) vs. exists but has no entry for this person
   - Recommendation: Treat "file not found" the same as "empty file" — start with `{ "schema_version": "1", "last_updated": "...", "people": {} }` and proceed. Document in command instructions.

5. **ISO week format on macOS — `%V` vs `%W` compatibility**
   - What we know: GNU date supports `%V` for ISO weeks; macOS BSD date may not
   - What's unclear: Needs verification on macOS where the skill typically runs
   - Recommendation: Wave 0 should include a test: `date +%Y-W%V` in a terminal on the target OS. If not supported, use `date +%Y-W%W` (non-ISO week 00–53) or derive the week in the command instructions from the injected date string.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None — pure markdown/JSON command file, no runnable code |
| Config file | n/a |
| Quick run command | Manual: invoke `/team-health:pulse` in Claude Code with GitHub MCP configured |
| Full suite command | Manual: run all 10 PULSE scenarios in a project with Phase 1 setup complete and at least one MCP source available |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PULSE-01 | `/team-health:pulse` runs for all configured team members | smoke/manual | Run command; verify output mentions every person in config.json team array | ❌ Wave 0 |
| PULSE-02 | Per-person scoring uses personal baseline, not team average | smoke/manual | Verify no team-relative comparisons in output; all flags reference person's own baseline | ❌ Wave 0 |
| PULSE-03 | Correct signals tracked per available source | smoke/manual | Run with GitHub only; verify 4 GitHub signals appear, 7 others marked unavailable | ❌ Wave 0 |
| PULSE-04 | Flags fire only on 2+ signals OR 1 signal >2 stddev | smoke/manual | Inject a scenario where 1 signal is at 1.5 stddev — verify no flag; then 2 signals at 2+ stddev — verify yellow flag | ❌ Wave 0 |
| PULSE-05 | Summary table has all members; detail only for flagged | smoke/manual | Run pulse with one flagged person; verify table has all members, detail section exists only for flagged person | ❌ Wave 0 |
| PULSE-06 | Flagged signal output states metric, delta, source | smoke/manual | Inspect flagged signal output; verify format: metric name, current value, baseline avg, delta, stddev delta, source | ❌ Wave 0 |
| PULSE-07 | Disclaimer appears verbatim at end of output | smoke/manual | Verify last paragraph of pulse output is exactly: "These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions." | ❌ Wave 0 |
| PULSE-08 | baselines.json written with provenance fields | smoke/manual | Run pulse; inspect `.team-health/baselines.json` — verify `computed_from`, `computed_to`, `source`, `schema_version` fields present for each person | ❌ Wave 0 |
| PULSE-09 | Pulse history snapshot written to pulse-history/ | smoke/manual | Run pulse; verify `.team-health/pulse-history/YYYY-WNN.md` exists with run date and team summary | ❌ Wave 0 |
| PULSE-10 | Graceful degradation names unavailable sources | smoke/manual | Run with no MCP sources configured; verify output names all 4 unavailable sources with PRIVACY.md degradation phrasing | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** Verify the specific deliverable (file exists, correct front matter, required sections present)
- **Per wave merge:** Run all 10 smoke scenarios in a fresh project with Phase 1 setup complete
- **Phase gate:** All 10 scenarios pass before marking Phase 3 complete

### Wave 0 Gaps

- [ ] `docs/testing/phase-3-smoke-tests.md` — step-by-step manual test procedure for all 10 PULSE scenarios above
- [ ] `.claude/commands/team-health/pulse.md` — the command file itself (primary deliverable)
- [ ] Verify `date +%Y-W%V` works on target OS (macOS) — add to Wave 0 checklist
- [ ] Verify `mkdir -p .team-health/pulse-history` is supported in `allowed-tools` Bash scoping

*(No automated framework needed — no runnable code; all tests are manual)*

---

## Sources

### Primary (HIGH confidence)

- `.planning/phases/01-foundation/state-schemas.json` — locked canonical schemas: `baselines.json` shape, `config.json` shape including `sources` map, `schema_version` fields, provenance fields
- `.claude/team-health/SIGNALS.md` — all 11 signals, thresholds, two-signal rule, severity levels — fully authored in Phase 1
- `.claude/team-health/BASELINES.md` — full baseline update algorithm, first-run behavior, provenance field requirements, worked examples — fully authored in Phase 1
- `.claude/team-health/PRIVACY.md` — output language rules, required disclaimer, graceful degradation language — fully authored in Phase 1
- `.planning/phases/01-foundation/01-RESEARCH.md` — command file structure, front matter keys, colon-namespace, Pre-flight Check boilerplate, MCP probe-by-namespace pattern
- `.planning/phases/02-people-log/02-RESEARCH.md` — confirmed patterns for state-writing commands: Read-mutate-write, `disable-model-invocation: true`, date injection

### Secondary (MEDIUM confidence)

- [GitHub Blog — Practical guide to GitHub MCP server](https://github.blog/ai-and-ml/generative-ai/a-practical-guide-on-how-to-use-the-github-mcp-server/) — confirmed 77 tools, toolsets include `pull_requests`, `repos`; tool names include `list_pull_requests`, `get_pull_request_reviews`
- [npm search — Jira MCP packages, March 2026](https://www.npmjs.com/package/@aashari/mcp-server-atlassian-jira) — confirmed `@aashari/mcp-server-atlassian-jira` (v3.3.0, actively maintained) supports JQL queries; `jira_ls_issues`, `jira_get_issue` tool names
- [Slack MCP Server docs](https://docs.slack.dev/ai/slack-mcp-server/) — official Slack MCP exists; public channel participation queryable

### Tertiary (LOW confidence — verify at implementation)

- MCP tool names for Jira, Slack, Calendar — community implementations vary; probe by namespace per Phase 1 pattern; exact tool names cannot be confirmed without live session
- `date +%Y-W%V` on macOS — standard POSIX `%V` flag may behave differently on BSD `date`; verify before using in Wave 0
- Calendar MCP availability — no evidence of a reliable, maintained Calendar MCP as of March 2026; treat as optional/deferred

---

## Metadata

**Confidence breakdown:**
- Command file format and front matter: HIGH — direct from Phase 1 implementation, validated in Phase 2
- State schemas (baselines.json, config.json): HIGH — locked in state-schemas.json, unchanged since Phase 1
- Signal definitions and thresholds: HIGH — fully authored in SIGNALS.md; pulse command reads and applies verbatim
- Baseline algorithm: HIGH — fully authored in BASELINES.md with worked examples; pulse command reads and applies verbatim
- Output format (dashboard, history snapshot): MEDIUM-HIGH — logical design derived from requirements; not pre-existing; may need iteration based on first use
- MCP tool names: MEDIUM — GitHub official MCP confirmed with general tool categories; Jira/Slack/Calendar tool names depend on which community package is installed
- Calendar MCP: LOW — no mature implementation confirmed; treat as sources.calendar = false for v1 users

**Research date:** 2026-03-13
**Valid until:** 2026-06-13 (stable foundations; locked schemas and reference docs will not change without a schema_version bump; MCP landscape may evolve faster — re-verify Jira/Calendar options at implementation)
