# Phase 2: People Log - Research

**Researched:** 2026-03-13
**Domain:** Claude Code command authoring, JSON state read/write, free-form note structuring, natural language query over flat JSON
**Confidence:** HIGH - all critical patterns verified against Phase 1 implementation and locked schemas

---

## Summary

Phase 2 delivers the `/team-health:log <name>` command - the longitudinal memory layer the whole skill depends on. It is a write-and-query command: the manager types free-form notes, the skill structures them into `people/<slug>.json` using the schema locked in Phase 1, and the same command can answer natural language questions about what was recorded.

The foundation is already in place. Phase 1 locked the canonical `person` schema in `state-schemas.json`, established the Pre-flight Check boilerplate, and implemented the setup command that the log command gates on. Phase 2 is a single command file - `.claude/commands/team-health/log.md` - that reads, appends to, and queries `.team-health/people/<slug>.json`. No new infrastructure is needed. The only new behavior is: (1) name resolution (argument → slug → file path), (2) first-time file creation, (3) context display (last 3–5 entries before prompting), (4) entry structuring (free-form → category + content + date), and (5) natural language query mode.

The most important design decision is how the command decides whether the manager wants to add a note vs. query the log. The cleanest approach: if `$ARGUMENTS` contains a question mark, or starts with "when", "what", "show", "list", or "find", treat it as a query; otherwise treat it as an append session. This keeps a single command file and avoids a separate `/team-health:log-query` command.

