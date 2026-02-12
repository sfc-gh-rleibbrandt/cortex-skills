# Recurring Level 1: All Snowflake (External Paid)

**Period:** Mar 2025 → Jan 2026  
**Filter:** Recurring queries only, External accounts (excludes Internal, Trial, Partner Access)  
**Run Date:** 2026-02-12

## Results

| Scope | Mar 2025 Jobs | Jan 2026 Jobs | Growth | Mar Cr/1K | Jan Cr/1K | Credits Δ | Mar Avg ms | Jan Avg ms | Exec Δ |
|-------|---------------|---------------|--------|-----------|-----------|-----------|------------|------------|--------|
| **Recurring** | 87.9B | 174.8B | **+98.9%** | 0.7535 | 0.463 | **+38.6%** | 1,195 ms | 791 ms | **+33.8%** |

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
mar AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
jan AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    'Recurring (External Paid)' AS scope,
    f.total_jobs AS "Mar 2025 Jobs",
    j.total_jobs AS "Jan 2026 Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 1) AS "Jobs Growth %",
    ROUND(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 4) AS "Mar Cr/1K",
    ROUND(j.total_credits * 1000 / NULLIF(j.total_jobs, 0), 4) AS "Jan Cr/1K",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) AS "Mar Avg ms",
    ROUND(j.total_dur_ms / NULLIF(j.total_jobs, 0), 0) AS "Jan Avg ms",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM mar f, jan j
```
