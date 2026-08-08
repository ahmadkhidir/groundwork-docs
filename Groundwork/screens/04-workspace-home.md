# Screen 4: Workspace Home

**Route:** `/workspaces/:id`
**Platform:** Both
**Purpose:** The main hub for a workspace — shows engine status, lets the user ask a question, surfaces recent reports and quick access to the semantic layer, and lists the workspace's datasets with a way to remove one ([[features]] FEAT-011, FEAT-012, FEAT-013, FEAT-034, FEAT-019, FEAT-048).
**Entry Points:** [[03-workspace-list|Workspace List]], [[07-ingest-confirmation|Ingest Confirmation]] (after first ingest), [[09-report-view|Report View]] ("back to workspace"), [[16-notifications|Notifications]] (deep link).
**Exit Points:** [[05-file-upload|File Upload]], [[08-run-viewer|Run Viewer]] (on question submit), [[09-report-view|Report View]] (recent report list), [[13-workspace-settings|Workspace Settings]], [[15-semantic-layer|Semantic Layer / Corrections]].

### Layout

**Mobile layout:**
- Navigation Shell
- Engine Status Pill (shared component) at top, always visible
- Ask panel: large `TextField` ("Ask a question about your data...") with a Submit `IconButton`
- Question suggestions strip: horizontally scrollable row of suggested-question chips below the Ask panel, shown when data exists, per [[features]] FEAT-050
- "Upload data" `Button` if no files ingested yet
- Stacked `ListView` of recent report `Card`s below the Ask panel

**Web layout:**
- Navigation Shell with workspace `Sidebar` (Home, Semantic Layer, Settings)
- Engine Status Pill top-right of the content header
- Prominent Ask panel (full-width `TextInput` + Submit `Button`) centered near the top
- Question suggestions strip: a row of suggested-question links/chips directly below the Ask panel, shown when data exists, per [[features]] FEAT-050
- Below: two-column layout — recent reports `Grid`/list (left, majority width) and a stacked right column with a compact semantic layer summary panel ("12 relationships, 4 metric definitions — view all") and a compact Datasets panel (filename, upload time, row count, per-row delete action) beneath it

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Engine Status Pill | Shared component | Both | Shows `sleeping`/`waking`/`ready` honestly at all times | GET `/api/workspaces/:id` (polled or streamed) |
| Ask input | TextInput / TextField | Both | Entry point for a new question, triggers [[08-run-viewer\|Run Viewer]] | User input |
| Question suggestions strip | Chip / link list | Both | Shows 5–10 suggested questions generated from the workspace's datasets and semantic layer; clicking fills the Ask input (editable, not auto-submitted), per [[features]] FEAT-050 | GET `/api/workspaces/:id/suggestions` |
| Recent report card | Card | Both | Title, timestamp, one-line summary, thumbnail chart | GET `/api/workspaces/:id/reports?limit=10` |
| Semantic layer summary | Panel | Web only (compact) | Quick counts + link to [[15-semantic-layer\|Semantic Layer / Corrections]] ([[features]] FEAT-019) | GET `/api/workspaces/:id/semantic-layer/summary` |
| Upload data prompt | Button | Both | Shown when no files ingested yet, routes to [[05-file-upload\|File Upload]] | GET `/api/workspaces/:id` (`has_datasets` field) |
| Datasets panel | Panel + List | Both (collapsible on mobile) | Lists this workspace's datasets (filename, `created_at`, row count, `ingest_status`) with a per-row delete `IconButton`, per [[features]] FEAT-048; two datasets sharing a filename (FEAT-049 "Keep both") are distinguished by `created_at` here too | GET `/api/workspaces/:id/datasets` |

### Data Requirements

**Data Needed:**
- Workspace metadata: name, engine status, `has_datasets` (boolean, true once at least one `Dataset` exists for this workspace, per [[schema]]) — from GET `/api/workspaces/:id`
- Recent reports (id, title, created_at, summary snippet, thumbnail chart spec) — from GET `/api/workspaces/:id/reports?limit=10`
- Semantic layer counts (relationship count, metric definition count) — from GET `/api/workspaces/:id/semantic-layer/summary`
- Dataset list (id, `original_filename`, `created_at`, `row_count`, `ingest_status`) — from GET `/api/workspaces/:id/datasets`, backing the Datasets panel per [[features]] FEAT-048
- Suggested questions (text) — from GET `/api/workspaces/:id/suggestions`, per [[features]] FEAT-050

