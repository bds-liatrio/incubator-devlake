# 01-tasks-bounded-cycle-time-dashboard

## Overview

This task list implements the Bounded Cycle Time Dashboard as specified in `01-spec-bounded-cycle-time-dashboard.md`. The dashboard provides configurable Jira status boundaries for measuring cycle time between any two workflow statuses.

**Reference Dashboard**: `grafana/dashboards/DORADetails-LeadTimeforChanges.json`
**Target File**: `grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json`

## Relevant Files

- `grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json` - **New file** - The main dashboard JSON to be created
- `grafana/dashboards/DORADetails-LeadTimeforChanges.json` - Reference dashboard for structure, variable patterns, and SQL query patterns
- `grafana/dashboards/DORA.json` - Parent DORA dashboard (for "Go Back" link reference)
- `grafana/dashboards/DORADetails-DeploymentFrequency.json` - Reference for bar chart panel patterns
- `backend/plugins/issue_trace/models/issue_status_history.go` - Model definition for `issue_status_history` table (columns: `issue_id`, `original_status`, `start_date`, `end_date`, `status_time_minutes`)
- `backend/core/models/domainlayer/ticket/issue.go` - Model definition for `issues` table (columns: `id`, `issue_key`, `title`, `original_type`, `original_status`, `url`)
- `backend/core/models/domainlayer/crossdomain/pull_request_issue.go` - Model definition for `pull_request_issues` junction table
- `backend/core/models/domainlayer/crossdomain/project_pr_metric.go` - Model definition for `project_pr_metrics` table (columns: `pr_coding_time`, `pr_pickup_time`, `pr_review_time`, `pr_deploy_time`)

### Notes

- Dashboard JSON follows Grafana 11.x schema version 39
- All SQL queries use MySQL dialect with Grafana macros (`$__timeFilter()`, `${variable}`)
- Panel IDs should start from 100+ to avoid conflicts with existing dashboards
- Time metrics in `project_pr_metrics` are stored in minutes; divide by 60 for hours display
- Use `COALESCE()` for nullable metric columns to prevent display issues
- Color scheme: green for overall metrics, blue for sub-metrics (PR components)

## Tasks

### [ ] 1.0 Create Dashboard Foundation with Status Variables

Establish the dashboard JSON structure with configurable Jira status dropdown variables that allow users to select start and end status boundaries.

#### 1.0 Proof Artifact(s)

- File: `grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json` exists with valid JSON structure
- Variables: JSON contains `project`, `start_status`, `end_status`, `issue_type` variable definitions with correct queries
- Import: Dashboard imports into Grafana without errors (manual verification or `cat` the JSON and validate structure)
- Text Panel: Markdown intro panel explains dashboard purpose and status bounding concept

#### 1.0 Tasks

- [ ] 1.1 Create base JSON file by copying structure from `DORADetails-LeadTimeforChanges.json` (annotations, editable, fiscalYearStartMonth, graphTooltip, liveNow, schemaVersion, time, timezone, weekStart)
- [ ] 1.2 Configure dashboard metadata: set `title` to "DORA Details - Bounded Lead Time for Changes", `uid` to "Bounded-lead-time-for-changes", `tags` to ["DORA"]
- [ ] 1.3 Add "Go Back" navigation link to main DORA dashboard (`/d/qNo8_0M4z/dora?orgId=1`) with `keepTime: true`
- [ ] 1.4 Add `project` variable (type: query, multi: true, includeAll: true) with query: `SELECT DISTINCT name FROM projects`
- [ ] 1.5 Add `start_status` variable (type: query) with query: `SELECT DISTINCT original_status FROM issue_status_history ORDER BY original_status` and default value "In Progress"
- [ ] 1.6 Add `end_status` variable (type: query) with query: `SELECT DISTINCT original_status FROM issue_status_history ORDER BY original_status` and default value "Verified"
- [ ] 1.7 Add `issue_type` variable (type: query, multi: true, includeAll: true) with query: `SELECT DISTINCT original_type FROM issues ORDER BY original_type`
- [ ] 1.8 Add markdown text panel (id: 100, gridPos: h:3, w:24, x:0, y:0) explaining dashboard purpose: "This dashboard shows bounded cycle time between configurable Jira statuses. Select start and end statuses to measure time for specific workflow phases."
- [ ] 1.9 Validate JSON syntax using `python -m json.tool` or equivalent

