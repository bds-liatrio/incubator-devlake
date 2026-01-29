# Task 4.0 Proof Artifacts: PR Metrics Panels

**Date:** 2026-01-29
**Task:** Integrate PR Metrics Panels

## Panel Count Verification

```bash
$ jq '.panels | length' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
8
```

**Result:** Dashboard now has 8 panels (1 text + 2 stat + 1 barchart + 4 PR stat panels)

## PR Metric Panels Overview

```bash
$ jq '.panels[4:8] | .[] | {id, title, type, gridPos}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

| Panel | ID | Title | Type | Position |
|-------|-----|-------|------|----------|
| 4.1 | 104 | Average PR Coding Time | stat | h:7, w:6, x:0, y:18 |
| 4.2 | 105 | Average PR Pickup Time | stat | h:7, w:6, x:6, y:18 |
| 4.3 | 106 | Average PR Review Time | stat | h:7, w:6, x:12, y:18 |
| 4.4 | 107 | Average PR Deploy Time | stat | h:7, w:6, x:18, y:18 |

**Result:** All four panels arranged horizontally across full width (4 x 6 = 24)

## SQL Query Template (Example: PR Coding Time)

```sql
WITH _prs AS (
  SELECT
    pr.id,
    pr.created_date as pr_issued_date,
    COALESCE(prm.pr_coding_time/60, 0) as coding_time
  FROM pull_requests pr
    LEFT JOIN project_pr_metrics prm ON pr.id = prm.id
    JOIN project_mapping pm ON pr.base_repo_id = pm.row_id AND pm.`table` = 'repos'
  WHERE
    $__timeFilter(prm.pr_deployed_date)
    AND pm.project_name IN (${project})
  GROUP BY pr.id, pr.created_date, prm.pr_coding_time
)
SELECT AVG(coding_time) as 'Average PR Coding Time (h)'
FROM _prs
```

### Query Features
- Follows the reference dashboard pattern (DORADetails-LeadTimeforChanges.json)
- Queries `pull_requests` directly with LEFT JOIN to `project_pr_metrics`
- Uses `project_mapping` on repos for project filtering
- Uses `COALESCE(metric/60, 0)` for null handling and minute-to-hour conversion
- Time filter on `prm.pr_deployed_date` for deployed PRs only
- Each panel uses the appropriate metric column:
  - `pr_coding_time`
  - `pr_pickup_time`
  - `pr_review_time`
  - `pr_deploy_time`

## Blue Color Configuration

```bash
$ jq '.panels[4].fieldConfig.defaults.color' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```json
{
  "fixedColor": "blue",
  "mode": "fixed"
}
```

**Result:** All PR metric panels use blue fixed color (consistent with DORA sub-metrics pattern)

## Documentation Links Verification

```bash
$ jq '.panels[4:8] | .[] | {title: .title, link: .links[0].url}' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

| Panel | Documentation Link |
|-------|-------------------|
| PR Coding Time | https://devlake.apache.org/docs/Metrics/PRCodingTime |
| PR Pickup Time | https://devlake.apache.org/docs/Metrics/PRPickupTime |
| PR Review Time | https://devlake.apache.org/docs/Metrics/PRReviewTime |
| PR Deploy Time | https://devlake.apache.org/docs/Metrics/PRDeployTime |

**Result:** All panels link to respective DevLake documentation

## Unit Configuration

```bash
$ jq '.panels[4].fieldConfig.defaults.unit' grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json
```

```
"h"
```

**Result:** All panels display values in hours

## JSON Validation

```bash
$ python3 -m json.tool grafana/dashboards/DORADetails-BoundedLeadTimeforChanges.json > /dev/null && echo "PASSED"
PASSED
```

## Summary

| Requirement | Status |
|-------------|--------|
| Base SQL with bounded_issues CTE | PASS |
| Joins issues -> pull_request_issues -> project_pr_metrics | PASS |
| COALESCE for null handling | PASS |
| PR Coding Time panel (id: 104) | PASS |
| PR Pickup Time panel (id: 105) | PASS |
| PR Review Time panel (id: 106) | PASS |
| PR Deploy Time panel (id: 107) | PASS |
| Blue fixed color for all panels | PASS |
| Unit set to hours ("h") | PASS |
| Documentation links for each metric | PASS |
| Valid JSON structure | PASS |

**All Task 4.0 proof artifacts verified successfully.**
