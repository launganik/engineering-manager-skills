# Phase 1: Foundation - Research

**Researched:** 2026-03-10
**Domain:** Claude Code skill authoring, file-based state schemas, MCP availability detection, reference document patterns
**Confidence:** HIGH — All critical questions verified against current official Claude Code documentation (code.claude.com/docs, March 2026)

---

## Summary

Phase 1 delivers the structural skeleton every subsequent phase depends on: command files registered as Claude Code skills, locked JSON state schemas, a first-run setup flow, MCP capability detection, and four reference documents encoding shared domain logic. Nothing else can be built until this layer exists — state schema drift and missing setup gates are the two failure modes most likely to require rewrites later.

The prior project research flagged two questions as needing verification before Phase 1 shipped: (1) whether `allowed-tools` is the correct YAML front matter key, and (2) whether `.claude/commands/team-health/*.md` creates the `/team-health:` namespace. Both are now confirmed. The `allowed-tools` key is correct per current Claude Code skill documentation. The subdirectory-to-colon-namespace mapping is confirmed: `.claude/commands/team-health/prep.md` registers as `/team-health:prep`. Additionally, the ecosystem has evolved — skills (`.claude/skills/<name>/SKILL.md`) are now the preferred authoring format, but the old `.claude/commands/` path continues to work and both are equivalent. This phase should adopt the skills format for new files to access supporting-file bundling, while understanding that legacy command files are still valid.

The primary design choice for Phase 1 is whether to use `.claude/commands/team-health/*.md` (legacy, simpler) or `.claude/skills/team-health-*/SKILL.md` (modern, supports bundled supporting files). The skills format is recommended for this phase because reference documents can be co-located as supporting files rather than requiring a separate `.claude/team-health/` directory tree. This reduces install surface.

**Primary recommendation:** Use `.claude/skills/team-health-{command}/SKILL.md` for each command, with reference docs as supporting files in the skill directory. Legacy `.claude/commands/` flat files work but do not support co-located supporting files.

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| SETUP-01 | Skill detects which MCP sources are available (GitHub, Jira, Slack, Calendar) on first command invocation and reports capability gaps | MCP probe pattern confirmed: attempt minimal tool call per source, handle tool-not-available gracefully; Claude sees only configured tools in its tool list |
| SETUP-02 | First-run setup flow collects team roster (names, roles, 1:1 cadence) and stores in `.team-health/config.json` | Standard JSON write via `Write` tool; schema defined in this phase |
| SETUP-03 | All commands check `config.json: setup_complete` gate before executing and route to setup if not configured | Pattern: "Step 0: Read .team-health/config.json; if missing or setup_complete !== true, follow setup flow before proceeding" in every SKILL.md |
| SETUP-04 | State file schemas defined with `schema_version` fields for `config.json`, `people/<name>.json`, `baselines.json` | Schemas defined in this research; locked before any command writes state |
| SETUP-05 | `.team-health/` directory is gitignored by default | Requires writing a `.gitignore` entry; setup flow should check and append if absent |
| REF-01 | `SIGNALS.md` reference document defines each signal, how it's computed, and thresholds | Reference doc authored in this phase; loaded by later command skills |
| REF-02 | `BASELINES.md` reference document defines rolling 8-week baseline computation methodology | Reference doc authored in this phase; provides inline computation instructions for Claude |
| REF-03 | `PRIVACY.md` reference document encodes output language rules | Reference doc authored in this phase; loaded by every command skill that generates output |
| REF-04 | `SKILL.md` installation guide covers MCP prerequisites, first-run steps, and what each command does | Top-level `SKILL.md` (or `README.md`) for the skill repo — human-facing, not loaded by Claude Code natively |
</phase_requirements>

---

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| Claude Code Skills (SKILL.md files) | Current (code.claude.com) | Slash command registration | Native mechanism — directory of .md files IS the skill; no runtime, no build, no package |
| YAML front matter in SKILL.md | n/a | Command metadata: description, allowed-tools, argument-hint, disable-model-invocation | Controls skill behavior, autocomplete hints, tool restrictions — verified against official docs |
| JSON files in `.team-health/` | n/a | Structured state: config, baselines, pulse history | Machine-readable by Claude, human-inspectable, no infrastructure required |
| Markdown files in `.team-health/people/` | n/a | People logs (longitudinal notes per person) | Human-readable, appendable, appropriate for free-form managerial notes |
| `.gitignore` entry for `.team-health/` | n/a | Privacy invariant: people data never in VCS | Required — logs contain personal employment observations |

