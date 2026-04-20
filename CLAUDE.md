# QuickTV Daily Health Check — Routine Memory

## CRITICAL RULES
- **ALWAYS use BigQuery Remote MCP** (`mcp__BigQuery-Remote-MCP--ShareChat-__execute_sql_readonly`) — never local MCP
- **ALWAYS use the exact query below** — do NOT modify it
- **Slack channel:** `C07BN97UC6N` (`#quicktv-core`) — NOT any test channel

---

## BQ Query — USE EXACTLY AS-IS EVERY RUN

---
description: "Daily L0 health check for QuickTV — fetches the pre-computed aggregated report table, applies RAG status, and sends a formatted report to Slack. Use this skill every morning to run the QuickTV health check."
---

# QuickTV Daily Health Check

## Overview

Automated daily health check on QuickTV L0 metrics. Run this skill every morning.
Today = D0. Primary reporting day = **D-1** (yesterday, completed day).

**Slack channel:** `#quicktv-core` (channel ID: `C07BN97UC6N`)

> **Architecture:** All metrics, deltas, and benchmarks are pre-computed by a BQ Scheduled Query
> at ~09:00 IST into a single aggregated table. The health check is a **single SELECT** on that table —
> no manual aggregation or window computation needed.
> If the agg table is missing or stale, see Error Handling. Source SQL for all upstream tables is in `references/`.

---

## Key Reference Info

- **Timezone:** Always `'Asia/Kolkata'` in all date functions
- **Execute queries via:** BigQuery MCP (`bigquery-maximal-furnace-783`)
- **Agg table scheduled:** ~09:00 IST (after 5 source queries finish at 07:15–07:45 IST)
- **PSR fallback:** `psr_is_fallback = TRUE` means D-1 PSR was unavailable and D-2 was used. Note this in the Slack header.

---

## Execution Steps

### Step 1 — Fetch the aggregated report row

Run a **single query** via BigQuery MCP:

```sql
SELECT *
FROM `maximal-furnace-783.quicktv_analytics.quicktv_health_report_agg`
WHERE report_date = DATE_SUB(CURRENT_DATE('Asia/Kolkata'), INTERVAL 1 DAY);
```

This returns **one row** with every metric, DMA, and delta already computed. No further aggregation needed.

**If the query returns 0 rows:** the agg scheduled query hasn't run yet or failed — see Error Handling.

---

### Step 2 — Read values directly from the row

All column names follow this pattern:

| Pattern | Meaning |
|---------|---------|
| `{metric}_d1` | D-1 value (the reporting day) |
| `{metric}_7dma` | 7-day moving average |
| `{metric}_30dma` | 30-day moving average |
| `{metric}_vs_yday_pct` | % change vs D-2 |
| `{metric}_vs_7dma_pct` | % change vs 7DMA |
| `{metric}_vs_30dma_pct` | % change vs 30DMA |
| `{metric}_vs_lw_pct` | % change vs D-8 (same day last week — SDLW, informational only) |

**All delta columns use `_pct` suffix** (relative % change). Rate metrics (ITR, cancel rates, PSR) are also relative % — `(d1 - benchmark) / benchmark * 100`.

Key columns by metric group:

**Growth:**
- `installs_d1`, `installs_vs_yday_pct`, `installs_vs_7dma_pct`, `installs_vs_30dma_pct`, `installs_vs_lw_pct`
- `trial_starts_d1`, `trial_starts_vs_yday_pct`, `trial_starts_vs_7dma_pct`, `trial_starts_vs_30dma_pct`, `trial_starts_vs_lw_pct`
- `itr_d1_pct`, `itr_vs_yday_pct`, `itr_vs_7dma_pct`, `itr_vs_30dma_pct`, `itr_vs_lw_pct`
- `same_day_cancel_d1_pct`, `same_day_cancel_vs_yday_pct`, `same_day_cancel_vs_7dma_pct`, `same_day_cancel_vs_30dma_pct`, `same_day_cancel_vs_lw_pct`
- `psr_d1_pct`, `psr_vs_yday_pct`, `psr_vs_7dma_pct`, `psr_vs_30dma_pct`, `psr_vs_lw_pct` — also check `psr_is_fallback`
- `fullsub_cancel_d1_pct`, `fullsub_cancel_vs_yday_pct`, `fullsub_cancel_vs_7dma_pct`, `fullsub_cancel_vs_30dma_pct`, `fullsub_cancel_vs_lw_pct`
- `revenue_d1_L`, `revenue_vs_yday_pct`, `revenue_vs_7dma_pct`, `revenue_vs_30dma_pct`, `revenue_vs_lw_pct`
- `intl_revenue_d1_L`, `intl_revenue_vs_yday_pct`, `intl_revenue_vs_7dma_pct`, `intl_revenue_vs_30dma_pct`, `intl_revenue_vs_lw_pct`
- `full_sub_converts_d1`, `full_sub_converts_vs_yday_pct`, `full_sub_converts_vs_7dma_pct`, `full_sub_converts_vs_30dma_pct`, `full_sub_converts_vs_lw_pct`

