# Screen 15: Semantic Layer / Corrections

**Route:** `/workspaces/:id/semantic-layer`
**Platform:** Both (desktop-optimized; mobile supports viewing, editing is easier on desktop)
**Purpose:** Let the user view and manage the persisted relationships, grain, metric definitions, and accumulated corrections that ground every question in the workspace ([[features]] FEAT-019, FEAT-020).
**Entry Points:** [[04-workspace-home|Workspace Home]] (Sidebar/summary panel link), [[08-run-viewer|Run Viewer]] (link after a correction is saved during a run).
**Exit Points:** [[04-workspace-home|Workspace Home]].

### Layout

**Mobile layout:**
- Navigation Shell
- `Tabs`: "Relationships," "Metric Definitions," "Corrections"
- Each tab shows a stacked `ListView` of entries (relationship: two columns + confidence; metric: name + definition; correction: what was corrected + when)

**Web layout:**
- Navigation Shell with workspace `Sidebar`
- `Tabs` as above, each rendering a `Table` for denser review
- Relationships tab: source column, target column, confidence, status (confirmed/rejected), "Edit" action
- Metric Definitions tab: metric name, SQL expression (SQL / Code Block, read-only with an "Edit" action), last modified
- Corrections tab: a chronological log — what was corrected, the original vs. corrected logic, which run it originated from (linking to [[08-run-viewer|Run Viewer]])

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Relationship row | Table row / List item | Both | Shows a confirmed/rejected join relationship | GET `/api/workspaces/:id/semantic-layer/relationships` |
| Metric definition row | Table row / List item | Both | Shows a named, defined business metric | GET `/api/workspaces/:id/semantic-layer/metrics` |
| Correction log entry | Table row / List item | Both | Shows a persisted user correction and its origin | GET `/api/workspaces/:id/semantic-layer/corrections` |
| Edit action | IconButton / Link | Both | Opens an inline or modal editor for a relationship or metric | — |

### Data Requirements

**Data Needed:**
- Relationships: source column, target column, confidence, status — from GET `/api/workspaces/:id/semantic-layer/relationships`
- Metric definitions: name, expression, description, last_modified — from GET `/api/workspaces/:id/semantic-layer/metrics`
- Corrections: description, original vs. corrected logic, originating run id, timestamp — from GET `/api/workspaces/:id/semantic-layer/corrections`

**Data Flow:**
1. Screen mounts → each tab's data loads independently (lazy-loaded on tab open, web).
2. Editing a metric definition or relationship submits a PATCH; on success the row updates in place.
3. Viewing a correction's origin navigates to the specific [[08-run-viewer|Run Viewer]] run it came from (read-only historical view of that run).

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Tab's data being fetched | Skeleton table/list |
| **Empty** | No relationships/metrics/corrections yet (new workspace, single file, no questions asked yet) | Shared Empty State per tab, e.g. "No corrections yet — they'll appear here as you refine answers" |
| **Loaded** | Data present | Full table/list |
| **Editing** | User editing a metric or relationship | Inline edit form/modal, Save/Cancel actions |
| **Error** | Fetch or update failed | Shared Error State; failed edit reverts to prior value with an inline error |

**Edge Cases:**
- Editing a metric definition that's actively referenced by an in-progress run → the edit is saved but does not retroactively alter the in-progress run's already-generated SQL; it applies from the next run onward, consistent with [[architecture]]'s "steps are pure functions of their persisted input" rule.
- A correction entry whose originating run has since been deleted (e.g., very old run pruned by retention policy) → the correction itself remains valid and listed, but the "view originating run" link is disabled with an explanatory tooltip rather than broken.
- Read-only shared member (FEAT-045) → can view all three tabs but has no Edit actions available anywhere on this screen.
