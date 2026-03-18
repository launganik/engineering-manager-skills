# Feature Landscape

**Domain:** Engineering Manager productivity / people analytics Claude Code skill
**Researched:** 2026-03-09
**Confidence:** MEDIUM - grounded in EM practice knowledge; web research tools unavailable, so claims based on training data rather than current sources. Core EM practice is stable; specific tool comparisons would benefit from live verification.

---

## Table Stakes

Features users expect. Missing = the skill is useless or no better than a blank doc.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| 1:1 prep sheet generation | EMs prep before every 1:1; a skill that doesn't help here has no obvious entry point | Medium | Needs: recent commits, open PRs, Jira velocity, previous topics from people log |
| Per-person signal summary | "Tell me what's going on with X" is the atomic unit of EM cognition | Medium | Must surface *changes* vs. baseline, not raw counts |
| Recent work summary | What shipped, what's blocked, what's in-flight - the factual layer every 1:1 starts with | Low | GitHub + Jira. Purely factual; low risk |
| Standing topics / follow-up tracking | EMs keep mental queues of things promised, things to revisit | Low | From people log: open items, manager commitments, IC asks |
| Tone calibration - signal, not diagnosis | Outputs framed as "signals to explore" not "this person is burned out" | Low (design) | Non-negotiable for legal/ethical safety |
| Team roster config | Skill is useless without knowing who the team is | Low | First-run setup: names, roles, 1:1 cadence |
| Graceful degradation to available tools | EM may only have GitHub; skill shouldn't fail if Jira/Slack absent | Medium | Capability detection on first run, clear gap reporting |

---

## Differentiators

Features that set this apart from a shared Notion template or a generic AI prompt.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Personal baseline scoring (8-week rolling) | Flags *changes* not absolute values. "PRs down 40% from your norm" beats "3 PRs this week" | High | Requires baseline snapshots in `.team-health/baselines.json`. Inline Claude computation avoids Python dependency |
| Multi-signal convergence requirement | Reduces false positives. A flag only fires if 2+ signals align or one is >2 std devs | Medium (prompt design) | Core to trust. Single-metric alerts cause alarm fatigue |
| Longitudinal people log with structured categories | Wins, growth edges, stated preferences, life context, manager commitments, career goals | Medium | The memory layer that makes 1:1s feel human. Structured so Claude can query across time |
| Attrition / disengagement early warning | Composite signal: PR review lag increase + meeting camera off trend + decreased async participation + reduced initiative | High | This is the killer feature. Requires multi-source data and multi-week patterns |
| Upward reporting mode (skip-level brief) | Converts team data into the format a VP/director wants: delivery health, risks, team mood, asks | Medium | Separates what's for skip-level (themes, not people) from what stays in people log |
| Sprint retro seeding from real data | Pre-populates retro with: cycle time, PR review wait, blocked tickets, recurring failure modes - not blank sticky notes | Medium | Turns retro from memory exercise into evidence-based conversation |
| Privacy-gated skip-level output | People log entries explicitly excluded from skip-level unless EM chooses to include them | Low (design) | Differentiates from naive "summarize everything" approaches |
| Calendar-aware prep timing | Knows when 1:1 is scheduled; can be triggered day-before automatically | Low | Calendar MCP integration |
| Named signal definitions, not opaque scores | Every flag explains *which signals* contributed and *why* they're notable | Medium (prompt design) | Transparency is the trust mechanism. "PR review lag: +3.2 days vs 8wk avg" not "health score: 4/10" |

---

## Anti-Features

Things to deliberately NOT build. Explicit scope exclusions that protect legal safety, manager-IC trust, and product coherence.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Sentiment scoring from Slack DMs | Privacy violation; legally fraught; destroys trust if discovered | Use Slack metadata only: message frequency, response latency, channel participation trends |
| Hard attrition probability scores ("72% likely to quit") | Creates liability; false precision; EMs act on it inappropriately | Frame as "pattern warranting a conversation": list the signals, suggest talking point |
| Team-relative scoring ("bottom 20% of team") | Surveillance/ranking; damages team culture if ever surfaced | Personal baseline only: each person measured against their own historical norm |
| Automated direct-report-facing output | This is a manager tool; IC trust requires they not feel surveilled | All outputs are manager-side only, never shown to ICs without explicit EM decision |
| Psychological labeling | "Alex seems depressed" is not EM's job and creates legal risk | "Multiple signals suggest Alex may be struggling - worth a direct check-in" |
| Performance rating automation | Replaces manager judgment; creates compliance/fairness issues | Surface relevant evidence; EM writes the actual review |
| Persistent tracking of DM content | No DM content, ever. Not stored, not summarized, not referenced | Metadata patterns only: timing, frequency, not content |
| Cross-team benchmarking | Comparing your team's velocity to other teams is politically loaded and context-free | Intra-team historical trends only |
| "Productivity score" rollups | Gameable, reductive, and misalign what EMs actually care about | Named signal categories surfaced separately, never summed into a single number |

