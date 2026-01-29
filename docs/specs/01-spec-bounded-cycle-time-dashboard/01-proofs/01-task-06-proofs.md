# Task 6.0 Proof Artifacts: Detailed Issue Table

**Date:** 2026-01-29
**Task:** Build Detailed Issue Table

## Panel Configuration Verification

```bash
$ jq '.panels[9] | {id, title, type, gridPos}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "id": 109,
  "title": "6. Issue Details",
  "type": "table",
  "gridPos": { "h": 12, "w": 24, "x": 0, "y": 33 }
}
```

**Result:** Table panel created with correct id, type, and full-width position

## SQL Query

```sql
WITH bounded_issues AS (
  SELECT
    i.id as issue_id,
    i.issue_key,
    i.title,
    i.url,
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
  GROUP BY i.id, i.issue_key, i.title, i.url
  HAVING start_time IS NOT NULL
    AND end_time IS NOT NULL
    AND end_time > start_time
    AND $__timeFilter(end_time)
),
status_times AS (
  SELECT
    ish.issue_id,
    SUM(CASE WHEN ish.original_status LIKE '%Dev%' OR ish.original_status LIKE '%Progress%' OR ish.original_status LIKE '%Code%' THEN ish.status_time_minutes ELSE 0 END) / 60.0 as dev_time,
    SUM(CASE WHEN ish.original_status LIKE '%QA%' OR ish.original_status LIKE '%Test%' OR ish.original_status LIKE '%Verif%' THEN ish.status_time_minutes ELSE 0 END) / 60.0 as sqa_time
  FROM issue_status_history ish
  JOIN bounded_issues bi ON ish.issue_id = bi.issue_id
  WHERE ish.start_date >= bi.start_time AND ish.start_date <= bi.end_time
  GROUP BY ish.issue_id
)
SELECT
  bi.issue_key as 'Issue Key',
  bi.title as 'Title',
  bi.url as url_hidden,
  ROUND(TIMESTAMPDIFF(MINUTE, bi.start_time, bi.end_time) / 60.0, 2) as 'Bounded Cycle Time',
  ROUND(COALESCE(st.dev_time, 0), 2) as 'Dev Time',
  ROUND(COALESCE(st.sqa_time, 0), 2) as 'SQA Time'
FROM bounded_issues bi
LEFT JOIN status_times st ON bi.issue_id = st.issue_id
ORDER BY TIMESTAMPDIFF(MINUTE, bi.start_time, bi.end_time) DESC
```

### Query Features
- Returns issue-level data with key, title, and URL
- Calculates bounded cycle time for each issue
- Uses LIKE patterns to categorize dev-related and QA-related statuses
- LEFT JOIN ensures issues without status times still appear
- Default sort by bounded cycle time descending (slowest first)
- URL column included but hidden for linking

## Field Overrides Verification

```bash
$ jq '.panels[9].fieldConfig.overrides | length' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

**Result:** 4 field overrides configured

### Override 1: Bounded Cycle Time Color
```json
{
  "matcher": { "id": "byName", "options": "Bounded Cycle Time" },
  "properties": [
    { "id": "custom.cellOptions", "value": { "type": "color-text" } }
  ]
}
```

### Override 2: Issue Key Link
```json
{
  "matcher": { "id": "byName", "options": "Issue Key" },
  "properties": [
    { "id": "links", "value": [{ "targetBlank": true, "title": "Open in Jira", "url": "${__data.fields[\"url_hidden\"]}" }] }
  ]
}
```

### Override 3: Hidden URL Column
```json
{
  "matcher": { "id": "byName", "options": "url_hidden" },
  "properties": [
    { "id": "custom.hidden", "value": true }
  ]
}
```

### Override 4: Title Width
```json
{
  "matcher": { "id": "byName", "options": "Title" },
  "properties": [
    { "id": "custom.width", "value": 400 }
  ]
}
```

## Table Options Verification

```bash
$ jq '.panels[9].options' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "cellHeight": "sm",
  "footer": { "countRows": true, "enablePagination": false, "reducer": ["count"], "show": false },
  "showHeader": true,
  "sortBy": [{ "desc": true, "displayName": "Bounded Cycle Time" }]
}
```

**Result:**
- `cellHeight: "sm"` for compact display
- `showHeader: true`
- Default sort by Bounded Cycle Time descending

## Filterable Configuration

```bash
$ jq '.panels[9].fieldConfig.defaults.custom.filterable' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```
true
```

**Result:** Column filtering enabled

## Thresholds Configuration

```bash
$ jq '.panels[9].fieldConfig.defaults.thresholds' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

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

**Result:** Thresholds set (green < 24h, orange 24-168h, red >= 168h)

## JSON Validation

```bash
$ python3 -m json.tool grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json > /dev/null && echo "PASSED"
PASSED
```

## Summary

| Requirement | Status |
|-------------|--------|
| SQL returns issue_key, title, url_hidden | PASS |
| Bounded Cycle Time calculated | PASS |
| Dev Time from dev-related statuses | PASS |
| SQA Time from QA-related statuses | PASS |
| Table panel (id: 109, type: table) | PASS |
| gridPos: h:12, w:24, x:0, y:33 | PASS |
| filterable: true | PASS |
| cellHeight: sm | PASS |
| showHeader: true | PASS |
| Issue Key links to url_hidden | PASS |
| url_hidden column hidden | PASS |
| Bounded Cycle Time color-text | PASS |
| Thresholds: green/orange(24)/red(168) | PASS |
| Title width 400px | PASS |
| Default sort by Bounded Cycle Time desc | PASS |

**All Task 6.0 proof artifacts verified successfully.**