### [ ] 2.0 Implement Bounded Cycle Time Stat Panels

Add stat panels displaying average and median bounded cycle time calculated from selected start status to end status, with appropriate color thresholds.

#### 2.0 Proof Artifact(s)

- SQL: Query calculates time from first entry into `start_status` to first entry into `end_status` using `issue_status_history` table
- Panels: Two stat panels render in Grafana showing "Average Bounded Cycle Time" and "Median Bounded Cycle Time" in hours
- Thresholds: Color coding applies correctly (green < 24h, orange < 168h, red >= 168h)
- Filtering: Results respect `$project`, `$start_status`, `$end_status`, `$issue_type`, and `$__timeFilter()` parameters

#### 2.0 Tasks

- [ ] 2.1 Write CTE-based SQL query for bounded cycle time that:
  - Joins `issues` to `board_issues` to `project_mapping` for project filtering
  - Joins `issue_status_history` twice (for start and end status)
  - Uses `MIN()` to get first entry into each status (ignoring rework cycles)
  - Calculates `TIMESTAMPDIFF(MINUTE, start_time, end_time) / 60.0` as hours
  - Filters by `${project}`, `${start_status}`, `${end_status}`, `${issue_type}`
  - Uses `$__timeFilter()` on an appropriate date column
- [ ] 2.2 Create "Average Bounded Cycle Time" stat panel (id: 101, gridPos: h:7, w:8, x:0, y:3) with:
  - `AVG()` aggregation in SQL
  - Green fixed color, unit: "h"
  - Threshold steps: null=green, 24=orange, 168=red
  - Link to DevLake cycle time documentation
- [ ] 2.3 Create "Median Bounded Cycle Time" stat panel (id: 102, gridPos: h:7, w:8, x:8, y:3) with:
  - `PERCENT_RANK()` window function for median calculation (WHERE ranks <= 0.5)
  - Same color and threshold configuration as average panel
- [ ] 2.4 Test both queries return numeric values with sample variable values

### [ ] 3.0 Add Time Per Status Breakdown Panel

Implement bar chart showing average time spent in each status within the selected bounds to identify workflow bottlenecks.

#### 3.0 Proof Artifact(s)

- SQL: Query returns status names with corresponding time values from `issue_status_history` table
- Chart: Bar chart renders with labeled bars for each status showing hours spent
- Filtering: Only statuses between start and end bounds appear in the chart
- Ordering: Statuses appear in workflow sequence (not alphabetically)

#### 3.0 Tasks

- [ ] 3.1 Write SQL query that:
  - Selects `original_status` and `SUM(status_time_minutes) / 60.0 / COUNT(DISTINCT issue_id)` as average hours
  - Filters issues that have both start_status and end_status in their history
  - Groups by `original_status`
  - Orders by a CASE statement defining workflow sequence (e.g., "Open"=1, "In Progress"=2, "Code Review"=3, etc.) or by first occurrence time
  - Filters by project, issue_type, and time range
- [ ] 3.2 Create bar chart panel (id: 103, type: "barchart", gridPos: h:8, w:24, x:0, y:10) with:
  - X-axis: status names
  - Y-axis: average hours
  - `fillOpacity: 80`, `barWidth: 0.7`
  - Distinct colors per bar (use palette-classic mode)
  - `xTickLabelRotation: 45` for readability
- [ ] 3.3 Configure legend placement at bottom, tooltip mode single

### [ ] 4.0 Integrate PR Metrics Panels

Add stat panels displaying PR timing metrics (coding, pickup, review, deploy times) for issues within the bounded time range.

#### 4.0 Proof Artifact(s)

- SQL: Queries join `project_pr_metrics` to issues via `pull_request_issues` table
- Panels: Four stat panels render showing Average PR Coding Time, PR Pickup Time, PR Review Time, PR Deploy Time in hours
- Styling: Panels use blue color consistent with existing DORA sub-metrics pattern
- Null Handling: Issues without linked PRs display appropriately (0 or N/A)

