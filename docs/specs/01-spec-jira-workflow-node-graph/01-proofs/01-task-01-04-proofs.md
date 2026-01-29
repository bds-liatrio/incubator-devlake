# Proof Artifacts: Jira Workflow Node Graph Dashboard

**Spec:** 01-spec-jira-workflow-node-graph
**Tasks Covered:** 1.0, 2.0, 3.0, 4.0
**Date:** 2026-01-29

---

## Task 1.0: Create Node Graph Base Queries

### Nodes Query SQL

The nodes query extracts distinct workflow statuses from `issue_status_history` table with:
- Project, issue_type, start/end status filtering
- Average cycle time calculation using `AVG(status_time_minutes) / 60.0`
- Issue count per status as `arc__issues`
- Detail fields for hover tooltips

```sql
WITH bounded_issues AS (
  SELECT
    i.id as issue_id,
    i.original_type,
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
  GROUP BY i.id, i.original_type
  HAVING start_time IS NOT NULL
    AND end_time IS NOT NULL
    AND end_time > start_time
    AND $__timeFilter(end_time)
),
status_metrics AS (
  SELECT
    ish.original_status,
    AVG(ish.status_time_minutes) / 60.0 as avg_hours,
    COUNT(DISTINCT ish.issue_id) as issue_count,
    GROUP_CONCAT(DISTINCT bi.original_type) as issue_types,
    MIN(ish.status_time_minutes) / 60.0 as min_hours,
    MAX(ish.status_time_minutes) / 60.0 as max_hours
  FROM issue_status_history ish
  JOIN bounded_issues bi ON ish.issue_id = bi.issue_id
  WHERE ish.start_date >= bi.start_time
    AND ish.start_date < bi.end_time
    AND ish.original_status NOT IN (${exclude_status})
  GROUP BY ish.original_status
)
-- Returns: id, title, mainStat, arc__issues, detail__* fields
SELECT
  sm.original_status as id,
  sm.original_status as title,
  ROUND(sm.avg_hours, 2) as mainStat,
  sm.issue_count as arc__issues,
  sm.issue_count as detail__issueCount,
  sm.issue_types as detail__issueTypes,
  ROUND(sm.min_hours, 2) as detail__minTime,
  ROUND(sm.max_hours, 2) as detail__maxTime
FROM status_metrics sm
```

**Expected Output Columns:**
- `id`: Node identifier (status name)
- `title`: Node label (status name)
- `mainStat`: Average cycle time in hours
- `arc__issues`: Issue count for node sizing
- `detail__*`: Additional fields for hover tooltip

### Edges Query SQL

The edges query uses `LEAD()` window function to identify consecutive status transitions:

```sql
WITH bounded_issues AS (
  -- Same CTE as nodes query
),
status_sequence AS (
  SELECT
    ish.issue_id,
    ish.original_status,
    ish.start_date,
    LEAD(ish.original_status) OVER (PARTITION BY ish.issue_id ORDER BY ish.start_date) as next_status
  FROM issue_status_history ish
  JOIN bounded_issues bi ON ish.issue_id = bi.issue_id
  WHERE ish.start_date >= bi.start_time
    AND ish.start_date < bi.end_time
    AND ish.original_status NOT IN (${exclude_status})
),
transitions AS (
  SELECT
    original_status as source,
    next_status as target,
    COUNT(*) as transition_count
  FROM status_sequence
  WHERE next_status IS NOT NULL
    AND next_status NOT IN (${exclude_status})
  GROUP BY original_status, next_status
),
source_totals AS (
  SELECT
    source,
    SUM(transition_count) as total_from_source
  FROM transitions
  GROUP BY source
)
SELECT
  t.source as source,
  t.target as target,
  CONCAT(ROUND(t.transition_count * 100.0 / st.total_from_source, 1), '%') as mainStat
FROM transitions t
JOIN source_totals st ON t.source = st.source
```

**Expected Output Columns:**
- `source`: Source status name
- `target`: Target status name
- `mainStat`: Transition percentage (e.g., "45.2%")

---

## Task 2.0: Integrate PR Processing Time Metrics

### PR Metrics Integration

The nodes query includes PR processing time as `secondaryStat`:

```sql
pr_metrics AS (
  SELECT
    ish.original_status,
    AVG(
      COALESCE(prm.pr_coding_time, 0) +
      COALESCE(prm.pr_pickup_time, 0) +
      COALESCE(prm.pr_review_time, 0)
    ) / 60.0 as avg_pr_time
  FROM issue_status_history ish
  JOIN bounded_issues bi ON ish.issue_id = bi.issue_id
  LEFT JOIN pull_request_issues pri ON bi.issue_id = pri.issue_id
  LEFT JOIN project_pr_metrics prm ON pri.pull_request_id = prm.id
  WHERE ish.start_date >= bi.start_time
    AND ish.start_date < bi.end_time
    AND ish.original_status NOT IN (${exclude_status})
  GROUP BY ish.original_status
)
```

