# Phase 4: 1:1 Prep - Research

**Researched:** 2026-03-17
**Domain:** Claude Code command authoring, people log synthesis, MCP signal collection, talking point generation from structured data
**Confidence:** HIGH for architecture and patterns (all from project-internal locked sources); MEDIUM for prep sheet UX decisions (Claude's discretion per CONTEXT.md)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Lookback window**
- Use "since last 1:1" as the primary signal window — reads `last_1on1_date` from the person's people log entry or config
- Fallback when no prior date is recorded: 14-day default
- After generating prep, auto-append a `prep_run` timestamp to the people log so the next invocation knows where to look back from
- Do NOT prompt the manager to enter the date — infer it automatically

**Talking points**
- Claude generates 3-5 talking points from the data — not raw data only
- Sources for talking point generation (all four):
  1. Signal flags: anomalies from baseline comparison → "PR review lag up 40% — ask about workload"
  2. Open commitments: unresolved manager commitments from the log → "You committed to intro-ing Alex to design — did that happen?"
  3. Career context: goals, time since last promo discussion, role aspirations from people log
  4. Log patterns: recurring themes in recent entries (same concern mentioned 3x in 6 weeks)
- Tone: frame as questions to ask, not scripts — stays in manager's voice
- Never diagnose; always behavioral observation → question form

**Patterns carried forward from prior phases**
- Pre-flight check (config.json + setup_complete gate) before executing
- Required Reference Reads: SIGNALS.md, BASELINES.md, PRIVACY.md — must be read before any signal collection or output
- Slug always resolved from config.json team array, never re-derived from argument
- Graceful degradation language from PRIVACY.md when sources unavailable
- Output (prep sheet rendered) before state writes (prep_run timestamp appended)
- `disable-model-invocation: true` front matter

### Claude's Discretion
- Exact prep sheet formatting (spacing, headers, visual weight)
- How many log entries to surface as "standing items" vs. summary
- Whether to group talking points by topic (signals / commitments / career) or rank by urgency

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope.

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| PREP-01 | `/team-health:prep <name>` generates a prep sheet scannable in 2 minutes | Single command file `.claude/commands/team-health/prep.md`; argument-based name resolution from config.json team array |
| PREP-02 | Output structure: (1) status snapshot, (2) signal flags, (3) standing items from people log, (4) suggested talking points, (5) context reminders from people log | Five-section output format defined in this research; each section maps to a distinct data source |
| PREP-03 | Pulls recent GitHub activity: PRs merged/open, review participation, stale PRs (>48h without review), commit pattern | Same GitHub signal collection pattern as pulse.md Phase B; lookback window scoped to `since last 1:1` instead of "past 7 days" |
| PREP-04 | Pulls Jira signals when available: tickets in-progress >2 sprints, velocity vs. baseline, blocked tickets | Same Jira signal collection as pulse.md; absolute-threshold signals work without a baseline window |
| PREP-05 | Pulls Calendar signals when available: meeting load % vs. baseline, 1:1 scheduling adherence | Same Calendar signal collection as pulse.md; `sources.calendar` gates this section |
| PREP-06 | Pulls Slack metadata signals when available: channel participation trend, response latency trend (not DM content) | Same Slack signal collection as pulse.md; PRIVACY.md Rule 2 hard constraint (public channel metadata only) |
| PREP-07 | Surfaces open manager commitments and IC asks from the people log | Read `open_commitments` array from `.team-health/people/<slug>.json`; filter for `status: "open"` |
| PREP-08 | Surfaces career context: last noted goals, time since last promotion discussion (from people log) | Read `career_context` object from people log; compute days since `last_promo_discussion` |
| PREP-09 | Tone is direct and factual — no diagnoses, no loaded language; all signals framed as behavioral observations | PRIVACY.md Rules 1–4 apply; required disclaimer verbatim at end |
| PREP-10 | Degrades gracefully to available sources; states what signals are absent | `sources` map from config.json gates each MCP section; PRIVACY.md Graceful Degradation Language section provides exact phrasing |
</phase_requirements>

---

## Summary

Phase 4 delivers `/team-health:prep <name>` — a command that synthesizes live MCP signals and the person's people log into a structured 2-minute prep sheet before a 1:1. The command is architecturally similar to `pulse.md` but scoped to a single person, with a dynamic lookback window ("since last 1:1"), and enriched with people log synthesis: open commitments, career context, and recurring entry themes.

All the hard technical problems are already solved. Signal collection, baseline comparison, two-signal rule, graceful degradation language, output language rules, name resolution, and pre-flight check are all defined in Phase 1–3 reference documents and implemented in `pulse.md` and `log.md`. Phase 4's distinguishing work is: (1) dynamically computing the lookback window from `last_1on1_date`, (2) reading and synthesizing the people log for three types of content (open commitments, career context, log patterns), (3) generating 3-5 opinionated talking points from all four source types, and (4) appending a `prep_run` timestamp to the people log after the prep sheet is rendered.

The primary structural insight is that prep is a per-person synthesis command, not a team loop like pulse. It reads one person's data deeply (people log + MCP signals) and produces a cohesive briefing. The "trusted chief-of-staff" framing from CONTEXT.md is the right mental model: the output should feel like briefing notes, not a data dump. This guides formatting decisions where Claude has discretion.

**Primary recommendation:** One command file (`.claude/commands/team-health/prep.md`) with Phases A–E structure adapted from `pulse.md`. Phase structure: A (pre-flight + reference reads + resolve person), B (read people log + derive lookback window), C (MCP signal collection for that person over the lookback window), D (prep sheet output — five sections), E (append `prep_run` to people log). `disable-model-invocation: true` is mandatory.

---

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| `.claude/commands/team-health/prep.md` | n/a | Slash command file for `/team-health:prep` | Established colon-namespace via `.claude/commands/team-health/` — same as all prior commands |
| `.claude/team-health/SIGNALS.md` | n/a | Signal taxonomy, thresholds, two-signal rule | Must be read before any signal collection — locked in Phase 1 |
| `.claude/team-health/BASELINES.md` | n/a | Rolling baseline computation algorithm | Must be read before any deviation comparison — locked in Phase 1 |
| `.claude/team-health/PRIVACY.md` | n/a | Output language rules, required disclaimer, degradation language | Must be read before any output generation — locked in Phase 1 |
| `.team-health/people/<slug>.json` | schema_version: "1" | Per-person people log: entries, open_commitments, career_context | Written by Phase 2 `log.md`; consumed by prep for log synthesis |
| `.team-health/baselines.json` | schema_version: "1" | Per-person rolling baseline windows for all 11 signals | Written by Phase 3 `pulse.md`; read by prep for deviation comparison |
| `.team-health/config.json` | schema_version: "1" | Sources map, team roster with slugs, 1:1 cadence | Pre-flight gate and name resolution source |
| `Read` tool | n/a | Load config, baselines, reference docs, people log | Pre-approved in allowed-tools |
| `Write` tool | n/a | Append `prep_run` timestamp to people log after output | Pre-approved in allowed-tools |
| `Bash(date +%Y-%m-%d)` | n/a | Inject today's date for prep_run timestamp and date arithmetic | Pre-approved in allowed-tools; date injection pattern established in Phase 1 |

### Supporting

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| `disable-model-invocation: true` | n/a | Prevent Claude from auto-running the prep command | Required — command writes state (prep_run timestamp); mandatory for all state-writing commands |
| `argument-hint: "<name>"` | n/a | UI hint for manager | Shows in Claude Code command palette |
| MCP tools (GitHub, Jira, Slack, Calendar) | varies | Query live signals per person for the lookback window | Controlled by `config.json` `sources` map; probe by namespace |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Single `prep.md` command file | Separate signal-collect and synthesis steps | Single file constraint (skills can't call each other); Claude orchestrates full pipeline in one session |
| `last_1on1_date` field in people log | Separate `prep-state.json` file | Storing prep state in the people log keeps all per-person state together; one fewer file to manage |
| Auto-derived lookback from `prep_run` | Explicit `last_1on1_date` | CONTEXT.md decision: use `last_1on1_date` as primary; `prep_run` is how the next invocation infers this date — the first invocation after a 1:1 updates `last_1on1_date`, subsequent invocations before the next 1:1 re-use the same anchor |

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
      pulse.md          # Phase 3 — exists
      prep.md           # Phase 4 — this phase creates this file

  team-health/
    SIGNALS.md          # Phase 1 — read by prep.md at runtime
    BASELINES.md        # Phase 1 — read by prep.md at runtime
    PRIVACY.md          # Phase 1 — read by prep.md at runtime

.team-health/           # Per-manager state (gitignored)
  config.json           # Read by prep: sources map, team roster, 1:1 cadence
  baselines.json        # Read by prep: baseline windows for signal deviation comparison
  people/
    alice-chen.json     # Read AND written by prep (appends prep_run)
                        # Written by log (Phase 2); prep reads for synthesis
```

### New Field: `last_1on1_date` in People Log

The people log schema (Phase 1 `state-schemas.json`) does not include `last_1on1_date` or `prep_run` fields. Phase 4 introduces these as additions to the `career_context` object or as a top-level field. The CONTEXT.md decision is:

- `last_1on1_date`: read to determine the lookback window; updated by prep_run after each invocation
- `prep_run`: the timestamp written by prep after output is generated; used as `last_1on1_date` on the next invocation

**Recommended schema placement:**

```json
{
  "schema_version": "1",
  "name": "Alice Chen",
  "slug": "alice-chen",
  "last_updated": "2026-03-17",
  "last_1on1_date": "2026-03-10",
  "prep_run": "2026-03-17",
  "entries": [...],
  "open_commitments": [...],
  "career_context": {
    "stated_goals": ["Staff Engineer track"],
    "last_promo_discussion": "2026-01-20",
    "notes": ""
  }
}
```

Both `last_1on1_date` and `prep_run` are top-level fields on the person object (not nested in `career_context`) because they are operational metadata, not career data. The prep command reads `last_1on1_date` first; if absent, falls back to `prep_run`; if absent, uses 14-day default.

**Lookback window derivation logic (locked in CONTEXT.md):**
```
1. Read person file: check for `last_1on1_date` field
2. If present: lookback_start = last_1on1_date
3. Else: check for `prep_run` field
4. If present: lookback_start = prep_run
5. Else: lookback_start = today - 14 days
6. After rendering output: set person.last_1on1_date = prep_run (today's date)
   and set person.prep_run = today's date
```

Wait — re-reading CONTEXT.md more carefully: `prep_run` is what gets auto-appended. The next invocation reads `last_1on1_date`. This implies that after running prep, `prep_run` is set to today. On the next invocation, the command reads `last_1on1_date`. So either: (a) the manager updates `last_1on1_date` manually via `log.md`, or (b) `prep_run` IS the `last_1on1_date` for the next call.

**Simplest consistent interpretation (use this):**
- After generating output: write `prep_run = today` to the person file
- On next invocation: read `last_1on1_date` first; if absent, read `prep_run` as the fallback; if absent, use 14-day default
- The manager can explicitly log a `last_1on1_date` via `/team-health:log` if they want to anchor it to the actual meeting date rather than the prep date

### Pattern 1: Command File Structure for `/team-health:prep`

**Front matter:**
```yaml
---
description: "Generate a 2-minute 1:1 prep sheet for a direct report: live signal snapshot, standing items from people log, and suggested talking points."
argument-hint: "<name>"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
```

**Why no ISO week:** Prep does not write a weekly history file, so `date +%Y-W%V` is not needed.

**Why `disable-model-invocation: true`:** Prep writes the `prep_run` timestamp to the people log. Mandatory for state-writing commands per Phase 1 convention.

### Pattern 2: Execution Phase Structure (A–E)

Adapted from `pulse.md` Phase A–E structure:

```
## Phase A — Setup and person resolution

1. Pre-flight Check (verbatim boilerplate from log.md/pulse.md)
2. Name Resolution (verbatim boilerplate from log.md)
3. Required Reference Reads: SIGNALS.md, BASELINES.md, PRIVACY.md (verbatim from pulse.md)
4. Read .team-health/baselines.json
   - If not found: treat as { "schema_version": "1", "last_updated": "", "people": {} }

## Phase B — People log read and lookback window derivation

1. Read .team-health/people/<slug>.json
   - If not found: no people log entries, open_commitments = [], career_context = empty
2. Derive lookback_start:
   - If person.last_1on1_date exists: use it
   - Else if person.prep_run exists: use it
   - Else: use today - 14 days (CONTEXT.md fallback)
3. Record lookback_start for use in Phase C and output
4. Extract from people log:
   - open_commitments: entries where status = "open"
   - career_context: stated_goals, last_promo_discussion, notes
   - log_patterns: scan entries since lookback_start for category and content patterns
     (same concern category ≥3 times in 6 weeks counts as a recurring theme)

## Phase C — MCP signal collection (for this person, since lookback_start)

For each available source (sources.<name> is true in config.json):
  Note: signal window is "since lookback_start", not the fixed 7/14-day windows in SIGNALS.md
  Apply the same signal definitions from SIGNALS.md but with the dynamic date range.

  GitHub signals (if sources.github is true):
  - github_prs_merged: count of PRs merged since lookback_start
  - github_pr_review_count: count of PRs reviewed since lookback_start
  - github_pr_review_lag_days: any stale PRs (open >48h with no review — absolute threshold)
  - github_commit_days: count of distinct commit days since lookback_start

  Jira signals (if sources.jira is true):
  - jira_tickets_closed: count closed since lookback_start
  - jira_tickets_in_progress_over_2_sprints: count (absolute threshold — no lookback needed)
  - jira_blocked_tickets: count (absolute threshold — no lookback needed)

  Slack signals (if sources.slack is true):
  PRIVACY CONSTRAINT: public channel metadata only.
  - slack_channel_participation: channel count since lookback_start
  - slack_response_latency: median @-mention response time

  Calendar signals (if sources.calendar is true):
  - calendar_meeting_load_pct: meeting load % since lookback_start
  - calendar_1on1_adherence: was 1:1 held in the expected cadence window?

Compare each collected signal to baseline from baselines.json using BASELINES.md deviation algorithm.
Apply two-signal rule from SIGNALS.md Section 5 to determine flag status.

## Phase D — Prep sheet output

Render the prep sheet. Full format defined in Pattern 3 below.
Output before writing any state.

## Phase E — State write

After output is complete:
1. Read the current person file from memory (already loaded in Phase B)
2. Set person.last_1on1_date = today's date (from injected date)
3. Set person.prep_run = today's date
4. Set person.last_updated = today's date
5. Write the complete updated person file using Write tool
   IMPORTANT: Write the complete file — do not truncate or omit existing entries.
6. Confirm: "Prep sheet generated for [Name]. Next prep will look back from today."
```

### Pattern 3: Prep Sheet Output Format (PREP-02)

The five sections are:

```markdown
# 1:1 Prep: [Full Name] — [today YYYY-MM-DD]
*Lookback: [lookback_start] to today ([N days])*

---

## 1. Status Snapshot

[1–3 sentences: current signal status. If signals flagged: name the flags factually.
If no signals flagged: "No signal anomalies since [date]. [N] signals checked ([sources])."
If all signals baseline-pending: "Insufficient baseline history — raw values shown, no flags applied."]

Available signals: [list active sources] | Unavailable: [list inactive sources with PRIVACY.md degradation phrasing]

---

## 2. Signal Flags

[If any signals flagged:]
[For each flag, use the same format as pulse.md Phase D detail section:]
- **[Human-readable metric name]:** [current_value] [unit] vs. 8-week avg [mean] [unit] ([delta:+/-N.N], [stddev_delta:+/-N.N] stddev)
  Source: [GitHub / Jira / Slack / Calendar] MCP

[If no signals flagged:]
*No flags. All available signals within personal baseline.*

[If signals unavailable:]
*[PRIVACY.md graceful degradation language for each unavailable source]*

---

## 3. Standing Items from People Log

[Open commitments — if any:]
**Open commitments (you owe them):**
- [date] — [content] *(open since [N days])*

[IC asks — if any open items categorized as questions/asks from IC in recent entries:]
**Open asks (they need from you):**
- [date] — [content]

[If no open items:]
*No open commitments in people log.*

---

## 4. Suggested Talking Points

[3-5 talking points, framed as questions the manager could ask.
Each labeled with its source: (signal) / (commitment) / (career) / (pattern)]

1. [Question] *(source: signal — PR review lag up 40%)*
2. [Question] *(source: commitment — intro to design team pending)*
3. [Question] *(source: career — 6 months since last promo discussion)*
4. [Question] *(source: pattern — concern entries 3x in past 6 weeks)*
5. [Question] *(source: signal — 2 blocked tickets since [date])*

---

## 5. Context Reminders

[Career context from people log:]
**Goals:** [stated_goals list, or "None recorded"]
**Last promo discussion:** [last_promo_discussion date, or "Not recorded"] [if recorded: "(N months ago)"]
**Notes:** [career_context.notes, or "—"]

[Recent log entries since lookback_start — up to 5, newest first:]
**Recent log entries:**
- [YYYY-MM-DD] · [category] — [content]
...

---

*These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.*
```

**Format decisions (Claude's discretion per CONTEXT.md):**
- Five sections always appear in order 1–5, even if a section has no data (show the "no data" state)
- Standing items: surface up to 5 open commitments; if more than 5, show oldest 3 + newest 2 with "and N others" note
- Talking points: group by urgency (signal flags first, then overdue commitments, then career, then patterns) — this is the recommended default, but the planner may override
- Context reminders: cap at 5 most recent log entries since lookback; don't surface entries older than 6 months

### Pattern 4: Lookback Window Language

The prep sheet header must state the lookback window clearly so the manager knows what data range the prep covers.

```
*Lookback: 2026-03-03 to 2026-03-17 (14 days — no prior 1:1 date recorded)*
*Lookback: 2026-03-10 to 2026-03-17 (7 days — since last prep on 2026-03-10)*
```

The parenthetical notes the source of the window so the manager can correct it if needed (e.g., they had a 1:1 earlier and the system doesn't know yet).

### Pattern 5: Talking Point Generation Rules

Claude generates talking points, not the data itself. The CONTEXT.md framing: "a trusted chief-of-staff briefing you before a meeting."

**Rules for talking point generation:**
1. Frame every point as a question to ask — not a statement about the person
2. Never diagnose; always behavioral observation → question
3. Label each point with its source type in parentheses
4. Order by urgency: (a) RED/YELLOW signal flags, (b) overdue open commitments, (c) career context overdue, (d) log patterns, (e) GREEN signals that are notable
5. 3 points minimum, 5 points maximum — if fewer than 3 sources have anything to surface, generate points from whatever is available

**Talking point templates (to be adapted, not copy-pasted):**

| Source | Template |
|--------|----------|
| Signal flag (low output) | "Commit/PR/review activity is down since [date] — what's been taking your focus?" |
| Signal flag (high lag) | "A few PRs have been waiting on your review for a while — anything blocking you from getting to them?" |
| Signal flag (blocked ticket) | "You've got [N] blocked tickets — what do you need from me to unblock those?" |
| Open commitment | "I committed to [content] on [date] — have I followed through on that?" |
| Career (promo due) | "We haven't talked about your promotion path in [N months] — is that still a priority for you?" |
| Career (goal check) | "You mentioned [goal] as a goal. How's that going? Is there anything I can do to support it?" |
| Log pattern (concern) | "I've noticed a theme in our conversations around [topic] — is that still something weighing on you?" |

### Anti-Patterns to Avoid

- **Prompting the manager for the 1:1 date:** CONTEXT.md explicitly prohibits this. Infer the lookback window automatically.
- **Writing prep_run before output is rendered:** Output must be complete before any state writes (same ordering rule as pulse.md).
- **Re-deriving the slug from the argument text:** Always resolve slug from config.json team array using name prefix match — same as log.md. The slug drives the people log file path.
- **Surfacing DM or private channel Slack content:** PRIVACY.md Rule 2 is a hard constraint, never configurable.
- **Comparing this person to other team members:** PRIVACY.md Rule 3 prohibits all team-relative comparisons.
- **Using psychological language in talking points:** All talking points must pass the "behavioral observation → question" test. No labels like "disengaged", "burned out", "checked out".
- **Writing multiple Write calls for the people log update:** Build the full updated person object in memory, then write with a single Write call. Prevents partial writes.
- **Omitting the disclaimer:** PRIVACY.md required disclaimer must appear verbatim at the end of every prep sheet output.
- **MCP signals scoped to fixed 7-day window:** Prep uses a dynamic lookback window ("since last 1:1"), not the fixed 7-day window used in pulse. The command instructions must pass `lookback_start` as the date filter for all MCP queries.
- **Flagging baseline-pending signals:** Same rule as pulse — if `len(window) < 3`, do not flag; surface raw value with "(baseline pending)" note. Most people will have baselines from Phase 3 pulse runs, but new team members will not.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Talking point generation | Template engine, rule-based system | Claude synthesizes from structured inputs per Pattern 5 rules | This is exactly what Claude is for; the rules are simple enough to specify in the command prompt |
| Log pattern detection | Text classifier, NLP pipeline | Claude scans entries array for category counts and content themes | Data volume: ≤50 entries per person; trivially inline |
| Baseline deviation for prep | Custom computation | BASELINES.md Comparing to Baseline algorithm (verbatim) | Already locked and working in pulse.md |
| Lookback date arithmetic | Date library | Claude computes `today - 14 days` inline from injected date | Trivial; no library needed |
| People log schema | New data model | `state-schemas.json` person schema (extend with top-level `last_1on1_date` and `prep_run` fields only) | Schema locked in Phase 1 — extend minimally, do not replace |
| Graceful degradation messaging | Feature flag system | `sources` map in config.json + PRIVACY.md degradation phrasing | Same pattern as pulse.md — works verbatim |

**Key insight:** Phase 4 adds no new infrastructure. The command file is the only deliverable. All underlying machinery — signal definitions, baseline math, output rules, schema, people log format — was built in Phases 1–3. The Phase 4 command is an orchestration and synthesis file.

---

## Common Pitfalls

### Pitfall 1: Dynamic Lookback Window Not Passed to MCP Queries

**What goes wrong:** MCP queries use a hardcoded 7-day or 14-day window (from SIGNALS.md) instead of the dynamic "since last 1:1" window. The prep sheet covers the wrong time range.

**Why it happens:** The command instructions don't explicitly replace the SIGNALS.md date ranges with the dynamic `lookback_start`.

**How to avoid:** The command instructions must say: "Use `lookback_start` (derived in Phase B) as the date filter for ALL MCP queries — not the fixed 7-day or 14-day windows defined in SIGNALS.md. SIGNALS.md defines what to query, not when."

**Warning signs:** A prep run the day after a 1:1 shows data from the previous two weeks instead of just the past day.

### Pitfall 2: prep_run Written Before Output Is Rendered

**What goes wrong:** The `prep_run` timestamp is appended to the people log before the prep sheet is rendered. If output generation fails mid-way, the next invocation uses today's date as the lookback anchor, missing the period that was never fully covered.

**Why it happens:** State writes done in the wrong order.

**How to avoid:** Strict Phase D → Phase E ordering. Output is complete before any Write call to the people log. This is the same ordering rule from pulse.md and is explicit in CONTEXT.md.

**Warning signs:** Prep sheet shows a very short lookback window even though no 1:1 was held.

### Pitfall 3: People Log Not Found (First Use)

**What goes wrong:** The person has no people log file yet (Phase 2 log.md was never used for them). The command errors when trying to read `.team-health/people/<slug>.json`.

**Why it happens:** No graceful first-use handling.

**How to avoid:** The command instructions must handle "file not found" gracefully: treat as empty people log with `entries = []`, `open_commitments = []`, `career_context = empty`. State in the prep sheet: "No people log entries recorded. Standing items and context reminders unavailable." Still proceed with MCP signals.

**Warning signs:** The prep command errors on a new team member.

### Pitfall 4: Talking Points Become Scripts, Not Questions

**What goes wrong:** Talking points are phrased as conclusions or directives: "Alex needs to fix their PR review lag" instead of "Alex's PR review lag is up since last 1:1 — ask about workload."

**Why it happens:** The "question form" constraint from CONTEXT.md is not enforced in the command instructions.

**How to avoid:** The command instructions must explicitly state: "Frame every talking point as a question the manager asks, not a statement about the person. The talking point is a prompt to open a conversation, not a script. Never include diagnosis language."

**Warning signs:** Talking points use declarative statements ("X needs to...", "X is...") instead of questions.

### Pitfall 5: Surfacing Too Many Log Entries as "Standing Items"

**What goes wrong:** The prep sheet surfaces 15 log entries as context, making it no longer scannable in 2 minutes. The 2-minute constraint is a hard design requirement (PREP-01).

**Why it happens:** No cap on the number of entries surfaced.

**How to avoid:** Pattern 3 caps standing items at 5 recent entries since lookback. For context reminders, cap at 5 entries and add "and N older entries not shown" if more exist. Open commitments: cap at 5 with "and N others" note if more.

**Warning signs:** Prep output exceeds ~60 lines.

### Pitfall 6: Missing PRIVACY.md Compliance in Talking Points

**What goes wrong:** A talking point uses language like "Alex seems disengaged lately — ask why they're checked out." Violates PRIVACY.md Rules 1 and 4.

**Why it happens:** Talking point generation synthesizes patterns across signals and log entries; psychological language can creep into synthesized text.

**How to avoid:** After generating talking points, the command instructions must include an explicit compliance check: "Review each talking point against PRIVACY.md Rules 1 (signals not diagnoses) and 4 (no psychological labels). Rewrite any talking point that contains interpretive or diagnostic language." This self-review step is in the command prompt.

**Warning signs:** Talking points use words like "burned out", "disengaged", "struggling", "checked out", "overwhelmed."

### Pitfall 7: Partial People Log Write

**What goes wrong:** The Write call to append `prep_run` to the people log only writes the new field, losing existing entries and open_commitments.

**Why it happens:** Write tool overwrites the entire file — if only a partial object is written, data is lost.

**How to avoid:** Always build the COMPLETE updated person object in memory (loading Phase B's in-memory copy, updating only `last_1on1_date`, `prep_run`, and `last_updated`), then write the full object. This is the same "atomic write" discipline from Phase 2 and Phase 3.

**Warning signs:** People log shows missing entries after a prep run.

---

## Code Examples

Verified patterns from existing command implementations:

### Command Front Matter

```yaml
---
description: "Generate a 2-minute 1:1 prep sheet for a direct report: live signal snapshot, standing items from people log, and suggested talking points."
argument-hint: "<name>"
allowed-tools: Read, Write, Bash(date +%Y-%m-%d)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
```

Source: Phase 1/2/3 RESEARCH.md command file structure pattern; confirmed in `pulse.md` and `log.md`.

### Pre-flight Check (copy verbatim from log.md/pulse.md)

```
## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.
```

Source: `.claude/commands/team-health/log.md` and `pulse.md` — verbatim boilerplate.

### Name Resolution (copy verbatim from log.md)

```
## Name Resolution

1. Extract the person name from $ARGUMENTS.
2. Do NOT re-derive the slug from the argument text. Instead: read the `team` array from config.json
   (already read in Pre-flight Check), find the entry whose `name` field is a case-insensitive prefix
   match for the typed name. Use the `slug` field from that config entry.
3. Fuzzy prefix matching: "alice" matches "Alice Chen"; "alice chen" matches "Alice Chen".
   Match is case-insensitive. If multiple team members match, list them and ask to clarify.
4. If no match found: "No team member matching '[typed name]' found. Configured team: [list].
   Did you mean one of these?" — stop.
5. File path: `.team-health/people/<slug>.json`
```

Source: `.claude/commands/team-health/log.md` — verbatim boilerplate.

### Required Reference Reads (copy verbatim from pulse.md)

```
## Required Reference Reads

Before doing any analysis, read these files using the Read tool (in order):
1. .claude/team-health/SIGNALS.md — 11 signals, thresholds, two-signal rule, severity levels
2. .claude/team-health/BASELINES.md — baseline computation algorithm, first-run behavior, provenance fields
3. .claude/team-health/PRIVACY.md — output language rules, required disclaimer, degradation language

Follow these documents exactly. Do not improvise signal thresholds, flagging rules, or output language.
```

Source: `.claude/commands/team-health/pulse.md` — verbatim boilerplate.

### Lookback Window Derivation

```
## Lookback Window Derivation

After reading the person file:
1. If person.last_1on1_date exists and is a valid YYYY-MM-DD date:
   - lookback_start = person.last_1on1_date
   - lookback_source = "since last 1:1 on [date]"
2. Else if person.prep_run exists and is a valid YYYY-MM-DD date:
   - lookback_start = person.prep_run
   - lookback_source = "since last prep on [date]"
3. Else:
   - lookback_start = today - 14 days (compute from injected date)
   - lookback_source = "14-day default (no prior 1:1 date recorded)"
4. Record lookback_start and lookback_source for use in output header and MCP queries.
```

### State Write After Output (Phase E)

```
## Phase E — State Write

Execute only after prep sheet output is complete.

1. Take the person object loaded in Phase B (already in memory).
2. Set: person.last_1on1_date = [today's date from injected date]
3. Set: person.prep_run = [today's date from injected date]
4. Set: person.last_updated = [today's date from injected date]
5. Write the COMPLETE updated person object to .team-health/people/<slug>.json
   using the Write tool. Do NOT truncate entries, open_commitments, or career_context.
6. Confirm: "Prep sheet generated for [Name]. Next prep will look back from today."
```

Source: Phase 2 "atomic write" pattern from `log.md` Step 9; Phase 3 "output before state" rule from `pulse.md` Phase E.

### People Log Schema with New Fields

```json
{
  "schema_version": "1",
  "name": "Alice Chen",
  "slug": "alice-chen",
  "created": "2026-03-10",
  "last_updated": "2026-03-17",
  "last_1on1_date": "2026-03-17",
  "prep_run": "2026-03-17",
  "entries": [...],
  "open_commitments": [
    {
      "id": "2026-03-10-001",
      "date": "2026-03-10",
      "content": "Will connect Alice with the platform team lead by end of month.",
      "status": "open"
    }
  ],
  "career_context": {
    "stated_goals": ["Staff Engineer track", "More system design ownership"],
    "last_promo_discussion": "2026-01-20",
    "notes": ""
  }
}
```

Source: `.planning/phases/01-foundation/state-schemas.json` (base schema) + CONTEXT.md additions.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Fixed 7-day rolling window (pulse) | Dynamic "since last 1:1" window (prep) | Phase 4 decision (CONTEXT.md) | Prep covers exactly the gap between meetings, not an arbitrary calendar window |
| Raw data dump | Synthesized talking points in question form | Phase 4 decision (CONTEXT.md) | Prep reads like briefing notes, not a spreadsheet extract |
| One-way read of people log (log.md writes, others read) | Prep writes prep_run timestamp back to people log | Phase 4 decision (CONTEXT.md) | Zero-friction lookback anchor — the manager gets the right window automatically on next run |
| Signals-only output (pulse) | Signals + people log synthesis in one view (prep) | Phase 4 design | Prep is the first command to combine live MCP signals with qualitative log data |

---

## Open Questions

1. **Where does `last_1on1_date` live in the person file — top-level or inside `career_context`?**
   - What we know: `state-schemas.json` locked in Phase 1; `career_context` contains `stated_goals`, `last_promo_discussion`, `notes`; no prep-related fields exist
   - What's unclear: Whether `last_1on1_date` is operational metadata (top-level) or career-adjacent (inside `career_context`)
   - Recommendation: Top-level field on the person object — it's operational state, not career data. This keeps it distinct from career context and easier to read/write. The planner should adopt this placement.

2. **Should the prep command update baselines.json the way pulse does?**
   - What we know: Pulse updates baselines.json after every run; prep reads baselines.json but CONTEXT.md doesn't mention updating it
   - What's unclear: If prep collects fresh MCP signal data, should it contribute that data to the rolling baseline?
   - Recommendation: No — prep should be read-only for baselines.json. Updating baselines is pulse's responsibility; it runs weekly and handles window management. If prep also updates baselines, the window could include data from multiple sources with different lookback ranges, corrupting the rolling computation. Keep prep as a consumer of baselines, not a producer.

3. **How should the prep command handle a person with an empty people log but an existing baseline?**
   - What we know: MCP signals and baselines work without a people log; the people log is optional data
   - What's unclear: Should sections 3 and 5 be hidden entirely, or show placeholder text?
   - Recommendation: Always render all five sections; show "No people log entries recorded" in sections 3 and 5. This makes the output structure consistent and signals to the manager that they could start logging.

4. **What is the exact talking point ranking algorithm when there are more than 5 candidate sources?**
   - What we know: CONTEXT.md says 3–5 talking points; urgency ordering is Claude's discretion
   - What's unclear: Priority order when more than 5 talking points could be generated
   - Recommendation: Priority order — (1) RED signal flags, (2) overdue open commitments (>30 days open), (3) YELLOW signal flags, (4) career context overdue (>90 days since last promo discussion), (5) log patterns (same concern ≥3x in 6 weeks), (6) recent notable wins or positive signals worth acknowledging. Cap at 5, picking highest priority.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None — pure markdown/JSON command file, no runnable code |
| Config file | n/a |
| Quick run command | Manual: invoke `/team-health:prep <name>` in Claude Code with at least one MCP source configured and a person file present |
| Full suite command | Manual: run all 10 PREP scenarios in a project with Phase 1–3 setup complete |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PREP-01 | `/team-health:prep <name>` generates a scannable prep sheet | smoke/manual | Run command; verify output has 5 sections and fits within ~60 lines | ❌ Wave 0 |
| PREP-02 | Output has all five sections: status snapshot, signal flags, standing items, talking points, context reminders | smoke/manual | Inspect output structure; verify all 5 section headers present | ❌ Wave 0 |
| PREP-03 | GitHub signals appear in prep output when GitHub MCP is active | smoke/manual | Run with GitHub MCP; verify PR and commit data appears in signal flags section | ❌ Wave 0 |
| PREP-04 | Jira signals appear when Jira MCP is active; absent with degradation message when not | smoke/manual | Run with Jira disabled; verify degradation message; run with Jira enabled; verify ticket data | ❌ Wave 0 |
| PREP-05 | Calendar signals appear when Calendar MCP is active; absent otherwise | smoke/manual | Run with Calendar disabled; verify degradation message | ❌ Wave 0 |
| PREP-06 | Slack signals use public channel metadata only | smoke/manual | Verify output contains no DM references; verify Slack degradation message when Slack disabled | ❌ Wave 0 |
| PREP-07 | Open commitments surfaced from people log | smoke/manual | Add a commitment entry via `log.md`; run prep; verify commitment appears in section 3 | ❌ Wave 0 |
| PREP-08 | Career context appears: goals, time since last promo discussion | smoke/manual | Add a career entry via `log.md`; run prep; verify career context in section 5 | ❌ Wave 0 |
| PREP-09 | No diagnosis language in any section; all signals framed as observations | smoke/manual | Inspect output; verify no words: "burned out", "disengaged", "struggling", "checked out", "overwhelmed", "depressed", "anxious" | ❌ Wave 0 |
| PREP-10 | Prep sheet states which signals are absent when sources are unavailable | smoke/manual | Run with all sources false; verify all 4 unavailable sources named with PRIVACY.md degradation phrasing | ❌ Wave 0 |

### Additional Test Scenarios

| Scenario | Behavior | Verification |
|----------|----------|--------------|
| First use — no people log | Prep runs without error; sections 3 and 5 show "no log entries" | No error; sections present with placeholder text |
| First use — no prior prep_run | Lookback defaults to 14 days; header shows "(14-day default — no prior 1:1 date recorded)" | Header text matches pattern |
| prep_run written after output | Run prep; interrupt before confirmation line; re-run; verify lookback window is from original run date, not today | State write ordering verification |
| Talking point compliance check | No talking point contains diagnostic language | All talking points in question form; no psychological labels |

### Sampling Rate

- **Per task commit:** Verify deliverable exists with correct front matter, required phase structure sections, and five output sections defined
- **Per wave merge:** Run all 10 smoke scenarios in a fresh project with Phase 1–3 setup complete
- **Phase gate:** All 10 scenarios pass before marking Phase 4 complete

### Wave 0 Gaps

- [ ] `docs/testing/phase-4-smoke-tests.md` — step-by-step manual test procedure for all 10 PREP scenarios above
- [ ] `.claude/commands/team-health/prep.md` — the command file itself (primary deliverable)
- [ ] Confirm person schema extension (`last_1on1_date`, `prep_run` as top-level fields) — verify no conflict with `state-schemas.json` schema_version

*(No automated framework needed — no runnable code; all tests are manual CLI invocations)*

---

## Sources

### Primary (HIGH confidence)

- `.claude/commands/team-health/pulse.md` — Phase A–E structure, baseline comparison pattern, graceful degradation implementation, state write ordering (output before state), MCP source gating
- `.claude/commands/team-health/log.md` — Name resolution pattern, slug-from-config, pre-flight check boilerplate, people log schema, open_commitments structure, career_context fields, atomic write discipline
- `.claude/team-health/SIGNALS.md` — All 11 signals, thresholds, two-signal rule, severity levels — consumed verbatim by prep
- `.claude/team-health/BASELINES.md` — Baseline comparison algorithm, first-run behavior, window thinness rules — consumed verbatim by prep
- `.claude/team-health/PRIVACY.md` — Output language rules (4 rules), required disclaimer, graceful degradation language patterns
- `.planning/phases/01-foundation/state-schemas.json` — Locked person schema: entries, open_commitments, career_context structure; config.json team array with slug field
- `.planning/phases/04-1-1-prep/04-CONTEXT.md` — All locked decisions: lookback window logic, talking point sources and tone, patterns carried forward, Claude's discretion scope

### Secondary (MEDIUM confidence)

- `.planning/phases/03-team-pulse/03-RESEARCH.md` — MCP tool name findings, probe-by-namespace pattern, Calendar MCP maturity assessment (no official Calendar MCP as of March 2026)
- `.planning/REQUIREMENTS.md` — PREP-01 through PREP-10 requirement text; confirms no additional signal types beyond the 11 in SIGNALS.md

### Tertiary (LOW confidence — verify at implementation)

- MCP tool names for individual signal queries with dynamic date ranges — the prep lookback window is dynamic; test whether GitHub/Jira MCP tools accept arbitrary date ranges or only relative windows
- Calendar MCP availability — same blocker as Phase 3; treat `sources.calendar = false` as the expected default for most v1 users

---

## Metadata

**Confidence breakdown:**
- Command file format and front matter: HIGH — direct from Phase 1/2/3 implementations, unchanged conventions
- Pre-flight check, name resolution, reference reads: HIGH — verbatim boilerplate from existing commands
- People log read patterns: HIGH — directly from log.md implementation
- Lookback window derivation: HIGH — locked in CONTEXT.md; implementation is straightforward date comparison
- Signal collection (adapted from pulse.md): HIGH for pattern; MEDIUM for exact MCP tool call syntax with dynamic date filters
- Talking point generation rules: MEDIUM — design is new to Phase 4; first implementation; may need iteration
- Output format (prep sheet five sections): MEDIUM-HIGH — design derived from PREP-02 requirement and CONTEXT.md; not pre-existing
- MCP tool names with dynamic date ranges: MEDIUM — probe-by-namespace handles naming variation, but dynamic date range support needs verification per tool

**Research date:** 2026-03-17
**Valid until:** 2026-06-17 (stable foundations; locked schemas and reference docs will not change without schema_version bump; MCP landscape may evolve faster — re-verify Calendar MCP options at implementation)
