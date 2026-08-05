# Screen 9: Report View

**Route:** `/workspaces/:id/reports/:reportId`
**Platform:** Both — fully first-class on mobile per [[branding]] and [[features]] FEAT-041.
**Purpose:** Present the narrated, charted answer to a question, with reconciliation transparency and clickable provenance ([[features]] FEAT-015, FEAT-016, FEAT-017, FEAT-029, FEAT-030, FEAT-034).
**Entry Points:** [[08-run-viewer|Run Viewer]] (on completion), [[04-workspace-home|Workspace Home]] (recent reports list), [[16-notifications|Notifications]] (deep link), shared report link (unauthenticated, read-only — see [[user_flow]] Global Flows).
**Exit Points:** [[10-provenance-panel|Provenance Panel]] (click a claim), [[11-export-dialog|Export Dialog]], [[08-run-viewer|Run Viewer]] (ask a follow-up question, forks the run).

### Layout

**Mobile layout:**
- Navigation Shell
- Report title + timestamp header
- Reconciliation Notice (shared component) if applicable, shown near the top, above the narrative
- Narrative text body (`body-large` token), with Provenance-Linked Number styling inline on every traceable figure
- Rendered chart below the narrative, full-width, touch-scrollable/zoomable if applicable
- Bottom action bar: "Export" and "Ask a follow-up" buttons
- Accessible data table equivalent for the chart, collapsed under a "View as table" disclosure

**Web layout:**
- Navigation Shell with workspace `Sidebar`
- Same content structure as mobile but in a wider, centered reading column (max-width for narrative legibility) with the chart rendered larger, side-by-side with a persistent action panel (Export, Ask a follow-up, "View source run") rather than a bottom bar

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Narrative text | Rich text block | Both | The plain-language answer | Report `narrative` field |
| Provenance-Linked Number | Shared component | Both | Clickable, verifiable figures within the narrative | Report `provenance_links` array |
| Chart | First-party chart renderer | Both | Visual representation per the chart spec | Report `chart_spec` field |
| Accessible data table | Table (collapsed by default) | Both | Screen-reader/text equivalent of the chart | Chart spec's underlying result data |
| Reconciliation Notice | Shared component | Both | Unprompted disclosure of significant row exclusions | Report `reconciliation_summary` field |
| Ask a follow-up | Button | Both | Opens a question input scoped to this report, routes to [[08-run-viewer\|Run Viewer]] on submit (forks) | — |
| Export | Button | Both | Opens [[11-export-dialog\|Export Dialog]] | — |
| Parent report link | Link | Both | Shown if this report was forked from another, per [[features]] FEAT-025 | Report `parent_report_id` field |

### Data Requirements

**Data Needed:**
- Report object: id, title, narrative text, chart spec, provenance links array (each linking a text span to a query + specific rows), reconciliation summary (rows in/out per stage, any exclusion warnings), parent report reference if forked, created_at — from GET `/api/workspaces/:id/reports/:reportId`

**Data Flow:**
1. Screen mounts → Loading state.
2. Fetch the report object in one call (it is fully self-contained per [[architecture]]'s Report Store — no engine required to view it).
3. On success → render narrative, chart, and reconciliation notice if present.
4. On failure → Error state with retry.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Report being fetched | Skeleton for narrative + chart |
| **Loaded** | Report fetched successfully | Full narrative, chart, and any reconciliation notice |
| **Loaded (no notice)** | No filter crossed the exclusion threshold | Report shown without the Reconciliation Notice component at all — not shown-and-empty |
| **Error** | Fetch failed | Shared Error State with retry |
| **Read-Only (shared link)** | Accessed via an unauthenticated shared report link ([[features]] FEAT-046) | Same content, but "Ask a follow-up" and "Export" actions are hidden or replaced with a "Sign in to interact" prompt |

**Edge Cases:**
- A claim's underlying query/rows have since been deleted (e.g., workspace was partially modified) → the Provenance-Linked Number becomes non-interactive with a tooltip explaining the source is no longer available, rather than erroring the whole report.
- Very long narrative text on mobile → no truncation; the report reads as a normal scrollable document, since narrative is the primary reading surface per [[branding]].
- Chart spec references a chart type not yet supported by the renderer (future-proofing) → falls back to the accessible data table as the primary view with a note, rather than failing to render anything.
- Forked report (via follow-up question) → Parent report link is always shown and navigable, per [[features]] FEAT-025, so report history reads as a traceable tree, not a dead end.
