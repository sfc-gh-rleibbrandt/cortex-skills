# Level 3a: Analytics by Use Case (External Paid)

**Period:** Feb 2025 → Jan 2026  
**Filter:** External accounts only (excludes Internal, Trial, Partner Access)  
**Run Date:** 2026-02-12

## Results

| Use Case | Feb 2025 Jobs | Jan 2026 Jobs | Growth % | Credits Δ % | Exec Δ % |
|----------|---------------|---------------|----------|-------------|----------|
| Business Intelligence | 34.9B | 81.6B | +134% | +46.0% | +45.3% |
| Interactive & Powered By Analytics | 5.1B | 9.5B | +86% | +34.1% | +24.6% |
| BI Tools (3P) | 2.6B | 7.5B | +186% | +58.8% | +62.1% |
| Applied Analytics | 962M | 1.5B | +55% | -10.8% | -10.2% |
| Lakehouse Analytics | 26M | 420M | **+1,547%** | +66.8% | +61.2% |

## Key Observations

- **Lakehouse Analytics** explosive growth (+1,547%) with excellent efficiency gains
- **BI Tools (3P)** best execution improvement (+62.1%) - Tableau, Power BI, Looker
- **Applied Analytics** only use case with regression (-10.2% exec)
- **Business Intelligence** largest volume (81.6B jobs), strong improvement

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
feb AS (SELECT * FROM metrics WHERE month = '2025-02-01'),
jan AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(f.use_case, j.use_case) AS use_case,
    f.total_jobs AS "Feb Jobs",
    j.total_jobs AS "Jan Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 0) AS "Growth %",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f
FULL OUTER JOIN jan j ON f.use_case = j.use_case
ORDER BY "Jan Jobs" DESC
```
