# Team Health — Baseline Computation Reference

Loaded by: /team-health:pulse (read and follow before computing any baselines)
Purpose: Instructions for computing and updating 8-week rolling baselines inline.

---

## What a Baseline Is

For each person × metric pair, the baseline is:
- A sliding window of at most 8 weekly observations (configurable via `baseline_window_weeks` in config.json; default 8)
- Precomputed mean and standard deviation stored in `.team-health/baselines.json`
- A deviation threshold: current value vs mean ± N*stddev determines flag status

Baselines are personal — Alice's baseline is derived from Alice's history only, never the team average. Do not mix person-level data when computing baselines.

---

## Reading a Baseline

**Step 1:** Read `.team-health/baselines.json` using the Read tool.

**Step 2:** Find the person by their `slug` field (e.g., `"alice-chen"`). The slug is derived from the `slug` field in config.json's `team` array.

**Step 3:** For each metric in that person's entry under `metrics`, the baseline record contains:

```json
{
  "window": [2, 3, 1, 2, 2, 3, 2, 2],
  "mean": 2.125,
  "stddev": 0.6,
  "last_value": 2
}
```

- `window` — array of weekly values, oldest first, at most `window_weeks` entries long
- `mean` — precomputed arithmetic mean of the window values
- `stddev` — precomputed population standard deviation of the window values
- `last_value` — the most recent value added (same as the last element of the window)

**Step 4:** Use `mean` and `stddev` directly for flag determination. Do not recompute from the window unless you are also updating it.

**If a person has no entry in baselines.json:** They have no baseline history. See "First-Run Baseline" section below.

**If a metric is missing from a person's entry:** That metric has never been computed for them. Treat the same as no history — do not flag, surface raw value only.

---

## Updating a Baseline (After a Pulse Run)

Follow these steps exactly, in order, for each person × metric pair after fetching the current week's data from the MCP sources.

**Step 1:** Fetch the current week's value for this metric from the appropriate MCP source (as defined in SIGNALS.md).

**Step 2:** Read the existing window from `baselines.json` for this person and metric. If no entry exists, start with an empty array `[]`.

**Step 3:** Append the new value to the end of the window array.
- Example: window was `[2, 3, 1, 2, 2, 3, 2, 2]`, new value is `1` → append → `[2, 3, 1, 2, 2, 3, 2, 2, 1]` (9 elements).

**Step 4:** If the window length now exceeds `baseline_window_weeks` (from config.json, default 8), remove the first (oldest) element.
- Example: `[2, 3, 1, 2, 2, 3, 2, 2, 1]` has 9 elements, max is 8 → remove first → `[3, 1, 2, 2, 3, 2, 2, 1]`.

**Step 5:** Recompute mean.
- Formula: `mean = sum(window) / len(window)`
- Example: `(3+1+2+2+3+2+2+1) / 8 = 16 / 8 = 2.0`

**Step 6:** Recompute population standard deviation.
- Formula: `stddev = sqrt( sum( (x - mean)^2 for each x in window ) / len(window) )`
- Special case: If the window contains only 1 value, set `stddev = 0`.
- Example (continuing from step 5):
  - Squared deviations: `(3-2)^2=1`, `(1-2)^2=1`, `(2-2)^2=0`, `(2-2)^2=0`, `(3-2)^2=1`, `(2-2)^2=0`, `(2-2)^2=0`, `(1-2)^2=1`
  - Sum of squared deviations: `1+1+0+0+1+0+0+1 = 4`
  - Variance: `4 / 8 = 0.5`
  - Stddev: `sqrt(0.5) ≈ 0.71`

**Step 7:** Set `last_value` to the new value that was just appended.
- Example: `last_value = 1`

**Step 8:** Update `computed_to` to today's date in ISO format (YYYY-MM-DD).

**Step 9:** Write the updated record back to `baselines.json` using the Write tool.

---

## Worked Example — Full Baseline Update

**Scenario:** Alice Chen, metric `github_prs_merged_per_week`, window size 8.

**Before update:**
```json
{
  "window": [2, 3, 1, 2, 2, 3, 2, 2],
  "mean": 2.125,
  "stddev": 0.6,
  "last_value": 2
}
```

**Current week's observed value:** 1 merged PR.

**Step 3 — Append:** `[2, 3, 1, 2, 2, 3, 2, 2, 1]` (9 elements)

**Step 4 — Trim oldest (window_weeks=8):** `[3, 1, 2, 2, 3, 2, 2, 1]` (8 elements)

**Step 5 — Recompute mean:**
- Sum: `3+1+2+2+3+2+2+1 = 16`
- Mean: `16 / 8 = 2.0`

