# SPI Execution Index

**Period:** Feb 2025 → Jan 2026  
**Run Date:** 2026-02-12

## Results

| Metric | Feb 2025 | Jan 2026 | Point Δ | % Improvement |
|--------|----------|----------|---------|---------------|
| **Execution Index** | -45.7 | -59.0 | **+13.2** | **+29%** |

**Interpretation:**
- Baseline = 0
- More negative = faster (improvement)
- Point Δ positive = improved (index got more negative/faster)

## SQL Query

```sql
WITH monthly AS (
    SELECT 
        DATE_TRUNC('month', ds) AS month,
        MAX(MONTH_INDEX_XP) AS execution_index
    FROM snowscience.job_analytics.spi_tracker_index 
    WHERE DATE_TRUNC('month', ds) IN ('2025-02-01', '2026-01-01')
    GROUP BY 1
),
feb AS (SELECT * FROM monthly WHERE month = '2025-02-01'),
jan AS (SELECT * FROM monthly WHERE month = '2026-01-01')
SELECT 
    'SPI Execution Index' AS metric,
    ROUND(f.execution_index, 1) AS "Feb 2025",
    ROUND(j.execution_index, 1) AS "Jan 2026",
    ROUND(ABS(j.execution_index) - ABS(f.execution_index), 1) AS "Point Δ",
    ROUND((ABS(j.execution_index) - ABS(f.execution_index)) / NULLIF(ABS(f.execution_index), 0) * 100, 0) AS "% Improvement"
FROM feb f, jan j
```
