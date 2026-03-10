# Phase 1 Smoke Tests

> Manual test procedure for all Phase 1 requirements. Run this end-to-end before marking Phase 1 complete. No automated test runner — pure markdown/JSON skill verification is manual.

---

## Prerequisites

Before starting any test scenario:

1. **A git-initialized project directory** — This must be a directory *other than* the skill repo itself. Example: `mkdir ~/test-project && cd ~/test-project && git init`
2. **Claude Code installed** — Confirm with `claude --version`.
3. **Skill files installed in the project** — Copy or symlink the skill files into your test project:
   ```bash
   # From the antigravity-skills repo root:
   cp -r .claude ~/test-project/.claude
   cp SKILL.md ~/test-project/SKILL.md
   ```
4. **No prior team-health state** — Confirm `.team-health/` does not exist: `ls ~/test-project/.team-health 2>&1` should return "No such file or directory".
5. **Claude Code open in the test project** — Open Claude Code with the `~/test-project` directory as the working directory before running any commands.

---

## Test Scenarios

### Scenario 1 — MCP detection: no MCPs configured (SETUP-01)

**Requirement:** SETUP-01

**Setup:**
1. Confirm no GitHub, Jira, Slack, or Calendar MCP servers are configured in Claude Code.
2. Run `claude mcp list` — the output should show no team-health-relevant MCPs (github, jira, slack, or google-calendar entries).
3. If any are configured, remove them: `claude mcp remove <name>` for each.
4. Ensure `.team-health/` does not exist in the test project (delete if present: `rm -rf ~/test-project/.team-health`).

**Action:**
1. Open Claude Code in the test project directory.
2. Run `/team-health:setup`.
3. When prompted for team members, add at least one person (any name, role, and 1-on-1 cadence).
4. Complete the full setup flow until setup reports it is done.

**Expected:**
- The setup summary output lists each source (GitHub, Jira, Slack, Calendar) as unavailable or "not detected".
- `.team-health/config.json` is created.
- The `sources` block in config.json has all four sources set to `false`.

**Pass criteria:**
```bash
cat ~/test-project/.team-health/config.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
s = d['sources']
assert s['github'] == False, f'github should be false, got {s[\"github\"]}'
assert s['jira'] == False, f'jira should be false, got {s[\"jira\"]}'
assert s['slack'] == False, f'slack should be false, got {s[\"slack\"]}'
assert s['calendar'] == False, f'calendar should be false, got {s[\"calendar\"]}'
print('PASS: all four sources are false')
"
```

---

### Scenario 2 — MCP detection: GitHub MCP configured (SETUP-01)

**Requirement:** SETUP-01

**Setup:**
1. Add the GitHub MCP server to Claude Code:
   ```bash
   claude mcp add github -- npx -y @modelcontextprotocol/server-github
   ```
   (Adjust the package/command if your GitHub MCP uses a different installer.)
2. Confirm it is listed: `claude mcp list` should show a `github` entry.
3. Remove or reset any existing `.team-health/` directory: `rm -rf ~/test-project/.team-health`.

**Action:**
1. Run `/team-health:setup` (or `/team-health:setup --reset` if setup was previously completed).
2. Complete the setup flow.

**Expected:**
- The setup summary reports GitHub as available/detected.
- `sources.github` is `true` in config.json.
- The remaining three sources (jira, slack, calendar) are `false` (since only GitHub MCP is configured).

**Pass criteria:**
```bash
cat ~/test-project/.team-health/config.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
s = d['sources']
assert s['github'] == True, f'github should be true, got {s[\"github\"]}'
assert s['jira'] == False, 'jira should be false'
assert s['slack'] == False, 'slack should be false'
assert s['calendar'] == False, 'calendar should be false'
print('PASS: github=true, others=false')
"
```

**Cleanup:** Remove the GitHub MCP after this scenario if you want a clean slate:
```bash
claude mcp remove github
```

---

### Scenario 3 — Setup writes correct config.json (SETUP-02)

**Requirement:** SETUP-02

**Setup:**
1. Remove any existing `.team-health/` directory: `rm -rf ~/test-project/.team-health`.
2. Ensure at least one MCP is absent (or all absent is fine — sources accuracy was tested above).

