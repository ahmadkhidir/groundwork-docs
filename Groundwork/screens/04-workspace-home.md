# Screen 4: Workspace Home

**Route:** `/workspaces/:id`
**Platform:** Both
**Purpose:** The main hub for a workspace — shows engine status, lets the user ask a question, surfaces recent reports and quick access to the semantic layer, and lists the workspace's datasets with a way to remove one ([[features]] FEAT-011, FEAT-012, FEAT-013, FEAT-034, FEAT-019, FEAT-048, FEAT-050, FEAT-051).
**Entry Points:** [[03-workspace-list|Workspace List]], [[07-ingest-confirmation|Ingest Confirmation]] (after first ingest), [[09-report-view|Report View]] ("back to workspace"), [[16-notifications|Notifications]] (deep link).
**Exit Points:** [[05-file-upload|File Upload]], [[08-run-viewer|Run Viewer]] (on question submit), [[09-report-view|Report View]] (recent report list), [[13-workspace-settings|Workspace Settings]], [[15-semantic-layer|Semantic Layer / Corrections]].

### Layout

**Mobile layout:**
- Navigation Shell
- Engine Status Pill (shared component) at top, always visible
- Ask panel: large `TextField` ("Ask a question about your data...") with a Submit `IconButton`
- Question suggestions strip: horizontally scrollable container holding a **multiline** (`flex-wrap`) chip layout with TWO groups side by side, shown when data exists — the deterministic "Suggested questions" group per [[features]] FEAT-050 and the "Smart questions" group from the workspace's persisted bucket per [[features]] FEAT-051; the accumulated bucket wraps onto several rows rather than one endless single-line scroll
- Smart generation affordance: a secondary "Generate smarter questions" action beside the strip, per [[features]] FEAT-051 — active on Pro/Team (appends to the bucket), locked Upgrade Prompt linking to `/settings` on Free (viewing the bucket is free on every tier)
- "Upload data" `Button` if no files ingested yet
- Stacked `ListView` of recent report `Card`s below the Ask panel

