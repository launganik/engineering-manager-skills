# Roadmap: team-health

## Overview

The skill is built in strict dependency order: reference docs and state schemas first (nothing else can exist without them), then the people log which is the memory layer all commands consume, then team pulse which writes the baselines all prep depends on, then 1:1 prep which synthesizes everything, then skip-level and retro-prep which consume pulse history. Five phases, each delivering a complete and independently usable capability. Skipping or reordering phases produces broken states — baselines would not exist when prep runs, privacy gates would not exist when skip-level runs.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Foundation** - Reference docs, state schemas, setup flow, and MCP capability detection (completed 2026-03-11)
- [x] **Phase 2: People Log** - `/team-health:log` command — the memory layer all other commands consume (completed 2026-03-13)
- [x] **Phase 3: Team Pulse** - `/team-health:pulse` with baseline computation and storage (completed 2026-03-17)
- [x] **Phase 4: 1:1 Prep** - `/team-health:prep` synthesizing people log and baselines (completed 2026-03-17)
- [x] **Phase 5: Skip-Level and Retro Prep** - `/team-health:skip-level` and `/team-health:retro-prep` consuming pulse history (completed 2026-03-17)

## Phase Details

### Phase 1: Foundation
**Goal**: The skill can be installed by any EM, detects what MCP sources are available, guides first-run setup, and provides all shared reference docs and locked state schemas that subsequent commands depend on
**Depends on**: Nothing (first phase)
**Requirements**: SETUP-01, SETUP-02, SETUP-03, SETUP-04, SETUP-05, REF-01, REF-02, REF-03, REF-04
**Success Criteria** (what must be TRUE):
  1. An EM can install the skill and run first-time setup by providing team roster and 1:1 cadence — config is saved and all subsequent commands skip setup
  2. Running any command without completing setup routes the user to the setup flow instead of erroring
  3. The skill detects which of GitHub, Jira, Slack, and Calendar MCPs are available and communicates which signals will be absent
  4. All state files (`config.json`, `people/<name>.json`, `baselines.json`) have documented schemas with `schema_version` fields
  5. `.team-health/` is gitignored by default; `SIGNALS.md`, `BASELINES.md`, `PRIVACY.md`, and `SKILL.md` exist and are readable
**Plans**: 4 plans

Plans:
- [ ] 01-00-PLAN.md — Wave 0 scaffolding: smoke test checklist + canonical state schemas
- [ ] 01-01-PLAN.md — Setup command: /team-health:setup with MCP detection, roster collection, config.json write, gitignore guard
- [ ] 01-02-PLAN.md — Reference docs: SIGNALS.md, BASELINES.md, PRIVACY.md
- [ ] 01-03-PLAN.md — Installation guide: top-level SKILL.md

### Phase 2: People Log
**Goal**: Managers can record and query longitudinal notes about each direct report — the memory layer that 1:1 prep and commitment tracking depend on
**Depends on**: Phase 1
**Requirements**: LOG-01, LOG-02, LOG-03, LOG-04, LOG-05, LOG-06
**Success Criteria** (what must be TRUE):
  1. Manager can run `/team-health:log <name>` and append a free-form note — the skill structures it with date, category tag, and content in the person's log file
  2. Opening the log shows the last 3-5 entries as context before prompting for new input
  3. Manager can ask natural language queries ("when did I last discuss promotion with Alex?") and get accurate answers from the log
  4. Running the log command for a person with no existing log creates the file gracefully without errors
**Plans**: 2 plans

Plans:
- [ ] 02-00-PLAN.md — Wave 0: phase-2 smoke test document (manual test scenarios for LOG-01 through LOG-06)
- [ ] 02-01-PLAN.md — /team-health:log command: append + query modes, Pre-flight Check, name resolution, category inference, commitment dual-write

