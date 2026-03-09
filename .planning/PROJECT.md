# team-health

## What This Is

A Claude Code skill for engineering managers that synthesizes real signals from connected tools (GitHub, Jira, Slack, Calendar) into situational, actionable intelligence — without relying on surveys or self-reporting. It provides 1:1 prep sheets, longitudinal people tracking, team pulse monitoring with early attrition/burnout warnings, sprint retro seeding, and upward reporting. Designed to be installed by any EM, not tied to any specific team setup.

## Core Value

Help managers be MORE human — remembering what their people care about, noticing when someone might be struggling, keeping their commitments — by surfacing what a good manager with infinite attention span would already notice.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] `/team-health:prep <name>` — 1:1 prep sheet from tool signals + people log context
- [ ] `/team-health:pulse` — weekly team health dashboard with per-person signal scoring vs. personal baselines
- [ ] `/team-health:log <name>` — append-to and query a structured longitudinal people log
- [ ] `/team-health:skip-level` — upward-facing team status brief for manager's own manager
- [ ] `/team-health:retro-prep` — sprint retrospective agenda seeded with real data
- [ ] Graceful degradation: detect available MCPs on first run, communicate capability gaps
- [ ] First-run setup flow: collect team roster, 1:1 cadence, source preferences
- [ ] Signal scoring vs. personal 8-week rolling baseline (not team averages), computed inline by Claude
- [ ] Privacy-first output: signals not diagnoses, no DM content, no auto-surfacing of people log in skip-level
- [ ] State storage in `.team-health/` directory within user's project

### Out of Scope

- Runnable baseline computation scripts — Claude computes baselines inline each run
- Direct report-facing output — this is entirely manager-side tooling
- Psychological labeling or diagnosis of individuals
- Slack DM content surfacing (metadata only)
- Mobile or web UI — pure CLI/markdown output

## Context

The skill is delivered as a set of markdown skill files (SKILL.md per command, reference docs, prompt templates). It uses MCP tool calls to pull data from GitHub, Jira, Slack, and Calendar. State is written to `.team-health/` in the user's project directory — people logs, baseline snapshots, pulse history, config.

Target team size: 3–15 direct reports (typical EM span of control).

The skill must work across different team setups. Any EM should be able to install it and have it detect what's available and guide setup. Full functionality requires all four MCP sources; it degrades gracefully down to GitHub-only.

Baseline computation: Claude reads historical data from `.team-health/baselines.json` (maintained as a simple rolling window) and computes rolling averages + standard deviations inline. No external scripts.

## Constraints

- **Distribution**: Shareable skill — must work for any EM regardless of their specific team or tool configuration
- **Privacy**: People data stays in the manager's context only; skip-level output excludes people log content unless explicitly chosen
- **Tone**: Chief-of-staff briefing, not HR surveillance. Every output should pass "would I be comfortable if my direct report saw this?"
- **Signal philosophy**: No single metric means anything in isolation; flags require multiple aligned signals or a strong outlier (>2 std devs from personal baseline)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Claude computes baselines inline | Simpler, no Python dependency, works across any install | — Pending |
| Shareable skill (not personal) | Broader value, forces good defaults and graceful degradation | — Pending |
| Signals not diagnoses | Legal/ethical safety, preserves manager judgment | — Pending |
| State in .team-health/ (user's project) | Keeps people data with the manager, not in skill install | — Pending |

---
*Last updated: 2026-03-09 after initialization*
