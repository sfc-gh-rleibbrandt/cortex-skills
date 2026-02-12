# Recurring Level 1: All Snowflake (External Paid)
**Period:** Mar 2025 → Jan 2026  
**Generated:** 2026-01-16

> Note: Recurring analysis uses Mar 2025 as baseline (is_recurrent flag stabilized then)

## Results

| Scope | Mar 2025 Jobs | Jan 2026 Jobs | Growth | Credits Δ | Exec Δ |
|-------|---------------|---------------|--------|-----------|--------|
| **Recurring** | 87.9B | 174.8B | **+98.9%** | **+38.6%** | **+33.8%** |

## Key Metrics
- **Mar 2025 Cr/1K Jobs:** 0.7535
- **Jan 2026 Cr/1K Jobs:** 0.4630
- **Mar 2025 Avg Exec:** 1,195 ms
- **Jan 2026 Avg Exec:** 791 ms

## SQL Query
```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
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
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1
),
p1 AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
p2 AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    'Recurring' AS scope,
    p1.total_jobs AS "Mar 2025 Jobs",
    p2.total_jobs AS "Jan 2026 Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 4) AS "Mar Cr/1K",
    ROUND(p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0), 4) AS "Jan Cr/1K",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) AS "Mar Avg ms",
    ROUND(p2.total_dur_ms / NULLIF(p2.total_jobs, 0), 0) AS "Jan Avg ms",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM p1, p2;
```