### Supporting

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| `!`bash-command`` syntax in SKILL.md | n/a | Dynamic context injection at invocation time | Use sparingly — runs before Claude sees the prompt; good for injecting current date, reading a file pre-load |
| `$ARGUMENTS` / `$0`, `$1` | n/a | Pass invocation arguments to skill content | Use in skills that take a person name argument: `/team-health:prep alex` |
| `${CLAUDE_SKILL_DIR}` | n/a | Reference skill-bundled files by absolute path | Use in skills that reference supporting files co-located with SKILL.md |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `.claude/skills/team-health-{cmd}/SKILL.md` | `.claude/commands/team-health/{cmd}.md` | Legacy command files work identically but cannot bundle supporting files in the same directory. Skills format preferred for new development. |
| Separate `.claude/team-health/` reference dir | Supporting files in skill directory | Skills format lets reference docs live alongside SKILL.md, reducing install surface. But: reference docs shared by multiple skills cannot be co-located — put them in `.claude/team-health/` or reference them via relative paths from each skill's directory. |

**Installation — no package needed:**

```bash
# Clone the skill repo into the manager's project directory
git clone https://github.com/your-org/antigravity-skills .

# Or copy the skill files
cp -r /path/to/antigravity-skills/.claude ./

# Run first-time setup in Claude Code
/team-health:setup
```

---

## Architecture Patterns

### Recommended Project Structure

The skills format (current) and commands format (legacy) produce equivalent results. Use skills format for Phase 1:

```
.claude/
  skills/
    team-health-setup/
      SKILL.md              # /team-health:setup command
      references/
        SIGNALS.md          # Signal taxonomy (read by prep, pulse)
        BASELINES.md        # Baseline computation guide (read by pulse)
        PRIVACY.md          # Output language rules (read by all)

    team-health-prep/
      SKILL.md              # /team-health:prep <name>

    team-health-pulse/
      SKILL.md              # /team-health:pulse

    team-health-log/
      SKILL.md              # /team-health:log <name>

    team-health-skip-level/
      SKILL.md              # /team-health:skip-level

    team-health-retro-prep/
      SKILL.md              # /team-health:retro-prep

.team-health/               # Per-manager state (gitignored contents)
  config.json               # Team roster, cadences, MCP availability, setup_complete
  baselines.json            # 8-week rolling window per person per signal
  people/
    <slug>.json             # Per-person structured log (Phase 2+)
  pulse-history/
    YYYY-WNN.md             # Weekly pulse snapshots (Phase 3+)
  retro-history/
    YYYY-MM-DD.json         # Sprint retro seeds (Phase 5+)

SKILL.md                    # Skill-level README / installation guide (human-facing)
```

**Note on command naming:** Because `.claude/skills/` uses directory names for skill names, `team-health-setup` creates `/team-health-setup` (hyphen) not `/team-health:setup` (colon). To get the colon-namespace `/team-health:setup`, use the legacy commands path:

```
.claude/commands/team-health/setup.md   → /team-health:setup
.claude/commands/team-health/prep.md    → /team-health:prep
.claude/commands/team-health/pulse.md   → /team-health:pulse
```

**Decision required:** The project uses colon-namespaced commands (`/team-health:prep`). This means using `.claude/commands/team-health/` (legacy format), not `.claude/skills/` (modern format). The legacy format supports the same YAML front matter but cannot bundle supporting files in a sibling directory. Reference documents must live in a separate `.claude/team-health/` directory.

**Recommended resolution:** Use `.claude/commands/team-health/` for commands (colon namespace), and `.claude/team-health/` for reference documents. Both are part of the skill's distribution artifact.

### Pattern 1: SKILL.md / Command File Structure

**What:** Each slash command is a markdown file with YAML front matter followed by prompt instructions.

**Verified front matter keys (from official docs, March 2026):**

```yaml
---
name: team-health-setup           # Optional; uses filename/dirname if omitted
description: "First-run setup..."  # Recommended: Claude uses for auto-discovery
argument-hint: "[name]"            # Optional: shown in autocomplete
allowed-tools: Read, Write, Bash(git *)  # Optional: tools usable without approval
disable-model-invocation: true     # Optional: prevent Claude auto-invoking; use for side-effect commands
user-invocable: true               # Optional: false hides from / menu
context: fork                      # Optional: "fork" runs in isolated subagent
model: sonnet                      # Optional: override model for this skill
---

Prompt instructions here...
```