**Action:**
1. Run `/team-health:setup`.
2. When prompted for your name/identity, enter: `Jordan Lee` (or any name).
3. When prompted for team members, add exactly **2 team members**:
   - Member 1: Name `Alice Chen`, Role `Senior Engineer`, 1-on-1 cadence `1` (weekly)
   - Member 2: Name `Bob Kim`, Role `Staff Engineer`, 1-on-1 cadence `2` (bi-weekly)
4. Complete the setup flow.

**Expected:**
- `.team-health/config.json` is created.
- The file is valid JSON.
- Required fields are present: `schema_version`, `setup_complete`, `team`, `sources`.
- `schema_version` equals `"1"`.
- `setup_complete` equals `true`.
- `team` is an array with 2 entries.
- Each team member entry has: `name`, `slug`, `role`, `1on1_cadence_weeks`.

**Pass criteria:**
```bash
cat ~/test-project/.team-health/config.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
assert d.get('schema_version') == '1', f'schema_version should be \"1\", got {d.get(\"schema_version\")}'
assert d.get('setup_complete') == True, 'setup_complete should be true'
assert isinstance(d.get('team'), list), 'team should be a list'
assert len(d['team']) == 2, f'team should have 2 members, got {len(d[\"team\"])}'
for m in d['team']:
    assert 'name' in m, f'member missing name: {m}'
    assert 'slug' in m, f'member missing slug: {m}'
    assert 'role' in m, f'member missing role: {m}'
    assert '1on1_cadence_weeks' in m, f'member missing 1on1_cadence_weeks: {m}'
assert 'sources' in d, 'sources block missing'
print(f'PASS: valid config.json with {len(d[\"team\"])} team members, schema_version={d[\"schema_version\"]}')
"
```

---

### Scenario 4 — Setup gate routes to setup (SETUP-03)

**Requirement:** SETUP-03

**Setup:**
1. Either delete config.json or rename it so it does not exist:
   ```bash
   rm -f ~/test-project/.team-health/config.json
   # or if .team-health/ never existed, that's fine too
   ```
2. Do NOT run `/team-health:setup` first.

**Action:**
1. Run `/team-health:prep alice` (or any team-health command other than `setup`, e.g. `/team-health:pulse` or `/team-health:log alice`).

**Expected:**
- The command does NOT attempt to generate prep content, run a pulse check, or log an entry.
- Claude outputs a message directing you to run setup first. The message should resemble:
  > "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
- The command stops without producing any health or prep output.

**Pass criteria (human review):**
- [ ] No prep content, pulse content, or log content was generated.
- [ ] A clear setup redirect message was displayed.
- [ ] The message references `/team-health:setup` explicitly.
- [ ] The command did not error out with a stack trace or cryptic message.

**Note:** This test requires human visual review of Claude's output. Automated pass criteria are not applicable.

---

### Scenario 5 — All state files have schema_version (SETUP-04)

**Requirement:** SETUP-04

**Setup:**
1. Complete setup first (run Scenario 3 steps if not already done — config.json must exist with `setup_complete: true`).
2. Run `/team-health:pulse` once to generate `baselines.json`. Confirm the run completes.
3. Run `/team-health:log alice` (or the slug for a team member you added) to generate a people entry. Add any note when prompted.

**Action:**
1. Inspect all three state files for the `schema_version` field.

**Expected:**
- `config.json` has `"schema_version": "1"` at the top level.
- `baselines.json` has `"schema_version": "1"` at the top level.
- `people/<slug>.json` has `"schema_version": "1"` at the top level.

**Pass criteria:**
```bash
# Check config.json
python3 -c "
import json
d = json.load(open('$HOME/test-project/.team-health/config.json'))
assert d.get('schema_version') == '1', f'config.json: schema_version wrong: {d.get(\"schema_version\")}'
print('config.json: PASS schema_version=1')
"

# Check baselines.json (run after /team-health:pulse)
python3 -c "
import json
d = json.load(open('$HOME/test-project/.team-health/baselines.json'))
assert d.get('schema_version') == '1', f'baselines.json: schema_version wrong: {d.get(\"schema_version\")}'
print('baselines.json: PASS schema_version=1')
"

# Check people/<slug>.json (replace alice-chen with actual slug)
python3 -c "
import json, glob
files = glob.glob('$HOME/test-project/.team-health/people/*.json')
assert files, 'No people/*.json files found'
for f in files:
    d = json.load(open(f))
    assert d.get('schema_version') == '1', f'{f}: schema_version wrong: {d.get(\"schema_version\")}'
    print(f'{f}: PASS schema_version=1')
"
```

