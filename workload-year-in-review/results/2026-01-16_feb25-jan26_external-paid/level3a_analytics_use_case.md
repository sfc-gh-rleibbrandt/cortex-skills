# Level 3a: Analytics by Use Case (External Paid)
**Period:** Feb 2025 → Jan 2026  
**Generated:** 2026-01-16

## Results

| Use Case | Feb Jobs | Jan Jobs | Growth | Credits Δ | Exec Δ |
|----------|----------|----------|--------|-----------|--------|
| Business Intelligence | 34.9B | 81.6B | +134% | +46.0% | +45.3% |
| Interactive & Powered By | 5.1B | 9.5B | +86% | +34.1% | +24.6% |
| BI Tools (3P) | 2.6B | 7.5B | +186% | +58.8% | **+62.1%** |
| Applied Analytics | 962M | 1.5B | +55% | -10.8% | -10.2% |
| Lakehouse Analytics | 26M | 420M | **+1,547%** | +66.8% | +61.2% |

## Highlights
- **Explosive growth:** Lakehouse Analytics (+1,547%)
- **Best efficiency gains:** Lakehouse Analytics (+66.8% credits), BI Tools (+62.1% exec)
- **Regression:** Applied Analytics showed slight efficiency decline

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
    WHERE r.ds >= '2025-02-01' AND r.ds < '2026-02-01'
      AND DATE_TRUNC('month', r.ds) IN ('2025-02-01', '2026-01-01')
      AND r.feature_vector:"Product Category"::STRING = 'Analytics'
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
p1 AS (SELECT * FROM metrics WHERE month = '2025-02-01'),
p2 AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(p1.use_case, p2.use_case) AS use_case,
    p1.total_jobs AS "Feb Jobs",
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