#### 4.0 Tasks

- [ ] 4.1 Write base SQL query template that:
  - Joins `issues` → `pull_request_issues` → `project_pr_metrics`
  - Filters by project, issue_type, and time range
  - Uses `COALESCE(metric/60, 0)` to convert minutes to hours with null handling
- [ ] 4.2 Create "Average PR Coding Time" stat panel (id: 104, gridPos: h:7, w:6, x:0, y:18) with:
  - `AVG(COALESCE(pr_coding_time/60, 0))` query
  - Blue fixed color (`fixedColor: "blue"`)
  - Unit: "h"
- [ ] 4.3 Create "Average PR Pickup Time" stat panel (id: 105, gridPos: h:7, w:6, x:6, y:18) with same styling
- [ ] 4.4 Create "Average PR Review Time" stat panel (id: 106, gridPos: h:7, w:6, x:12, y:18) with same styling
- [ ] 4.5 Create "Average PR Deploy Time" stat panel (id: 107, gridPos: h:7, w:6, x:18, y:18) with same styling
- [ ] 4.6 Add links to respective DevLake documentation pages for each metric

### [ ] 5.0 Create Trend Chart for Historical Analysis

Add time series panel showing bounded cycle time trend over time with weekly/monthly aggregation.

#### 5.0 Proof Artifact(s)

- SQL: Query returns time-bucketed data using Grafana time bucketing macros
- Chart: Time series chart renders with trend line showing average bounded cycle time over selected time range
- Labels: Appropriate axis labels and legend display correctly
- Interaction: Chart respects dashboard time range picker

#### 5.0 Tasks

- [ ] 5.1 Write SQL query that:
  - Uses `DATE_FORMAT()` or similar for weekly/monthly time bucketing
  - Calculates average bounded cycle time per time bucket
  - Returns two columns: time bucket (as datetime) and average hours
  - Filters by all dashboard variables
- [ ] 5.2 Create time series panel (id: 108, type: "timeseries", gridPos: h:8, w:24, x:0, y:25) with:
  - `drawStyle: "line"`, `lineInterpolation: "linear"`
  - `fillOpacity: 10` for subtle area fill
  - Green color for the trend line
- [ ] 5.3 Configure Y-axis label as "Hours" and legend at bottom
- [ ] 5.4 Enable tooltip with `mode: "single"` for clean display

### [ ] 6.0 Build Detailed Issue Table

Implement table panel showing individual issues with all metrics for drill-down analysis, including clickable links to Jira.

#### 6.0 Proof Artifact(s)

- SQL: Query returns issue-level data with all metric columns (Issue Key, Title, Bounded Cycle Time, Dev Time, SQA Time, PR metrics)
- Table: Table renders with sortable, filterable columns
- Links: Issue Key column links to original Jira issue URL (opens in new tab)
- Colors: Cycle time column uses color-coded values based on thresholds

#### 6.0 Tasks

- [ ] 6.1 Write comprehensive SQL query returning columns:
  - `issue_key` as "Issue Key"
  - `title` as "Title"
  - Bounded cycle time (hours) as "Bounded Cycle Time"
  - Time in "In Dev" statuses as "Dev Time"
  - Time in "In SQA" statuses as "SQA Time"
  - PR metrics (coding, pickup, review, deploy) as individual columns
  - `url` as hidden column for linking
- [ ] 6.2 Create table panel (id: 109, type: "table", gridPos: h:12, w:24, x:0, y:33) with:
  - `custom.filterable: true` for column filtering
  - `cellHeight: "sm"` for compact display
  - `showHeader: true`
- [ ] 6.3 Add field override for "Issue Key" column with link to `${__data.fields["url_hidden"]}`
- [ ] 6.4 Add field override to hide the URL column (`custom.hidden: true` for "url_hidden")
- [ ] 6.5 Add field override for "Bounded Cycle Time" column with:
  - `custom.cellOptions: { type: "color-text" }`
  - Threshold colors: green < 24h, orange < 168h, red >= 168h
- [ ] 6.6 Set fixed width for "Title" column (~400px) to prevent table overflow
- [ ] 6.7 Configure default sort by "Bounded Cycle Time" descending to show slowest issues first