---

### Scenario 6 — .team-health/ is gitignored (SETUP-05)

**Requirement:** SETUP-05

**Setup:**
1. Use a fresh git-initialized project that has skill files installed but NO prior `.team-health/` or `.gitignore` content from the skill.
2. Confirm `git status` shows a clean working tree (or only untracked skill files — NOT `.team-health/`).

**Action:**
1. Run `/team-health:setup` in the test project.
2. Complete the setup flow (add at least one team member).
3. After setup completes, run `git status` from the project root.

**Expected:**
- `.gitignore` in the project root contains `.team-health/`.
- `git status` does NOT show `.team-health/` as an untracked directory.
- The setup summary mentions the gitignore entry was added.

**Pass criteria:**
```bash
# Verify .gitignore contains the entry
grep ".team-health" ~/test-project/.gitignore && echo "PASS: .team-health/ found in .gitignore" || echo "FAIL: .team-health/ missing from .gitignore"

# Verify git status doesn't list .team-health/
cd ~/test-project && git status --short | grep ".team-health" && echo "FAIL: .team-health/ appears in git status" || echo "PASS: .team-health/ not in git status"
```

---

### Scenario 7 — SIGNALS.md exists and is substantive (REF-01)

**Requirement:** REF-01

**Setup:**
- Skill files must be installed (`.claude/team-health/SIGNALS.md` should exist).

**Action:**
1. Read the file:
   ```bash
   cat ~/test-project/.claude/team-health/SIGNALS.md
   ```

**Expected:**
The file exists and contains all four signal source sections:

1. **GitHub signals** — must describe:
   - PR frequency (PRs merged per week)
   - Review participation (review comments or approvals per week)
   - Commit days per week
   - How each signal is computed (what data query/MCP call to use)
   - Threshold guidance (e.g., what delta from baseline is notable)

2. **Jira signals** — must describe:
   - Tickets closed per week
   - Open blockers/impediments
   - How each signal is computed
   - Threshold guidance

3. **Slack signals** — must describe:
   - Channel participation (messages per week in team channels)
   - Response latency (avg time to reply to mentions)
   - How each signal is computed
   - Threshold guidance

4. **Calendar signals** — must describe:
   - Meeting load (hours/week in meetings)
   - 1:1 adherence (% of scheduled 1:1s held)
   - How each signal is computed
   - Threshold guidance

**Pass criteria (human review):**
- [ ] File exists at `.claude/team-health/SIGNALS.md`
- [ ] All four source sections are present (GitHub, Jira, Slack, Calendar)
- [ ] Each signal has a computation description (not just a name)
- [ ] Each signal has threshold guidance for what is notable vs. baseline
- [ ] Data source for each signal is named (which MCP tool or API to use)

---

### Scenario 8 — BASELINES.md exists and is substantive (REF-02)

**Requirement:** REF-02

**Setup:**
- Skill files must be installed (`.claude/team-health/BASELINES.md` should exist).

**Action:**
1. Read the file:
   ```bash
   cat ~/test-project/.claude/team-health/BASELINES.md
   ```

**Expected:**
The file exists and contains:
- Explanation of the rolling 8-week window concept
- Step-by-step instructions for computing mean (arithmetic, written out)
- Step-by-step instructions for computing standard deviation (arithmetic, written out — suitable for inline computation by Claude without external libraries)
- The two-sigma flag threshold rule (when to flag a metric as anomalous)
- At least one worked example with real numbers showing the computation end-to-end

**Pass criteria (human review):**
- [ ] File exists at `.claude/team-health/BASELINES.md`
- [ ] Rolling 8-week window is defined
- [ ] Mean computation is written out with inline arithmetic steps
- [ ] Standard deviation computation is written out (not just "use stddev formula")
- [ ] Two-sigma threshold is stated explicitly
- [ ] A worked example with concrete numbers is present
- [ ] A Claude instance with no extra context could follow these instructions to compute a baseline

