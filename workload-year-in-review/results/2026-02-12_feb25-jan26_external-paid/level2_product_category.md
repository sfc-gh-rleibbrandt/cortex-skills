# Level 2: By Product Category (External Paid)

**Period:** Feb 2025 → Jan 2026  
**Filter:** External accounts only (excludes Internal, Trial, Partner Access)  
**Run Date:** 2026-02-12

## Results

| Product Category | Feb 2025 Jobs | Jan 2026 Jobs | Growth % | Credits Δ % | Exec Δ % |
|------------------|---------------|---------------|----------|-------------|----------|
| Data Engineering | 52.0B | 113.3B | +118% | +38.6% | +34.6% |
| Analytics | 43.6B | 100.5B | +130% | +43.3% | +42.4% |
| OLTP | 621M | 3.4B | **+440%** | +46.1% | +50.5% |
| Apps & Collaboration | 1.3B | 3.6B | +169% | +40.8% | +41.9% |
| AI/ML | 357M | 1.6B | +350% | +35.8% | +60.6% |
| Platform | 146M | 427M | +192% | +30.1% | +14.9% |

## Key Observations

- **OLTP** shows massive growth (+440%) and strong improvement (+50.5% exec) when excluding internal accounts
- **AI/ML** has best execution improvement (+60.6%)
- **All categories show positive improvement** (unlike with internal accounts where OLTP was negative)

## SQL Query

```sql
WITH metrics AS (
    SELECT 
        DATE_TRUNC('month', r.ds) AS month,
        r.feature_vector:"Product Category"::STRING AS product_category,
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
      AND r.feature_vector:"Product Category"::STRING IS NOT NULL
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
feb AS (SELECT * FROM metrics WHERE month = '2025-02-01'),
jan AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(f.product_category, j.product_category) AS product_category,
    f.total_jobs AS "Feb Jobs",
    j.total_jobs AS "Jan Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 0) AS "Growth %",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM feb f
FULL OUTER JOIN jan j ON f.product_category = j.product_category
ORDER BY "Jan Jobs" DESC
```
