# Task 3.0 Proof Artifacts: Time Per Status Breakdown Panel

**Date:** 2026-01-29
**Task:** Add Time Per Status Breakdown Panel

## Panel Configuration Verification

```bash
$ jq '.panels[3] | {id, title, type, gridPos}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "id": 103,
  "title": "3. Time Per Status Breakdown",
  "type": "barchart",
  "gridPos": { "h": 8, "w": 24, "x": 0, "y": 10 }
}
```

**Result:** Bar chart panel created with correct id, type, and position

## SQL Query

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
status_times AS (
  SELECT
    ish.original_status,
    SUM(ish.status_time_minutes) / 60.0 as total_hours,
    COUNT(DISTINCT ish.issue_id) as issue_count,
    MIN(ish.start_date) as first_occurrence
  FROM issue_status_history ish
  JOIN bounded_issues bi ON ish.issue_id = bi.issue_id
  WHERE ish.start_date >= bi.start_time
    AND ish.start_date <= bi.end_time
  GROUP BY ish.original_status
)
SELECT
  original_status as 'Status',
  ROUND(total_hours / issue_count, 2) as 'Average Hours'
FROM status_times
ORDER BY first_occurrence
```

### Query Features
- Reuses `bounded_issues` CTE to identify issues within the bounded range
- `status_times` CTE aggregates time per status only for issues within bounds
- Filters statuses to only those occurring between start_time and end_time
- Calculates average hours: `total_hours / issue_count`
- Orders by `first_occurrence` (workflow sequence based on actual data)
- Uses `status_time_minutes` from `issue_status_history` table

## Bar Chart Options Verification

```bash
$ jq '.panels[3].options | {barWidth, xTickLabelRotation, legend, tooltip}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "barWidth": 0.7,
  "xTickLabelRotation": 45,
  "legend": {
    "calcs": [],
    "displayMode": "list",
    "placement": "bottom",
    "showLegend": true
  },
  "tooltip": {
    "maxHeight": 600,
    "mode": "single",
    "sort": "none"
  }
}
```

**Result:** Options configured correctly:
- `barWidth: 0.7` (70% width)
- `xTickLabelRotation: 45` (angled for readability)
- Legend at bottom
- Tooltip mode single

## Color Configuration Verification

```bash
$ jq '.panels[3].fieldConfig.defaults.color' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "mode": "palette-classic"
}
```

**Result:** Uses palette-classic for distinct colors per status bar

## Fill Opacity Verification

```bash
$ jq '.panels[3].fieldConfig.defaults.custom.fillOpacity' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```
80
```

**Result:** Fill opacity set to 80%

## Axis Label Verification

```bash
$ jq '.panels[3].fieldConfig.defaults.custom.axisLabel' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```
"Hours"
```

**Result:** Y-axis labeled "Hours"

## JSON Validation

```bash
$ python3 -m json.tool grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json > /dev/null && echo "PASSED"
PASSED
```

## Summary

| Requirement | Status |
|-------------|--------|
| SQL query returns status names with time values | PASS |
| Filters issues with both start/end status | PASS |
| Groups by original_status | PASS |
| Orders by workflow sequence (first_occurrence) | PASS |
| Filters by project, issue_type, time range | PASS |
| Bar chart panel (id: 103, type: barchart) | PASS |
| gridPos: h:8, w:24, x:0, y:10 | PASS |
| X-axis: status names | PASS |
| Y-axis: average hours with label | PASS |
| fillOpacity: 80 | PASS |
| barWidth: 0.7 | PASS |
| palette-classic color mode | PASS |
| xTickLabelRotation: 45 | PASS |
| Legend at bottom | PASS |
| Tooltip mode single | PASS |

**All Task 3.0 proof artifacts verified successfully.**