### Phase 3: Team Pulse
**Goal**: Managers can run a weekly team health scan that scores each person against their personal 8-week baseline, flags anomalies conservatively, and stores baselines and pulse history for downstream commands
**Depends on**: Phase 1
**Requirements**: PULSE-01, PULSE-02, PULSE-03, PULSE-04, PULSE-05, PULSE-06, PULSE-07, PULSE-08, PULSE-09, PULSE-10
**Success Criteria** (what must be TRUE):
  1. Manager can run `/team-health:pulse` and receive a dashboard with a per-person green/yellow/red summary table — flagged individuals have detail sections, unflagged do not
  2. Every flag states the specific metric, delta, and source ("PR review lag: +3.2 days vs. 8-week avg") — no opaque scores
  3. Flags only fire when 2+ signals align or a single signal exceeds 2 standard deviations from the person's personal baseline — not team averages
  4. After each run, computed baselines are stored in `baselines.json` with `computed_from`, `source`, and `schema_version` provenance fields
  5. Pulse output includes the disclaimer about signals not being diagnoses and degrades gracefully when MCP sources are absent, naming which signals are unavailable
**Plans**: 2 plans

Plans:
- [ ] 03-00-PLAN.md — Wave 0: phase-3 smoke test document (manual test scenarios for PULSE-01 through PULSE-10)
- [ ] 03-01-PLAN.md — /team-health:pulse command: full scan pipeline, baseline comparison, two-signal flagging, dashboard output, baselines.json write, pulse-history snapshot

### Phase 4: 1:1 Prep
**Goal**: Managers can generate a scannable 2-minute prep sheet for any direct report that synthesizes live tool signals against personal baselines and surfaces standing items from the people log
**Depends on**: Phase 2, Phase 3
**Requirements**: PREP-01, PREP-02, PREP-03, PREP-04, PREP-05, PREP-06, PREP-07, PREP-08, PREP-09, PREP-10
**Success Criteria** (what must be TRUE):
  1. Manager can run `/team-health:prep <name>` and receive a structured prep sheet with: status snapshot, signal flags, standing items from people log, suggested talking points, and context reminders
  2. All signal flags are framed as behavioral observations ("PR review participation down 40% vs. 8-week baseline") — no diagnostic language
  3. Open manager commitments and IC asks are surfaced from the people log; career context (goals, time since last promotion discussion) is included
  4. The prep sheet degrades gracefully to available MCP sources and states which signals are absent
**Plans**: 2 plans

Plans:
- [ ] 04-00-PLAN.md — Wave 0: phase-4 smoke test document (manual test scenarios for PREP-01 through PREP-10)
- [ ] 04-01-PLAN.md — /team-health:prep command: per-person prep sheet with signal snapshot, people log synthesis, talking points, lookback window, prep_run state write

### Phase 5: Skip-Level and Retro Prep
**Goal**: Managers can generate an upward-facing team brief for their own manager and a data-seeded sprint retro agenda — both consuming pulse history with explicit privacy gates that prevent people log content from surfacing without opt-in
**Depends on**: Phase 3
**Requirements**: SKIP-01, SKIP-02, SKIP-03, SKIP-04, SKIP-05, RETRO-01, RETRO-02, RETRO-03, RETRO-04
**Success Criteria** (what must be TRUE):
  1. Manager can run `/team-health:skip-level` and receive a team-level brief (delivery status, risks, themes, escalation asks, wins) — people log content is excluded by default and individual names do not appear in pulse aggregations without explicit `--include-person` opt-in
  2. The skip-level output passes the "would I be comfortable if my entire team saw this?" test by design
  3. Manager can run `/team-health:retro-prep` and receive a retro agenda seeded with sprint facts and discussion prompts attributed to work items, never to individuals
  4. Retro output preserves blank space for team conversation — data seeds the agenda, it does not fill it
**Plans**: 3 plans

Plans:
- [ ] 05-00-PLAN.md — Wave 0: phase-5 smoke test document (manual test scenarios for SKIP-01 through SKIP-05, RETRO-01 through RETRO-04)
- [ ] 05-01-PLAN.md — /team-health:skip-level command: privacy-gated team brief with pulse history aggregation, anonymized themes, lookback window, last_skip_level state write
- [ ] 05-02-PLAN.md — /team-health:retro-prep command: sprint retro agenda with work-item-attributed seeds, blank discussion space + SKILL.md update

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation | 4/4 | Complete   | 2026-03-11 |
| 2. People Log | 2/2 | Complete   | 2026-03-13 |
| 3. Team Pulse | 2/2 | Complete   | 2026-03-17 |
| 4. 1:1 Prep | 2/2 | Complete   | 2026-03-17 |
| 5. Skip-Level and Retro Prep | 3/3 | Complete   | 2026-03-17 |
