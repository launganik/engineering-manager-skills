# Domain Pitfalls

**Domain:** Claude Code skills + MCP integrations + people analytics / EM tooling
**Project:** team-health
**Researched:** 2026-03-09
**Confidence note:** WebSearch and WebFetch were unavailable during this research session. All findings are drawn from training data (knowledge cutoff August 2025) plus direct analysis of the PROJECT.md constraints. Confidence levels reflect this limitation.

---

## Critical Pitfalls

Mistakes that cause rewrites, manager rejection, or harm to direct reports.

---

### Pitfall 1: Treating MCP Availability as Binary

**What goes wrong:** The skill assumes all four MCPs (GitHub, Jira, Slack, Calendar) are available or fails ungracefully when one is absent. Commands abort with cryptic errors. The manager who only has GitHub configured gets nothing.

**Why it happens:** Skills are written and tested against a full stack. The "detect what's available" logic is deferred to "later" and never gets built properly. The graceful degradation path is treated as an edge case rather than the norm.

**Consequences:** The skill works for the author and nobody else. Distribution fails. EMs with partial tool configs abandon it on first run.

**Prevention:**
- Build MCP detection and graceful degradation in Phase 1, not as a polish step.
- Every command must specify its MCP dependencies and what it does without each one.
- Test each command against: GitHub-only, GitHub+Jira, GitHub+Slack, all four, none.
- Use a `capabilities` object computed at session start that gates feature branches within each command.

**Detection (warning signs):**
- Setup instructions say "requires GitHub, Jira, Slack, and Calendar MCPs" without qualification.
- Commands have no fallback branches — single code path assumes all tools present.
- Testing only done with author's own MCP config.

**Phase:** Address in Phase 1 (foundation/setup flow). Every subsequent phase inherits the degradation framework.

---

### Pitfall 2: Conflating Signal with Diagnosis — The "Surveillance Label" Problem

**What goes wrong:** The skill surfaces an insight like "Jordan has been disengaged this week" or "Maria shows signs of burnout." The manager reads it, acts on it, and the direct report eventually finds out their manager is using an AI to psychologically profile them. The tool gets labeled as surveillance. The manager loses trust with their team. The skill gets uninstalled.

**Why it happens:** Developers optimize for insight sharpness and forget that the output will be read by a human who may act on it as fact, not hypothesis. Language like "shows signs of," "appears to be," "is likely experiencing" reads as diagnosis to a manager under stress.

**Consequences:** Direct report trust erosion if the tool's existence becomes known. Manager makes poor judgment calls based on AI-generated labels. In regulated industries (EU, CCPA-adjacent), could create legal exposure. Reputational damage to the skill itself.

**Prevention:**
- Every signal output must be framed as observable behavior, not internal state. "Commit frequency dropped 40% vs. 8-week baseline" not "Jordan seems disengaged."
- Add an explicit output review step in the prompt: "Before outputting, verify: does this read as observation or diagnosis?"
- Establish a phrase blocklist for the prompt templates: "signs of burnout," "disengaged," "struggling," "checked out," "low morale" — replace with behavioral descriptions.
- The PROJECT.md constraint ("signals not diagnoses") must be encoded into every command's system prompt, not just stated in docs.

**Detection (warning signs):**
- Output contains adjectives about internal states ("seems," "appears," "feels," "showing signs of").
- Skip-level output includes people log content without explicit manager opt-in.
- Any output that would be uncomfortable if the direct report saw it.

**Phase:** Address in every phase that generates output. Build a shared output review prompt fragment in Phase 1 and reuse it across all commands.

---

### Pitfall 3: Baseline Drift and Statistical Incoherence

**What goes wrong:** The rolling 8-week baseline is computed inconsistently across runs. Claude reads different data windows each time (depending on what GitHub/Jira returns), computes different averages, and generates contradictory signals. A person flagged as "unusually active" one week is flagged "unusually quiet" the next — for the same behavior.