**For team-health commands:** All setup/config-modifying commands should use `disable-model-invocation: true` to prevent Claude from running them automatically. Information-only commands (prep, pulse) can use defaults.

**Example — setup command:**

```yaml
---
description: "First-run setup for team-health skill. Collects team roster, 1:1 cadence, and detects available MCP sources. Run once per manager."
argument-hint: ""
allowed-tools: Read, Write, Bash(git log --oneline -1)
disable-model-invocation: true
---

## /team-health:setup

Step 0: Check if .team-health/config.json exists and has setup_complete: true.
If yes, report "Setup already complete. Run with --reset to reconfigure." and stop.

Step 1: Detect MCP availability by attempting minimal probe calls:
...
```

### Pattern 2: Setup Gate in Every Command

**What:** Every command file begins with a check for `config.json: setup_complete` before doing any other work. If absent or false, it routes to the setup flow.

**When to use:** All commands that depend on the team roster (prep, pulse, log, skip-level, retro-prep).

**Implementation (verbatim block to copy into every command's SKILL.md):**

```
## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.
```

**Why not a shared include:** Claude Code skill files are self-contained — one skill cannot `import` another. The gate must be copy-pasted into each command file. This is intentional; reference docs (read at runtime) carry shared logic, but the gate is load-bearing prompt structure.

### Pattern 3: MCP Availability Detection

**What:** Claude only sees tools from configured MCP servers in its tool list. If a server is not configured, its tools simply do not appear. There is no error; the tool is absent.

**Runtime behavior (HIGH confidence from MCP docs):**
- Claude Code reads MCP server configurations at session start
- Each configured MCP server's tools are loaded into Claude's available tool list
- If a server is not configured or fails to connect, none of its tools appear
- Claude cannot call a tool that is not in its tool list — attempting to would fail at the tool invocation level

**Detection approach:** The setup flow instructs Claude to attempt a minimal call to each MCP's most-fundamental tool and observe whether it succeeds or returns a "tool not found" / connection error. Write results to `config.json`'s `sources` block.

```
MCP Probe Instructions (for use in /team-health:setup):

For each source, attempt the listed probe and record the result:

1. GitHub: Use the github MCP list_repositories tool (or equivalent minimal read tool).
   - If it succeeds: sources.github = true
   - If the tool does not exist or errors: sources.github = false
   - Note: "GitHub MCP not connected — PR/commit signals will be unavailable."

2. Jira: Use the jira MCP list_projects tool.
   - If it succeeds: sources.jira = true
   - If the tool does not exist or errors: sources.jira = false
   - Note: "Jira MCP not connected — ticket velocity/blocker signals will be unavailable."

3. Slack: Use the slack MCP list_channels tool.
   - If it succeeds: sources.slack = true
   - If the tool does not exist or errors: sources.slack = false
   - Note: "Slack MCP not connected — participation metadata signals will be unavailable."

4. Calendar: Use the gcal or google-calendar MCP list_events tool.
   - If it succeeds: sources.calendar = true
   - If the tool does not exist or errors: sources.calendar = false
   - Note: "Calendar MCP not connected — meeting load and 1:1 adherence signals will be unavailable."

After probing all sources, present a capabilities summary to the user before proceeding.
```

**Important:** The exact tool names for each MCP server depend on which MCP server package is installed. The setup flow should instruct Claude to list available tools from each source namespace and report what it finds, rather than assuming specific tool names. This makes the detection robust to different community implementations.

### Pattern 4: Reference Document Injection

**What:** Commands are instructed to read specific reference documents at the start of their execution. This separates evolving domain logic (signal weights, baseline math, privacy rules) from command structure.

**When:** Every command that generates scored output (prep, pulse) or output with privacy implications (skip-level, retro-prep).

**Pattern block to include in command SKILL.md:**

```
## Required Reference Reads

Before doing any analysis, read these files using the Read tool:
- .claude/team-health/SIGNALS.md — signal taxonomy, scoring weights, and thresholds
- .claude/team-health/PRIVACY.md — mandatory output language rules

If this command computes or reads baselines, also read:
- .claude/team-health/BASELINES.md — baseline computation and comparison methodology
```

### Pattern 5: Progressive Setup Disclosure

**What:** The setup flow asks only the minimum required questions to reach a usable state. Optional configuration is prompted contextually on first use.

**Required questions (maximum 3):**
1. Team roster: names and roles of direct reports
2. Primary GitHub organization or username (for MCP targeting)
3. Default 1:1 cadence in weeks (default: 1)

**Everything else is optional** and can be added via `config.json` manually or through future commands.

### Anti-Patterns to Avoid

- **Importing one skill from another:** Claude Code skills cannot call each other. Shared logic must be in reference docs read at runtime.
- **Setup questions in every command:** The gate should route to the setup command, not re-ask setup questions inline. Keep the gate a hard stop.
- **Storing people data in git:** `.team-health/` must be gitignored. The setup flow should check for and create the `.gitignore` entry.
- **Using `context: fork` for stateful commands:** Forked subagents do not share the parent conversation context. Commands that write state (setup, pulse) should run inline, not forked.
- **Hardcoding MCP tool names:** Different community MCP implementations expose different tool names. Probe by namespace, not by specific tool name.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Rolling mean/stddev computation | Python script, external service | Claude inline arithmetic over `baselines.json` window array | The PROJECT.md explicitly rules out Python. Claude handles simple statistics over ≤8 values × ≤15 people × ≤10 signals = ≤1,200 data points without accuracy concerns. |
| Slash command registration | Custom CLI tooling, npm package | `.claude/commands/team-health/*.md` files | This IS how Claude Code skills work — markdown files are the registration mechanism |
| MCP server configuration | In-skill config wizard | User's own `claude mcp add` commands + documented in SKILL.md | MCP config is a user/workspace concern, not a skill concern. The skill detects what's there; it doesn't configure it. |
| People data database | SQLite, Postgres, external API | Flat JSON + Markdown in `.team-health/` | No infrastructure, no auth, human-inspectable, works offline, keeps data with the manager |
| Schema validation | Zod, JSON Schema library | Explicit field checks in each read instruction + `schema_version` field | Claude can validate field presence inline; no library needed for this data volume |

**Key insight:** This skill has zero runtime dependencies beyond Claude Code itself. Every "infrastructure" problem is solved by the operating environment (Claude's tools) or by being unnecessary at this data scale.

---

## Concrete State Schemas

These schemas are locked in Phase 1. All subsequent phases read/write these exact shapes.

### `config.json` Schema

```json
{
  "schema_version": "1",
  "setup_complete": true,
  "created": "2026-03-10",
  "last_updated": "2026-03-10",
  "manager": {
    "name": "Jordan Lee",
    "github_username": "jlee",
    "timezone": "America/New_York"
  },
  "team": [
    {
      "name": "Alice Chen",
      "slug": "alice-chen",
      "role": "Senior Engineer",
      "github_username": "achen",
      "jira_user_id": "alice@company.com",
      "slack_user_id": "U0123ALICE",
      "1on1_cadence_weeks": 1,
      "start_date": "2024-06-01"
    }
  ],
  "sources": {
    "github": true,
    "jira": false,
    "slack": true,
    "calendar": false
  },
  "baseline_window_weeks": 8,
  "github_org": "acme-corp",
  "sprint_cadence_weeks": 2,
  "last_skip_level": null
}
```

**Required fields for setup_complete: true:** `schema_version`, `setup_complete`, `team` (non-empty array), `sources`, `github_org`.

### `people/<slug>.json` Schema

```json
{
  "schema_version": "1",
  "name": "Alice Chen",
  "slug": "alice-chen",
  "created": "2026-03-10",
  "last_updated": "2026-03-15",
  "entries": [
    {
      "id": "2026-03-15-001",
      "date": "2026-03-15",
      "category": "feedback-given",
      "content": "Gave specific positive feedback on the auth refactor PR — clean API design.",
      "tags": ["technical", "positive"],
      "created_at": "2026-03-15T14:32:00Z"
    }
  ],
  "open_commitments": [
    {
      "id": "2026-03-15-002",
      "date": "2026-03-15",
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

**Valid categories:** `feedback-given`, `feedback-received`, `career`, `commitment`, `concern`, `win`, `note`

### `baselines.json` Schema

```json
{
  "schema_version": "1",
  "last_updated": "2026-03-10",
  "people": {
    "alice-chen": {
      "computed_from": "2025-12-01",
      "computed_to": "2026-02-23",
      "source": ["github", "jira"],
      "window_weeks": 8,
      "metrics": {
        "github_prs_merged_per_week": {
          "window": [2, 3, 1, 2, 2, 3, 2, 2],
          "mean": 2.125,
          "stddev": 0.6,
          "last_value": 2
        },
        "github_pr_review_count_per_week": {
          "window": [4, 5, 3, 4, 5, 4, 4, 5],
          "mean": 4.25,
          "stddev": 0.7,
          "last_value": 4
        },
        "github_commit_days_per_week": {
          "window": [4, 5, 4, 4, 3, 5, 4, 4],
          "mean": 4.125,
          "stddev": 0.6,
          "last_value": 4
        },
        "jira_tickets_closed_per_week": {
          "window": [3, 2, 3, 4, 3, 2, 3, 3],
          "mean": 2.875,
          "stddev": 0.6,
          "last_value": 3
        }
      }
    }
  }
}
```

**Notes on schema design:**
- `window` arrays hold at most `window_weeks` values; oldest is dropped when a new value is added
- `mean` and `stddev` are precomputed and stored to avoid recomputation on every read
- `computed_from` / `computed_to` provide provenance — if MCP data was partial, this is auditable
- `source` lists which MCPs contributed to this baseline — if a source is removed, downstream reads know the baseline may be stale

---

## Common Pitfalls

### Pitfall 1: Schema Drift Before Commands Are Written

**What goes wrong:** The setup command writes `config.json` in one shape; the pulse command reads it expecting different field names. After a few phases, `sources.github` vs `source.github` vs `sources["github"]` creates silent failures.

**Why it happens:** Schemas evolve during implementation without a canonical reference.

**How to avoid:** Lock the schemas above before writing any command SKILL.md. Commands reference the canonical schema in this document. Changes to schema require a version bump (`schema_version: "2"`) and a migration note.

**Warning signs:** A command reads a field that is undefined — particularly `config.setup_complete` vs `config.setup_complete` vs a nested path.

### Pitfall 2: Setup Gate Missing from a Command

**What goes wrong:** A later command (pulse, prep) runs before setup is complete and fails to find team roster data. Instead of routing to setup, it produces an empty or broken output.

**Why it happens:** Each command is written in isolation; the gate block is not copy-pasted in.

**How to avoid:** The "Pre-flight Check" block above is mandatory in every command SKILL.md. Treat it as boilerplate — include it verbatim, do not paraphrase.

### Pitfall 3: Using `disable-model-invocation: false` on State-Writing Commands

**What goes wrong:** Claude automatically invokes the setup or pulse command in the background because the description matches a user's question. The setup flow runs unsolicited, overwriting existing config.

**Why it happens:** Default behavior allows Claude to auto-invoke any skill without this flag.

**How to avoid:** Set `disable-model-invocation: true` in the front matter of any skill that writes state (`setup`, `pulse`, `log`). Information-only skills (`prep`, `skip-level`) can use the default.

### Pitfall 4: MCP Tool Name Assumptions

**What goes wrong:** The setup probe is written to call `mcp_github_list_repositories` specifically. But the user has installed a different GitHub MCP implementation that exposes `github_list_repos`. The probe fails even though GitHub is connected.

**Why it happens:** Community MCP server implementations use different tool naming conventions.

**How to avoid:** Write probe instructions that check for the namespace/prefix (e.g., "any github-prefixed tool") rather than a specific tool name. Alternatively, instruct Claude to run `/mcp` internally and report what tools are listed per namespace.

### Pitfall 5: Gitignore Not Created on Installation

**What goes wrong:** Manager clones the skill, runs setup, and their `.team-health/config.json` gets committed to the project repo. GitHub indexing can pick up team roster data and 1:1 notes.

**Why it happens:** `.gitignore` is not modified during setup. The directory is created but not protected.

**How to avoid:** The setup command's first write action must check for `.gitignore` and append `.team-health/` if absent. Instructions should also tell the manager to verify this manually.

---

## Code Examples

Verified patterns from official sources:

### Command File with Full Front Matter

```yaml
---
description: "First-time setup for team-health skill. Collects team roster, 1:1 cadence, and detects available MCP sources. Must be run before any other team-health command."
argument-hint: ""
allowed-tools: Read, Write, Bash(git status)
disable-model-invocation: true
---

## /team-health:setup

[prompt body here]
```

Source: [code.claude.com/docs/en/skills — Frontmatter reference](https://code.claude.com/docs/en/skills)

### Colon-Namespace from Subdirectory

```
.claude/commands/team-health/setup.md     →  /team-health:setup
.claude/commands/team-health/prep.md      →  /team-health:prep
.claude/commands/team-health/pulse.md     →  /team-health:pulse
.claude/commands/team-health/log.md       →  /team-health:log
.claude/commands/team-health/skip-level.md  →  /team-health:skip-level
.claude/commands/team-health/retro-prep.md  →  /team-health:retro-prep
```

Source: [danielcorin.com/til/anthropic/custom-slash-commands-hierarchy](https://www.danielcorin.com/til/anthropic/custom-slash-commands-hierarchy/)

### Arguments in Command Files

```yaml
---
description: "Generate a 1:1 prep sheet for a direct report."
argument-hint: "[name]"
allowed-tools: Read, Write
disable-model-invocation: true
---

Generate a 1:1 prep sheet for $ARGUMENTS.

...
```

### Dynamic Context Injection (Pre-invocation Bash)

```yaml
---
name: team-health-pulse
description: "Weekly team health pulse"
allowed-tools: Read, Write
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`

Run the weekly team health pulse...
```

Source: [code.claude.com/docs/en/skills — Inject dynamic context](https://code.claude.com/docs/en/skills)

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `.claude/commands/*.md` flat files | `.claude/skills/<name>/SKILL.md` directory structure | 2025-2026 | Skills support bundled supporting files; commands format still works and is equivalent for basic use |
| Separate `SKILL.md` concept in older GSD framework | SKILL.md is now the native Claude Code skill entry point | 2025 | SKILL.md as skill manifest is now official, not just a GSD convention |
| Manual MCP tool permission approval per use | `allowed-tools` in front matter pre-approves tool use | Confirmed current | Critical for skills that call MCP tools repeatedly — prevents approval dialogs per call |
| Plugin namespace: `plugin-name:skill-name` | Colon namespace from subdirectory in commands format | Confirmed current | `/team-health:prep` pattern confirmed via subdirectory structure |

**Key clarification:** SKILL.md (the file at the root of this repo used as an installation guide) and SKILL.md (the entry point file in a `.claude/skills/<name>/` directory) are now the same concept. The GSD framework's convention and the Claude Code native format have converged. The top-level `SKILL.md` in the skill repo serves as both a human-readable installation guide and, if placed correctly, can be Claude Code's skill entrypoint.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None detected — this is a pure markdown/JSON skill with no runnable code |
| Config file | n/a |
| Quick run command | Manual: invoke each command in Claude Code and verify output |
| Full suite command | Manual: run all commands in a fresh project with no config, then with full config |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SETUP-01 | MCP detection reports correct availability | smoke/manual | Run `/team-health:setup` without any MCPs configured; verify all four report as unavailable | ❌ Wave 0 |
| SETUP-01 | MCP detection reports GitHub available when GitHub MCP configured | smoke/manual | Run `/team-health:setup` with GitHub MCP configured; verify sources.github = true in config.json | ❌ Wave 0 |
| SETUP-02 | Setup writes config.json with team roster | smoke/manual | Run `/team-health:setup` with 2 team members; verify `.team-health/config.json` contains correct schema | ❌ Wave 0 |
| SETUP-03 | Setup gate routes to setup when not configured | smoke/manual | Delete config.json; run `/team-health:prep alice`; verify response routes to setup, does not error or proceed | ❌ Wave 0 |
| SETUP-04 | All state files have schema_version field | schema review | Inspect generated config.json, people/<slug>.json, baselines.json for schema_version field presence | ❌ Wave 0 |
| SETUP-05 | .team-health/ is gitignored | smoke/manual | Run setup in a git repo; verify `.gitignore` contains `.team-health/` | ❌ Wave 0 |
| REF-01 | SIGNALS.md exists and is readable | file check | Verify `.claude/team-health/SIGNALS.md` or equivalent supporting file exists and contains signal definitions | ❌ Wave 0 |
| REF-02 | BASELINES.md exists and is readable | file check | Same for BASELINES.md | ❌ Wave 0 |
| REF-03 | PRIVACY.md exists and is readable | file check | Same for PRIVACY.md | ❌ Wave 0 |
| REF-04 | SKILL.md installation guide is readable and accurate | review | Human review of top-level SKILL.md for accuracy and completeness | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** Manually verify the specific deliverable for that task (file exists, JSON is valid, correct schema fields present)
- **Per wave merge:** Run full smoke test: fresh project, no config, run each command created so far, verify routing and output
- **Phase gate:** All smoke tests pass; all reference documents exist and are substantively complete; `schema_version` present in all state file templates

### Wave 0 Gaps

All tests are manual in this phase — there is no automated test runner for a pure markdown skill.

- [ ] Smoke test script checklist: `docs/testing/phase-1-smoke-tests.md` — step-by-step manual test procedure for all 10 scenarios above
- [ ] JSON schema reference: `.planning/phases/01-foundation/state-schemas.json` — canonical schemas the planner can reference to verify task outputs

*(No automated framework install needed — no runnable code in Phase 1)*

---

## Open Questions

1. **Colon namespace vs hyphen namespace: which does the project want?**
   - What we know: `.claude/commands/team-health/prep.md` gives `/team-health:prep` (colon); `.claude/skills/team-health-prep/SKILL.md` gives `/team-health-prep` (hyphen)
   - What's unclear: PROJECT.md and REQUIREMENTS.md use colon syntax (`/team-health:prep`) consistently — this locks the format to `.claude/commands/team-health/` (legacy commands path)
   - Recommendation: Use `.claude/commands/team-health/` for command files, `.claude/team-health/` for reference docs. The legacy format is fully supported and produces the desired colon-namespace UX.

2. **Where do reference documents live?**
   - What we know: Skills format supports co-located supporting files; legacy commands format does not. Reference docs needed by multiple commands cannot be co-located with any single command file.
   - What's unclear: No current evidence that commands can reference files from a sibling directory using relative paths vs `${CLAUDE_SKILL_DIR}`
   - Recommendation: Put reference docs in `.claude/team-health/` (a sibling of `commands/`). Reference them with absolute paths in command prompt bodies: "Read .claude/team-health/SIGNALS.md"

3. **MCP tool names for probe calls**
   - What we know: Claude only sees tools from configured MCP servers; if a server is absent, its tools simply don't appear
   - What's unclear: Exact tool names differ by MCP implementation. Official GitHub MCP at `api.githubcopilot.com/mcp/` may expose different tool names than `@modelcontextprotocol/server-github`
   - Recommendation: Write probe instructions that are implementation-agnostic — instruct Claude to check whether any github-namespaced tool is available, rather than calling a specific tool name

---

## Sources

### Primary (HIGH confidence)

- [code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills) — Official Claude Code skills documentation (March 2026): full frontmatter reference, SKILL.md structure, skills vs commands equivalence, `allowed-tools` syntax confirmed, `disable-model-invocation` behavior
- [code.claude.com/docs/en/mcp](https://code.claude.com/docs/en/mcp) — Official Claude Code MCP documentation (March 2026): MCP tool availability at runtime, probe call behavior, configuration scopes, `/mcp` command

### Secondary (MEDIUM confidence)

- [danielcorin.com/til/anthropic/custom-slash-commands-hierarchy](https://www.danielcorin.com/til/anthropic/custom-slash-commands-hierarchy/) — Subdirectory-to-colon-namespace mapping confirmed: `.claude/commands/posts/new.md` → `/posts:new`
- WebSearch cross-reference: multiple independent sources confirm `allowed-tools: Read, Write, Bash(gh *)` syntax; glob-style tool scoping confirmed

### Tertiary (LOW confidence — verify at implementation)

- Community MCP server tool names for GitHub, Jira, Slack, Calendar — exact tool names differ by implementation; verify at Phase 3 when MCP-calling commands are built
- `.mcp.json` project-scope config path — confirmed in MCP docs but exact behavior when both user-scope and project-scope are configured needs validation

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — official docs verified, both commands and skills paths confirmed
- Architecture: HIGH — file structure, front matter keys, colon namespace all verified against current Claude Code documentation
- State schemas: HIGH — derived from project requirements and standard JSON state patterns; no external dependency to verify
- MCP detection pattern: MEDIUM-HIGH — confirmed that Claude only sees tools from configured servers; specific probe tool names are LOW confidence until verified at Phase 3

**Research date:** 2026-03-10
**Valid until:** 2026-06-10 (stable platform — skills API unlikely to break, but verify front matter keys if Claude Code has a major release)
