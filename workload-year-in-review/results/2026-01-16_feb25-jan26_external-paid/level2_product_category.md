# Level 2: By Product Category (External Paid)
**Period:** Feb 2025 → Jan 2026  
**Generated:** 2026-01-16

## Results

| Category | Feb Jobs | Jan Jobs | Growth | Credits Δ | Exec Δ |
|----------|----------|----------|--------|-----------|--------|
| Data Engineering | 52.0B | 113.3B | +118% | +38.6% | +34.6% |
| Analytics | 43.6B | 100.5B | +130% | +43.3% | +42.4% |
| Apps & Collaboration | 1.3B | 3.6B | +169% | +40.8% | +41.9% |
| OLTP | 621M | 3.4B | **+440%** | +46.1% | +50.5% |
| AI/ML | 357M | 1.6B | +350% | +35.8% | **+60.6%** |
| Platform | 146M | 427M | +192% | +30.1% | +14.9% |

## Highlights
- **Fastest growing:** OLTP (+440%), AI/ML (+350%)
- **Best execution improvement:** AI/ML (+60.6%)
- **Largest by volume:** Data Engineering (113B jobs)

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
p1 AS (SELECT * FROM metrics WHERE month = '2025-02-01'),
p2 AS (SELECT * FROM metrics WHERE month = '2026-01-01')
SELECT 
    COALESCE(p1.product_category, p2.product_category) AS product_category,
    p1.total_jobs AS "Feb Jobs",
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