**Why it happens:** Inline baseline computation (no external script) is powerful but fragile. If the data window fetched from MCP tools varies — due to API pagination, date range interpretation, or tool rate limits — the baseline shifts. Without a stable stored baseline, every computation is a one-shot estimate.

**Consequences:** Manager loses confidence in the signals. "The tool said Alex was doing great last week and now it's saying he's flagged — what changed?" The answer is "nothing, the baseline shifted." The skill becomes noise.

**Prevention:**
- Store computed baselines in `.team-health/baselines.json` after each successful pulse run. Use the stored baseline for signal comparison; only recompute the baseline itself on explicit refresh or after N weeks.
- Document the exact data window and source for each baseline value inside the JSON (e.g., `"computed_from": "2025-11-01 to 2026-01-24", "source": ["github", "jira"]`).
- Treat baseline recomputation as an explicit user action (`/team-health:pulse --rebaseline`), not automatic per run.
- When MCP data is incomplete (partial window), do not recompute — use stored baseline and note the gap.

**Detection (warning signs):**
- Baselines are computed fresh every command invocation.
- No version/timestamp stored with baseline values.
- Signals flip week-to-week for people whose actual behavior hasn't changed.

**Phase:** Address in Phase 2 (baseline + pulse). The baseline storage schema must be locked before the pulse command is built.

---

### Pitfall 4: State File Corruption and Schema Drift

**What goes wrong:** The `.team-health/` state files (people log, baselines, config, pulse history) are written as freeform JSON or markdown by Claude. After a few months of use, entries are inconsistent — some have fields others lack, schema has evolved across versions, and reads produce unexpected nulls. Querying the log fails silently.

**Why it happens:** Claude writes files conversationally. Without an enforced schema and a migration strategy, the format drifts with each new feature. Early entries have different structure than later ones. The skill has no version field in its state files.

**Consequences:** `/team-health:log <name>` returns garbled or incomplete history. Baseline computation reads malformed data and produces wrong averages. Manager loses longitudinal context — the core value proposition.

**Prevention:**
- Define all state file schemas explicitly in a reference doc (`STATE_SCHEMA.md`) before writing any commands that touch state.
- Include a `schema_version` field in every state file root.
- All writes must append (log) or replace (config, baseline) full valid entries — no partial writes.
- Each command that reads state must validate required fields before using them, with clear error messages when fields are missing.
- On schema version mismatch, prompt the manager to run a migration step rather than silently failing.

**Detection (warning signs):**
- People log entries have inconsistent fields across time.
- State files have no schema version.
- Commands silently succeed even when expected fields are absent.

**Phase:** Address in Phase 1 (state schema definition) before any command that writes state. All subsequent phases are blocked on a clean schema.

---

### Pitfall 5: Over-reliance on Single-Source Signals

**What goes wrong:** The skill surfaces a flag based on a single metric — PR count dropped this week, or no Slack messages on Friday. The manager has a 1:1 conversation with their report based on this single data point. The report explains they took Friday off (calendar not connected) and submitted one giant PR instead of many small ones. The manager looks foolish. The tool loses credibility.

**Why it happens:** Easier to implement and more dramatic-seeming to surface single-metric spikes. Multi-signal correlation is harder to reason about in a prompt.

**Consequences:** False positives erode manager trust rapidly. After two or three wrong flags, the tool is ignored. Signal fatigue sets in. The skill stops being used.

**Prevention:**
- The PROJECT.md constraint ("no single metric means anything in isolation; flags require multiple aligned signals or a strong outlier >2 std devs") must be encoded directly in the pulse command prompt, not just documented.
- Pulse output should explicitly show which signals are aligned before raising a flag. "3 of 4 signals suggest reduced engagement" is credible. "PR count is down" is not.
- When only one or two data sources are available (degraded mode), lower the sensitivity threshold or add a disclaimer: "Limited data sources — treat with caution."

**Detection (warning signs):**
- Pulse output flags someone based on a single metric.
- No "signal agreement" count shown alongside flags.
- Flags appear frequently (more than 1-2 per person per month in normal operation).

