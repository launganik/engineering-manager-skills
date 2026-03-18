# Project Research Summary

**Project:** team-health (Claude Code skill for engineering managers)
**Domain:** Claude Code custom skill with MCP integration and longitudinal people analytics
**Researched:** 2026-03-09
**Confidence:** MEDIUM - Stack and architecture patterns are HIGH confidence from direct Claude Code knowledge; MCP server maturity and community package maintenance are MEDIUM; features and pitfall thresholds draw on established EM practice (HIGH confidence on fundamentals, MEDIUM on specific calibrations).

## Executive Summary

The team-health skill is a Claude Code custom command set that gives engineering managers signal-based 1:1 prep, team health dashboards, and longitudinal people logs - all from data already flowing through their existing tools (GitHub, Jira, Slack, Google Calendar). The expert approach for this domain is: pure markdown command files registered as Claude Code slash commands, file-based JSON/Markdown state in `.team-health/` (no database, no external service), MCP servers as the data layer, and inline Claude reasoning for baseline statistics. The recommended stack requires zero runtime dependencies beyond Claude Code itself, which is a deliberate constraint that enables simple installation and keeps all people data with the manager.

The central architectural challenge is multi-source graceful degradation: the skill must deliver value with GitHub alone and incrementally unlock capability as Jira, Slack, and Calendar MCPs are added. This degradation logic must be built first - not retrofitted. The second major challenge is the privacy model: every output must frame signals as observable behavioral patterns (not psychological diagnoses), the people log must never surface in skip-level output by default, and Slack access must remain limited to channel metadata, never DM content. Both constraints are non-negotiable - violations erode manager-IC trust and create legal exposure.

The recommended build order, derived from hard architectural dependencies, is: foundation (reference docs + state schemas + setup flow) first, then the people log command, then team pulse with baseline computation, then 1:1 prep which consumes both, then skip-level and retro-prep which build on pulse history. Skipping this order is the most common way skills in this category fail - specifically, building prep before baselines exist leads to signals computed against nothing, and building skip-level without a locked privacy gate leads to people log content leaking upward.

## Key Findings

### Recommended Stack

The skill is purely markdown files - no build step, no npm package, no binary. Claude Code custom slash commands are registered from `.claude/commands/team-health/*.md` files, each containing YAML front matter (description, allowed-tools) and a complete prompt template. Shared logic (signal taxonomy, baseline math, privacy rules) lives in reference documents the skill prompts read at runtime, enabling logic updates without touching all command files.

State lives in `.team-health/` (JSON for structured data, Markdown for people logs), gitignored to prevent accidental commit of people data. The four MCP servers handle data collection: GitHub MCP is official and stable, Slack MCP is official, Jira and Calendar MCPs are community-maintained and need verification at implementation time. MCP configuration is a user/workspace concern - the skill detects what's available via probe calls and degrades accordingly.

**Core technologies:**
- Claude Code custom commands (`.claude/commands/team-health/*.md`): slash command registration - no runtime needed, markdown IS the skill
- JSON state files (`.team-health/`): structured state for config, baselines, pulse history - machine-readable, diffable, zero infrastructure
- Markdown people logs (`.team-health/people/`): longitudinal per-person notes - human-readable, appendable, privacy-appropriate
- `@modelcontextprotocol/server-github`: PR activity, review participation, commit cadence - official, stable
- `@modelcontextprotocol/server-slack`: channel metadata only (not DM content) - official, Slack API rate limits are the real constraint
- Community Jira MCP: ticket velocity and blocker signals - verify package name and maintenance status at implementation time
- Community Google Calendar MCP: meeting load and 1:1 adherence signals - LOW confidence on availability; may require building a thin wrapper

### Expected Features

See `.planning/research/FEATURES.md` for full signal specifications per command.

**Must have (table stakes):**
- Team roster configuration - skill is inoperable without knowing who the team is
- 1:1 prep sheet generation - the primary entry point; combines people log + GitHub signals + baseline comparison
- Per-person signal summary framed as behavioral observations, not diagnoses - non-negotiable for legal/ethical safety
- People log (`/team-health:log`) - standing topics, follow-up tracking, manager commitments; the memory layer
- Graceful degradation to available MCP sources - skill must not fail when only GitHub is configured