---

### Scenario 9 — PRIVACY.md exists and is substantive (REF-03)

**Requirement:** REF-03

**Setup:**
- Skill files must be installed (`.claude/team-health/PRIVACY.md` should exist).

**Action:**
1. Read the file:
   ```bash
   cat ~/test-project/.claude/team-health/PRIVACY.md
   ```

**Expected:**
The file exists and contains all four privacy categories from REQUIREMENTS.md:

1. **Behavioral observation language** — rule against diagnostic or psychological labeling; approved phrasing examples (e.g., "commit frequency dropped 40% vs. baseline") and prohibited phrasing examples (e.g., "seems burned out", "appears disengaged")

2. **DM content exclusion** — explicit rule that Slack DM content is never to be read or cited; only public channel participation metadata

3. **No team-relative scoring** — rule against comparing individuals to each other (no "Alice is the lowest performer"); comparisons are only against the individual's own baseline

4. **No psychological labels** — explicit prohibition of clinical or quasi-clinical terms (burnout, anxiety, disengagement as diagnostic conclusions)

**Pass criteria (human review):**
- [ ] File exists at `.claude/team-health/PRIVACY.md`
- [ ] All four privacy categories are present
- [ ] Each category has concrete approved phrasing examples
- [ ] Each category has concrete prohibited phrasing examples
- [ ] Rules are written as explicit prohibitions, not vague guidance

---

### Scenario 10 — SKILL.md installation guide is accurate (REF-04)

**Requirement:** REF-04

**Setup:**
- Use a completely fresh terminal session and a new empty directory.
- Do NOT use any project that already has the skill files installed.

**Action:**
1. Open SKILL.md from the skill repo:
   ```bash
   cat ~/path/to/antigravity-skills/SKILL.md
   ```
2. Follow the installation steps **exactly as written**, step by step, in the fresh project directory.
3. After completing the installation steps, open Claude Code in the new project directory.
4. Verify that `/team-health:setup` is available (type `/team-health:` and confirm autocomplete or help shows the command).
5. Run the setup flow through to completion.

**Expected:**
- MCP prerequisites are listed (any MCPs needed, and that the skill works without them too)
- Install steps result in the `.claude/` directory being present with team-health content
- All six commands described in SKILL.md are available in Claude Code after installation:
  - `/team-health:setup`
  - `/team-health:prep`
  - `/team-health:pulse`
  - `/team-health:log`
  - `/team-health:skip-level`
  - `/team-health:digest`
- The first-run setup flow completes without errors

**Pass criteria (human review):**
- [ ] MCP prerequisites are documented (install optional, graceful degradation described)
- [ ] Install steps work end-to-end in a fresh environment
- [ ] All six commands are described in SKILL.md
- [ ] The commands listed in SKILL.md match the actual command files in `.claude/commands/team-health/`
- [ ] Setup flow completes successfully following only the SKILL.md instructions

---

## Test Matrix

| Scenario | Requirement | Status |
|----------|-------------|--------|
| 1 — MCP detection (no MCPs) | SETUP-01 | ⬜ |
| 2 — MCP detection (GitHub MCP) | SETUP-01 | ⬜ |
| 3 — Config.json written correctly | SETUP-02 | ⬜ |
| 4 — Setup gate routes | SETUP-03 | ⬜ |
| 5 — schema_version in all state files | SETUP-04 | ⬜ |
| 6 — .team-health/ gitignored | SETUP-05 | ⬜ |
| 7 — SIGNALS.md substantive | REF-01 | ⬜ |
| 8 — BASELINES.md substantive | REF-02 | ⬜ |
| 9 — PRIVACY.md substantive | REF-03 | ⬜ |
| 10 — SKILL.md accurate | REF-04 | ⬜ |

---

## Pass/Fail

Phase 1 is complete when all 10 scenarios show ✅.

Fill this in after running end-to-end:
- Date tested: ___
- Tester: ___
- Scenarios passed: ___/10
- Issues found: ___