**Web layout:**
- Navigation Shell with workspace `Sidebar` (Home, Semantic Layer, Settings)
- Engine Status Pill top-right of the content header
- Prominent Ask panel (full-width `TextInput` + Submit `Button`) centered near the top
- Question suggestions strip: horizontally scrollable container with a **multiline** (`flex-wrap`) chip layout holding two groups directly below the Ask panel, shown when data exists — the deterministic "Suggested questions" group per [[features]] FEAT-050 and the "Smart questions" group from the workspace's persisted bucket per [[features]] FEAT-051
- Smart generation affordance: a secondary "Generate smarter questions" button beside the strip, per [[features]] FEAT-051 — active on Pro/Team (appends to the bucket), locked Upgrade Prompt linking to `/settings` on Free (viewing the bucket is free on every tier)
- Below: two-column layout — recent reports `Grid`/list (left, majority width) and a stacked right column with a compact semantic layer summary panel ("12 relationships, 4 metric definitions — view all") and a compact Datasets panel (filename, upload time, row count, per-row delete action) beneath it

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Engine Status Pill | Shared component | Both | Shows `sleeping`/`waking`/`ready` honestly at all times | GET `/api/workspaces/:id` (polled or streamed) |
| Ask input | TextInput / TextField | Both | Entry point for a new question, triggers [[08-run-viewer\|Run Viewer]] | User input |
| Question suggestions strip | Chip / link list | Both | Shows TWO groups side by side in a horizontally scrollable, multiline (`flex-wrap`) container: the deterministic "Suggested questions" chips (5–10, generated from the workspace's datasets and semantic layer; clicking fills the Ask input, editable, not auto-submitted), per [[features]] FEAT-050, and the "Smart questions" group from the workspace's persisted bucket, per [[features]] FEAT-051 | GET `/api/workspaces/:id/suggestions` |
| Smart suggestion chip | Chip + delete control | Both | A persisted smart question from the bucket. Clicking fills the Ask input (editable, not auto-submitted); each chip carries its own delete (×) control that removes that one bucket row via `DELETE /suggestions/:id`. A failed delete keeps the chip with an honest inline error; a 404 delete is treated as already-removed | GET `/api/workspaces/:id/suggestions` → `smart` array |
| Smart generation affordance | Button (secondary) | Both | On Pro/Team: triggers one on-demand model-assisted generation (wake → profile → generate) that APPENDS the model's questions to the persisted bucket (deduplicated, never replacing the strip); on Free: locked Upgrade Prompt ("Upgrade to Pro/Team", linking to `/settings`) per [[screens]] shared-component copy rule, never silently absent, waking no engine when clicked while locked, per [[features]] FEAT-051 | POST `/api/workspaces/:id/suggestions/smart` |
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
- Suggested questions (text) — from GET `/api/workspaces/:id/suggestions`, per [[features]] FEAT-050; the deterministic strip is computed at read time from persisted metadata and is not stored
- Smart-generated questions (id + question text) — from the `smart` array of the same GET `/api/workspaces/:id/suggestions` response (the persisted per-workspace bucket), per [[features]] FEAT-051; viewing the bucket is free on every tier, including Free
- New smart questions — from POST `/api/workspaces/:id/suggestions/smart`, per [[features]] FEAT-051; request is gated server-side to Pro/Team, triggers the on-demand generation, and the response appends the model's questions to the bucket, returning the full updated `{ suggestions, smart, model_generated: true }`
- Delete a smart question — `DELETE /api/workspaces/:id/suggestions/:id`, per [[features]] FEAT-051; owner/writable members only (read-only shared members get 403 `read_only_member`), a missing or cross-workspace id returns 404 `not_found`, and there is no tier gate on deletion

**Data Flow:**
1. Screen mounts → Loading state for reports list; engine status shown immediately from cached/last-known value while a fresh fetch confirms it.
2. Question suggestions are fetched in parallel with workspace metadata; the strip shows a skeleton while loading and, when the workspace has data, renders the deterministic 5–10 suggestion chips below the Ask input (hidden in the Empty no-data state), with the persisted "Smart questions" bucket group beside them on every tier. Generation is served from Postgres alone per [[features]] FEAT-050 — it never wakes a sleeping engine.
3. Focusing the Ask input triggers wake-on-intent (FEAT-012) if engine status is `sleeping` — status pill updates to `waking` without blocking the input.
4. Submitting a question while `waking` queues the question and navigates to [[08-run-viewer|Run Viewer]] immediately; the run shows `waking` as its first visible step rather than blocking here. Clicking a suggestion fills the Ask input with its exact text (never auto-submits); the user can edit before submitting, and submission follows this same path.
5. Recent reports and the dataset list are fetched in parallel with workspace metadata.
6. The "Upload data" button's visibility is driven directly by `has_datasets` from the same workspace-metadata fetch (step 1) — shown while `false`, hidden once `true` — rather than rendering unconditionally.
7. Deleting a dataset from the Datasets panel opens the shared Delete Confirmation Modal, stating what will be removed (dataset, its columns, any dependent relationships/joins, both storage objects) and, if any column on the dataset carries a type override or a reviewed PII decision, that this review work will be permanently lost and is not carried forward to any future re-upload, per [[features]] FEAT-048; on confirm, the row is removed from the list and, if it was the workspace's only remaining dataset, `has_datasets` reverts to driving the Empty (no data) state.
8. Returning to Workspace Home from [[07-ingest-confirmation|Ingest Confirmation]] after a new upload, or after a dataset deletion (step 7), re-fetches suggestions so they reflect the workspace's current datasets and semantic layer.
9. Smart generation ([[features]] FEAT-051, Pro/Team only): clicking the smart affordance POSTs `/api/workspaces/:id/suggestions/smart`. Server-side, a sleeping engine wakes (status transitions visible per FEAT-011), cheap read-only profiling queries run against safe columns, the model is called once through the shared payload-construction function, and the call is written to the audit/egress log as a `model_call` entry. The client shows the honest `waking` → `profiling` → `generating` sequence and disables the affordance until it resolves; on success the returned full updated `{ suggestions, smart, model_generated: true }` is rendered — the generated questions are appended to the "Smart questions" bucket group beside the unchanged deterministic strip (deduplicated, never a replacement); on failure the shared Error State with retry is shown in the strip and the deterministic set, the existing bucket, and the Ask input remain usable. The Free-tier locked affordance makes no request; viewing the bucket's existing rows is free on every tier.

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
| **Smart generating** | Smart generation in flight (Pro/Team) | Smart affordance disabled showing `waking` → `profiling` → `generating`; the deterministic strip, the existing bucket group, and the Ask input remain fully usable |
| **Smart bucket shown** | Workspace has persisted smart questions (any tier) | "Smart questions" group renders beside the deterministic "Suggested questions" group — viewing the bucket is free on every tier, including Free |
| **Smart generation error** | Wake, profiling, or model call failed | Shared Error State with retry in the strip; the deterministic set, the existing bucket, and the Ask input remain usable |
| **Smart delete error** | Deleting a smart question failed | The chip stays in the bucket group with an honest inline error; a 404 response is treated as already-removed (chip leaves the group, no error) |
| **Smart locked (Free)** | Free-tier account | Smart affordance renders as the locked Upgrade Prompt ("Upgrade to Pro/Team" linking to `/settings`); clicking it makes no request and wakes no engine — but the bucket group's existing rows remain viewable |

**Edge Cases:**
- User submits a question, then immediately navigates away before the run starts — the run still proceeds server-side per [[architecture]] ("runs are server-side state; the UI is a viewer attached to them"), and a durable notification ([[16-notifications|Notifications]]) surfaces completion if the user isn't watching live.
- Very long recent-report titles → truncate with ellipsis, full title on hover/tap.
- Read-only shared access ([[features]] FEAT-045): Ask input and Upload CTA are hidden or disabled with an explanatory tooltip ("You have read-only access to this workspace"); recent reports, semantic layer, and the Datasets panel remain viewable, but the per-row delete action is hidden or disabled for read-only members. The question suggestions strip is hidden alongside the Ask input, since a read-only member cannot submit a suggestion.
- Suggestions are visible on a `sleeping` engine — generation reads Postgres only per [[features]] FEAT-050, so the strip never causes or waits on a wake.
- Smart generation wakes a sleeping engine — unlike the deterministic strip, the on-demand path requires engine profiling, so its loading state shows the honest `waking` → `profiling` → `generating` sequence per [[features]] FEAT-051/FEAT-011; the deterministic strip continues to render instantly on the same sleeping workspace.
- Read-only shared access ([[features]] FEAT-045): the smart affordance is hidden alongside the Ask input and strip, since a read-only member can neither submit a suggestion nor trigger generation.
- Double-click on the smart affordance → disabled during generation (no double-generation); each click after completion starts a fresh, metered, egress-logged generation that appends to the bucket — never a replacement, and never a duplicate of a question already on the list (deduplicated against the deterministic set AND the existing bucket).
- Deleting a smart question fails (network/5xx) → the chip stays in the bucket group with an honest inline error and the user can retry; a 404 response (already removed elsewhere, e.g., another member) is treated as already-removed — the chip leaves the group without erroring. Read-only shared members get no delete control (server rejects with 403 `read_only_member` regardless).
- Deleting a dataset that a confirmed `Relationship` or `MetricDefinition` depends on ([[features]] FEAT-048) → the Delete Confirmation Modal states this dependency explicitly before the user commits, rather than silently orphaning a semantic-layer entry.
- Deleting a dataset whose columns carry a type override ([[features]] FEAT-006) or a reviewed PII decision ([[features]] FEAT-007) → the Delete Confirmation Modal states that this review work will be permanently lost, distinct from (and in addition to) any dependent `Relationship`/`MetricDefinition` warning above; Phase 1 has no mechanism to carry it forward onto a future re-upload.
- Deleting the dataset currently shown as `ingest_status: failed` (the retry-cleanup case) → same delete action and confirmation, since a failed dataset still has rows to remove even if it has no usable data.