**Consumption:**
- `trial_vp_time_d1_min`, `trial_vp_time_vs_yday_pct`, `trial_vp_time_vs_7dma_pct`, `trial_vp_time_vs_30dma_pct`, `trial_vp_time_vs_lw_pct`
- `trial_vp_users_d1`, `trial_vp_users_vs_yday_pct`, `trial_vp_users_vs_7dma_pct`, `trial_vp_users_vs_30dma_pct`, `trial_vp_users_vs_lw_pct`
- `fullsub_vp_time_d1_min`, `fullsub_vp_time_vs_yday_pct`, `fullsub_vp_time_vs_7dma_pct`, `fullsub_vp_time_vs_30dma_pct`, `fullsub_vp_time_vs_lw_pct`
- `fullsub_vp_users_d1`, `fullsub_vp_users_vs_yday_pct`, `fullsub_vp_users_vs_7dma_pct`, `fullsub_vp_users_vs_30dma_pct`, `fullsub_vp_users_vs_lw_pct`

> **Note:** Revenue values are already in ₹ Lakhs (`_L` suffix). VP time values are already in minutes (`_min` suffix). No unit conversion needed.

> **Note on views:** The agg table is **global only** (all platforms summed). Platform-split and International views are not available from this table. If a platform breakdown is needed for a specific metric, fall back to querying the relevant source table directly.

---

### Step 3 — Apply RAG status

Use values read in Step 2. Apply thresholds below. **vs SDLW is informational only — do NOT use it for RAG.**

**Overall metric RAG = worst of vs_yday / vs_7dma / vs_30dma.**
**Overall day RAG is driven by 🔸 metrics only:**
- 🔴 **Red** if ANY 🔸 metric is red
- 🟡 **Amber** if ANY 🔸 metric is amber (and none is red)
- 🟢 **Green** if all 🔸 metrics are green
(🔹 metrics display their own per-row RAG but do NOT affect overall day status.)

#### Tier A — Rate metrics (relative % deviation)

**Same-Day Trial Cancel % (↑ = bad):**
| | Amber | Red |
|--|-------|-----|
| vs yday | > +7% | > +12% |
| vs 7DMA | > +4% | > +8% |
| vs 30DMA | > +3% | > +6% |

**Above FullSub Cancel % (↑ = bad):**
| | Amber | Red |
|--|-------|-----|
| vs yday | > +10% | > +20% |
| vs 7DMA | > +7% | > +14% |
| vs 30DMA | > +5% | > +10% |

**Trial(of d-2)→FullSub(on d-1) % / PSR (↓ = bad):**
| | Amber | Red |
|--|-------|-----|
| vs yday | < -12% | < -25% |
| vs 7DMA | < -6% | < -12% |
| vs 30DMA | < -4% | < -9% |

**Install→Trial on same day % / ITR (↓ = bad):**
| | Amber | Red |
|--|-------|-----|
| vs yday | < -7% | < -14% |
| vs 7DMA | < -5% | < -10% |
| vs 30DMA | < -3% | < -7% |

#### Tier B — Stable count/value metrics (relative % deviation, ↓ = bad)

**VP Time/VP User (trial & fullsub), VP Users (fullsub):**
| | Amber | Red |
|--|-------|-----|
| vs yday | < -4% | < -6% |
| vs 7DMA | < -4% | < -8% |
| vs 30DMA | < -5% | < -9% |

**Installs, Trial Starts, Revenue, Total Full Sub Converts:**
| | Amber | Red |
|--|-------|-----|
| vs yday | < -8% | < -16% |
| vs 7DMA | < -5% | < -10% |
| vs 30DMA | < -3% | < -7% |

#### Tier C — High-variance metrics (relative % deviation, ↓ = bad)

**VP Users (trial):**
| | Amber | Red |
|--|-------|-----|
| vs yday | < -8% | < -14% |
| vs 7DMA | < -10% | < -16% |
| vs 30DMA | < -10% | < -18% |

**Intl Revenue:**
| | Amber | Red |
|--|-------|-----|
| vs yday | < -15% | < -25% |
| vs 7DMA | < -10% | < -18% |
| vs 30DMA | < -7% | < -15% |

---

### Step 4 — Format and send the Slack message