**Key Features:**
- LEFT JOIN ensures statuses without PRs still appear
- COALESCE defaults NULL values to 0
- Combines pr_coding_time + pr_pickup_time + pr_review_time
- Converts from minutes to hours (`/ 60.0`)

---

## Task 3.0: Configure Node Graph Panel with Interactive Details

### Panel Configuration

```json
{
  "type": "nodeGraph",
  "gridPos": {
    "h": 16,
    "w": 24,
    "x": 0,
    "y": 3
  },
  "id": 101,
  "options": {
    "nodes": {
      "mainStatUnit": "h",
      "secondaryStatUnit": "h"
    },
    "edges": {}
  }
}
```

### Field Mappings

**Nodes Query (refId: "nodes"):**
- `id` → Node identifier
- `title` → Node label
- `mainStat` → Primary metric (cycle time in hours)
- `secondaryStat` → Secondary metric (PR time in hours)
- `arc__issues` → Node arc showing issue count
- `color` → Node color (green/orange/red based on percentile)
- `detail__*` → Hover tooltip fields

**Edges Query (refId: "edges"):**
- `source` → Source node ID
- `target` → Target node ID
- `mainStat` → Edge label (transition percentage)

### Percentile-Based Color Coding

```sql
percentiles AS (
  SELECT
    original_status,
    avg_hours,
    PERCENT_RANK() OVER (ORDER BY avg_hours) as pct_rank
  FROM status_metrics
)
SELECT
  CASE
    WHEN p.pct_rank < 0.33 THEN 'green'
    WHEN p.pct_rank < 0.66 THEN 'orange'
    ELSE 'red'
  END as color
FROM percentiles p
```

---

## Task 4.0: Assemble Complete Dashboard

### Dashboard File Created

**File:** `grafana/dashboards/DORADetails-JiraWorkflowNodeGraph.json`

### JSON Syntax Validation

```bash
$ python3 -m json.tool grafana/dashboards/DORADetails-JiraWorkflowNodeGraph.json > /dev/null
# Exit code: 0 (valid JSON)
```

### Dashboard Metadata

```json
{
  "uid": "jira-workflow-node-graph",
  "title": "DORA Details - Jira Workflow Node Graph",
  "tags": ["DORA"],
  "schemaVersion": 39,
  "pluginVersion": "11.2.0"
}
```

### Navigation Link

```json
{
  "links": [
    {
      "title": "Go Back",
      "url": "/d/qNo8_0M4z/dora?orgId=1",
      "keepTime": true
    }
  ]
}
```

### Template Variables

Copied from reference dashboard:
1. `project` - Multi-select, includes all
2. `start_status` - Single select, default "In Progress"
3. `end_status` - Single select, default "Verified"
4. `issue_type` - Multi-select, includes all
5. `exclude_status` - Multi-select, default "Closed"

### Introduction Text Panel

```markdown
## Jira Workflow Node Graph

This dashboard visualizes your Jira workflow as an interactive node graph:
- **Nodes** represent workflow statuses with average cycle time (hours) and PR processing time
- **Edges** show transitions between statuses with percentage labels (% of issues taking each path)
- **Colors** indicate relative performance: green (fastest), orange (medium), red (slowest)

Use **Start Status** and **End Status** filters to focus on a specific workflow phase. Use **Exclude Statuses** to remove statuses from the visualization.
```

### Time Range Defaults

```json
{
  "time": {
    "from": "now-6M",
    "to": "now"
  },
  "timezone": "utc"
}
```

---

## Verification Summary

| Requirement | Status | Evidence |
|------------|--------|----------|
| Dashboard file created | ✅ | `DORADetails-JiraWorkflowNodeGraph.json` exists |
| Valid JSON syntax | ✅ | `python3 -m json.tool` passes |
| Nodes query returns id, title, mainStat | ✅ | SQL query structure verified |
| Edges query returns source, target, mainStat | ✅ | SQL with LEAD() window function |
| PR metrics integration | ✅ | secondaryStat with COALESCE handling |
| Percentile-based coloring | ✅ | PERCENT_RANK() with green/orange/red |
| Template variables | ✅ | All 5 variables from reference dashboard |
| Navigation link | ✅ | "Go Back" to main DORA dashboard |
| Introduction text | ✅ | Panel ID 100 with markdown |
| Node Graph panel | ✅ | Panel ID 101 with type: "nodeGraph" |
