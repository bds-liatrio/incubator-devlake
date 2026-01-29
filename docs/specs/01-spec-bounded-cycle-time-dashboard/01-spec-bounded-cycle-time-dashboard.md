# 01-spec-bounded-cycle-time-dashboard

## Introduction/Overview

This specification defines a new Grafana dashboard for Apache DevLake that displays bounded lead time for changes with configurable Jira status and GitLab MR state boundaries. The dashboard enhances the existing "DORA Details - Lead Time for Changes" dashboard by allowing users to measure cycle time between any two Jira statuses (e.g., "In Progress" to "Verified") while integrating PR/MR lifecycle metrics. This addresses the need to understand actual development cycle time rather than just creation-to-resolution time.

## Goals

- **Provide configurable status bounding**: Allow users to select start and end Jira statuses via dropdown variables to calculate cycle time for any workflow phase
- **Integrate Jira and GitLab data**: Combine issue status history with PR/MR metrics (coding, pickup, review, deploy times) for comprehensive cycle analysis
- **Identify process bottlenecks**: Display time-per-status breakdown to reveal which workflow phases take the longest
- **Support trend analysis**: Show both current averages and historical trends for metrics over time
- **Maintain DORA alignment**: Follow existing DevLake dashboard patterns and naming conventions for consistency

## User Stories

- **As a Development Team Lead**, I want to see how long issues spend between "In Progress" and "Verified" statuses so that I can understand our actual development cycle time excluding backlog wait time.

- **As a Process Improvement Engineer**, I want to see time breakdown per status so that I can identify bottlenecks in our workflow and target specific phases for optimization.

- **As a Delivery Manager**, I want to filter metrics by issue type (Story, Bug, Task) so that I can compare cycle times across different work item categories.

- **As a DevOps Engineer**, I want to see PR metrics (coding, pickup, review, deploy times) alongside issue metrics so that I can correlate code review efficiency with overall delivery time.

## Demoable Units of Work

### Unit 1: Dashboard Foundation with Status Variables

**Purpose:** Establish the dashboard structure with configurable Jira status dropdown variables that allow users to select start and end status boundaries.

**Functional Requirements:**
- The system shall create a new Grafana dashboard JSON file named `DORADetails-BoundedLeadTimeforChanges.json` in `grafana/dashboards/`
- The dashboard shall include a `project` variable populated from `SELECT DISTINCT name FROM projects`
- The dashboard shall include a `start_status` variable populated from `SELECT DISTINCT original_status FROM issue_status_history ORDER BY original_status`
- The dashboard shall include an `end_status` variable populated from `SELECT DISTINCT original_status FROM issue_status_history ORDER BY original_status`
- The dashboard shall include an `issue_type` variable (multi-select) populated from `SELECT DISTINCT original_type FROM issues`
- The `start_status` variable shall default to "In Progress" if available
- The `end_status` variable shall default to "Verified" if available
- The dashboard shall include a markdown text panel explaining the dashboard purpose and status bounding concept

**Proof Artifacts:**
- Dashboard JSON: File exists at `grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json`
- Variables Query: Variables section in JSON contains all four variable definitions with correct queries
- Import Test: Dashboard can be imported into Grafana without errors

### Unit 2: Bounded Cycle Time Stat Panels

**Purpose:** Display primary bounded cycle time metrics as stat panels, showing average and median cycle time from the selected start status to end status.

**Functional Requirements:**
- The dashboard shall include a stat panel showing "Average Bounded Cycle Time" in hours
- The dashboard shall include a stat panel showing "Median Bounded Cycle Time" in hours
- The cycle time calculation shall use first entry into start status to first entry into end status (ignoring rework cycles)
- The SQL query shall filter by selected `$project`, `$start_status`, `$end_status`, and `$issue_type` variables
- The SQL query shall use Grafana's `$__timeFilter()` macro for time range filtering
- The stat panels shall display values with appropriate color thresholds (green < 24h, orange < 168h, red >= 168h)

**Proof Artifacts:**
- SQL Validation: Query returns numeric values when run against DevLake database with sample data
- Panel Display: Stat panels render with correct formatting and color coding

### Unit 3: Time Per Status Breakdown Panel

**Purpose:** Display a bar chart showing average time spent in each status within the selected bounds to identify bottlenecks.

**Functional Requirements:**
- The dashboard shall include a bar chart panel showing average time (hours) per status
- The chart shall only include statuses that fall between the selected start and end status in the workflow
- The SQL query shall calculate `SUM(status_time_minutes) / 60.0` for each status, grouped by `original_status`
- The chart shall order statuses by workflow sequence (not alphabetically)
- The chart shall use distinct colors for each status bar

**Proof Artifacts:**
- SQL Validation: Query returns status names with corresponding time values
- Chart Display: Bar chart renders with labeled bars for each status

### Unit 4: PR Metrics Integration Panels

**Purpose:** Display PR/MR timing metrics (coding, pickup, review, deploy) for issues within the bounded time range.

