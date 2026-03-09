# Technology Stack

**Project:** team-health (Claude Code skill for engineering managers)
**Researched:** 2026-03-09
**Confidence note:** WebFetch, WebSearch, and Bash tools were not available in this research session. All findings are drawn from training data (cutoff August 2025) and first-hand knowledge of the Claude Code skill ecosystem. Confidence levels reflect this constraint.

---

## Recommended Stack

### Claude Code Skill Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Claude Code custom commands | Current (≥1.0) | Slash command registration | Native mechanism — no runtime needed, commands are pure markdown files that Claude reads as system context |
| Markdown command files (`.md`) | n/a | Skill definition per command | Each slash command maps 1:1 to a `.md` file; the file IS the prompt template |
| YAML front matter in command files | n/a | Command metadata (description, allowed-tools) | Lets Claude Code display descriptions in autocomplete and restrict which MCP tools a command may call |

**Confidence: HIGH** — This is how Claude Code custom slash commands work as of August 2025, based on direct experience with the system.

#### Slash Command File Structure

Claude Code registers custom slash commands from two locations:

- **User-global commands:** `~/.claude/commands/<namespace>/<command>.md`
- **Project-local commands:** `.claude/commands/<namespace>/<command>.md`

For a shareable skill (installable by any EM), the files live in the project directory at install time. The install step is: copy the skill's `.claude/commands/team-health/` tree into the target project.

Each command file uses this structure:

```
---
description: "1:1 prep sheet from GitHub/Jira/Slack/Calendar signals"
allowed-tools: [mcp__github, mcp__jira, mcp__slack, mcp__gcal]
---

# /team-health:prep

[Full prompt template that Claude executes]
```

The `allowed-tools` front matter key restricts which MCP tools Claude will call for that command — critical for the privacy model (e.g., `:skip-level` should not allow mcp__slack).

**No SKILL.md concept exists in core Claude Code.** The term "SKILL.md" is a GSD framework convention (used by the `get-shit-done` scaffolding tool that orchestrated this research). In Claude Code itself, the skill is the collection of command `.md` files. The GSD framework likely uses SKILL.md as a skill-level manifest or README — write one as documentation, but it is not loaded by Claude Code natively.

### MCP Server Layer

| MCP Server | Purpose | Source | Notes |
|------------|---------|--------|-------|
| `@modelcontextprotocol/server-github` | GitHub signal collection | Official Anthropic/MCP org | PR activity, review participation, commit cadence, issue engagement |
| `mcp-server-jira` (community) | Jira ticket signal collection | Community (multiple implementations) | Ticket velocity, blocked time, comment activity |
| `@modelcontextprotocol/server-slack` | Slack metadata (not content) | Official or community | Channel activity metadata; NOT DM content — enforce in prompt |
| Google Calendar MCP / `mcp-google-calendar` | Calendar load signals | Community implementations | Meeting density, focus block availability, after-hours patterns |

**Confidence on GitHub MCP: HIGH** — Official server, well-documented, stable.
**Confidence on Jira MCP: MEDIUM** — Multiple community implementations exist; no single canonical official server as of Aug 2025. Requires validation at implementation time to choose a maintained fork.
**Confidence on Slack MCP: MEDIUM** — Official MCP Slack server exists; Slack API rate limits and workspace permissions are the real constraint.
**Confidence on Calendar MCP: LOW** — Google Calendar MCP is community-built. Availability, maintenance status, and OAuth flow need verification at implementation time. May require building a thin wrapper.

#### MCP Configuration Pattern

