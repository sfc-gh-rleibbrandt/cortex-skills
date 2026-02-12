# SPI Execution Index
**Period:** Feb 2025 → Jan 2026  
**Generated:** 2026-01-16

## Results

| Metric | Feb 2025 | Jan 2026 | Point Δ | % Improvement |
|--------|----------|----------|---------|---------------|
| **SPI Execution Index** | -45.7 | -59.0 | **+13.2** | **+29%** |

> Note: More negative = better. The index measures execution efficiency relative to baseline.

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
p1 AS (SELECT * FROM monthly WHERE month = '2025-02-01'),
p2 AS (SELECT * FROM monthly WHERE month = '2026-01-01')
SELECT 
    'SPI Execution Index' AS metric,
    ROUND(p1.execution_index, 1) AS "Feb 2025",
    ROUND(p2.execution_index, 1) AS "Jan 2026",
    ROUND(ABS(p2.execution_index) - ABS(p1.execution_index), 1) AS "Point Δ",
    ROUND((ABS(p2.execution_index) - ABS(p1.execution_index)) / NULLIF(ABS(p1.execution_index), 0) * 100, 0) AS "% Improvement"
FROM p1, p2;
```