**Phase:** Address in Phase 2 (pulse command design). The multi-signal aggregation logic must be explicit in the prompt template.

---

### Pitfall 6: Skip-Level Output Leaking People Log Content

**What goes wrong:** `/team-health:skip-level` generates a brief for the manager's own manager. It pulls context from the people log — including sensitive 1:1 notes, personal context, or performance concerns — and surfaces them upward. The manager's manager now has information the direct reports never consented to share at that level.

**Why it happens:** The prompt for skip-level pulls from "all available context" without scoping what's excluded. The people log is available in context; Claude uses it.

**Consequences:** Direct reports feel surveilled. If it leaks (screenshots, forwarded emails), legal and HR consequences are possible. The manager who installed the tool is now in an awkward position.

**Prevention:**
- Skip-level command must explicitly exclude people log content from its context window — not rely on Claude to know not to use it.
- The prompt template for skip-level should scope its data sources: "Use only aggregate metrics, sprint outcomes, delivery data, and anything explicitly marked as skip-level-safe."
- Add an explicit manager opt-in gate: "This brief will include only delivery metrics and aggregate team health. To include specific context, run `/team-health:skip-level --include-context` and confirm."

**Detection (warning signs):**
- Skip-level output mentions individual people's situations, personal context, or 1:1 notes.
- The skip-level prompt pulls from people log without filtering.
- No distinction between aggregate and individual-level data in the prompt template.

**Phase:** Address when skip-level command is built. The data scoping rule must be in the prompt template from day one.

---

### Pitfall 7: MCP Rate Limits and Stale Data Silently Corrupting Pulse

**What goes wrong:** GitHub, Jira, or Slack MCPs hit rate limits mid-run or return cached/stale data. Claude proceeds with partial data and computes a pulse as if it were complete. The manager gets a pulse output that silently reflects only 60% of the data window. Signals are wrong but confidently presented.

**Why it happens:** MCP tool call failures are treated as exceptions rather than expected conditions. The skill has no mechanism to detect partial data collection.

**Consequences:** Pulse output is wrong but looks authoritative. Downstream decisions are made on bad data. The skill loses trust when the manager notices inconsistencies.

**Prevention:**
- Every MCP data collection step must return a data completeness signal: how many records were fetched, what date range, whether the call succeeded fully.
- The pulse prompt should include a pre-computation step: "Summarize data completeness before analyzing signals. If any source returned fewer than expected records or failed, flag this in the output."
- Treat rate-limit failures as "source unavailable" for that run — fall back to stored baseline, do not recompute from partial data.

**Detection (warning signs):**
- Pulse runs silently succeed even when MCP calls time out or return errors.
- No data completeness summary in pulse output.
- Signals vary dramatically across runs without behavioral changes.

**Phase:** Address in Phase 2 (pulse). Data completeness checks must be in the prompt before the signal computation step.

---

## Moderate Pitfalls

---

### Pitfall 8: People Log Becoming a Liability Document

**What goes wrong:** The people log accumulates candid managerial notes over months. The manager writes things like "Alex is clearly checked out, thinking about whether to PIP," or "Jordan mentioned their marriage is struggling." This log is stored in the project directory — it is discoverable in a legal or HR investigation.

**Why it happens:** The log interface is too permissive. The EM treats it like a private diary. No guidance on what's appropriate to record.

**Prevention:**
- The log command prompt and documentation must include explicit guidance: "This log is not attorney-client privileged and may be discoverable. Record observable behaviors and outcomes, not diagnoses or personal disclosures."
- The `/team-health:log` command should surface this reminder on first use and annually.
- Suggest a "review and prune" step when logs reach a certain age.

**Phase:** Address when log command is built. The guidance must be in the UI, not just the docs.

---

### Pitfall 9: First-Run Setup Abandonment

**What goes wrong:** The first-run setup flow asks too many questions upfront. The manager answers 3 of 12 questions, gets distracted, and never finishes. The skill is in a half-configured state. Subsequent runs fail with confusing errors about missing config.