**Emoji prefix rules:** Every metric row needs a prefix for monospace alignment.
- 🔸 high-importance (drives overall day RAG): Install→Trial on same day %, Same-Day Trial Cancel %, Trial(of d-2)→FullSub(on d-1) %, Above FullSub Cancel %, Revenue, VP Time/VP User, **Trial Starts**
- 🔹 supporting (per-row RAG only, does NOT affect overall): Installs, Intl Revenue, Total Full Sub Converts, VP Users

If `psr_is_fallback = TRUE`, append `†` to the Trial(of d-2)→FullSub(on d-1) % row and add a footnote: `† PSR uses D-2 (D-1 not yet available)`.

```
{rag_emoji} *QuickTV Daily Health Check — {Day, DD Mon YYYY}*
*Overall: {GREEN/AMBER/RED}*  ·  Reporting D-1: {date}  ·  All times IST

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📈 *GLOBAL GROWTH*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[code block]
Metric                                D-1        vs Yday   vs 7DMA   vs 30DMA   vs SDLW   Status
──────────────────────────────────────────────────────────────────────────────────────────────────
🔹 Installs                           {val}       {%}       {%}       {%}        {%}       {rag}
🔸 Trial Starts                       {val}       {%}       {%}       {%}        {%}       {rag}
🔸 Install→Trial on same day %        {val}%      {%}       {%}       {%}        {%}       {rag}
🔸 Same-Day Trial Cancel %            {val}%      {%}       {%}       {%}        {%}       {rag}
🔸 Trial(of d-2)→FullSub(on d-1) %   {val}%      {%}       {%}       {%}        {%}       {rag}
🔸 Above FullSub Cancel %             {val}%      {%}       {%}       {%}        {%}       {rag}
🔸 Revenue (₹)                        {val}L      {%}       {%}       {%}        {%}       {rag}
🔹 Intl Revenue (₹)                   {val}L      {%}       {%}       {%}        {%}       {rag}
🔹 Total Full Sub Converts            {val}       {%}       {%}       {%}        {%}       {rag}
[/code block]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📺 *CONSUMPTION — Active Trial*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[code block]
Metric               D-1       vs Yday   vs 7DMA   vs 30DMA   vs SDLW   Status
────────────────────────────────────────────────────────────────────────────────
🔸 VP Time/VP User   {val}min  {%}       {%}       {%}        {%}       {rag}
🔹 VP Users          {val}     {%}       {%}       {%}        {%}       {rag}
[/code block]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📺 *CONSUMPTION — Active Full Sub*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[code block]
Metric               D-1       vs Yday   vs 7DMA   vs 30DMA   vs SDLW   Status
────────────────────────────────────────────────────────────────────────────────
🔸 VP Time/VP User   {val}min  {%}       {%}       {%}        {%}       {rag}
🔹 VP Users          {val}     {%}       {%}       {%}        {%}       {rag}
[/code block]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 *KEY INSIGHTS*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• {insight_1}
• {insight_2}
• {insight_3}
```

---
name: Slack code block formatting
description: Slack multi-line code blocks require triple backticks on their own lines, not glued to content
type: feedback
originSessionId: ce6bb13a-e057-49d6-b20a-29f2ca1b049a
---
When posting Slack messages with multi-line code blocks (e.g. the QuickTV daily health check tables), the opening and closing triple backticks must each be on their own line. Do NOT write ` ```Metric ... ` or ` ... 🟢``` ` with backticks attached to content on the same line.

**Why:** When ``` is glued to text on the same line, Slack parses it inconsistently — section headings and content outside the intended table end up wrapped inside code blocks, breaking the layout. This happened on the 2026-04-18 daily health check post to #quicktv-core.

**How to apply:** For any Slack message using ```...``` fenced code blocks, structure as:

```
Heading line outside block

```
table content line 1
table content line 2
```

Next heading outside block
```

Each ``` goes on its own line with nothing else on it. Applies to `slack_send_message`, `slack_send_message_draft`, and canvas tools.
---



Send to `#quicktv-core` (C07BN97UC6N) via Slack MCP.


---

## Error Handling

- **Agg table returns 0 rows:** the `quicktv_health_report_agg` scheduled query hasn't run yet or failed. Wait until after 09:15 IST and retry. Do NOT fall back to querying the 5 source tables manually — flag `:warning: Agg table not yet available for {date}` and note the delay.
- **`psr_is_fallback = TRUE`:** D-1 PSR was unavailable; D-2 value used instead. Note in Slack header: `PSR date: {psr_report_date} (D-2 fallback)`.
- **Any metric column is NULL:** indicates the upstream source table had a gap for that date. Mark that metric row as `:warning: N/A` and continue with remaining metrics.
- **Agg table appears stale** (i.e., you run the query and `generated_at` is from a prior day): flag `:warning: Agg table stale, generated_at = {timestamp}` and do not send partial data.
