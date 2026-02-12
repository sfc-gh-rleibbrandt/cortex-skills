# Recurring Level 2: By Product Category (External Paid)

**Period:** Mar 2025 → Jan 2026  
**Filter:** Recurring queries only, External accounts (excludes Internal, Trial, Partner Access)  
**Run Date:** 2026-02-12

## Results

| Product Category | Mar 2025 Jobs | Jan 2026 Jobs | Growth % | Credits Δ % | Exec Δ % |
|------------------|---------------|---------------|----------|-------------|----------|
| Analytics | 46.2B | 86.0B | +86% | +34.8% | +35.3% |
| Data Engineering | 40.0B | 82.2B | +106% | +41.9% | +34.6% |
| OLTP | 324M | 2.5B | **+667%** | +65.6% | +50.1% |
| Apps & Collaboration | 974M | 2.5B | +157% | +49.0% | +46.9% |
| AI/ML | 403M | 1.3B | +211% | +54.1% | +59.0% |
| Platform | 78M | 313M | +301% | +51.6% | +52.4% |

## Key Observations

- **OLTP recurring** shows exceptional improvement (+65.6% credits, +50.1% exec)
- **All categories positive** - stable production workloads all improved
- **AI/ML** strong execution gains (+59.0%)

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
    WHERE r.ds >= '2025-03-01' AND r.ds < '2026-02-01'
      AND DATE_TRUNC('month', r.ds) IN ('2025-03-01', '2026-01-01')
      AND r.feature_vector:is_recurrent::STRING = 'true'
      AND r.feature_vector:"Product Category"::STRING IS NOT NULL
      AND a.snowflake_account_type <> 'Internal'
      AND a.agreement_type NOT IN ('Trial', 'Partner Access')
    GROUP BY 1, 2
),
mar AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
jan AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(f.product_category, j.product_category) AS product_category,
    f.total_jobs AS "Mar Jobs",
    j.total_jobs AS "Jan Jobs",
    ROUND((j.total_jobs - f.total_jobs) * 100.0 / NULLIF(f.total_jobs, 0), 0) AS "Growth %",
    ROUND((f.total_credits * 1000 / NULLIF(f.total_jobs, 0) - j.total_credits * 1000 / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_credits * 1000 / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((f.total_dur_ms / NULLIF(f.total_jobs, 0) - j.total_dur_ms / NULLIF(j.total_jobs, 0)) 
          / NULLIF(f.total_dur_ms / NULLIF(f.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM mar f
FULL OUTER JOIN jan j ON f.product_category = j.product_category
ORDER BY "Jan Jobs" DESC
```
