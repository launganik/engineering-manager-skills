# Phase 4: 1:1 Prep - Context

**Gathered:** 2026-03-17
**Status:** Ready for planning

<domain>
## Phase Boundary

Generate a per-person 2-minute prep sheet by synthesizing live MCP signals (GitHub, Jira, Slack, Calendar) against personal baselines and surfacing standing items from the people log. Single command: `/team-health:prep <name>`. Skip-level and retro prep are separate phases.

</domain>

<decisions>
## Implementation Decisions

### Lookback window
- Use "since last 1:1" as the primary signal window - reads `last_1on1_date` from the person's people log entry or config
- Fallback when no prior date is recorded: 14-day default
- After generating prep, auto-append a `prep_run` timestamp to the people log so the next invocation knows where to look back from
- Do NOT prompt the manager to enter the date - infer it automatically

### Talking points
- Claude generates 3-5 talking points from the data - not raw data only
- Sources for talking point generation (all four):
  1. Signal flags: anomalies from baseline comparison → "PR review lag up 40% - ask about workload"
  2. Open commitments: unresolved manager commitments from the log → "You committed to intro-ing Alex to design - did that happen?"
  3. Career context: goals, time since last promo discussion, role aspirations from people log
  4. Log patterns: recurring themes in recent entries (same concern mentioned 3x in 6 weeks)
- Tone: frame as questions to ask, not scripts - stays in manager's voice
- Never diagnose; always behavioral observation → question form

### Patterns carried forward from prior phases
- Pre-flight check (config.json + setup_complete gate) before executing
- Required Reference Reads: SIGNALS.md, BASELINES.md, PRIVACY.md - must be read before any signal collection or output
- Slug always resolved from config.json team array, never re-derived from argument
- Graceful degradation language from PRIVACY.md when sources unavailable
- Output (prep sheet rendered) before state writes (prep_run timestamp appended)
- `disable-model-invocation: true` front matter

### Claude's Discretion
- Exact prep sheet formatting (spacing, headers, visual weight)
- How many log entries to surface as "standing items" vs. summary
- Whether to group talking points by topic (signals / commitments / career) or rank by urgency

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Signal definitions and thresholds
- `.claude/team-health/SIGNALS.md` - all 11 signals, per-source grouping, flag thresholds, two-signal rule

### Baseline computation
- `.claude/team-health/BASELINES.md` - 8-week rolling baseline algorithm, first-run behavior, provenance fields

### Output language rules
- `.claude/team-health/PRIVACY.md` - mandatory language rules, signals-not-diagnoses framing, graceful degradation language, disclaimer

### Reference implementations (existing commands)
- `.claude/commands/team-health/pulse.md` - Phase A→E structure, baseline comparison pattern, graceful degradation implementation, state write ordering
- `.claude/commands/team-health/log.md` - people log schema, name resolution pattern, slug-from-config pattern, open_commitments structure, career_context fields

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- Pre-flight check block (from `log.md` / `pulse.md`): config.json gate - copy verbatim
- Name resolution block (from `log.md`): slug-from-config, fuzzy prefix match, multi-match disambiguation - copy verbatim
- Required Reference Reads block (from `pulse.md`): SIGNALS.md + BASELINES.md + PRIVACY.md read pattern - copy verbatim
- Graceful degradation per-source check (from `pulse.md` Phase A): sources map + PRIVACY.md language - adapt for prep context

### Established Patterns
- Phase A→E execution structure (pulse.md): pre-flight → reference reads → setup → per-person data collection → output → state writes
- Atomic file write: build full content in memory, single Write call - prevent partial writes
- Output before state: render prep sheet fully before appending prep_run timestamp
- `last_1on1_date` field: must be read from people log's `career_context` or a prep-specific field (new field to define)

### Integration Points
- `.team-health/people/<slug>.json` - people log files; `open_commitments`, `career_context`, `entries` arrays consumed by prep
- `.team-health/baselines.json` - read for per-person baseline values (same as pulse)
- `.team-health/config.json` - sources map, team array, 1:1 cadence per person

</code_context>

<specifics>
## Specific Ideas

- "Since last 1:1" anchor is critical - the prep should feel like it's covering the gap between meetings, not an arbitrary 7-day window
- Talking points should feel like a trusted chief-of-staff briefing you before a meeting, not a data dump
- The auto-recording of prep_run timestamp is a zero-friction design choice - manager gets the right lookback automatically next time

</specifics>

<deferred>
## Deferred Ideas

None - discussion stayed within phase scope.

</deferred>

---

*Phase: 04-1-1-prep*
*Context gathered: 2026-03-17*