MCP servers are registered in Claude Code's `claude_desktop_config.json` (or project-level `.mcp/config.json`):

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "jira": {
      "command": "npx",
      "args": ["-y", "mcp-server-jira"],
      "env": {
        "JIRA_HOST": "${JIRA_HOST}",
        "JIRA_EMAIL": "${JIRA_EMAIL}",
        "JIRA_TOKEN": "${JIRA_TOKEN}"
      }
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": { "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}" }
    },
    "gcal": {
      "command": "npx",
      "args": ["-y", "mcp-google-calendar"],
      "env": { "GOOGLE_CREDENTIALS": "${GOOGLE_CREDENTIALS}" }
    }
  }
}
```

**The skill itself does not configure MCP servers** — MCP is a user/workspace concern. The skill's first-run flow (`setup` command) should detect which MCP servers are available and communicate what's missing.

### State Storage Layer

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| JSON files in `.team-health/` | n/a | Structured state (config, baselines, pulse history) | Machine-readable by Claude, diffable in git, no external dependencies |
| Markdown files in `.team-health/people/` | n/a | People logs (longitudinal notes per person) | Human-readable, appendable, natural for free-text observations |
| `.gitignore` entry for `.team-health/` | n/a | Keep people data out of version control | Privacy invariant — people logs must never be accidentally committed |

**Confidence: HIGH** — File-based state is the standard pattern for stateful Claude Code skills. No database, no external storage.

#### State File Layout

```
.team-health/
  config.json           # Team roster, 1:1 cadence, MCP source preferences
  baselines.json        # 8-week rolling window per person, per signal dimension
  pulse-history/
    YYYY-MM-DD.json     # Weekly pulse snapshot
  people/
    <slug>.md           # Longitudinal log per direct report
  setup-complete        # Presence flag; absence triggers first-run flow
