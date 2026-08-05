# Screen 4: Workspace Home

**Route:** `/workspaces/:id`
**Platform:** Both
**Purpose:** The main hub for a workspace — shows engine status, lets the user ask a question, and surfaces recent reports and quick access to the semantic layer ([[features]] FEAT-011, FEAT-012, FEAT-013, FEAT-034).
**Entry Points:** [[03-workspace-list|Workspace List]], [[07-ingest-confirmation|Ingest Confirmation]] (after first ingest), [[09-report-view|Report View]] ("back to workspace"), [[16-notifications|Notifications]] (deep link).
**Exit Points:** [[05-file-upload|File Upload]], [[08-run-viewer|Run Viewer]] (on question submit), [[09-report-view|Report View]] (recent report list), [[13-workspace-settings|Workspace Settings]], [[15-semantic-layer|Semantic Layer / Corrections]].

### Layout

**Mobile layout:**
- Navigation Shell
- Engine Status Pill (shared component) at top, always visible
- Ask panel: large `TextField` ("Ask a question about your data...") with a Submit `IconButton`
- "Upload data" `Button` if no files ingested yet
- Stacked `ListView` of recent report `Card`s below the Ask panel

**Web layout:**
- Navigation Shell with workspace `Sidebar` (Home, Semantic Layer, Settings)
- Engine Status Pill top-right of the content header
- Prominent Ask panel (full-width `TextInput` + Submit `Button`) centered near the top
- Below: two-column layout — recent reports `Grid`/list (left, majority width) and a compact semantic layer summary panel (right, "12 relationships, 4 metric definitions — view all")

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Engine Status Pill | Shared component | Both | Shows `sleeping`/`waking`/`ready` honestly at all times | GET `/api/workspaces/:id` (polled or streamed) |
| Ask input | TextInput / TextField | Both | Entry point for a new question, triggers [[08-run-viewer\|Run Viewer]] | User input |
| Recent report card | Card | Both | Title, timestamp, one-line summary, thumbnail chart | GET `/api/workspaces/:id/reports?limit=10` |
| Semantic layer summary | Panel | Web only (compact) | Quick counts + link to [[15-semantic-layer\|Semantic Layer / Corrections]] | GET `/api/workspaces/:id/semantic-layer/summary` |
| Upload data prompt | Button | Both | Shown when no files ingested yet, routes to [[05-file-upload\|File Upload]] | — |

### Data Requirements

**Data Needed:**
- Workspace metadata: name, engine status — from GET `/api/workspaces/:id`
- Recent reports (id, title, created_at, summary snippet, thumbnail chart spec) — from GET `/api/workspaces/:id/reports?limit=10`
- Semantic layer counts (relationship count, metric definition count) — from GET `/api/workspaces/:id/semantic-layer/summary`

**Data Flow:**
1. Screen mounts → Loading state for reports list; engine status shown immediately from cached/last-known value while a fresh fetch confirms it.
2. Focusing the Ask input triggers wake-on-intent (FEAT-012) if engine status is `sleeping` — status pill updates to `waking` without blocking the input.
3. Submitting a question while `waking` queues the question and navigates to [[08-run-viewer|Run Viewer]] immediately; the run shows `waking` as its first visible step rather than blocking here.
4. Recent reports fetched in parallel with workspace metadata.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Initial mount | Skeleton for reports list; engine status pill shows last-known state or a neutral skeleton if truly first load |
| **Empty (no data)** | No files ever ingested | Shared Empty State: "Upload your first file to get started" + Upload CTA, Ask input disabled with a hint |
| **Empty (no reports yet)** | Data ingested but no questions asked yet | Ask panel prominent, reports section shows "Your reports will appear here" |
| **Loaded** | Reports present | Full layout with recent reports |
| **Engine sleeping** | No active engine process | Status pill shows `sleeping`; workspace remains fully browsable (reports, semantic layer) per [[user_flow]] Flow 4 |
| **Engine waking** | Wake triggered | Status pill shows `waking`; Ask input remains usable, submission queues |
| **Engine ready** | Engine responsive | Status pill shows `ready` |
| **Error** | Workspace fetch failed | Shared Error State with retry |

**Edge Cases:**
- User submits a question, then immediately navigates away before the run starts — the run still proceeds server-side per [[architecture]] ("runs are server-side state; the UI is a viewer attached to them"), and a durable notification ([[16-notifications|Notifications]]) surfaces completion if the user isn't watching live.
- Very long recent-report titles → truncate with ellipsis, full title on hover/tap.
- Read-only shared access ([[features]] FEAT-045): Ask input and Upload CTA are hidden or disabled with an explanatory tooltip ("You have read-only access to this workspace"); recent reports and semantic layer remain viewable.
