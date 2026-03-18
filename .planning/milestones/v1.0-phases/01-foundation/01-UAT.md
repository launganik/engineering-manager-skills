---
status: testing
phase: 01-foundation
source: 01-00-SUMMARY.md, 01-01-SUMMARY.md, 01-02-SUMMARY.md, 01-03-SUMMARY.md
started: 2026-03-11T10:00:00Z
updated: 2026-03-11T10:00:00Z
---

## Current Test

number: 1
name: SKILL.md — readable installation guide
expected: |
  Open SKILL.md at the project root. An EM with no prior context should be able to follow it to a working /team-health:setup invocation. It should cover: prerequisites (MCP installs), numbered setup steps with verify commands, a command reference for all 6 planned commands (with phase availability labels), and a privacy section.
awaiting: user response

## Tests

### 1. SKILL.md — readable installation guide
expected: Open SKILL.md at the project root. An EM with no prior context should be able to follow it to a working /team-health:setup invocation. It should cover: prerequisites (MCP installs), numbered setup steps with verify commands, a command reference for all 6 planned commands (with phase availability labels), and a privacy section.
result: [pending]

### 2. setup command file exists with correct structure
expected: .claude/commands/team-health/setup.md exists. It should have YAML front matter (including disable-model-invocation: true), and the body should describe a 6-step setup flow: already-configured guard, MCP probe, manager info collection, team roster collection, gitignore check, and config.json write.
result: [pending]

### 3. .gitignore protects .team-health/
expected: .gitignore at the project root contains a .team-health/ entry. Running `grep '.team-health' .gitignore` should return a match.
result: [pending]

### 4. MCP probe is namespace-based (not tool-name-based)
expected: Open .claude/commands/team-health/setup.md and find the MCP probe step. The probe should check for tool namespaces (e.g., "any tool containing 'github'") rather than specific tool names — making it robust to different community MCP implementations.
result: [pending]

### 5. setup command writes config.json with schema_version "1"
expected: The setup.md command file contains instructions to write a config.json that includes schema_version: "1", manager info (name, email, timezone), team roster (array of person objects with slug), sources object (github/jira/slack/calendar booleans), and setup_complete: true.
result: [pending]

### 6. SIGNALS.md covers all 11 signals
expected: .claude/team-health/SIGNALS.md defines signals across 4 sections: 4 GitHub signals, 3 Jira signals, 2 Slack signals, 2 Calendar signals — 11 total. The Two-Signal Rule section should state that a person is only flagged if 2+ signals are below threshold OR 1 signal exceeds 2 standard deviations from their personal baseline.
result: [pending]

### 7. BASELINES.md has a worked numerical example
expected: .claude/team-health/BASELINES.md includes a step-by-step update procedure AND a worked example with real numbers (window arrays, mean computation, stddev computation using sqrt). Someone following it could compute a baseline without any external tools.
result: [pending]

### 8. PRIVACY.md has required disclaimer verbatim
expected: .claude/team-health/PRIVACY.md contains the following text verbatim: "These signals are indicators, not diagnoses. Always talk to your people before drawing conclusions." It should also have 4 rules with prohibited and approved examples for each.
result: [pending]

### 9. Smoke test checklist covers all 10 scenarios
expected: docs/testing/phase-1-smoke-tests.md exists and covers all 10 Phase 1 scenarios: SETUP-01 through SETUP-05 (setup command behavior) and REF-01 through REF-04 (reference document content). Each scenario should have setup steps, actions, expected behavior, and pass criteria.
result: [pending]

### 10. Pre-flight Check pattern in setup.md
expected: .claude/commands/team-health/setup.md includes a guard step (Step 0 or similar) that checks for an existing config.json. If setup_complete is already true, the command should tell the user they're already set up and exit gracefully rather than running the full setup again.
result: [pending]

## Summary

total: 10
passed: 0
issues: 0
pending: 10
skipped: 0

## Gaps

[none yet]
