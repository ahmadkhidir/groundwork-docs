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
- Same three-step structure as `Tabs`, but each tab shows a `Table` (column name, inferred/active value, fit badge or join basis, action) instead of a list — denser, suited to reviewing many columns/joins at once. On the Types tab, the fit badge sits directly beside the override dropdown as a bare "N%" with no surrounding sentence, which is exactly why it must always match the dropdown's current value (see Edge Cases).
- Persistent "Use suggested defaults" (Ghost) and "Continue" (Primary) action bar at the bottom of the content area, not the viewport, so it stays reachable while scrolling a long table

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Type review table/list | Table (web) / List (mobile) | Both | Per-column inferred type with override control | GET `/api/workspaces/:id/ingest/:ingestId/types` |
| PII flag review | Table / List | Both | Per-column PII flag with accept/override toggle | GET `/api/workspaces/:id/ingest/:ingestId/pii-flags` |
| Candidate join review | Table / List | Both | Per-join confidence score, basis (name match, value overlap %), accept/reject/manual-adjust controls | GET `/api/workspaces/:id/ingest/:ingestId/candidate-joins` |
| Type fit badge | Badge | Both | Per-column HIGH/MEDIUM/LOW indicator on the Types step, always paired with the numeric "N% of values fit this type" text, never color-only. Always describes the column's currently active type — the override if the user has set one on that row, otherwise the inferred type — never the original inference once overridden. Updates in the same interaction when the override dropdown changes, via [[schema]] `Column.type_fit_rates` (no loading state, no fresh query — see [[features]] FEAT-005/FEAT-006) | Type review table/list data |
| Join confidence badge | Badge | Both | Visual confidence indicator (high/medium/low) on the Relationships step, always paired with the numeric % or basis text, never color-only | Candidate join data |
| Use suggested defaults | Button (Ghost) | Both | Applies sane defaults to types/PII, skips explicit per-item review; never auto-applies joins (see Edge Cases) | — |
| Continue | Button (Primary) | Both | Advances to next step, or completes confirmation on the last step | — |

### Data Requirements

**Data Needed:**
- Per-column: name, inferred type, `type_fit_rates` (fit rate for every candidate type, per [[schema]] `Column`), current override (if any), override control state — from GET `/api/workspaces/:id/ingest/:ingestId/types`. The badge rendered for a column is `type_fit_rates[override ?? inferred_type]`, banded HIGH/MEDIUM/LOW per [[features]] FEAT-005; changing the override in the table/list re-reads `type_fit_rates` client-side (data already fetched, no re-request) so the badge updates instantly.
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
- **Overriding a column's type changes the badge shown next to it, in the same interaction.** The Type fit badge never describes a type other than the one currently selected in that row's override dropdown (or the inferred type, if no override is set). On the web table view especially — where the badge sits as a bare "N%" beside the dropdown with no surrounding sentence — this matters: a badge left showing the old inference's fit rate would read as endorsing the newly-chosen type. A low percentage after an override (e.g. `text` → `whole number` at 62%) is shown as-is, using the same HIGH/MEDIUM/LOW styling as any other low score — this is the sole intended warning signal for a risky override; the screen does not add a second indicator on top of it.
- **The `string` (text) badge measures a different kind of fit than the others** — "N% of values fit this type" for `string` means N% of sampled values did *not* match a stricter type pattern (boolean/integer/number/date/timestamp), not that N% successfully "coerced" to text (all non-null values trivially coerce to text, so a coercion-based number would always read 100% and be meaningless). The label wording ("N% of values fit this type") is written to hold for both cases without needing the user to know which mechanism produced it; do not render coercion-specific phrasing (e.g. "N% parsed successfully") on the `string` badge.