```

**`config.json` schema (minimal):**

```json
{
  "team": [
    {
      "name": "Alice Chen",
      "slug": "alice-chen",
      "github_username": "achen",
      "jira_user_id": "alice@company.com",
      "slack_user_id": "U0123ABC",
      "oncall_email": "alice@company.com",
      "1on1_cadence_weeks": 1
    }
  ],
  "sources": {
    "github": true,
    "jira": false,
    "slack": true,
    "calendar": true
  },
  "baseline_window_weeks": 8,
  "created": "2026-03-09",
  "version": "1"
}
```

**`baselines.json` schema (minimal):**

```json
{
  "alice-chen": {
    "updated": "2026-03-09",
    "window_weeks": 8,
    "github": {
      "weekly_prs_merged": { "mean": 2.1, "stddev": 0.8 },
      "weekly_pr_review_count": { "mean": 4.5, "stddev": 1.2 },
      "weekly_commit_days": { "mean": 4.2, "stddev": 0.6 }
    },
    "jira": {
      "weekly_tickets_closed": { "mean": 3.0, "stddev": 1.1 }
    },
    "slack": {
      "weekly_message_days": { "mean": 4.8, "stddev": 0.4 }
    },
    "calendar": {
      "weekly_meeting_hours": { "mean": 12.0, "stddev": 2.5 }
    }
  }
}
```

### Skill Distribution Layer

| Artifact | Location in Repo | Purpose |
|----------|-----------------|---------|
| `.claude/commands/team-health/*.md` | Repo root | The slash commands themselves |
| `references/` | Repo root | Prompt reference docs Claude reads inline |
| `SKILL.md` | Repo root | Human-readable install guide and capability overview |
| `.team-health/.gitkeep` | Repo root | Ensures state directory exists on clone, gitignored for contents |
| `install.md` | Repo root | Step-by-step: copy commands, configure MCP, run setup |

**Distribution model:** The skill repo is cloned or the files are copied into the EM's project. There is no npm package, no binary, no build step. The skill is the markdown files.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| State storage | JSON + Markdown files | SQLite via MCP | No SQLite MCP exists for this use case; adds runtime dependency; overkill for team of 3–15 |
| State storage | JSON + Markdown files | External database (Postgres, Notion) | Requires network, auth, external service — violates "keeps data with the manager" principle |
| Baseline computation | Claude inline reasoning | Python script | Python requires environment setup; breaks cross-platform install; Claude is capable of computing mean/stddev over <50 data points |
| MCP for Calendar | Google Calendar MCP | Build custom calendar tool | Custom tool is correct fallback IF community MCP is unmaintained — but try community server first |
| Skill packaging | Raw markdown files | npm package | npm adds versioning complexity with no benefit; markdown files are trivially diff-able and installable via git clone |
| Command namespace | `team-health:` prefix | Flat commands like `/prep` | Namespacing prevents collision with other installed skills; required pattern for shareable skills |
| Jira MCP | Community server | Atlassian REST API via `curl` in prompt | Direct curl in prompts is fragile; MCP abstraction is the right layer; pick most-maintained community server |

---

## What NOT to Use

### Python / Node.js scripts for baseline computation
The PROJECT.md explicitly rules this out. Inline Claude reasoning over the baseline JSON is sufficient for rolling mean/stddev over 8 weeks × 15 people × ~5 signals = ~600 data points maximum. Claude handles this without external code.

### Survey-based or self-reporting inputs
Out of scope by design. Every signal must come from tool integrations, not from asking people to rate themselves.

### A database
`.team-health/` as flat files is the correct call. JSON is human-inspectable, git-trackable, and requires zero infrastructure.

### Slack DM content
The Slack MCP must only be queried for channel-level activity metadata (message count per day, days active). DM content is explicitly out of scope — enforce this in every prompt that calls the Slack MCP.

### Skip-level auto-population from people logs
The `:skip-level` command must not include people log content unless the manager explicitly invokes a flag. Enforce in the command's prompt, not left to Claude's judgment.

### Psychological labels in output
No "Alice seems depressed" — only observable signal language: "PR review participation is down 2.3 std devs from Alice's 8-week baseline." Enforce in output formatting instructions in every command.

---

## Installation

There is no package to install. The skill is installed by:

```bash
# 1. Clone the skill repo (or copy the files)
git clone https://github.com/your-org/antigravity-skills <target>

# 2. Copy command files to target project
cp -r <target>/.claude/commands/team-health/ <your-project>/.claude/commands/team-health/

# 3. Add MCP servers to your Claude Code config (see install.md)
# Edit ~/.config/claude/claude_desktop_config.json

# 4. Run first-time setup
# In Claude Code: /team-health:setup
```

MCP server npm packages needed:

```bash
# These are fetched at runtime via npx -y, no pre-install required
# But for offline/CI use:
npm install -g @modelcontextprotocol/server-github
npm install -g mcp-server-jira        # verify current package name at implementation time
npm install -g @modelcontextprotocol/server-slack
npm install -g mcp-google-calendar    # verify current package name at implementation time
```

---

## Graceful Degradation Matrix

The skill must detect which MCP sources are available and adjust:

| Sources Available | Commands Functional | Degraded Behavior |
|-------------------|--------------------|--------------------|
| GitHub only | `:prep` (partial), `:pulse` (partial) | Signals limited to code activity; calendar/ticket/social signals absent |
| GitHub + Jira | `:prep`, `:pulse`, `:retro-prep` | Strong engineering signals; social + calendar absent |
| GitHub + Slack | `:prep`, `:pulse` | Code + presence signals; ticket velocity absent |
| All four | Full capability | All five commands fully operational |
| None | `:log` only | Can still maintain people log; cannot generate signal-based prep |

Detection mechanism: The `:setup` command (or the first run of any command) attempts a minimal probe call to each MCP server and writes the results to `config.json`'s `sources` block. All other commands read `sources` and include/exclude sections accordingly.

---

## Sources

**Note:** This document is based on training data (August 2025 cutoff). The following are authoritative sources to verify at implementation time — web access was not available during this research session:

- Claude Code documentation: https://docs.anthropic.com/en/docs/claude-code
- Claude Code custom commands reference: https://docs.anthropic.com/en/docs/claude-code/slash-commands
- MCP specification and server registry: https://modelcontextprotocol.io
- Official MCP GitHub server: https://github.com/modelcontextprotocol/servers/tree/main/src/github
- Official MCP Slack server: https://github.com/modelcontextprotocol/servers/tree/main/src/slack
- Jira MCP community servers: search npm for `mcp-server-jira` and verify maintenance status
- Google Calendar MCP: search npm for `mcp-google-calendar` and verify maintenance status

**Pre-implementation verification checklist:**
- [ ] Confirm `.claude/commands/` is still the correct path for project-local commands
- [ ] Confirm `allowed-tools` is a valid front matter key (may be `tools` or similar)
- [ ] Confirm YAML front matter is supported in command files (vs. a different metadata mechanism)
- [ ] Verify current package name and maintenance status for Jira MCP server
- [ ] Verify current package name and maintenance status for Google Calendar MCP server
- [ ] Verify Slack MCP returns metadata (not DM content) by default, or requires explicit scoping