---

## Feature Dependency Map

```
Team roster config (first-run)
    └─> all other commands depend on this

People log (log command)
    └─> 1:1 prep sheet (provides standing topics, follow-up items, context)
    └─> skip-level brief (provides themes if EM opts in)

Signal baselines (computed from GitHub/Jira history)
    └─> pulse dashboard (per-person vs. baseline scoring)
    └─> 1:1 prep sheet (flags notable changes)
    └─> attrition early warning (multi-week pattern detection)

Pulse dashboard
    └─> retro-prep (aggregates team-level patterns for the sprint window)
    └─> skip-level brief (team health summary section)
```

---

## Feature Detail: 1:1 Prep (`/team-health:prep <name>`)

### What signals matter

**From GitHub:**
- PR count and merge rate vs. personal 8-week baseline
- PR review turnaround time (submitted to first review, submitted to merge)
- Review participation: PRs reviewed for others vs. baseline
- Commit frequency pattern (not count - pattern: burst-then-silence vs. steady)
- After-hours commit timestamps (trend, not single instances)
- PR size trends (very large PRs = possible scope creep or isolation behavior)

**From Jira/linear:**
- Tickets completed vs. committed (sprint velocity vs. personal baseline)
- Ticket age: how long are things sitting in-progress
- Blocked ticket count and duration
- Scope of work: feature vs. bug vs. tech-debt ratio shift

**From Calendar:**
- Meeting load this week vs. typical (over-meeting = less maker time)
- Meeting accept/decline pattern changes
- 1:1 gaps (if last 1:1 was skipped or rescheduled, note it)
- Focus time blocks: how much uninterrupted time existed

**From Slack (metadata only):**
- Response latency trend (slower than usual to respond to the manager specifically)
- Channel participation trend (quieter in team channels)
- Message volume trend (dramatically up or down)

**From people log:**
- Outstanding manager commitments ("you said you'd get them access to X")
- IC-stated goals or asks from last 1:1
- Life context flagged in previous entries (performance plan, family situation, stated preference)
- Growth topics in progress
- Recent wins to acknowledge

### What format works

The 1:1 prep sheet should be structured as a briefing, not a data dump:

1. **Status snapshot** - 2-3 sentence narrative of what this person has been up to
2. **Signal flags** - Bulleted notable changes with the specific metric and delta (only items outside normal range)
3. **Standing items** - Open manager commitments + IC asks from people log
4. **Suggested talking points** - 2-4 concrete questions to explore flagged signals
5. **Context reminders** - Life/career context from people log worth keeping in mind

---

## Feature Detail: Team Pulse (`/team-health:pulse`)

### Established metrics

**Delivery health signals:**
- Sprint velocity variance (team-level): % deviation from rolling 8-week mean
- PR cycle time: median days from open to merge
- Review queue depth: open PRs awaiting review > X days (configurable threshold)
- Blocked ticket age: tickets in "blocked" state > sprint duration

**Engagement signals (per-person, vs. personal baseline):**
- Code contribution frequency (commits, PRs opened)
- Review participation (PRs reviewed)
- Async communication activity (Slack message volume, response latency)
- Meeting attendance/camera trends

**Risk signals (multi-week patterns required, not single-week):**
- Contribution decline: 2+ consecutive weeks below baseline
- Increasing after-hours activity: 3+ consecutive weeks trending up
- Review participation drop: person stopping cross-review suggests isolation
- Meeting decline trend: increasing declines or camera-off trend
- Combined flag: 3+ signals moving in same direction = "worth a conversation"

### Signal thresholds (opinionated defaults, configurable)

| Signal | Flag threshold | Notes |
|--------|---------------|-------|
| Individual contribution | >30% below 8-week baseline | Sustained 2 weeks |
| After-hours commits | >25% of commits outside 8am-7pm for 3+ weeks | Not one late night |
| PR review lag | +2 days above personal baseline | Sustained |
| Meeting declines | 30%+ of team meetings declined in a week | |
| Response latency | 2x personal baseline for manager DMs | Metadata only |
| Multi-signal convergence | 3+ signals flagged in same direction | Escalate recommendation |

---

## Feature Detail: People Log (`/team-health:log <name>`)

### Categories of notes that matter

**What EMs actually track in well-maintained people logs:**