**Should have (differentiators):**
- Personal baseline scoring (8-week rolling window) - flags changes vs. the person's own norm, not team averages; this is the core analytical value
- Multi-signal convergence requirement - flags only when 2+ signals align or one is >2 std devs; prevents alarm fatigue
- Attrition/disengagement early warning composite - the "killer feature"; requires multi-source data and multi-week patterns
- Upward reporting mode (`/team-health:skip-level`) - delivery health, team mood themes, asks to escalate; explicitly privacy-gated from people log
- Sprint retro seeding (`/team-health:retro-prep`) - populates retro with real cycle time, blocked ticket data, carry-over patterns

**Defer (v2+):**
- Calendar integration - adds value but GitHub + Jira + people log already deliver core value; Calendar MCP maturity is uncertain
- After-hours commit analysis - high-sensitivity feature; needs multi-signal convergence logic mature before introducing
- Automated calendar-triggered prep - "day before 1:1" trigger; nice-to-have, not MVP

### Architecture Approach

The skill has four distinct layers: (1) entry point command files that define per-command prompt templates, (2) reference documents that encode shared domain logic (signal taxonomy, baseline math, privacy rules) and are read by skills at runtime, (3) persistent state files in `.team-health/` that accumulate longitudinal data, and (4) MCP data sources queried at runtime. Commands are self-contained and do not call each other - shared logic propagates through reference doc reads. This keeps each command independently debuggable while allowing shared logic to be updated in one place.

**Major components:**
1. Skill entry points (`.claude/commands/team-health/*.md`) - five command prompt templates; each defines full data gathering, computation, and output instructions
2. Reference documents (`.claude/team-health/`) - `SIGNALS.md`, `BASELINES.md`, `PRIVACY.md`, `SETUP.md`; shared domain logic updated independently of command files
3. Persistent state (`.team-health/`) - `config.json` (roster, cadences, MCP availability), `people/<name>.json` (people logs), `baselines.json` (rolling statistics), `pulse-history/` and `retro-history/` (run outputs)
4. MCP data sources (GitHub, Jira, Slack, Calendar) - external; queried at runtime; skill detects availability via probe calls and adjusts output scope accordingly

### Critical Pitfalls

1. **MCP availability treated as binary** - Build the capability detection and graceful degradation pattern in Phase 1. Every subsequent phase inherits it. Test every command against GitHub-only and no-MCP configurations, not just full-stack.

2. **Diagnosis language in outputs** - Enforce behavioral framing in every prompt template: "PR review participation down 40% vs. 8-week baseline" not "Jordan seems disengaged." Build a shared output review prompt fragment in Phase 1 and include it in every command that generates output.

3. **Baseline drift from inconsistent computation** - Store computed baselines in `baselines.json` after each pulse run. Use the stored baseline for comparison; recompute only on explicit `--rebaseline` flag or after N weeks. Never recompute from partial MCP data. Include `computed_from` and `source` provenance fields.

4. **State file schema drift** - Define all state file schemas with `schema_version` fields before writing any commands that touch state. All writes follow the canonical schema. Schema changes trigger a migration prompt, not silent failure.

5. **People log leaking into skip-level output** - The skip-level command must explicitly scope its data sources to aggregate metrics and pulse history; it must not read `people/` at all. This is a prompt-level enforcement, not a judgment call left to Claude.

6. **First-run setup abandonment** - Limit required setup questions to 3 or fewer (team roster, primary GitHub org, one optional MCP). Save config incrementally so partial setup is resumable. The skill must be usable with minimal config before all MCPs are wired.

## Implications for Roadmap

Based on combined research, the architectural dependency chain dictates a clear 5-phase build order. Deviating from this order produces broken states that are expensive to recover from.

### Phase 1: Foundation - Reference Docs, State Schemas, and Setup Flow

**Rationale:** Every subsequent command reads reference docs and writes state. Neither can exist without a locked schema and shared logic layer. The setup gate (checking `config.json: setup_complete`) must exist in all commands from day one - retrofitting it is fragile. This phase also locks in the privacy model and output language rules before any output-generating commands are built.

**Delivers:** `.claude/team-health/SIGNALS.md`, `BASELINES.md`, `PRIVACY.md`, `SETUP.md`; canonical state schemas for `config.json`, `people/<name>.json`, `baselines.json`; first-run setup flow (`/team-health:setup` or setup gate in all commands); MCP probe/capability detection pattern; skill installation docs (`SKILL.md`, `install.md`)

**Addresses:** Team roster config (table stakes), graceful degradation framework (table stakes)