**Why it happens:** Setup flows are designed by developers who have infinite patience. Real users have zero tolerance for long onboarding when they're not yet sure the tool is worth it.

**Prevention:**
- Design setup as progressive disclosure: minimal required config first (team roster, primary GitHub org), everything else optional and prompted contextually.
- The skill should be usable with minimal config — `/team-health:prep` with just a name and GitHub org, no Jira, no Slack.
- Save config incrementally after each question, so partial setup is resumable.
- Limit required setup questions to 3 or fewer.

**Phase:** Address in Phase 1 (setup flow). The flow must be validated against a "zero patience" user before shipping.

---

### Pitfall 10: Skill Prompt Context Window Bloat

**What goes wrong:** Each command loads the full people log for all team members, full baseline history, and full pulse history into the prompt before doing its analysis. For a team of 15 with 6 months of history, this consumes most of the context window before Claude has done any work. Quality degrades. Responses truncate.

**Why it happens:** Loading "all available context" is simpler to implement. Selective loading requires thinking about what each command actually needs.

**Prevention:**
- Each command specifies its exact state reading scope: `/team-health:prep <name>` reads only that person's log, not all logs.
- Pulse reads only the baseline snapshot and recent metrics, not full log history.
- Log entries should have a compact summary format for bulk loading and a full format for single-person queries.
- Design state file schemas with this distinction in mind from Phase 1.

**Phase:** Address in Phase 1 (schema design) and enforce in each command's prompt template.

---

### Pitfall 11: Skill Installed as Root CLAUDE.md (Global State Bleed)

**What goes wrong:** The skill is installed at the user's global CLAUDE.md level. Now every Claude Code session — including non-EM work — has team-health context loaded. The skill's state paths conflict with project-specific usage. The "team" being tracked bleeds across different projects.

**Why it happens:** Easy to install globally; scoping to per-project is a documentation and design problem.

**Prevention:**
- Skill documentation must explicitly recommend project-level installation, not global.
- State paths (`./team-health/`) are relative to project root — document this clearly.
- The first-run setup flow should warn if it detects it's running from a global context.

**Phase:** Address in Phase 1 (installation design and documentation).

---

## Minor Pitfalls

---

### Pitfall 12: Calendar MCP Timezone Handling

**What goes wrong:** Calendar MCP returns meeting times in UTC or the server timezone, not the manager's or report's local timezone. Meeting load analysis is wrong — a report in London appears to have all their meetings at midnight.

**Prevention:**
- Prompt templates for calendar analysis must specify timezone normalization: "Interpret all timestamps in the context of [user's timezone from config]."
- Store user timezone in config during setup.
- When timezone is unknown, note this limitation in the output.

**Phase:** Address when any calendar-based analysis is implemented.

---

### Pitfall 13: Jira Sprint Boundary Ambiguity

**What goes wrong:** Jira sprint data doesn't cleanly map to calendar weeks. If a sprint ends mid-week, ticket velocity metrics straddle two reporting periods. Pulse output for "this week" may mix two sprints or miss a sprint boundary entirely.

**Prevention:**
- Pulse prompt must specify whether it's analyzing by sprint or by calendar week, and document which sprint is "current."
- Config should store sprint cadence (2-week, 3-week) to help Claude reason about boundaries.
- When sprint data is ambiguous, note it explicitly in output rather than silently averaging.

**Phase:** Address in Phase 2 (pulse command, Jira integration).

---

### Pitfall 14: Retro Prep Command Creating a Paper Trail

**What goes wrong:** `/team-health:retro-prep` generates an agenda seeded with real data including individual-level signals. This agenda gets shared in a team retrospective meeting. People in the retro now know the manager is tracking individual-level signals. The tool's existence creates a team culture problem.

**Prevention:**
- Retro prep output must aggregate signals to team level — no individual names attached to flags.
- Document clearly: retro-prep generates team-level talking points, not individual-level reports.
- Prompt template must anonymize: "Describe themes at the team level. Do not attribute signals to specific individuals."