| Category | Examples | Why it matters |
|----------|---------|---------------|
| Wins + impact | Shipped X, unblocked team on Y, got external shoutout | 1:1 acknowledgment, perf review evidence |
| Growth edges | Identified tendency to under-communicate, working on scope estimation | Continuity across 1:1s, coaching thread |
| Career goals | "wants to go staff", "not interested in management", "wants to do more infra" | Alignment, opportunity routing |
| Life context | Caregiver situation, health thing, family change, timezone constraint | Contextualizing signal anomalies |
| Manager commitments | "I will get you access to X", "I'll raise your case in promo cycle" | Trust maintenance; most important to track |
| IC asks / open items | "they want more design exposure", "wants to pair with platform team" | Follow-through = manager credibility |
| Difficult conversations | Themes, outcomes, what was agreed - not verbatim | Pattern recognition, HR documentation if needed |
| Stated preferences | "prefers async feedback", "wants direct delivery", "morning person" | Personalization of management approach |

### How to structure them

- Timestamped entries, not a single blob
- Each entry: date, category tag, free-text note
- Query patterns: "last 3 months of wins for perf review", "all open manager commitments", "growth thread for Alex"
- No minimum length - "acknowledged shipping X" is a valid entry

---

## Feature Detail: Upward Reporting (`/team-health:skip-level`)

### What skip-level briefs actually contain

Based on what engineering VPs and directors actually ask for:

1. **Delivery status** - Are we on track for the quarter? Any risks to committed work?
2. **Team health summary** - General mood/energy, not individual flags. "Team is heads-down post-launch, energy is good but vacation backlog is real"
3. **Staffing risks** - Open headcount, retention concerns (stated as themes, not named individuals unless EM chooses)
4. **Top asks / blockers requiring escalation** - What does the EM need from their manager to succeed
5. **Team wins** - What shipped, what's worth celebrating at the org level
6. **Upcoming needs** - Interviews, promo cases in flight, org changes anticipated

### Privacy gate

- People log content is NOT included by default
- Individual signal flags are aggregated into themes: "two team members showing engagement dip worth monitoring" not named
- EM can explicitly choose to include a specific person's context for a specific conversation
- Output should pass: "would I be comfortable if my entire team saw this?"

---

## Feature Detail: Retro Prep (`/team-health:retro-prep`)

### What data seeding looks like

The value is converting the sprint from memory-dependent to evidence-seeded.

**Data inputs:**
- Sprint ticket list: completed vs. committed vs. carried over
- PR merge timestamps vs. sprint window: what shipped when
- Blocked ticket log: what got stuck and why (if captured in Jira)
- PR review wait times during sprint: bottlenecks
- Any recurring ticket labels: repeated bug categories, same component

**Output format:**

1. **Sprint facts** - Velocity, carry-over count, PR merge distribution (front-loaded vs. last-day crunch)
2. **What went well (seeded)** - Tickets shipped, reviews completed promptly, collab patterns
3. **What was hard (seeded)** - Blocks, late merges, carry-overs with root causes
4. **Patterns to name** - "Third sprint in a row with carry-over from X component" - signal for structural conversation
5. **Blank space** - Retro is still a team conversation; data seeds, not determines

---

## MVP Recommendation

Build in this order:

1. **People log** (`/team-health:log`) - The memory layer. Low complexity, high daily value, no MCP required. Proves out the state storage and log querying pattern.
2. **1:1 prep** (`/team-health:prep`) - The entry point. Combines people log + GitHub signals. Immediately useful, demonstrates the synthesis value.
3. **Team pulse** (`/team-health:pulse`) - Requires baseline computation and multi-source MCP. Build after single-person prep is solid.
4. **Retro prep** (`/team-health:retro-prep`) - Depends on Jira + pulse patterns. Lower frequency need but high-value moments.
5. **Skip-level brief** (`/team-health:skip-level`) - Builds on pulse + people log; needs the privacy gate logic proven first.

Defer:
- Calendar integration - adds value but GitHub+Jira+people-log already deliver core value
- After-hours commit analysis - high sensitivity feature; needs the multi-signal convergence logic mature before introducing

---

## Sources

Note: Web research tools were unavailable during this research session. Findings are grounded in training knowledge of EM practice, people management literature, and the engineering productivity tooling space (Waydev, LinearB, Swarmia, Jellyfish, Lattice, Leapsome feature sets as of training cutoff August 2025). Confidence is MEDIUM - EM practice fundamentals are stable; specific tool comparisons or recent feature additions should be verified against current sources.

Key reference frameworks this draws on:
- Manager Tools "One-on-One" methodology (high-confidence, long-established practice)
- DORA metrics (cycle time, deployment frequency, change failure rate) - HIGH confidence
- Engineering productivity signal literature (Forsgren et al., "Accelerate") - HIGH confidence
- People management frameworks from Lara Hogan, Will Larson ("An Elegant Puzzle") - HIGH confidence for people log categories
- Attrition signal patterns from HR analytics practice - MEDIUM confidence (domain is well-established; specific thresholds are opinionated)
