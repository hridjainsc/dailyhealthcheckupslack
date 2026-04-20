# QuickTV Daily Health Check — Routine Memory

## Overview
This repo stores the configuration and memory for the QuickTV daily health check routine.
Every session running this routine must read this file first and follow it exactly.

---

## BQ Query — USE EXACTLY AS-IS EVERY RUN

```sql
SELECT *
FROM `sharechat-production.quicktv_reporting.daily_health_check_agg`
WHERE report_date = (
  SELECT MAX(report_date)
  FROM `sharechat-production.quicktv_reporting.daily_health_check_agg`
)
LIMIT 1
```

- **Project ID:** `sharechat-production`
- **Tool:** `mcp__BigQuery-Remote-MCP--ShareChat-__execute_sql_readonly`
- Do NOT modify the query. Do NOT add location filters. Do NOT change table name.

---

## Slack Output

- **Channel ID:** `C0ATATHDERW` (`#qtv-claude-hrid-private`)
- **Tool:** `mcp__Slack__slack_send_message`

---

## Routine Steps

1. Run the BQ query above (read-only, `execute_sql_readonly`)
2. Parse all 106 metric values from the result row
3. Compute RAG status per metric using the thresholds below
4. Format the Slack message using the template below
5. Send to channel `C0ATATHDERW`

---

## Rate Metric Delta Computation

The DB stores rate metrics (ITR, Cancel %, PSR) as `_pp` (absolute percentage point difference).
RAG thresholds are **relative %**. Convert before comparing:

```
relative_pct = pp_value / benchmark_value × 100
```

For `vs_yday`: `benchmark = d1_value - pp_value` (i.e. yesterday's value)
For `vs_7dma`, `vs_30dma`: benchmark = the stored 7DMA / 30DMA value
For `vs_lw`: `benchmark = d1_value - lw_pp_value`

---

## RAG Thresholds

### 🔸 High-Importance Metrics

| Metric | Direction | vs Yday Amber | vs Yday Red | vs 7DMA Amber | vs 7DMA Red | vs 30DMA Amber | vs 30DMA Red |
|--------|-----------|---------------|-------------|---------------|-------------|----------------|--------------|
| Trial Starts | ↓ bad | < -10% | < -20% | < -7% | < -14% | < -5% | < -10% |
| Install→Trial % (ITR) | ↓ bad | < -7% | < -14% | < -5% | < -10% | < -4% | < -8% |
| Same-Day Cancel % | ↑ bad | > +7% | > +12% | > +4% | > +8% | > +3% | > +6% |
| Trial→FullSub % (PSR) | ↓ bad | < -12% | < -25% | < -6% | < -12% | < -4% | < -9% |
| Above FullSub Cancel % | ↑ bad | > +10% | > +20% | > +7% | > +14% | > +5% | > +10% |
| Revenue (₹) | ↓ bad | < -8% | < -16% | < -5% | < -10% | < -3% | < -7% |
| Trial VP Time/VP User | ↓ bad | < -4% | < -10% | < -4% | < -10% | < -5% | < -12% |
| FullSub VP Time/VP User | ↓ bad | < -4% | < -10% | < -4% | < -10% | < -5% | < -12% |

### 🔹 Supporting Metrics

| Metric | Direction | vs 7DMA Amber | vs 7DMA Red | vs 30DMA Amber | vs 30DMA Red |
|--------|-----------|---------------|-------------|----------------|--------------|
| Installs | ↓ bad | < -5% | < -10% | < -3% | < -6% |
| Intl Revenue (₹) | ↓ bad | < -10% | < -18% | < -7% | < -15% |
| Total Full Sub Converts | ↓ bad | < -5% | < -10% | < -3% | < -7% |
| Trial VP Users | ↓ bad | < -10% | < -20% | < -10% | < -20% |
| FullSub VP Users | ↓ bad | < -4% | < -9% | < -5% | < -10% |

**Overall Day RAG:** worst status among all 🔸 high-importance metrics.

---

## Slack Message Template

```
{rag_emoji} *QuickTV Daily Health Check — {Day, DD Mon YYYY}*
*Overall: {RED/AMBER/GREEN}*  ·  Reporting D-1: {report_date}  ·  All times IST

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📈 *GLOBAL GROWTH*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[monospace table: Metric | D-1 | vs Yday | vs 7DMA | vs 30DMA | vs SDLW | Status]
Rows: Installs, Trial Starts, Install→Trial %, Same-Day Cancel %,
      Trial(d-2)→FullSub(d-1) %, Above FullSub Cancel %, Revenue (₹),
      Intl Revenue (₹), Total Full Sub Converts

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📺 *CONSUMPTION — Active Trial*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[monospace table: VP Time/VP User, VP Users]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📺 *CONSUMPTION — Active Full Sub*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[monospace table: VP Time/VP User, VP Users]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 *KEY INSIGHTS*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• [3–5 bullet points on RED/AMBER metrics and notable positives]

_Note: Rate metric deltas (ITR, Cancel %, PSR) are shown as relative % deviation: (D-1 − benchmark) / benchmark × 100._
```

---

## Column → Metric Mapping (field order in BQ result)

| Index | Column | Metric |
|-------|--------|--------|
| 0 | report_date | Report date |
| 4 | installs_d1 | Installs D-1 |
| 7–10 | installs_vs_*_pct | Installs deltas |
| 11 | trial_starts_d1 | Trial Starts D-1 |
| 14–17 | trial_starts_vs_*_pct | Trial Starts deltas |
| 18 | itr_d1_pct | ITR D-1 |
| 21–24 | itr_vs_*_pp | ITR deltas (pp → convert to relative %) |
| 36 | same_day_cancel_d1_pct | Same-Day Cancel D-1 |
| 39–42 | same_day_cancel_vs_*_pp | Same-Day Cancel deltas (pp → relative %) |
| 43 | psr_d1_pct | PSR D-1 |
| 44–45 | psr_7dma_pct / psr_30dma_pct | PSR benchmarks |
| 46–49 | psr_vs_*_pp | PSR deltas (pp → relative %) |
| 50 | fullsub_cancel_d1_pct | FullSub Cancel D-1 |
| 53–56 | fullsub_cancel_vs_*_pp | FullSub Cancel deltas (pp → relative %) |
| 57 | revenue_d1_L | Revenue D-1 (Lakhs) |
| 60–63 | revenue_vs_*_pct | Revenue deltas |
| 64 | intl_revenue_d1_L | Intl Revenue D-1 |
| 67–70 | intl_revenue_vs_*_pct | Intl Revenue deltas |
| 71 | full_sub_converts_d1 | Full Sub Converts D-1 |
| 74–77 | full_sub_converts_vs_*_pct | Full Sub Converts deltas |
| 78 | trial_vp_time_d1_min | Trial VP Time D-1 |
| 81–84 | trial_vp_time_vs_*_pct | Trial VP Time deltas |
| 85 | trial_vp_users_d1 | Trial VP Users D-1 |
| 88–91 | trial_vp_users_vs_*_pct | Trial VP Users deltas |
| 92 | fullsub_vp_time_d1_min | FullSub VP Time D-1 |
| 95–98 | fullsub_vp_time_vs_*_pct | FullSub VP Time deltas |
| 99 | fullsub_vp_users_d1 | FullSub VP Users D-1 |
| 102–105 | fullsub_vp_users_vs_*_pct | FullSub VP Users deltas |