**Functional Requirements:**
- The dashboard shall include stat panels for: Average PR Coding Time, Average PR Pickup Time, Average PR Review Time, Average PR Deploy Time
- The PR metrics shall be linked to issues via `pull_request_issues` table
- The PR metrics shall use values from `project_pr_metrics` table (pr_coding_time, pr_pickup_time, pr_review_time, pr_deploy_time)
- The panels shall display time in hours with unit formatting
- The panels shall use consistent styling with existing DORA dashboard (blue color for sub-metrics)

**Proof Artifacts:**
- SQL Validation: Queries return PR metric values joined to issue data
- Panel Display: Four stat panels render with hour values

### Unit 5: Trend Charts for Historical Analysis

**Purpose:** Display time series charts showing how bounded cycle time and PR metrics trend over time.

**Functional Requirements:**
- The dashboard shall include a time series panel showing bounded cycle time trend (weekly or monthly aggregation)
- The time series shall plot average bounded cycle time over the selected time range
- The chart shall include appropriate axis labels and legend
- The SQL query shall group results by time period using Grafana time bucketing

**Proof Artifacts:**
- SQL Validation: Query returns time-bucketed data with metric values
- Chart Display: Time series chart renders with trend line

### Unit 6: Detailed Issue Table

**Purpose:** Display a detailed table showing individual issues with all metrics for drill-down analysis.

**Functional Requirements:**
- The dashboard shall include a table panel listing individual issues
- The table shall display columns: Issue Key, Title, Bounded Cycle Time (hours), Dev Time (hours), SQA Time (hours), PR Coding (hours), PR Pickup (hours), PR Review (hours), PR Deploy (hours)
- The Issue Key column shall link to the original Jira issue URL
- The table shall be sortable by any column
- The table shall be filterable via Grafana's built-in table filtering
- The cycle time column shall use color-coded values based on thresholds

**Proof Artifacts:**
- SQL Validation: Query returns issue-level data with all metric columns
- Table Display: Table renders with clickable links and sortable columns

## Non-Goals (Out of Scope)

1. **Backend DevLake Changes**: No modifications to DevLake plugins, models, or data collection. Dashboard uses existing tables only.
2. **Real-time Updates**: No live data streaming or automatic refresh beyond standard Grafana capabilities.
3. **Cross-Project Comparison**: Dashboard shows one project at a time; side-by-side project comparison is out of scope.
4. **Custom Status Ordering**: Status workflow order is assumed; custom drag-and-drop ordering is not included.
5. **Export/Reporting Features**: No PDF export or scheduled report generation beyond Grafana's built-in features.

## Design Considerations

- Follow existing DevLake dashboard visual patterns from `DORADetails-LeadTimeforChanges.json`
- Use consistent color scheme: green for overall metrics, blue for sub-metrics
- Panel layout should follow left-to-right flow: Overall → Breakdown → Details
- Use stat panels for single values, bar charts for comparisons, tables for details
- Include "Go Back" link to main DORA dashboard consistent with existing pattern
- Dashboard should be readable at standard 1920x1080 resolution

## Repository Standards

- **Dashboard Location**: `grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json`
- **Naming Convention**: Follow existing pattern `DORADetails-*.json`
- **JSON Formatting**: Match structure and formatting of existing dashboards
- **Datasource**: Use `mysql` as datasource name (standard DevLake configuration)
- **Variable Naming**: Use lowercase with underscores (e.g., `start_status`, `end_status`)
- **Panel IDs**: Use sequential integers starting from a high number (e.g., 100+) to avoid conflicts

## Technical Considerations

- **SQL Dialect**: Queries must work with MySQL (DevLake's default database)
- **Grafana Version**: Dashboard should be compatible with Grafana 11.x (based on existing dashboard pluginVersion)
- **Time Filtering**: Use `$__timeFilter()` macro for all time-bounded queries
- **Variable Interpolation**: Use `${variable}` syntax for multi-value variables, `${variable:raw}` where needed
- **Performance**: Queries should include appropriate indexes (issue_id, status, date columns are indexed)
- **Null Handling**: Use `COALESCE()` for nullable metric columns to prevent display issues

## Security Considerations

- No specific security considerations identified
- Dashboard uses read-only SQL queries against existing DevLake tables
- No credentials or API keys are stored in dashboard JSON
- Access control managed by Grafana's built-in authentication/authorization

## Success Metrics

1. **Functional Completeness**: All six demoable units pass their proof artifact criteria
2. **Query Performance**: All dashboard queries execute in under 5 seconds for typical data volumes
3. **User Adoption**: Dashboard can be successfully imported and used with existing DevLake deployments
4. **Data Accuracy**: Bounded cycle time calculations match manual calculations from raw data

## Open Questions

1. **Status Workflow Order**: Should status ordering in the time-per-status chart be hardcoded based on typical Jira workflows, or derived dynamically from the data? (Current assumption: Define common order in query)

2. **Default Time Range**: What should the default time range be? (Current assumption: 6 months, matching existing DORA dashboard)

3. **PR-less Issues**: How should issues without linked PRs be displayed in the PR metrics columns? (Current assumption: Show as 0 or N/A)