**Data Flow:**
1. Screen mounts → Loading state for reports list; engine status shown immediately from cached/last-known value while a fresh fetch confirms it.
2. Question suggestions are fetched in parallel with workspace metadata; the strip shows a skeleton while loading and, when the workspace has data, renders 5–10 suggestion chips below the Ask input (hidden in the Empty no-data state). Generation is served from Postgres alone per [[features]] FEAT-050 — it never wakes a sleeping engine.
3. Focusing the Ask input triggers wake-on-intent (FEAT-012) if engine status is `sleeping` — status pill updates to `waking` without blocking the input.
4. Submitting a question while `waking` queues the question and navigates to [[08-run-viewer|Run Viewer]] immediately; the run shows `waking` as its first visible step rather than blocking here. Clicking a suggestion fills the Ask input with its exact text (never auto-submits); the user can edit before submitting, and submission follows this same path.
5. Recent reports and the dataset list are fetched in parallel with workspace metadata.
6. The "Upload data" button's visibility is driven directly by `has_datasets` from the same workspace-metadata fetch (step 1) — shown while `false`, hidden once `true` — rather than rendering unconditionally.
7. Deleting a dataset from the Datasets panel opens the shared Delete Confirmation Modal, stating what will be removed (dataset, its columns, any dependent relationships/joins, both storage objects) and, if any column on the dataset carries a type override or a reviewed PII decision, that this review work will be permanently lost and is not carried forward to any future re-upload, per [[features]] FEAT-048; on confirm, the row is removed from the list and, if it was the workspace's only remaining dataset, `has_datasets` reverts to driving the Empty (no data) state.
8. Returning to Workspace Home from [[07-ingest-confirmation|Ingest Confirmation]] after a new upload, or after a dataset deletion (step 7), re-fetches suggestions so they reflect the workspace's current datasets and semantic layer.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Initial mount | Skeleton for reports list; engine status pill shows last-known state or a neutral skeleton if truly first load |
| **Suggestions loading** | Suggestions fetch in flight | Skeleton chips below the Ask input; the Ask input itself is unaffected |
| **Empty (no data)** | No files ever ingested | Shared Empty State: "Upload your first file to get started" + Upload CTA, Ask input disabled with a hint |
| **Empty (no reports yet)** | Data ingested but no questions asked yet | Ask panel prominent, reports section shows "Your reports will appear here" |
| **Suggestions empty** | Metadata present but fewer than 5 safe suggestions possible | Suggestions strip hidden (never padded with duplicates or unsafe columns); the Ask input remains the entry point |
| **Loaded** | Reports present | Full layout with recent reports |
| **Engine sleeping** | No active engine process | Status pill shows `sleeping`; workspace remains fully browsable (reports, semantic layer) per [[user_flow]] Flow 4 |
| **Engine waking** | Wake triggered | Status pill shows `waking`; Ask input remains usable, submission queues |
| **Engine ready** | Engine responsive | Status pill shows `ready` |
| **Error** | Workspace fetch failed | Shared Error State with retry |
| **Suggestions error** | Suggestions fetch failed (including offline) | Shared Error State with retry in the strip; the Ask input remains fully usable |

**Edge Cases:**
- User submits a question, then immediately navigates away before the run starts — the run still proceeds server-side per [[architecture]] ("runs are server-side state; the UI is a viewer attached to them"), and a durable notification ([[16-notifications|Notifications]]) surfaces completion if the user isn't watching live.
- Very long recent-report titles → truncate with ellipsis, full title on hover/tap.
- Read-only shared access ([[features]] FEAT-045): Ask input and Upload CTA are hidden or disabled with an explanatory tooltip ("You have read-only access to this workspace"); recent reports, semantic layer, and the Datasets panel remain viewable, but the per-row delete action is hidden or disabled for read-only members. The question suggestions strip is hidden alongside the Ask input, since a read-only member cannot submit a suggestion.
- Suggestions are visible on a `sleeping` engine — generation reads Postgres only per [[features]] FEAT-050, so the strip never causes or waits on a wake.
- Deleting a dataset that a confirmed `Relationship` or `MetricDefinition` depends on ([[features]] FEAT-048) → the Delete Confirmation Modal states this dependency explicitly before the user commits, rather than silently orphaning a semantic-layer entry.
- Deleting a dataset whose columns carry a type override ([[features]] FEAT-006) or a reviewed PII decision ([[features]] FEAT-007) → the Delete Confirmation Modal states that this review work will be permanently lost, distinct from (and in addition to) any dependent `Relationship`/`MetricDefinition` warning above; Phase 1 has no mechanism to carry it forward onto a future re-upload.
- Deleting the dataset currently shown as `ingest_status: failed` (the retry-cleanup case) → same delete action and confirmation, since a failed dataset still has rows to remove even if it has no usable data.
