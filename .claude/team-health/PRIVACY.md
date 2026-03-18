# Team Health - Output Language Rules

Loaded by: ALL commands that generate output (/team-health:pulse, /team-health:prep, /team-health:skip-level, /team-health:retro-prep)
Purpose: Mandatory language rules. These rules protect your direct reports from being described in ways that are inaccurate, unfair, or harmful to their careers.

## The Core Principle

Team Health surfaces behavioral observations, not interpretations. The manager decides what to do with them. This tool does not diagnose, score, or judge people.

Before generating any output, read and internalize these four rules. Every statement you write about a team member must pass all four rules.

---

## Rule 1: Signals, Not Diagnoses

Describe what the data shows. Do not interpret what it means about the person's state of mind, motivation, or wellbeing.

**Prohibited:**
- "Alex seems burned out"
- "Jordan is disengaged"
- "Sam appears to be struggling"
- "Casey looks like they might be checked out"

**Approved:**
- "Alex's PR review participation is down 40% vs. 8-week baseline"
- "Jordan's commit activity dropped from 4 days/week to 1 day/week"
- "Sam has 2 tickets blocked for >1 sprint"
- "Casey merged 0 PRs this week (personal baseline: 2.3/week)"

**Why:** Diagnoses are irreversible labels. Behavioral observations are facts that can be discussed, corrected, and verified. A manager who acts on a diagnosis without talking to the person is working from assumption, not information.

---

## Rule 2: No DM Content

Slack signals use public channel metadata only. Never surface anything from direct messages or private channels.

**Prohibited:**
- Any reference to private message content
- Direct message sentiment or tone
- "Alex seems quieter in DMs"
- "Jordan's private messages suggest..."
- Content, topics, or sentiment from any non-public Slack channel

**Approved:**
- Public channel participation counts
- @-mention response latency in public channels only
- "Jordan posted in 2 public channels this week (baseline: 4.8)"

**Why:** Accessing DM content without consent is a privacy violation and destroys manager-IC trust if discovered. The Slack MCP is configured to return metadata only. Do not attempt to infer private channel content from any available data.

---

## Rule 3: No Team-Relative Scoring

Compare each person to their own history. Never compare people to each other or to team averages.

**Prohibited:**
- "Alex is the least active committer on the team"
- "Jordan is in the bottom quartile for PR reviews"
- "Team average is X, Sam is below average"
- "Casey has fewer code reviews than anyone else"
- "The highest performer this week was..."

**Approved:**
- "Alex's commit days dropped from their personal 8-week average of 4.1 to 1 this week"
- "Jordan's PR review count is below their personal baseline (current: 1, baseline mean: 3.2)"
- "Sam's ticket closure rate is consistent with their baseline"

**Why:** Team-relative scoring creates surveillance culture and penalizes people in high-performing teams for being "merely good." It also creates perverse incentives - a person can improve while still being flagged because a teammate improved faster. Personal baselines measure each person against themselves only.

---

## Rule 4: No Psychological Labels

Do not name mental health states, personality traits, or psychological conditions - even tentatively.

**Prohibited:**
- "Alice may be experiencing burnout"
- "Bob's behavior suggests low morale"
- "Carol seems anxious about the project"
- "David might be depressed"
- "Eve is showing signs of stress"

**Approved:**
- "Alice's Slack participation dropped significantly this week (2 channels vs. baseline 6.1)"
- "Bob has not merged any PRs in 2 weeks (personal baseline: 2.1/week)"
- "Carol has 3 blocked tickets open"
- "David's calendar meeting load is 72% this week (threshold: 60%)"
- "Eve's @-mention response latency is up to 8.4 hours (baseline: 2.1 hours)"

**Why:** Psychological labeling is outside an EM's scope, creates legal risk, and is usually wrong. The same behavioral signals can have many causes - illness, personal circumstances, context switching, or simply a slow week. Surface what the data shows. The manager's job is to have the conversation, not reach a conclusion before it.

---

## Required Disclaimer

Every pulse and prep output must include this disclaimer, verbatim:

> These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions.

**Placement:** At the end of the output, after the main content. Do not bury it. Do not paraphrase it.

---

## Graceful Degradation Language

When an MCP source is unavailable, state clearly what data is absent and why it matters. Do not make the limitation about the tool.

**Pattern:** "[Source] signals are unavailable - [MCP name] MCP is not configured. [What data cannot be included]."

**Examples:**
- "GitHub signals are unavailable - GitHub MCP is not configured. PR and commit data cannot be included."
- "Jira signals are unavailable - Jira MCP is not configured. Ticket velocity and blocked-ticket data cannot be included."
- "Slack signals are unavailable - Slack MCP is not configured. Channel participation and response latency cannot be included."
- "Calendar signals are unavailable - Calendar MCP is not configured. Meeting load and 1:1 adherence cannot be checked."

**Do not say:** "I can't access GitHub" - this focuses on the tool's limitation.
**Do say:** What data is absent and what questions the manager cannot answer as a result.

When fewer than all four sources are available, note at the top of the output which sources are active and which are unavailable, then proceed with the available data only.
