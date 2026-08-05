# Screen 10: Provenance Panel

**Route:** Drawer/overlay on [[09-report-view|Report View]] (`?claim=:claimId`)
**Platform:** Both
**Purpose:** Show exactly which query and rows produced a specific number the user clicked on — the concrete mechanism behind claim-level provenance ([[features]] FEAT-030).
**Entry Points:** [[09-report-view|Report View]] (clicking any Provenance-Linked Number).
**Exit Points:** Closes back to [[09-report-view|Report View]]; "View full run" link to [[08-run-viewer|Run Viewer]] for the originating run.

### Layout

**Mobile layout:**
- `ModalBottomSheet` sliding up from the bottom, covering ~80% of viewport height, scrollable
- Header: the claim's stated figure (large, `h2`) + the sentence it came from, for context
- Body: the exact query (SQL / Code Block, read-only) that produced it
- Result rows: a scrollable `Table` showing the specific rows/cells backing the number, with the relevant cell highlighted
- "View full run" `Link` at the bottom

**Web layout:**
- Right-side `Drawer`, ~480px wide, overlaying part of the Report View without fully hiding it
- Same content structure as mobile: claim header, read-only SQL, result rows table with the relevant cell highlighted
- Close `IconButton` top-right of the drawer

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Claim header | Text block | Both | Restates the number and its narrative context | Provenance link data |
| Source query | SQL / Code Block (shared, read-only) | Both | The exact SQL that produced the figure | Provenance link's linked query |
| Result rows table | Table | Both | The specific backing rows/cells, with the relevant cell highlighted | Provenance link's linked result rows |
| View full run | Link | Both | Routes to [[08-run-viewer\|Run Viewer]] for the originating run | Provenance link's `run_id` |

### Data Requirements

**Data Needed:**
- Provenance link detail: claim text, stated value, source query text, result row set (with the specific cell/row identified), originating run id — from GET `/api/workspaces/:id/reports/:reportId/provenance/:claimId`

**Data Flow:**
1. User clicks a Provenance-Linked Number on [[09-report-view|Report View]].
2. Panel opens, `?claim=:claimId` reflected in the URL so the panel state is linkable/shareable.
3. Fetch provenance detail for that specific claim.
4. On success → render query + rows, with the relevant cell visually highlighted (using `provenance-light` background per [[branding]]).
5. Closing the panel returns to the unmodified Report View underneath; the URL drops the `?claim=` param.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Provenance detail being fetched | Skeleton within the drawer/sheet |
| **Loaded** | Detail fetched successfully | Full query + highlighted result rows |
| **Error** | Fetch failed | Shared Error State inline within the panel (not a full-page error, since the underlying report remains visible) |
| **Source Unavailable** | The underlying query/rows have since been deleted or the workspace's data changed materially | Explanatory message: "This source data is no longer available" instead of an error — an honest, distinct state from a fetch failure |

**Edge Cases:**
- Very large result row set behind a single claim (e.g., an aggregate over thousands of rows) → the table paginates or virtualizes rather than rendering all rows at once; the aggregate computation itself is still shown clearly at the top regardless of row count.
- Panel opened via a direct link with `?claim=:claimId` but the user isn't authenticated and the report isn't a public shared link → Authentication Gate applies (see [[user_flow]] Global Flows).
