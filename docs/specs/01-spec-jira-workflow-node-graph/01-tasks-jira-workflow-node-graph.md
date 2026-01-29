# Tasks: Jira Workflow Node Graph Dashboard

**Spec:** [01-spec-jira-workflow-node-graph.md](./01-spec-jira-workflow-node-graph.md)

## Relevant Files

- `grafana/dashboards/DORADetails-JiraWorkflowNodeGraph.json` - New dashboard file to create containing the Node Graph visualization
- `grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json` - Reference dashboard for template variables, SQL patterns, and panel structure

### Notes

- Dashboard JSON files follow Grafana's export format with panels, templating, and metadata sections
- Use MySQL datasource with `editorMode: "code"` and `rawQuery: true` for all SQL queries
- Panel IDs should start at 100+ following existing conventions
- Template variables can be copied from the reference dashboard and adjusted as needed
- Node Graph panel requires two queries: one for nodes (refId: "nodes") and one for edges (refId: "edges")

## Tasks

### [x] 1.0 Create Node Graph Base Queries

Develop SQL queries that generate node and edge data in the format required by Grafana's Node Graph panel. Nodes represent Jira workflow statuses with average cycle time, edges represent transitions between statuses with percentage labels.

#### 1.0 Proof Artifact(s)

- SQL Output: Nodes query returns rows with `id`, `title`, `mainStat` columns when run against DevLake database
- SQL Output: Edges query returns rows with `source`, `target`, `mainStat` (percentage) columns
- Screenshot: Raw query results in Grafana Explore showing correctly formatted node/edge data

#### 1.0 Tasks

- [x] 1.1 Create nodes SQL query that extracts distinct workflow statuses from `issue_status_history` table, filtered by project, issue_type, and within start_status/end_status bounds, excluding statuses in exclude_status variable
- [x] 1.2 Add average cycle time calculation to nodes query using `AVG(status_time_minutes) / 60.0` as `mainStat` (hours)
- [x] 1.3 Add issue count per status to nodes query as `arc__issues` field for node sizing
- [x] 1.4 Create edges SQL query using window functions (LAG/LEAD) to identify consecutive status transitions per issue
- [x] 1.5 Calculate transition percentages for edges: count transitions from source to target divided by total transitions from source, formatted as `mainStat`
- [x] 1.6 Test both queries in Grafana Explore panel to verify output format matches Node Graph requirements (id, title, mainStat for nodes; source, target, mainStat for edges)

---

### [x] 2.0 Integrate PR Processing Time Metrics

Add PR lifecycle metrics (first commit to merge time) to each status node by joining issue status data with PR metrics via the `pull_request_issues` linking table. Display as secondary stat on nodes.

#### 2.0 Proof Artifact(s)

- SQL Output: Query returns both `mainStat` (cycle time) and `secondaryStat` (PR time) per node
- Screenshot: Query results showing status nodes with both timing metrics populated
- Screenshot: Nodes with no associated PRs showing graceful fallback (0 or null)

#### 2.0 Tasks

- [x] 2.1 Extend nodes query to JOIN with `pull_request_issues` table linking issues to PRs
- [x] 2.2 JOIN with `project_pr_metrics` table to access PR timing fields (pr_coding_time, pr_pickup_time, pr_review_time, pr_deploy_time)
- [x] 2.3 Calculate average PR total time per status as `secondaryStat`: sum of pr_coding_time + pr_pickup_time + pr_review_time converted to hours
- [x] 2.4 Use COALESCE to handle NULL values for statuses with no associated PRs, defaulting to 0
- [x] 2.5 Test query with statuses that have PRs and statuses without PRs to verify graceful handling

---

### [x] 3.0 Configure Node Graph Panel with Interactive Details

Configure the Grafana Node Graph panel with proper field mappings, color coding (relative/percentile-based), and hover tooltips showing expanded details (issue count, issue types, median time).

#### 3.0 Proof Artifact(s)

- Screenshot: Node graph visualization with colored nodes based on relative timing
- Screenshot: Hover tooltip showing expanded details for a node
- Screenshot: Edge labels showing transition percentages

#### 3.0 Tasks

- [x] 3.1 Create Node Graph panel JSON structure with `type: "nodeGraph"` and appropriate gridPos (full width, h: 16)
- [x] 3.2 Configure nodes query (refId: "nodes") with field mappings: `id` → node ID, `title` → node label, `mainStat` → primary metric display
- [x] 3.3 Configure edges query (refId: "edges") with field mappings: `source` → source node ID, `target` → target node ID, `mainStat` → edge label
- [x] 3.4 Add `secondaryStat` field mapping for PR processing time display on nodes
- [x] 3.5 Configure `arc__issues` field for node arc visualization showing issue count
- [x] 3.6 Add detail fields to nodes query for hover tooltip: `detail__issueCount`, `detail__issueTypes`, `detail__medianTime`, `detail__minTime`, `detail__maxTime`
- [x] 3.7 Configure color field using percentile calculation: compute rank of each node's cycle time and map to color scale (green < 33rd percentile, orange 33-66th, red > 66th)
- [x] 3.8 Set panel options for layout direction (left-to-right) and zoom controls

---

### [x] 4.0 Assemble Complete Dashboard

Create the complete dashboard JSON file with Node Graph panel, introduction text panel, template variables (matching existing bounded lead time dashboard), navigation links, and proper metadata.

#### 4.0 Proof Artifact(s)

- File: `grafana/dashboards/DORADetails-JiraWorkflowNodeGraph.json` exists and is valid JSON
- Screenshot: Complete dashboard in Grafana with filters applied showing workflow visualization
- Screenshot: "Go Back" navigation link visible and functional

#### 4.0 Tasks

- [x] 4.1 Create dashboard JSON skeleton with required top-level fields: annotations, editable, fiscalYearStartMonth, graphTooltip, links, liveNow, panels, refresh, schemaVersion, tags, templating, time, timepicker, timezone, title, uid, version, weekStart
- [x] 4.2 Set dashboard metadata: `uid: "jira-workflow-node-graph"`, `title: "DORA Details - Jira Workflow Node Graph"`, `tags: ["DORA"]`, `schemaVersion: 39`
- [x] 4.3 Add "Go Back" navigation link to main DORA dashboard: `url: "/d/qNo8_0M4z/dora?orgId=1"`, `keepTime: true`
- [x] 4.4 Copy template variables from DORADetails-BoundedLeadTimeforChanges.json: project, start_status, end_status, issue_type, exclude_status
- [x] 4.5 Create introduction text panel (id: 100) with markdown explaining dashboard purpose, how to use start/end status filters, and what the visualization shows
- [x] 4.6 Add the configured Node Graph panel (id: 101) with both nodes and edges queries
- [x] 4.7 Set time range defaults: `from: "now-6M"`, `to: "now"`, `timezone: "utc"`
- [x] 4.8 Validate JSON syntax and test import into Grafana
- [x] 4.9 Verify all template variables populate correctly and filters affect the visualization
