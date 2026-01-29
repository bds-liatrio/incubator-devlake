# Task 2.0 Proof Artifacts: Bounded Cycle Time Stat Panels

**Date:** 2026-01-29
**Task:** Implement Bounded Cycle Time Stat Panels

## Panel Count Verification

```bash
$ jq '.panels | length' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
3
```

**Result:** Dashboard now has 3 panels (1 text intro + 2 stat panels)

## Average Bounded Cycle Time Panel (id: 101)

### SQL Query

```sql
WITH bounded_issues AS (
  SELECT
    i.id as issue_id,
    MIN(CASE WHEN ish.original_status = '${start_status}' THEN ish.start_date END) as start_time,
    MIN(CASE WHEN ish.original_status = '${end_status}' THEN ish.start_date END) as end_time
  FROM issues i
  JOIN board_issues bi ON i.id = bi.issue_id
  JOIN project_mapping pm ON bi.board_id = pm.row_id AND pm.`table` = 'boards'
  JOIN issue_status_history ish ON i.id = ish.issue_id
  WHERE
    pm.project_name IN (${project})
    AND i.original_type IN (${issue_type})
    AND ish.original_status IN ('${start_status}', '${end_status}')
  GROUP BY i.id
  HAVING start_time IS NOT NULL
    AND end_time IS NOT NULL
    AND end_time > start_time
    AND $__timeFilter(end_time)
)
SELECT
  AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time) / 60.0) as 'Average Bounded Cycle Time (h)'
FROM bounded_issues
```

### Query Features
- Uses CTE (`bounded_issues`) for clean query structure
- Joins: `issues` -> `board_issues` -> `project_mapping` for project filtering
- Uses `MIN()` with CASE to get first entry into each status (ignores rework)
- Calculates time difference in hours using `TIMESTAMPDIFF(MINUTE, ...) / 60.0`
- Filters by all dashboard variables: `${project}`, `${start_status}`, `${end_status}`, `${issue_type}`
- Uses `$__timeFilter(end_time)` for Grafana time range filtering
- HAVING clause ensures valid bounded issues only (both statuses exist, end > start)

### Panel Configuration

```bash
$ jq '.panels[1] | {id, title, type, gridPos}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "id": 101,
  "title": "1. Average Bounded Cycle Time",
  "type": "stat",
  "gridPos": { "h": 7, "w": 8, "x": 0, "y": 3 }
}
```

### Threshold Configuration

```json
{
  "mode": "absolute",
  "steps": [
    { "color": "green", "value": null },
    { "color": "orange", "value": 24 },
    { "color": "red", "value": 168 }
  ]
}
```

**Result:** Thresholds set correctly (green < 24h, orange 24-168h, red >= 168h)

## Median Bounded Cycle Time Panel (id: 102)

### SQL Query

```sql
WITH bounded_issues AS (
  SELECT
    i.id as issue_id,
    MIN(CASE WHEN ish.original_status = '${start_status}' THEN ish.start_date END) as start_time,
    MIN(CASE WHEN ish.original_status = '${end_status}' THEN ish.start_date END) as end_time
  FROM issues i
  JOIN board_issues bi ON i.id = bi.issue_id
  JOIN project_mapping pm ON bi.board_id = pm.row_id AND pm.`table` = 'boards'
  JOIN issue_status_history ish ON i.id = ish.issue_id
  WHERE
    pm.project_name IN (${project})
    AND i.original_type IN (${issue_type})
    AND ish.original_status IN ('${start_status}', '${end_status}')
  GROUP BY i.id
  HAVING start_time IS NOT NULL
    AND end_time IS NOT NULL
    AND end_time > start_time
    AND $__timeFilter(end_time)
),
ranked_issues AS (
  SELECT
    TIMESTAMPDIFF(MINUTE, start_time, end_time) / 60.0 as cycle_time_hours,
    PERCENT_RANK() OVER (ORDER BY TIMESTAMPDIFF(MINUTE, start_time, end_time)) as pct_rank
  FROM bounded_issues
)
SELECT
  AVG(cycle_time_hours) as 'Median Bounded Cycle Time (h)'
FROM ranked_issues
WHERE pct_rank <= 0.5
```

### Query Features
- Extends average query with `ranked_issues` CTE
- Uses `PERCENT_RANK()` window function to calculate percentile ranking
- Filters to bottom 50% (`pct_rank <= 0.5`) and averages for median approximation
- Same variable filtering as average panel

### Panel Configuration

```bash
$ jq '.panels[2] | {id, title, type, gridPos}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "id": 102,
  "title": "2. Median Bounded Cycle Time",
  "type": "stat",
  "gridPos": { "h": 7, "w": 8, "x": 8, "y": 3 }
}
```

**Result:** Panel positioned correctly next to average panel

## Documentation Links Verification

```bash
$ jq '.panels[1].links[0], .panels[2].links[0]' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "targetBlank": true,
  "title": "Cycle Time",
  "url": "https://devlake.apache.org/docs/Metrics/LeadTimeForChanges"
}
```

**Result:** Both panels link to DevLake documentation

## JSON Validation

```bash
$ python3 -m json.tool grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json > /dev/null && echo "PASSED"
PASSED
```

## Summary

| Requirement | Status |
|-------------|--------|
| CTE-based SQL query structure | PASS |
| Joins issues -> board_issues -> project_mapping | PASS |
| MIN() for first status entry | PASS |
| TIMESTAMPDIFF calculation in hours | PASS |
| All variable filters applied | PASS |
| $__timeFilter() macro used | PASS |
| Average panel (id: 101) created | PASS |
| Median panel (id: 102) with PERCENT_RANK | PASS |
| Thresholds: green/orange(24)/red(168) | PASS |
| Unit set to hours ("h") | PASS |
| Documentation links added | PASS |
| Valid JSON structure | PASS |

**All Task 2.0 proof artifacts verified successfully.**
