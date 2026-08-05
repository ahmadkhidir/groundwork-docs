# Screen 7: Ingest Confirmation

**Route:** `/workspaces/:id/ingest/confirm`
**Platform:** Both (desktop-optimized — the highest-density review screen in the product; usable but not primarily designed for mobile)
**Purpose:** The mandatory review step where the user confirms inferred types, reviews PII flags, and accepts/rejects candidate joins before anything is queryable ([[features]] FEAT-006, FEAT-007, FEAT-008, FEAT-009). Called out in the source spec as the highest-risk UX in the product.
**Entry Points:** [[06-ingest-progress|Ingest Progress]] (automatic, on success).
**Exit Points:** [[04-workspace-home|Workspace Home]].

### Layout

**Mobile layout:**
- Navigation Shell
- `Stepper`/`Tabs` across three steps: "Types," "Sensitive Columns," "Relationships" (last step only shown if 2+ files)
- Each step shows a scrollable list of items to review, with a per-item accept/adjust control
- Bottom action bar: "Use suggested defaults" (Ghost) and "Continue" (Primary, advances step or, on the last step, completes)

**Web layout:**
- Navigation Shell
- Same three-step structure as `Tabs`, but each tab shows a `Table` (column name, inferred value, confidence/basis, action) instead of a list — denser, suited to reviewing many columns/joins at once
- Persistent "Use suggested defaults" (Ghost) and "Continue" (Primary) action bar at the bottom of the content area, not the viewport, so it stays reachable while scrolling a long table

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Type review table/list | Table (web) / List (mobile) | Both | Per-column inferred type with override control | GET `/api/workspaces/:id/ingest/:ingestId/types` |
| PII flag review | Table / List | Both | Per-column PII flag with accept/override toggle | GET `/api/workspaces/:id/ingest/:ingestId/pii-flags` |
| Candidate join review | Table / List | Both | Per-join confidence score, basis (name match, value overlap %), accept/reject/manual-adjust controls | GET `/api/workspaces/:id/ingest/:ingestId/candidate-joins` |
| Confidence badge | Badge | Both | Visual confidence indicator (high/medium/low), always paired with the numeric % or basis text, never color-only | Candidate join data |
| Use suggested defaults | Button (Ghost) | Both | Applies sane defaults to types/PII, skips explicit per-item review; never auto-applies joins (see Edge Cases) | — |
| Continue | Button (Primary) | Both | Advances to next step, or completes confirmation on the last step | — |

### Data Requirements

**Data Needed:**
- Per-column: name, inferred type, confidence, override control state — from GET `/api/workspaces/:id/ingest/:ingestId/types`
- Per-column: PII flag (boolean + detected pattern type), override state — from GET `/api/workspaces/:id/ingest/:ingestId/pii-flags`
- Per candidate join (if 2+ files): source column, target column, confidence score, basis (name similarity/value overlap/cardinality), accept/reject/manual-adjust state — from GET `/api/workspaces/:id/ingest/:ingestId/candidate-joins`

**Data Flow:**
1. Screen mounts with all three data sets fetched in parallel (join data only if 2+ files).
2. User reviews/adjusts per item, or selects "Use suggested defaults" for a given step.
3. On final "Continue," confirmed types/flags/joins are submitted (POST) and seeded into the semantic layer ([[features]] FEAT-019).
4. On success → navigate to [[04-workspace-home|Workspace Home]].

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Ingest review data being fetched | Skeleton table/list per step |
| **Default (review)** | Data loaded, user actively reviewing | Full table/list with per-item controls |
| **Defaults Applied** | User chose "Use suggested defaults" for types/PII | Those steps marked complete with defaults visibly applied (not hidden — user can still open and inspect/override) |
| **Submitting** | Final Continue pressed | Action bar shows spinner, controls disabled |
| **Error** | Review data failed to load, or submission failed | Shared Error State with retry; on submission failure, user's in-progress selections are preserved, not lost |

**Edge Cases:**
- **Candidate joins are never auto-applied**, even under "Use suggested defaults" — that shortcut only applies to type inference and PII flags. Every join requires an explicit accept/reject action, per [[features]] FEAT-009's non-negotiable requirement.
- Rejecting a candidate join removes it from this session's list; it is not silently re-suggested identically on a future ingest of the same file pair per [[user_flow]] Flow 2.
- A column flagged as PII that the user overrides to "not sensitive" is logged in the semantic layer as an explicit user decision, distinguishable later from a column that was never flagged at all.
- Single-file upload → the "Relationships" step/tab is omitted entirely rather than shown empty.
