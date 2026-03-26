---
description: "First-time setup for team-health skill. Collects team roster, 1:1 cadence, and detects available MCP sources. Run once per manager before using any other team-health command."
argument-hint: ""
allowed-tools: Read, Write, Bash(git status), Bash(cat .gitignore), Bash(echo)
disable-model-invocation: true
---

## /team-health:setup

### Step 0 - Already configured check

Read `.team-health/config.json` using the Read tool.

If the file exists and `setup_complete` is `true`:
- Output: "Team Health is already configured. Your team has [N] members. Sources detected: [list active sources]. Run `/team-health:setup --reset` to reconfigure."
- Stop. Do not proceed with the rest of this command.

If `$ARGUMENTS` contains `--reset`, skip this check and proceed to Step 1 even if already configured.

---

### Step 1 - MCP availability probe

Detect which MCP sources are available by attempting minimal tool calls for each source. Because MCP server implementations vary, probe by namespace rather than specific tool name.

**GitHub:** Attempt to call any tool whose name contains "github" or starts with "mcp_github". If any such tool is available and responds (even with an error that is NOT "tool not found" or "unknown tool"), set `github = true`. If no github-namespaced tool exists at all, set `github = false`.

**Jira:** Attempt to call any tool whose name contains "jira". If any such tool responds (even with a non-fatal error), set `jira = true`. If no jira-namespaced tool exists, set `jira = false`.

**Slack:** Attempt to call any tool whose name contains "slack". If any such tool responds (even with a non-fatal error), set `slack = true`. If no slack-namespaced tool exists, set `slack = false`.

**Calendar:** Attempt to call any tool whose name contains "gcal", "google_calendar", or "calendar". If any such tool responds (even with a non-fatal error), set `calendar = true`. If no calendar-namespaced tool exists, set `calendar = false`.

**Confluence:** Attempt to call any tool whose name contains "confluence" or "atlassian". If any such tool responds (even with a non-fatal error), set `confluence = true`. If no confluence/atlassian-namespaced tool exists, set `confluence = false`. Note: The Atlassian MCP often provides both Jira and Confluence tools — if Jira was detected via an atlassian-namespaced tool, also probe for Confluence-specific tools (getConfluencePage, searchConfluenceUsingCql, etc.).

After probing all five sources, present a capabilities summary before asking any questions:

```
MCP Sources Detected:
- GitHub: [available / NOT AVAILABLE - PR/commit signals will be absent]
- Jira: [available / NOT AVAILABLE - ticket velocity signals will be absent]
- Slack: [available / NOT AVAILABLE - participation metadata signals will be absent]
- Calendar: [available / NOT AVAILABLE - meeting load signals will be absent]
- Confluence: [available / NOT AVAILABLE - documentation activity signals will be absent]
```

Note: Full functionality requires all five sources. The skill degrades gracefully - GitHub-only is a supported configuration. Confluence signals are informational (do not trigger flags) but provide valuable context about non-code contributions.

---

### Step 2 - Collect manager info

Ask the manager for all of the following in a single message (do not ask one field at a time):

1. Your full name
2. Your GitHub username
3. Your timezone (e.g. America/New_York) - optional, defaults to UTC if not provided
4. Your GitHub organization or username for the team repos (e.g. "acme-corp" or "jlee")

Wait for the manager's response before continuing.

---

### Step 3 - Collect team roster

Ask the manager to list their direct reports. For each person, collect:
- Full name (required)
- Role (required)
- GitHub username (optional - leave null if not provided)
- Default 1:1 cadence in weeks (optional - default is 1 if not specified)

Additional optional fields (skip if not provided, set to null):
- Jira user ID (usually their email)
- Slack user ID (format: U0123XXXXX)
- Start date (YYYY-MM-DD)

Derive each person's `slug` from their name: lowercase all letters, replace spaces with hyphens, remove all special characters. Examples:
- "Alice Chen" → "alice-chen"
- "Bob O'Brien" → "bob-obrien"
- "Carlos Ruiz-García" → "carlos-ruiz-garcia"

Before writing any files, confirm the roster with the manager:
"I'll set up team-health for [N] direct reports: [list full names]. Is this correct? (yes/no)"

Wait for confirmation before continuing.

---

### Step 4 - Gitignore check

Read `.gitignore` using the Read tool.

- If `.gitignore` does not exist or does not contain `.team-health/`:
  - Use Bash to append the entry: `echo "# team-health - people data must not be committed\n.team-health/" >> .gitignore`
  - Tell the user: ".team-health/ added to .gitignore to keep people data out of version control."
- If `.team-health/` is already present in `.gitignore`:
  - Note it silently and continue.

---

### Step 5 - Write config.json

Write `.team-health/config.json` using the Write tool. Use the exact schema shape below with `schema_version: "1"`.

Fill in fields from the information collected in Steps 1–3:

```json
{
  "schema_version": "1",
  "setup_complete": true,
  "created": "<today's date in YYYY-MM-DD format>",
  "last_updated": "<today's date in YYYY-MM-DD format>",
  "manager": {
    "name": "<manager name from Step 2>",
    "github_username": "<github username from Step 2>",
    "timezone": "<timezone from Step 2, or 'UTC' if not provided>"
  },
  "team": [
    {
      "name": "<full name>",
      "slug": "<derived slug>",
      "role": "<role>",
      "github_username": "<github username or null>",
      "jira_user_id": "<jira user id or null>",
      "slack_user_id": "<slack user id or null>",
      "1on1_cadence_weeks": <number, default 1>,
      "start_date": "<YYYY-MM-DD or null>"
    }
  ],
  "sources": {
    "github": <true or false from Step 1>,
    "jira": <true or false from Step 1>,
    "slack": <true or false from Step 1>,
    "calendar": <true or false from Step 1>,
    "confluence": <true or false from Step 1>
  },
  "baseline_window_weeks": 8,
  "github_org": "<github org or username from Step 2>",
  "sprint_cadence_weeks": 2,
  "last_skip_level": null
}
```

Required fields that must be present for `setup_complete: true`: `schema_version`, `setup_complete`, `team` (non-empty array), `sources`, `github_org`.

---

### Step 6 - Confirm and summarize

After writing `config.json`, output the following confirmation:

```
Team Health setup complete.

Team roster saved: [N] people
MCP sources: GitHub [yes/no], Jira [yes/no], Slack [yes/no], Calendar [yes/no]

Next steps:
- Run /team-health:log <name> to start keeping notes about your team
- Run /team-health:pulse to run your first team health scan (requires GitHub MCP)
- Run /team-health:prep <name> before your next 1:1

Config saved to .team-health/config.json
.team-health/ is gitignored - your people data stays local.
```

Replace [N] with the actual count of team members, and [yes/no] with the actual detection results from Step 1.
