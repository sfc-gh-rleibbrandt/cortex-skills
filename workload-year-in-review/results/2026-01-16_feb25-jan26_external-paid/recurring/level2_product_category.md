# Recurring Level 2: By Product Category (External Paid)
**Period:** Mar 2025 → Jan 2026  
**Generated:** 2026-01-16

## Results

| Category | Mar Jobs | Jan Jobs | Growth | Credits Δ | Exec Δ |
|----------|----------|----------|--------|-----------|--------|
| Analytics | 46.2B | 86.0B | +86% | +34.8% | +35.3% |
| Data Engineering | 40.0B | 82.2B | +106% | +41.9% | +34.6% |
| Apps & Collaboration | 974M | 2.5B | +157% | +49.0% | +46.9% |
| OLTP | 324M | 2.5B | **+667%** | **+65.6%** | +50.1% |
| AI/ML | 403M | 1.3B | +211% | +54.1% | +59.0% |
| Platform | 78M | 313M | +301% | +51.6% | +52.4% |

## Highlights
- **Fastest growing:** OLTP (+667%), Platform (+301%)
- **Best efficiency:** OLTP (+65.6% credits)
- **All categories positive:** Every category improved in both credits and execution

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
p1 AS (SELECT * FROM metrics WHERE month = '2025-03-01'),
p2 AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(p1.product_category, p2.product_category) AS product_category,
    p1.total_jobs AS "Mar Jobs",
    p2.total_jobs AS "Jan Jobs",
    ROUND((p2.total_jobs - p1.total_jobs) * 100.0 / NULLIF(p1.total_jobs, 0), 0) AS "Growth %",
    ROUND((p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0) - p2.total_credits * 1000 / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_credits * 1000 / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Credits Δ %",
    ROUND((p1.total_dur_ms / NULLIF(p1.total_jobs, 0) - p2.total_dur_ms / NULLIF(p2.total_jobs, 0)) 
          / NULLIF(p1.total_dur_ms / NULLIF(p1.total_jobs, 0), 0) * 100, 1) AS "Exec Δ %"
FROM p1
FULL OUTER JOIN p2 ON p1.product_category = p2.product_category
ORDER BY "Jan Jobs" DESC;
```