**Avoids:** State schema drift (Pitfall 4), first-run abandonment (Pitfall 9), global install bleed (Pitfall 11), MCP binary availability assumption (Pitfall 1)

**Research flag:** Standard patterns - no additional research phase needed. Claude Code command file structure and JSON state are well-documented patterns.

### Phase 2: People Log Command (`/team-health:log`)

**Rationale:** The people log is the memory layer all other commands depend on. It has no MCP dependencies, so it can be built and validated in isolation. It proves out state write/read patterns before more complex commands touch state. It also delivers immediate standalone value - EMs can start logging before any MCP is configured.

**Delivers:** `/team-health:log <name>` with append and query modes; per-person `.team-health/people/<name>.json`; legal guidance prompt on first use; compact summary vs. full-format loading strategy for context window management

**Addresses:** Standing topics / follow-up tracking (table stakes), people log longitudinal categories (differentiator)

**Avoids:** People log as liability document (Pitfall 8), context window bloat on log reads (Pitfall 10)

**Research flag:** Standard patterns - no additional research needed. Append-and-query log pattern is straightforward.

### Phase 3: Team Pulse with Baseline Computation (`/team-health:pulse`)

**Rationale:** Pulse must come before prep because prep depends on baselines. The pulse command is the baseline writer - it computes and stores the 8-week rolling window. Without at least one pulse run, prep has no baseline to compare against. Pulse is also the most architecturally complex command (multi-source MCP, inline statistics, state writes) and sets the pattern all later commands follow.

**Delivers:** `/team-health:pulse`; baseline computation and storage in `baselines.json`; pulse history snapshots; multi-signal aggregation logic with convergence thresholds; data completeness checks before signal computation; per-source degradation messaging

**Addresses:** Personal baseline scoring (differentiator), multi-signal convergence requirement (differentiator), team pulse dashboard (table stakes)

**Avoids:** Baseline drift (Pitfall 3), single-source false positives (Pitfall 5), MCP rate limits corrupting pulse (Pitfall 7), diagnosis language (Pitfall 2)

**Research flag:** Needs validation - Jira MCP package name, maintenance status, and sprint boundary data model should be verified before implementation. Calendar MCP availability assessment needed.

### Phase 4: 1:1 Prep Command (`/team-health:prep`)

**Rationale:** Prep is the primary user-facing entry point but depends on both people log (Phase 2) and baselines (Phase 3). Building it fourth ensures it has real data to synthesize. This is where the signal-to-talking-point synthesis is most visible and where output language discipline is most critical.

**Delivers:** `/team-health:prep <name>`; 5-section output format (status snapshot, signal flags, standing items, suggested talking points, context reminders); single-person scoped data loading; behavioral framing enforcement via shared output review fragment

**Addresses:** 1:1 prep sheet generation (table stakes), per-person signal summary (table stakes), suggested talking points (differentiator)

**Avoids:** Diagnosis framing (Pitfall 2), context window bloat scoped to single person (Pitfall 10), baseline drift (inherits Phase 3 fix)

**Research flag:** Standard patterns - no additional research phase needed. Signal synthesis prompt design is the main work; patterns are established.

### Phase 5: Skip-Level Brief and Retro Prep (`/team-health:skip-level`, `/team-health:retro-prep`)

**Rationale:** Both commands are consumers of pulse history (written in Phase 3). Skip-level reads aggregate pulse data and produces upward-facing output; retro-prep reads sprint-window pulse data and seeds a team conversation. Building these last means the privacy gate logic for skip-level can be validated against real pulse history data, and retro-prep can leverage mature Jira integration from Phase 3.

**Delivers:** `/team-health:skip-level` with explicit people log exclusion gate and opt-in flag for specific context; `/team-health:retro-prep` with team-level aggregation only (no individual attribution); `retro-history/` output files

**Addresses:** Upward reporting mode (differentiator), sprint retro seeding (differentiator), privacy-gated skip-level output (differentiator)

**Avoids:** People log leaking into skip-level (Pitfall 6), individual signals in shared retro doc (Pitfall 14)

**Research flag:** Standard patterns - privacy gate logic is already fully specified. No additional research phase needed.

### Phase Ordering Rationale

The dependency chain is strict: reference docs and schemas must precede all state-writing commands; people log must precede prep (provides log context); pulse must precede prep (provides baselines) and must precede skip-level and retro-prep (provides pulse history). This ordering also front-loads the highest-risk architectural decisions (state schemas, MCP degradation, privacy model) before any user-facing commands are built, reducing rewrite risk.