**Primary recommendation:** One command file handles both append and query modes. Mode detection is prompt-instruction-based (look at argument content and/or the manager's first message). State is read and written using `people/<slug>.json` per the locked Phase 1 schema. `disable-model-invocation: true` is required because this command writes state.

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| LOG-01 | `/team-health:log <name>` appends timestamped, tagged entries to `.team-health/people/<name>.md` | Note: requirements doc says `.md` but state-schemas.json specifies `.json` - use `.json` (canonical schema wins; `.md` is likely a documentation error from requirements draft) |
| LOG-02 | Entry categories supported: `feedback-given`, `feedback-received`, `career`, `commitment`, `concern`, `win`, `note` | Locked in state-schemas.json `valid_categories` - use those exact strings |
| LOG-03 | Manager types free-form notes; skill structures them into the log with date, category tag, and content | Standard prompt-engineering: Claude reads free-form input, infers category, asks for confirmation if ambiguous, writes structured entry |
| LOG-04 | Opening the log shows the last 3–5 entries as context before prompting for new input | Read the file (or detect file doesn't exist), render last 3–5 entries from `entries` array, then prompt |
| LOG-05 | Log supports natural language queries: "when did I last discuss promotion with Alex?", "what commitments have I made this quarter?" | Query mode: Claude reads the full `entries` array (and `open_commitments`) and answers inline - no search index needed at this data scale |
| LOG-06 | Skill handles first-time log creation for a person gracefully | If file does not exist, create it with empty `entries`, `open_commitments`, and `career_context` per schema |
</phase_requirements>

---

## Clarification: `.md` vs `.json` for people log files

REQUIREMENTS.md (LOG-01) says `.team-health/people/<name>.md`. The state-schemas.json says `.team-health/people/<slug>.json`. Phase 1 RESEARCH.md, the Standard Stack table, and the setup command all reference `.json`. The architecture note says "Markdown files in `.team-health/people/`" in a table row but the actual schema example is JSON.

**Resolution:** Use `.json`. The state-schemas.json file is the canonical locked reference (Phase 1 decision). The `.md` in LOG-01 is a documentation artifact from an early draft. The JSON schema supports structured querying and typed fields that a flat markdown file would not. The planner should use `people/<slug>.json` for all file operations.

---

## Standard Stack

### Core

| Technology | Version | Purpose | Why Standard |
|------------|---------|---------|--------------|
| `.claude/commands/team-health/log.md` | n/a | Slash command file for `/team-health:log` | Established in Phase 1: colon-namespace via `.claude/commands/team-health/` subdirectory |
| `$ARGUMENTS` in command file | n/a | Receives the person name (and optionally query text) typed after the command | Phase 1 confirmed: `$ARGUMENTS` is the variable name for everything after the command |
| `Read` tool | n/a | Read config.json (setup gate) and people/<slug>.json | Standard Claude Code tool, pre-approved in `allowed-tools` |
| `Write` tool | n/a | Write people/<slug>.json (create new or overwrite with updated entries array) | Standard Claude Code tool, pre-approved in `allowed-tools` |
| `Bash(date +%Y-%m-%dT%H:%M:%SZ)` | n/a | Get current timestamp for `created_at` field | Phase 1 pattern: `!`date +%Y-%m-%d`` for dynamic context injection |
| `people/<slug>.json` schema | schema_version "1" | Per-person log file format | Locked in Phase 1 `state-schemas.json` - must not deviate |

### Supporting

| Technology | Version | Purpose | When to Use |
|------------|---------|---------|-------------|
| `!`date +%Y-%m-%d`` pre-invocation bash | n/a | Inject current date into command context at invocation time | Use in front matter to give Claude today's date without a tool call - critical for entry `date` field accuracy |
| `disable-model-invocation: true` | n/a | Prevent Claude from auto-running this command | Required for all state-writing commands - Phase 1 established this rule |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Single `log.md` command (append + query) | Separate `log.md` and `log-query.md` | Single file is cleaner UX - manager doesn't need to know which sub-command to use. Mode detection from argument text is simple and reliable at this scope. |
| JSON file for people log | Markdown file (`.md`) | JSON is machine-queryable; Claude can reliably scan arrays and filter by category or date. Markdown is human-readable but harder to query structurally. JSON wins per state-schemas.json. |
| Full file overwrite on append | Append-only log | JSON files must be fully re-written to add an entry - this is the only option with Claude's `Write` tool. The workflow is: Read → mutate entries array in memory → Write full file. |

**No package installation needed.** This command uses only Claude's built-in tools.

---

## Architecture Patterns

### Recommended Structure

```
.claude/
  commands/
    team-health/
      setup.md          # Phase 1 - already exists
      log.md            # Phase 2 - this phase creates this file

.team-health/           # Per-manager state (gitignored)
  config.json           # Written by setup - log command reads this
  people/
    alice-chen.json     # Written by log command - created on first use
    bob-smith.json
```

### Pattern 1: Command File Structure for `/team-health:log`

**What:** The command file follows the exact front matter pattern established in Phase 1.

**Front matter:**
```yaml
---
description: "Append a note to a direct report's people log, or query their log history. Creates the log on first use."
argument-hint: "<name> [or natural language query]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%dT%H:%M:%SZ)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
```

**Why `disable-model-invocation: true`:** This command writes state. Phase 1 established this is mandatory for all state-writing commands to prevent Claude from auto-invoking during unrelated conversations.

**Why `Bash(date +%Y-%m-%dT%H:%M:%SZ)` in allowed-tools:** The `created_at` field in each entry requires an ISO 8601 timestamp. Pre-approving this specific bash call prevents an approval dialog on every log entry.

### Pattern 2: Name → Slug → File Path Resolution

**What:** The manager types `/team-health:log Alice Chen` or `/team-health:log alice`. The command must reliably resolve this to `alice-chen.json`.

**Resolution sequence:**
1. Read `$ARGUMENTS` (everything after the command)
2. If `$ARGUMENTS` contains a `?` or starts with a query word (`when`, `what`, `show`, `list`, `find`, `have`, `did`, `has`), extract the name from context - name is the first word(s) before the query begins, or the manager will be asked to clarify
3. Otherwise, treat `$ARGUMENTS` as the person name (may include query text too - see Mode Detection below)
4. Derive slug: lowercase all letters, replace spaces with hyphens, strip non-alphanumeric except hyphens
5. Look up slug in `config.json`'s `team` array to confirm person exists
6. File path: `.team-health/people/<slug>.json`

**If name not found in config:** Report "No team member matching '[name]' found. Configured team: [list names]. Did you mean one of these?" - do not proceed.

**Why check config.json:** Prevents creating orphaned log files for names that aren't on the team roster. Forces the manager to add people through setup first.

### Pattern 3: Mode Detection (Append vs. Query)

**What:** A single command handles both "add a note" and "answer a question about this person's history."

**Detection logic (in command prompt instructions):**

```
After resolving the person name:

If $ARGUMENTS (beyond the name) contains a question mark,
or if the text after the name starts with: when, what, show, list, find, have, did, has, how many, which -
  → QUERY MODE: read the full log and answer the question. Stop after answering.

Otherwise:
  → APPEND MODE: show last 3–5 entries, then prompt for a new note.
```

**Query mode examples:**
- `/team-health:log alice when did I last discuss promotion?` → query mode
- `/team-health:log alice what commitments are open?` → query mode
- `/team-health:log alice show me feedback from this quarter` → query mode

**Append mode examples:**
- `/team-health:log alice` → append mode (no extra text)
- `/team-health:log alice gave great feedback in the design review` → append mode (not a question)

**Edge case:** If ambiguous, default to append mode and let the manager clarify inline.

### Pattern 4: Context Display Before Append (LOG-04)

**What:** Before prompting for a new note, show the last 3–5 entries so the manager has context.

**Implementation:**
```
After reading people/<slug>.json:

If entries array has entries:
  Display the last min(5, len(entries)) entries in this format:

  ---
  Recent log for [Name]:

  [date] · [category] - [content]
  [date] · [category] - [content]
  ...
  ---

  What would you like to add?

If entries array is empty or file does not exist:
  Display: "No previous entries for [Name]. What would you like to log?"
```

**Why last 5, not all:** The full history can be arbitrarily long. Showing too many entries creates cognitive overhead. 3–5 is enough to remind the manager of recent context without overwhelming the prompt window.

### Pattern 5: Entry Structuring (LOG-03)

**What:** Manager types free-form. Claude infers the category and structures the entry.

**Category inference rules:**
| Keywords / signals | Inferred category |
|-------------------|-------------------|
| "gave feedback", "told them", "positive feedback", "constructive feedback" | `feedback-given` |
| "they said", "feedback from them", "they told me", "received" | `feedback-received` |
| "career", "promotion", "promo", "goals", "growth", "next level", "staff", "senior" | `career` |
| "I promised", "I'll follow up", "I will", "committed to", "going to connect" | `commitment` |
| "worried about", "concern", "seems off", "struggling", "flagging", "burnout" | `concern` |
| "shipped", "won", "great job", "nailed it", "promoted", "recognition" | `win` |
| (no strong signal) | `note` |

**Confirmation behavior:**
- If category is clearly inferred from keywords: use it without asking
- If ambiguous between two categories: show the inferred category and ask "Does that sound right? (or type the correct category)"
- If the manager types a `commitment` category: also add an entry to `open_commitments` array

**Entry ID format:** `<YYYY-MM-DD>-<NNN>` where NNN is a zero-padded sequence starting at 001 per day. Example: `2026-03-13-001`.

**ID collision avoidance:** Read existing entries for today's date, find the max sequence used, increment by 1.

### Pattern 6: First-Time File Creation (LOG-06)

**What:** Manager runs `/team-health:log alice` and no `alice-chen.json` exists yet.

**Behavior:**
1. Check if `.team-health/people/<slug>.json` exists using Read tool
2. If the Read returns an error or empty: create the file with the empty schema below, then proceed to append mode as normal
3. Do NOT error or warn excessively - just mention "Starting a new log for Alice Chen."

**Empty file template (per state-schemas.json):**
```json
{
  "schema_version": "1",
  "name": "<full name from config.json>",
  "slug": "<slug>",
  "created": "<today YYYY-MM-DD>",
  "last_updated": "<today YYYY-MM-DD>",
  "entries": [],
  "open_commitments": [],
  "career_context": {
    "stated_goals": [],
    "last_promo_discussion": null,
    "notes": ""
  }
}
```

**Why read full name from config.json:** The `name` field in the person file should match the canonical name stored during setup, not whatever case the manager typed in the argument.

### Pattern 7: Write-After-Read (Append Flow)

**What:** JSON files must be fully re-written. The append flow always reads first, mutates, then writes.

**Sequence:**
1. Read `.team-health/people/<slug>.json` (or create empty structure if absent)
2. Parse entries array
3. Build new entry object (generate ID, set date from injected `!`date +%Y-%m-%d``, infer category, set content)
4. If category is `commitment`: also push to `open_commitments` array
5. If note mentions career/promotion context: optionally update `career_context` fields
6. Append new entry to end of `entries` array
7. Update `last_updated` to today
8. Write full file back with `Write` tool

**Do not use Bash append:** `echo >> file` would corrupt the JSON. Always read-mutate-write.

### Anti-Patterns to Avoid

- **Writing to `.team-health/people/<slug>.md`:** Use `.json` per the locked schema, not `.md` as written in the requirements text.
- **Creating the log file before checking config.json:** Always run the Pre-flight Check and name resolution against config.json first - never create a log file for an unrecognized name.
- **Asking too many clarifying questions:** If category is inferable, infer it. Only ask when genuinely ambiguous. The log command should feel fast, not bureaucratic.
- **Using `context: fork`:** This command writes state (JSON file). Forked subagents may not persist file writes correctly in all Claude Code contexts. Run inline.
- **Truncating entries on write:** Read the full file, append to the entries array, write the full file. Never discard existing entries.
- **Querying with Bash grep:** All querying should be done by Claude reading the JSON and answering inline - not by shelling out. The data volume (max ~15 people × years of entries) is manageable in context.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Category classification | Multi-class classifier, LLM API call, rule engine | Claude inline inference from keywords in the note text | Claude handles this natively in the command prompt; the category list has only 7 values; keyword hints are sufficient |
| Natural language search | Elasticsearch, SQLite FTS, vector embeddings | Claude reads the full JSON and answers inline | At 3–15 people × years of entries × ~100 bytes each, the entire log fits in context window trivially; no search index needed |
| Entry ID uniqueness | UUID library, counter service, database sequence | Date prefix + daily sequence number (`2026-03-13-001`) | Human-readable, sortable, collision-free within a day, no external dependency |
| Schema validation | JSON Schema library, Zod | Explicit required-field checks in command prompt instructions | Claude validates presence of `schema_version`, `entries`, `slug` inline; 3 fields to check, no library needed |
| Slug normalization | `slugify` npm package | Inline instruction: lowercase, spaces to hyphens, strip non-alphanumeric | Simple enough for Claude to apply reliably without a library |

**Key insight:** This command is pure Claude-mediated state manipulation on a tiny JSON file. Every "infrastructure" need (search, validation, IDs, classification) is either trivially handled by the LLM inline or by a simple convention. There is nothing to install.

---

## Common Pitfalls

### Pitfall 1: Log File Path Mismatch

**What goes wrong:** Command creates `alice-chen.json` but Phase 4 (`/team-health:prep`) looks for it at a different path because the slug derivation differs.

**Why it happens:** Inconsistent slug rules (e.g., one command strips apostrophes, another doesn't; one lowercases Unicode, another doesn't).

**How to avoid:** Slug derivation rules must be verbatim-copied from setup.md into log.md. Do not paraphrase. The canonical rule: "lowercase all letters, replace spaces with hyphens, remove all special characters." The setup command already establishes the slug for each person and stores it in `config.json` - the log command should read the slug from `config.json` rather than re-deriving it from the argument, to guarantee consistency.

**Warning signs:** "No previous entries" shown when entries exist; `prep` can't find people log data.

### Pitfall 2: Appending to Wrong Data Structure

**What goes wrong:** Command writes a new entry to a top-level array instead of `entries`, or writes career context directly to the entry instead of `career_context`.

**Why it happens:** The person schema has three distinct arrays/objects (`entries`, `open_commitments`, `career_context`). Commands that handle `commitment` notes must write to both `entries` AND `open_commitments`.

**How to avoid:** The command instructions must explicitly state: "If category is `commitment`, append to both `entries` (with the full entry) and `open_commitments` (with id, date, content, status: 'open')." Reference the state-schemas.json example explicitly.

**Warning signs:** Phase 4 prep command says "no open commitments" when the manager logged commitments.

### Pitfall 3: Query Mode Returns Stale/Partial Data

**What goes wrong:** Manager asks "what commitments have I made this quarter?" and gets only entries from `entries` array, missing items in `open_commitments`.

**Why it happens:** Query mode instructions tell Claude to scan `entries` but don't mention `open_commitments` is a separate array.

**How to avoid:** Query mode instructions must explicitly say "Read all of: `entries`, `open_commitments`, and `career_context` when answering questions. Commitments may appear in both `entries` and `open_commitments`."

### Pitfall 4: Name Resolution Too Strict

**What goes wrong:** Manager types `/team-health:log Alice` and gets "No team member matching 'Alice' found" even though the config has "Alice Chen".

**Why it happens:** Exact string match on argument vs. config name.

**How to avoid:** Name resolution should do fuzzy prefix matching: if `$ARGUMENTS` (first word or words) is a case-insensitive prefix of any `name` in `config.json`, use that person. If multiple people match (e.g., two people named "Alex"), list them and ask the manager to disambiguate.

**Warning signs:** Managers constantly getting "not found" errors for people they know are on their team.

### Pitfall 5: Forgetting the Pre-flight Check

**What goes wrong:** Manager runs `/team-health:log alice` before setup, and the command either creates a file outside `.team-health/` or errors uncleanly when config.json doesn't exist.

**Why it happens:** The Pre-flight Check boilerplate from Phase 1 was not included.

**How to avoid:** The Pre-flight Check is mandatory boilerplate from Phase 1 - include it verbatim at the top of every command file, before any other logic.

### Pitfall 6: Date Drift

**What goes wrong:** Entries are logged with the wrong date because Claude uses its training cutoff date rather than the actual current date.

**Why it happens:** `$ARGUMENTS` and prompt context don't include today's date unless explicitly injected.

**How to avoid:** Use `!`date +%Y-%m-%d`` pre-invocation injection in the command front matter preamble. This runs at invocation time and injects the real system date into the prompt. The Phase 1 research confirmed this pattern works. Include it as the first line of the command body: `Current date: !`date +%Y-%m-%d``.

---

## Code Examples

Verified patterns from Phase 1 implementation and official sources:

### Pre-flight Check (verbatim from Phase 1 - copy exactly)

```
## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.
```

Source: Phase 1 RESEARCH.md Pattern 2

### Command Front Matter

```yaml
---
description: "Append a note to a direct report's people log, or query their log history. Creates the log on first use."
argument-hint: "<name> [or: <name> when did I last...?]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%dT%H:%M:%SZ)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`
```

Source: Phase 1 RESEARCH.md Pattern 1 (command file structure)

### Entry Object Shape (from state-schemas.json)

```json
{
  "id": "2026-03-13-001",
  "date": "2026-03-13",
  "category": "feedback-given",
  "content": "Gave specific positive feedback on the auth refactor PR - clean API design.",
  "tags": [],
  "created_at": "2026-03-13T14:32:00Z"
}
```

Source: `.planning/phases/01-foundation/state-schemas.json` (locked)

### Commitment Object Shape (from state-schemas.json)

```json
{
  "id": "2026-03-13-002",
  "date": "2026-03-13",
  "content": "Will connect Alice with the platform team lead by end of month.",
  "status": "open"
}
```

Source: `.planning/phases/01-foundation/state-schemas.json` (locked)

### Empty Person File Template

```json
{
  "schema_version": "1",
  "name": "Alice Chen",
  "slug": "alice-chen",
  "created": "2026-03-13",
  "last_updated": "2026-03-13",
  "entries": [],
  "open_commitments": [],
  "career_context": {
    "stated_goals": [],
    "last_promo_discussion": null,
    "notes": ""
  }
}
```

Source: `.planning/phases/01-foundation/state-schemas.json` (locked)

### Query Mode Prompt Block

```
## Query Mode

If $ARGUMENTS (beyond the person name) contains a question mark, or starts with:
when, what, show, list, find, have, did, has, how many, which

→ Read the full contents of .team-health/people/<slug>.json.
→ Answer the question using entries, open_commitments, and career_context.
→ Be specific: include dates, categories, and quoted content where relevant.
→ Do not prompt for a new entry after answering.
```

### Context Display Block (before append)

```
## Show Recent Context

Read .team-health/people/<slug>.json.

If entries array has 1 or more entries:
  Show the last min(5, total entries) in this format:

  ---
  Recent entries for [Name]:
  [YYYY-MM-DD] · [category] - [content]
  ---

  Then ask: "What would you like to log?"

If entries array is empty:
  Say: "No previous entries for [Name]. What would you like to log?"
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Separate append and query commands | Single command with mode detection | This phase establishes it | Fewer commands to remember; natural UX |
| Raw markdown people logs | Structured JSON with typed fields | Phase 1 decision | Enables structured querying; Phase 4 prep can reliably read career context and open commitments |
| Separate `open_commitments` array | Commitments only in entries | Phase 1 schema design | Prep command can pull open commitments without scanning all entries for keyword matches |

**Locked decisions from Phase 1 that constrain this phase:**
- `.team-health/people/<slug>.json` - file extension is `.json`, not `.md`
- `schema_version: "1"` - must be present in every file written
- `disable-model-invocation: true` - required on this command
- Pre-flight Check boilerplate - mandatory, copy verbatim
- Slug derivation: lowercase, spaces → hyphens, strip special chars

---

## Open Questions

1. **Does LOG-01's mention of `.md` vs `.json` need explicit acknowledgment in the command file?**
   - What we know: state-schemas.json says `.json`; REQUIREMENTS.md says `.md`; all Phase 1 architecture uses `.json`
   - What's unclear: whether the planner should add a note in the command explaining the discrepancy (useful for future maintainers)
   - Recommendation: Use `.json` silently (the schema is authoritative); add a comment in the PLAN explaining the resolution

2. **Should the log command update `career_context.last_promo_discussion` automatically?**
   - What we know: The schema has `last_promo_discussion` as a date field; LOG-03 says "structures them into the log"
   - What's unclear: whether automatic update of career_context is in scope for Phase 2 or Phase 4
   - Recommendation: Yes, include it - if the inferred category is `career` and the content mentions "promotion" or "promo", update `last_promo_discussion` to today's date. Keeps Phase 4 prep data fresh without requiring a separate command.

3. **What happens when a manager logs for someone who left the team?**
   - What we know: config.json `team` array is the canonical roster; if someone is removed from setup, their slug won't appear
   - What's unclear: whether the log command should allow logging for people not in config (might be useful for transition notes)
   - Recommendation: Gate on config.json roster for Phase 2. If the person is not in config, inform and stop. This can be relaxed in a future phase.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

| Property | Value |
|----------|-------|
| Framework | None - pure markdown/JSON skill, no runnable code |
| Config file | n/a |
| Quick run command | Manual: invoke `/team-health:log <name>` in Claude Code and verify output |
| Full suite command | Manual: run all log scenarios in a project with Phase 1 setup complete |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| LOG-01 | `/team-health:log alice` appends a new entry to `alice-chen.json` | smoke/manual | Run command, type a note, verify `.team-health/people/alice-chen.json` exists with one entry | ❌ Wave 0 |
| LOG-02 | All 7 category values appear correctly in written entries | smoke/manual | Log one note per category type; inspect JSON for correct category strings | ❌ Wave 0 |
| LOG-03 | Free-form note is structured with date, category, content | smoke/manual | Log "gave great code review feedback on the auth PR"; verify entry has date, inferred `feedback-given` category, original content | ❌ Wave 0 |
| LOG-04 | Last 3–5 entries shown before prompting | smoke/manual | Add 6 entries to a person; run log again; verify only last 5 are shown in context display | ❌ Wave 0 |
| LOG-05 | Natural language query returns accurate answer | smoke/manual | Log 3 entries including a career note; run `/team-health:log alice when did I last discuss promotion?`; verify date and content in answer | ❌ Wave 0 |
| LOG-06 | First-time log creation is graceful | smoke/manual | Run setup, do NOT log anything; run `/team-health:log bob` where bob has no existing log file; verify file is created cleanly with empty entries array | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** Verify the specific deliverable (file exists, JSON structure matches schema, command front matter correct)
- **Per wave merge:** Run all 6 smoke scenarios in a fresh project with Phase 1 setup complete
- **Phase gate:** All 6 scenarios pass before marking Phase 2 complete

### Wave 0 Gaps

- [ ] `docs/testing/phase-2-smoke-tests.md` - step-by-step manual test procedure for all 6 LOG scenarios above
- [ ] `.claude/commands/team-health/log.md` - the command file itself (this is the primary deliverable of Phase 2)

*(No automated framework needed - no runnable code; all tests are manual)*

---

## Sources

### Primary (HIGH confidence)

- `.planning/phases/01-foundation/state-schemas.json` - locked canonical schemas; `valid_categories`, `entries` shape, `open_commitments` shape, `career_context` shape all taken directly from this file
- `.planning/phases/01-foundation/01-RESEARCH.md` - established patterns (Pre-flight Check, front matter keys, `disable-model-invocation`, `!`date`` injection, colon-namespace commands, slug derivation)
- `.claude/commands/team-health/setup.md` - reference implementation showing the exact command file format in use

### Secondary (MEDIUM confidence)

- `.planning/REQUIREMENTS.md` - LOG-01 through LOG-06 requirements text (note: `.md` vs `.json` discrepancy documented above; schema wins)
- Phase 1 plan summaries (01-01-SUMMARY.md, 01-03-SUMMARY.md) - confirmed what was actually built in Phase 1 and what patterns are now in use

### Tertiary (LOW confidence - verify at implementation)

- Category inference keywords table above - reasonable heuristics based on common EM language; may need tuning after first use
- Fuzzy name matching recommendation - reasonable UX choice; implementation details may need adjustment based on edge cases encountered

---

## Metadata

**Confidence breakdown:**
- Command file format and front matter: HIGH - direct from Phase 1 implementation
- State schema: HIGH - locked in state-schemas.json, no ambiguity
- Mode detection pattern (append vs. query): MEDIUM-HIGH - logical design choice, not verified against an existing implementation; may need iteration
- Category inference rules: MEDIUM - keyword heuristics, reasonable but may need tuning
- Validation scenarios: HIGH - directly derived from LOG-01 through LOG-06 requirement text

**Research date:** 2026-03-13
**Valid until:** 2026-06-13 (stable foundations; Claude Code skills API unlikely to change; state schema is locked until Phase 1 schema_version bump)
