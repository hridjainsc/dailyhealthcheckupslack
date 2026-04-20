# QuickTV Daily Health Check — Routine Memory

## CRITICAL RULES
- **ALWAYS use BigQuery Remote MCP** (`mcp__BigQuery-Remote-MCP--ShareChat-__execute_sql_readonly`) — never local MCP
- **ALWAYS use the exact query below** — do NOT modify it
- **Slack channel:** `C07BN97UC6N` (`#quicktv-core`) — NOT any test channel

---

## BQ Query — USE EXACTLY AS-IS EVERY RUN

```sql
SELECT *
FROM `maximal-furnace-783.quicktv_analytics.quicktv_health_report_agg`
WHERE report_date = DATE_SUB(CURRENT_DATE('Asia/Kolkata'), INTERVAL 1 DAY);
```

- **Project ID:** `maximal-furnace-783`
- **Tool:** `mcp__BigQuery-Remote-MCP--ShareChat-__execute_sql_readonly`
- Do NOT change table name, project, or date filter.
- Returns **one row** — no further aggregation needed.

**If 0 rows returned:** agg scheduled query hasn't run yet (runs ~08:00 IST). Wait and retry. Do NOT query source tables manually. Post `:warning: Agg table not yet available for {date}`.

---

## Delta Column Convention

**All delta columns use `_pct` suffix — already relative % change.**
Rate metrics (ITR, Cancel %, PSR) are also stored as relative %: `(d1 - benchmark) / benchmark * 100`.
No manual pp→% conversion needed.

---

## Slack Output

- **Channel:** `C07BN97UC6N` (`#quicktv-core`)
- **Tool:** `mcp__Slack__slack_send_message`
- Code blocks: triple backticks **must be on their own lines** — never glued to content.

---

## RAG Thresholds

**Overall day RAG is driven by 🔸 metrics only.**
Per-metric RAG = worst of vs_yday / vs_7dma / vs_30dma (vs SDLW is informational only, do NOT use for RAG).

### 🔸 High-importance metrics (drive overall day RAG)
Trial Starts, Install→Trial %, Same-Day Cancel %, PSR, Above FullSub Cancel %, Revenue, VP Time/VP User (trial & fullsub)

### 🔹 Supporting metrics (per-row RAG only, do NOT affect overall)
Installs, Intl Revenue, Total Full Sub Converts, VP Users (trial & fullsub)

#### Rate Metrics (Tier A)

| Metric | Direction | vs Yday Amber | vs Yday Red | vs 7DMA Amber | vs 7DMA Red | vs 30DMA Amber | vs 30DMA Red |
|--------|-----------|---------------|-------------|---------------|-------------|----------------|--------------|
| Same-Day Cancel % | ↑ bad | > +7% | > +12% | > +4% | > +8% | > +3% | > +6% |
| Above FullSub Cancel % | ↑ bad | > +10% | > +20% | > +7% | > +14% | > +5% | > +10% |
| PSR (Trial→FullSub %) | ↓ bad | < -12% | < -25% | < -6% | < -12% | < -4% | < -9% |
| ITR (Install→Trial %) | ↓ bad | < -7% | < -14% | < -5% | < -10% | < -3% | < -7% |

#### Stable Count/Value Metrics (Tier B)

| Metric | Direction | vs Yday Amber | vs Yday Red | vs 7DMA Amber | vs 7DMA Red | vs 30DMA Amber | vs 30DMA Red |
|--------|-----------|---------------|-------------|---------------|-------------|----------------|--------------|
| VP Time/VP User (trial & fullsub), VP Users (fullsub) | ↓ bad | < -4% | < -6% | < -4% | < -8% | < -5% | < -9% |
| Installs, Trial Starts, Revenue, Full Sub Converts | ↓ bad | < -8% | < -16% | < -5% | < -10% | < -3% | < -7% |

#### High-Variance Metrics (Tier C)

| Metric | Direction | vs Yday Amber | vs Yday Red | vs 7DMA Amber | vs 7DMA Red | vs 30DMA Amber | vs 30DMA Red |
|--------|-----------|---------------|-------------|---------------|-------------|----------------|--------------|
| VP Users (trial) | ↓ bad | < -8% | < -14% | < -10% | < -16% | < -10% | < -18% |
| Intl Revenue | ↓ bad | < -15% | < -25% | < -10% | < -18% | < -7% | < -15% |

---

## Slack Message Format

```
{rag_emoji} *QuickTV Daily Health Check — {Day, DD Mon YYYY}*
*Overall: {GREEN/AMBER/RED}*  ·  Reporting D-1: {date}  ·  All times IST

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📈 *GLOBAL GROWTH*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
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
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📺 *CONSUMPTION — Active Trial*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Metric               D-1       vs Yday   vs 7DMA   vs 30DMA   vs SDLW   Status
────────────────────────────────────────────────────────────────────────────────
🔸 VP Time/VP User   {val}min  {%}       {%}       {%}        {%}       {rag}
🔹 VP Users          {val}     {%}       {%}       {%}        {%}       {rag}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📺 *CONSUMPTION — Active Full Sub*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
Metric               D-1       vs Yday   vs 7DMA   vs 30DMA   vs SDLW   Status
────────────────────────────────────────────────────────────────────────────────
🔸 VP Time/VP User   {val}min  {%}       {%}       {%}        {%}       {rag}
🔹 VP Users          {val}     {%}       {%}       {%}        {%}       {rag}
```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💡 *KEY INSIGHTS*
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
• {insight_1}
• {insight_2}
• {insight_3}
```

---

## PSR Fallback

If `psr_is_fallback = TRUE`: append `†` to the PSR row label and add footnote:
`† PSR uses D-2 (D-1 not yet available)`. Note in Slack header: `PSR date: {psr_report_date} (D-2 fallback)`.

---

## Error Handling

| Condition | Action |
|-----------|--------|
| 0 rows returned | Wait until after 08:15 IST, retry. Post `:warning: Agg table not yet available for {date}`. |
| `psr_is_fallback = TRUE` | Add `†` to PSR row + footnote |
| Any metric column NULL | Mark row `:warning: N/A`, continue |
| `generated_at` from prior day | Post `:warning: Agg table stale, generated_at = {timestamp}`, do not send data |
