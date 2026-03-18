# team-health

## What This Is

A Claude Code skill for engineering managers that synthesizes real signals from connected tools (GitHub, Jira, Slack, Calendar) into situational, actionable intelligence — without relying on surveys or self-reporting. It provides 1:1 prep sheets, longitudinal people tracking, team pulse monitoring with early attrition/burnout warnings, sprint retro seeding, and upward reporting. Designed to be installed by any EM, not tied to any specific team setup.

## Core Value

Help managers be MORE human — remembering what their people care about, noticing when someone might be struggling, keeping their commitments — by surfacing what a good manager with infinite attention span would already notice.

## Requirements

### Validated

- ✓ `/team-health:prep <name>` — 1:1 prep sheet from tool signals + people log context — v1.0
- ✓ `/team-health:pulse` — weekly team health dashboard with per-person signal scoring vs. personal baselines — v1.0
- ✓ `/team-health:log <name>` — append-to and query a structured longitudinal people log — v1.0
- ✓ `/team-health:skip-level` — upward-facing team status brief for manager's own manager — v1.0
- ✓ `/team-health:retro-prep` — sprint retrospective agenda seeded with real data — v1.0
- ✓ Graceful degradation: detect available MCPs on first run, communicate capability gaps — v1.0
- ✓ First-run setup flow: collect team roster, 1:1 cadence, source preferences — v1.0
- ✓ Signal scoring vs. personal 8-week rolling baseline (not team averages), computed inline by Claude — v1.0
- ✓ Privacy-first output: signals not diagnoses, no DM content, no auto-surfacing of people log in skip-level — v1.0
- ✓ State storage in `.team-health/` directory within user's project — v1.0

### Active

(None — defining in next milestone)

### Out of Scope

- Runnable baseline computation scripts — Claude computes baselines inline each run
- Direct report-facing output — this is entirely manager-side tooling
- Psychological labeling or diagnosis of individuals
- Slack DM content surfacing (metadata only)
- Mobile or web UI — pure CLI/markdown output

## Context

Shipped v1.0 with 16,117 LOC across 74 files (markdown skill files, reference docs, prompt templates, planning artifacts).

Tech stack: Claude Code slash commands (markdown), MCP tool calls (GitHub, Jira, Slack, Calendar), JSON state files in `.team-health/`.

All 6 commands delivered: `/team-health:setup`, `/team-health:log`, `/team-health:pulse`, `/team-health:prep`, `/team-health:skip-level`, `/team-health:retro-prep`.

Target team size: 3–15 direct reports (typical EM span of control). Full functionality requires all four MCP sources; degrades gracefully to GitHub-only.

## Constraints

- **Distribution**: Shareable skill — must work for any EM regardless of their specific team or tool configuration
- **Privacy**: People data stays in the manager's context only; skip-level output excludes people log content unless explicitly chosen
- **Tone**: Chief-of-staff briefing, not HR surveillance. Every output should pass "would I be comfortable if my direct report saw this?"
- **Signal philosophy**: No single metric means anything in isolation; flags require multiple aligned signals or a strong outlier (>2 std devs from personal baseline)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Claude computes baselines inline | Simpler, no Python dependency, works across any install | ✓ Good — worked cleanly across all commands |
| Shareable skill (not personal) | Broader value, forces good defaults and graceful degradation | ✓ Good — shaped design of setup flow and graceful degradation |
| Signals not diagnoses | Legal/ethical safety, preserves manager judgment | ✓ Good — enforced via PRIVACY.md reference doc consumed by all commands |
| State in .team-health/ (user's project) | Keeps people data with the manager, not in skill install | ✓ Good — gitignore guard added in Phase 1 |
| Wave 0 smoke test contracts per phase | Establishes verification anchor before implementation | ✓ Good — each phase had explicit test scenarios to verify against |
| Privacy gate in skip-level: no people log by default | People log is for EM context only; skip-level is upward-facing | ✓ Good — `--include-person` explicit opt-in pattern |
| Work-item attribution in retro-prep (not person attribution) | Retro seeds should name PRs/tickets, not individuals | ✓ Good — enforced by command design |

---
*Last updated: 2026-03-18 after v1.0 milestone*