**Phase:** Address when retro-prep command is built.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Phase 1: Foundation / Setup | First-run abandonment (Pitfall 9) | Progressive disclosure, max 3 required questions |
| Phase 1: State schema | Schema drift (Pitfall 4) | Lock schema + version field before writing any state commands |
| Phase 1: Installation docs | Global install bleed (Pitfall 11) | Explicit project-level install recommendation |
| Phase 2: Pulse command | Baseline incoherence (Pitfall 3) | Store baselines after first run, use stored for comparison |
| Phase 2: Pulse command | Single-source false positives (Pitfall 5) | Multi-signal agreement gate before flagging |
| Phase 2: Pulse command | Stale/partial MCP data (Pitfall 7) | Data completeness check before signal computation |
| Phase 2: Pulse command | Diagnosis language (Pitfall 2) | Behavioral language constraint in every output prompt |
| Phase 2: MCP integration | Rate limits corrupting pulse (Pitfall 7) | Treat tool errors as "source unavailable," not exceptions |
| Phase 3: 1:1 prep | Diagnosis framing in prep output (Pitfall 2) | Shared output review prompt fragment |
| Phase 3: 1:1 prep | Context window bloat (Pitfall 10) | Scope to single person's log only |
| Phase 4: Log command | People log as liability doc (Pitfall 8) | Legal guidance in UI on first use |
| Phase 4: Log command | Context window bloat on log reads (Pitfall 10) | Compact summary format for bulk, full format for single |
| Phase 5: Skip-level | People log leaking upward (Pitfall 6) | Explicit source scoping in skip-level prompt |
| Phase 6: Retro prep | Individual signals in shared doc (Pitfall 14) | Team-level aggregation only, no names |
| All phases | MCP availability assumptions (Pitfall 1) | Capabilities detection from Phase 1, inherited everywhere |

---

## Sources and Confidence

| Pitfall | Confidence | Basis |
|---------|------------|-------|
| MCP binary availability (1) | HIGH | Direct analysis of PROJECT.md + known MCP/Claude Code behavior patterns |
| Diagnosis vs. signal language (2) | HIGH | Well-documented failure mode in HR analytics; PROJECT.md explicitly names it |
| Baseline drift (3) | HIGH | Statistical/state management principle; PROJECT.md architecture implies the risk |
| State file schema drift (4) | HIGH | Universal problem with LLM-written state files; strongly implied by architecture |
| Single-source false positives (5) | HIGH | PROJECT.md explicitly names this constraint |
| Skip-level people log leak (6) | HIGH | Direct inference from PROJECT.md privacy constraints |
| MCP rate limits / stale data (7) | MEDIUM | Known MCP behavior pattern; verification would require current MCP docs |
| People log as liability doc (8) | HIGH | Well-established legal principle for employment documentation |
| First-run setup abandonment (9) | HIGH | Universal UX failure mode; well-documented in SaaS research |
| Context window bloat (10) | HIGH | Fundamental Claude Code constraint; universal for state-heavy skills |
| Global install bleed (11) | MEDIUM | Inferred from Claude Code's CLAUDE.md scoping model |
| Calendar timezone handling (12) | MEDIUM | Known API integration problem; specifics depend on MCP implementation |
| Jira sprint boundary ambiguity (13) | MEDIUM | Known Jira data model issue; specifics depend on MCP implementation |
| Retro prep individual attribution (14) | HIGH | Directly follows from privacy-first constraint in PROJECT.md |

**Note:** WebSearch and WebFetch were unavailable during this research session. Pitfalls were derived from: (1) direct analysis of PROJECT.md constraints and architecture decisions, (2) training knowledge of Claude Code skill patterns and MCP integration behavior through August 2025, (3) established principles in HR analytics tooling, employment law, and statistical signal processing. External verification of MCP-specific behaviors (rate limits, data completeness APIs) was not possible and those findings are marked MEDIUM confidence.
