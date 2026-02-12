# Recurring Level 3a: Analytics by Use Case (External Paid)
**Period:** Mar 2025 → Jan 2026  
**Generated:** 2026-01-16

## Results

| Use Case | Mar Jobs | Jan Jobs | Growth | Credits Δ | Exec Δ |
|----------|----------|----------|--------|-----------|--------|
| Business Intelligence | 37.6B | 69.5B | +85% | +34.7% | +36.5% |
| Interactive & Powered By | 4.9B | 8.3B | +69% | +21.8% | +21.7% |
| BI Tools (3P) | 3.0B | 6.6B | +122% | +55.1% | +60.1% |
| Applied Analytics | 685M | 1.3B | +90% | +25.4% | +20.7% |
| Lakehouse Analytics | 34M | 337M | **+894%** | +65.2% | +59.2% |

## Highlights
- **Explosive growth:** Lakehouse Analytics (+894%)
- **Best efficiency:** Lakehouse (+65.2% credits), BI Tools (+60.1% exec)
- **Recurring Applied Analytics improved** vs all-workloads (was -10.8%, now +25.4%)

## SQL Query
```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
        r.feature_vector['Analytics']['use_case']::STRING AS use_case,
        SUM(r.jobs) AS total_jobs,
        SUM(r.total_credits) AS total_credits,
        SUM(r.dur_xp_executing) AS total_dur_ms
    FROM snowscience.job_analytics.job_feature_daily_account_rollup r
    JOIN snowscience.dimensions.dim_accounts_history a 
      ON r.deployment = a.snowflake_deployment 
      AND r.account_id = a.snowflake_account_id 
      AND r.ds = a.general_date
    WHERE r.ds >= '2025-03-01' AND r.ds < '2026-02-01'
      AND DATE_TRUNC('month', r.ds) IN ('2025-03-01', '2026-01-01')
      AND r.feature_vector:is_recurrent::STRING = 'true'
      AND r.feature_vector:"Product Category"::STRING = 'Analytics'
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
p1 AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
p2 AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(p1.use_case, p2.use_case) AS use_case,
    p1.total_jobs AS "Mar Jobs",
    p2.total_jobs AS "Jan Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 0) AS "Growth %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM p1
FULL OUTER JOIN p2 ON p1.use_case = p2.use_case
ORDER BY "Jan Jobs" DESC;
```