The privacy model and output language rules are established in Phase 1 and applied to every subsequent phase. This is not a polish step - it is a foundation constraint. Building any output-generating command before the privacy rules are codified in reference docs risks baking in diagnostic framing that is expensive to purge later.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (Pulse / MCP integration):** Jira MCP package name, version, and maintenance status must be verified before implementation. Google Calendar MCP availability and OAuth flow need evaluation - may require building a thin wrapper. Slack MCP rate limits and workspace permission model need verification.

Phases with standard patterns (skip research-phase):
- **Phase 1 (Foundation):** Claude Code command file structure, JSON state, and YAML front matter patterns are well-established.
- **Phase 2 (People Log):** Append-and-query log with structured JSON is a standard pattern.
- **Phase 4 (1:1 Prep):** Signal synthesis prompt design follows documented EM practice and DORA metric frameworks.
- **Phase 5 (Skip-Level / Retro Prep):** Privacy gate and team-level aggregation patterns are fully specified in research.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Claude Code command file structure, MCP config pattern, and file-based state are direct training knowledge. Community Jira and Calendar MCP packages are MEDIUM - verify at implementation. |
| Features | MEDIUM-HIGH | Core EM practice (1:1 prep, people logs, DORA signals) is HIGH confidence from established literature. Specific signal thresholds (e.g., 30% contribution drop = flag) are opinionated defaults that should be tuned with real users. |
| Architecture | MEDIUM | Core layer structure and data flow patterns are HIGH confidence. MCP probe call behavior and `allowed-tools` front matter key syntax need verification against current Claude Code docs before Phase 1. |
| Pitfalls | HIGH | Most pitfalls are derived from first principles (state management, privacy law, statistical signal processing, UX research on setup abandonment). MCP-specific behaviors (rate limits, data completeness APIs) are MEDIUM - not externally verified. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- **`allowed-tools` front matter syntax:** Verify that `allowed-tools` is the correct YAML key in Claude Code command files (vs. `tools` or similar). Verify YAML front matter is supported at all in the current Claude Code version. Resolve before Phase 1 ships any command files.
- **Jira MCP package selection:** Multiple community implementations exist. Evaluate maintenance status, API coverage (sprint velocity, blocked tickets), and authentication model before Phase 3. Backup plan: Atlassian REST API calls if no maintained MCP is available.
- **Google Calendar MCP viability:** Community-built; availability and OAuth flow are uncertain. Evaluate in Phase 3 before committing to calendar-based features. Fallback: defer calendar integration to post-MVP.
- **Slack MCP scoping:** Verify that the official Slack MCP returns metadata (not DM content) by default, or requires explicit API scope configuration. This is a privacy constraint - resolve before Slack integration in Phase 3.
- **Baseline calibration thresholds:** The specific thresholds (30% drop = flag, 2 std devs = alert, 3 signals = escalate) are opinionated defaults based on HR analytics literature. These should be treated as starting points and made configurable in `config.json` for real-world tuning.

## Sources

### Primary (HIGH confidence)
- PROJECT.md (authoritative project requirements and constraints)
- Claude Code documentation (training knowledge, August 2025 cutoff) - command file structure, `.claude/commands/` path, YAML front matter
- MCP specification and official servers (training knowledge) - GitHub MCP, Slack MCP
- Manager Tools 1:1 methodology - 1:1 prep structure and standing topics model
- DORA metrics (Forsgren et al., "Accelerate") - delivery health signal definitions
- Will Larson ("An Elegant Puzzle"), Lara Hogan - people log category framework

### Secondary (MEDIUM confidence)
- HR analytics tooling feature survey (Waydev, LinearB, Swarmia, Jellyfish, Lattice, Leapsome) - feature landscape and anti-features; based on training data, not current product audits
- Engineering productivity signal literature - attrition pattern indicators; well-established domain, specific thresholds are opinionated
- SaaS onboarding research - setup abandonment failure mode; well-documented UX principle

### Tertiary (LOW confidence - verify at implementation)
- Community Jira MCP packages (npm: `mcp-server-jira`) - verify current package name, maintenance status, sprint data coverage
- Community Google Calendar MCP (npm: `mcp-google-calendar`) - verify current availability and OAuth model
- MCP rate limit behavior and data completeness APIs - not externally verified; handle defensively

---
*Research completed: 2026-03-09*
*Ready for roadmap: yes*