**Step 6 — Recompute stddev:**
- `(3-2)^2 = 1`
- `(1-2)^2 = 1`
- `(2-2)^2 = 0`
- `(2-2)^2 = 0`
- `(3-2)^2 = 1`
- `(2-2)^2 = 0`
- `(2-2)^2 = 0`
- `(1-2)^2 = 1`
- Sum: `1+1+0+0+1+0+0+1 = 4`
- Variance: `4 / 8 = 0.5`
- Stddev: `sqrt(0.5) ≈ 0.71`

**Step 7 — last_value:** `1`

**After update:**
```json
{
  "window": [3, 1, 2, 2, 3, 2, 2, 1],
  "mean": 2.0,
  "stddev": 0.71,
  "last_value": 1
}
```

---

## Comparing to Baseline (Flag Determination)

After fetching the current value and reading the stored baseline, determine whether to flag:

**Step 1:** Retrieve `mean` and `stddev` from the stored baseline.

**Step 2:** Compute the deviation score:
- `deviation = (current_value - mean) / stddev`
- Special case: If `stddev == 0`, set `deviation = 0` unless `current_value != mean`. If `stddev == 0` and `current_value != mean`, treat as a definite flag condition (deviation is infinite).

**Step 3:** Apply the direction-appropriate threshold from SIGNALS.md:
- **Low = bad** (PR count, commit days, review count, ticket close rate, channel participation):
  - Flag if `deviation < -2.0`
- **High = bad** (review lag, response latency):
  - Flag if `deviation > +2.0`

**Step 4:** For absolute-threshold signals (meeting load %, stuck tickets, blocked tickets, 1:1 adherence, PR open >48h):
- Compare current value directly to the stated absolute threshold in SIGNALS.md.
- No deviation computation needed.

**Step 5:** Apply the Two-Signal Rule from SIGNALS.md before surfacing a flag.

---

## First-Run Baseline (No History)

When a person has no entries in baselines.json yet:

**Rule:** Do not flag them. No baseline = no valid comparison. Surfacing a "flag" without a baseline would be misleading.

**After the first pulse run:**
- Write their current week's values as the initial window (a single-element array).
- Set `mean = current_value`, `stddev = 0`.
- The baseline record now exists but is too thin to use for flagging.

**When baselines become meaningful:**
- After 3 or more weeks of data, the window has enough variance information to be useful.
- Before 3 weeks: surface raw values in the pulse output labeled as "new — no baseline yet". Do not show flag status.
- After 3 weeks: begin computing deviation scores and applying thresholds.
- After 8 weeks: full rolling window baseline is in effect.

**Implementation note for pulse command:** When generating output, check `len(window)` before using the baseline. If `len < 3`, display raw value only with the label "(baseline pending — N weeks of data)".

---

## Provenance Fields

When writing any person's entry in `baselines.json`, always include these fields at the person level (not inside individual metric records):

```json
{
  "computed_from": "2025-12-01",
  "computed_to": "2026-03-10",
  "source": ["github", "jira"],
  "window_weeks": 8
}
```

- `computed_from` — ISO date (YYYY-MM-DD) of the earliest week in the current window
- `computed_to` — ISO date of the most recent week (today's date at time of update)
- `source` — array of MCP source names that contributed data to this person's baselines in this session (e.g., `["github", "jira"]`). If a source was unavailable this run, do not include it — the absence is informative.
- `window_weeks` — the window size configured in config.json at time of computation; stored here for auditability

**Why provenance matters:** If a source is removed from config.json later, the `source` array shows which baselines were computed with that source's data. If `source` no longer includes `"github"`, the GitHub metrics in the window are stale and should be noted as such in pulse output.

---

## Full baselines.json Structure Reference

```json
{
  "schema_version": "1",
  "last_updated": "2026-03-10",
  "people": {
    "alice-chen": {
      "computed_from": "2025-12-01",
      "computed_to": "2026-03-10",
      "source": ["github", "jira"],
      "window_weeks": 8,
      "metrics": {
        "github_prs_merged_per_week": {
          "window": [3, 1, 2, 2, 3, 2, 2, 1],
          "mean": 2.0,
          "stddev": 0.71,
          "last_value": 1
        },
        "github_pr_review_count_per_week": {
          "window": [4, 5, 3, 4, 5, 4, 4, 5],
          "mean": 4.25,
          "stddev": 0.66,
          "last_value": 5
        }
      }
    }
  }
}
```

After updating all persons and metrics, set `last_updated` to today's date and write the entire file back with the Write tool.
