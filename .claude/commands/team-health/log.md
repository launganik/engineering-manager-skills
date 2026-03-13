---
description: "Append a note to a direct report's people log, or query their log history. Creates the log on first use."
argument-hint: "<name> [or: <name> when did I last...?]"
allowed-tools: Read, Write, Bash(date +%Y-%m-%dT%H:%M:%SZ)
disable-model-invocation: true
---

Current date: !`date +%Y-%m-%d`

## Pre-flight Check

Before doing anything else:
1. Read .team-health/config.json using the Read tool.
2. If the file does not exist OR if setup_complete is not true:
   - Tell the user: "Team Health is not set up yet. Run /team-health:setup to configure your team roster and MCP sources."
   - Stop. Do not proceed with the rest of this command.
3. If setup_complete is true, continue.

## Name Resolution

1. Extract the person name from $ARGUMENTS. The name is everything before the first query keyword or question mark — if $ARGUMENTS is just a name, the entire value is the name.
2. Do NOT re-derive the slug from the argument text. Instead: read the `team` array from config.json (already read in Pre-flight Check), find the entry whose `name` field is a case-insensitive prefix match for the typed name. Use the `slug` field from that config entry. This guarantees slug consistency with setup.
3. Fuzzy prefix matching: "alice" matches "Alice Chen"; "alice chen" matches "Alice Chen". Match is case-insensitive. If multiple team members match (e.g. two people named "Alex"), list them and ask the manager to clarify.
4. If no match found: "No team member matching '[typed name]' found. Configured team: [list all names from config]. Did you mean one of these?" — stop.
5. File path: `.team-health/people/<slug>.json`

## Mode Detection

After resolving the person name:

If $ARGUMENTS (the text after the person name) contains a question mark,
OR if the text after the name starts with any of: when, what, show, list, find, have, did, has, how many, which

→ QUERY MODE: go to the Query section below.

Otherwise:
→ APPEND MODE: go to the Append section below.

If genuinely ambiguous, default to APPEND MODE.

## Query Mode

Read the full contents of .team-health/people/<slug>.json using the Read tool.

Answer the question using ALL of:
- entries array (full history)
- open_commitments array (may contain commitments not yet in entries, or duplicate entries still open)
- career_context object (stated_goals, last_promo_discussion, notes)

Be specific: include dates, category labels, and quoted content where relevant.
Format the answer as a short, direct response — no headers, no markdown tables unless the answer is a list.

Do NOT prompt for a new entry after answering. Stop after answering the question.

## Append Mode

### Step 1: Read or create the person file

Attempt to read .team-health/people/<slug>.json using the Read tool.

If the file does not exist (Read returns an error or empty):
  - Say: "Starting a new log for [Full Name from config]."
  - Initialize an in-memory person object using this exact template:
    {
      "schema_version": "1",
      "name": "[full name from config.json team entry]",
      "slug": "[slug]",
      "created": "[today YYYY-MM-DD from injected date]",
      "last_updated": "[today YYYY-MM-DD]",
      "entries": [],
      "open_commitments": [],
      "career_context": {
        "stated_goals": [],
        "last_promo_discussion": null,
        "notes": ""
      }
    }
  - Proceed to Step 2.

If the file exists:
  - Parse it into memory.
  - Proceed to Step 2.

### Step 2: Show recent context

If entries array has 1 or more entries:
  Show the last min(5, total entries count) entries in this format:

  ---
  Recent entries for [Name]:
  [YYYY-MM-DD] · [category] — [content]
  [YYYY-MM-DD] · [category] — [content]
  ...
  ---

  Then ask: "What would you like to log?"

If entries array is empty:
  Say: "No previous entries for [Name]. What would you like to log?"

Wait for the manager's response.

### Step 3: Infer category from the manager's note

Use these keyword signals to infer the category:

| Keywords / signals in the note | Inferred category |
|-------------------------------|-------------------|
| "gave feedback", "told them", "positive feedback", "constructive feedback", "gave", "reviewed" | feedback-given |
| "they said", "feedback from them", "they told me", "received feedback" | feedback-received |
| "career", "promotion", "promo", "goals", "growth", "next level", "staff", "senior" | career |
| "I promised", "I'll follow up", "I will", "committed to", "going to connect", "will connect", "will introduce" | commitment |
| "worried about", "concern", "seems off", "struggling", "flagging", "burnout" | concern |
| "shipped", "won", "great job", "nailed it", "promoted", "recognition", "shoutout" | win |
| (no strong signal from above) | note |

Inference behavior:
- If category is clearly inferred: use it without asking.
- If ambiguous between two categories: state the inferred category and ask "Does that sound right? (or type the correct category: feedback-given, feedback-received, career, commitment, concern, win, note)"
- If the manager types a category name explicitly: use it.

### Step 4: Build the entry and write the file

1. Get the current ISO timestamp: call Bash(date +%Y-%m-%dT%H:%M:%SZ)
2. Determine today's date from the injected date at the top of this command (YYYY-MM-DD format)
3. Compute the entry ID:
   - Count existing entries in the entries array whose `date` field equals today's date
   - ID format: <YYYY-MM-DD>-<NNN> where NNN is (count + 1) zero-padded to 3 digits
   - Example: if 0 entries today → "2026-03-13-001"; if 2 entries today → "2026-03-13-003"
4. Build the entry object:
   {
     "id": "<computed id>",
     "date": "<today YYYY-MM-DD>",
     "category": "<inferred or confirmed category>",
     "content": "<manager's note verbatim>",
     "tags": [],
     "created_at": "<ISO timestamp from Bash call>"
   }
5. Append the entry to the entries array.
6. If category is "commitment":
   - Also append to open_commitments array:
     {
       "id": "<same id as the entry>",
       "date": "<today YYYY-MM-DD>",
       "content": "<manager's note verbatim>",
       "status": "open"
     }
7. If category is "career" AND the note contains "promotion" or "promo":
   - Set career_context.last_promo_discussion to today's date (YYYY-MM-DD)
8. Set last_updated to today's date.
9. Write the full updated object to .team-health/people/<slug>.json using the Write tool.
   IMPORTANT: Write the complete file — do not truncate or omit existing entries.
10. Confirm: "Logged [category] entry for [Name]." (one line, no extra detail)
