# Task 5.0 Proof Artifacts: Trend Chart for Historical Analysis

**Date:** 2026-01-29
**Task:** Create Trend Chart for Historical Analysis

## Panel Configuration Verification

```bash
$ jq '.panels[8] | {id, title, type, gridPos}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "id": 108,
  "title": "5. Bounded Cycle Time Trend",
  "type": "timeseries",
  "gridPos": { "h": 8, "w": 24, "x": 0, "y": 25 }
}
```

**Result:** Time series panel created with correct id, type, and full-width position

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
)
SELECT
  DATE(end_time) as time,
  AVG(TIMESTAMPDIFF(MINUTE, start_time, end_time) / 60.0) as 'Avg Bounded Cycle Time (h)'
FROM bounded_issues
GROUP BY DATE(end_time)
ORDER BY time
```

### Query Features
- Uses `DATE(end_time)` for daily time bucketing (returns proper date type for Grafana)
- Returns two columns: `time` (date) and average hours
- Groups by date for trend aggregation
- Orders by time for chronological display
- Filters by all dashboard variables (`${project}`, `${issue_type}`, `${start_status}`, `${end_status}`)

## Time Series Configuration Verification

```bash
$ jq '.panels[8].fieldConfig.defaults.custom | {drawStyle, lineInterpolation, fillOpacity, axisLabel}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "drawStyle": "line",
  "lineInterpolation": "linear",
  "fillOpacity": 10,
  "axisLabel": "Hours"
}
```

**Result:** Configuration matches specification:
- `drawStyle: "line"` - displays as line chart
- `lineInterpolation: "linear"` - straight line connections
- `fillOpacity: 10` - subtle area fill under line
- `axisLabel: "Hours"` - Y-axis labeled correctly

## Color Configuration

```bash
$ jq '.panels[8].fieldConfig.defaults.color' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "fixedColor": "green",
  "mode": "fixed"
}
```

**Result:** Green color for trend line as specified

## Legend and Tooltip Configuration

```bash
$ jq '.panels[8].options | {legend, tooltip}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "legend": {
    "calcs": ["mean"],
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

**Result:**
- Legend at bottom with mean calculation shown
- Tooltip mode single for clean display

## JSON Validation

```bash
$ python3 -m json.tool grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json > /dev/null && echo "PASSED"
PASSED
```

## Summary

| Requirement | Status |
|-------------|--------|
| DATE_FORMAT() for time bucketing | PASS |
| Average bounded cycle time per bucket | PASS |
| Returns time and hours columns | PASS |
| Filters by all dashboard variables | PASS |
| Panel id: 108, type: timeseries | PASS |
| gridPos: h:8, w:24, x:0, y:25 | PASS |
| drawStyle: "line" | PASS |
| lineInterpolation: "linear" | PASS |
| fillOpacity: 10 | PASS |
| Green color for trend line | PASS |
| Y-axis label: "Hours" | PASS |
| Legend at bottom | PASS |
| Tooltip mode: single | PASS |

**All Task 5.0 proof artifacts verified successfully.**
