# Screen 17: Egress / Audit Log

**Route:** `/workspaces/:id/audit-log`
**Platform:** Both (desktop-optimized; the payload detail view is easier to read on a wider screen, but mobile supports the full list and filtering)
**Purpose:** Let the workspace owner see every payload sent to the model, every query executed, export, deletion, and login — the single visible model-egress/audit log that makes Groundwork's privacy claims verifiable rather than just trusted ([[features]] FEAT-042). `query_executed`, `model_call`, `export`, and `deletion` entries are scoped to this workspace; `login` entries are account-scoped (per [[schema]]) and this screen shows the current user's own login entries alongside them, labeled by scope rather than attributed to a workspace they didn't occur in.
**Entry Points:** [[13-workspace-settings|Workspace Settings]] ("Audit Log" tab/section), [[05-file-upload|File Upload]] (zero-retention notice link, per [[features]] FEAT-044).
**Exit Points:** [[13-workspace-settings|Workspace Settings]] (back navigation).

### Layout

**Mobile layout:**
- Navigation Shell
- Filter row: action-type filter `Chips` (`Query`, `Model Call`, `Export`, `Deletion`, `Login`), horizontally scrollable
- Stacked `ListView` of audit entry `Card`s (timestamp, user, action type badge, scope badge, one-line summary)
- Tapping a card expands it in place to show the full entry detail, including `payload_sent_to_model` for `model_call` entries
- "Load more" at list end (cursor-based)

**Web layout:**
- Navigation Shell with workspace `Sidebar`
- Filter/search bar: action-type filter `Chips`, date-range picker, all above a `Table`
- `Table` columns: Timestamp, User, Action Type (badge), Scope (`This workspace` / `Account`), Summary
- Row click opens a `Drawer` with full entry detail: action type, user, timestamp, scope, and — for `model_call` entries — the exact `payload_sent_to_model` rendered in the SQL / Code Block shared component (per [[screens]] Shared/Common Components), since it is JSON/text, not prose
- "Load more" / cursor-based pagination at table end

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Action-type filter chips | Chip group | Both | Filters entries by `query_executed`/`model_call`/`export`/`deletion`/`login` | Client-side filter param on the list request |
| Date-range picker | Date input | Web only | Narrows entries to a time window | Client-side filter param on the list request |
| Audit entry row/card | List item / Table row | Both | Shows one AuditLogEntry summary, with a Scope badge distinguishing this-workspace entries from the current user's account-scoped `login` entries | GET `/api/workspaces/:id/audit-log` |
| Entry detail drawer/expansion | Drawer (web) / inline expansion (mobile) | Both | Shows full entry detail including `payload_sent_to_model` | Already present on the fetched entry — no separate request |
| Payload viewer | SQL / Code Block (shared component) | Both | Renders `payload_sent_to_model` as read-only formatted JSON | Field on the fetched AuditLogEntry |
| Load more | Button (Ghost) | Both | Fetches the next page of entries | GET `/api/workspaces/:id/audit-log?cursor=...` |

### Data Requirements

**Data Needed:**
- Paginated list of `AuditLogEntry` records for the workspace, plus the current user's own account-scoped `login` entries — `id`, `workspace_id` (empty string for `login`), `user_id` (resolved to display name), `action_type`, `payload_sent_to_model`, `timestamp` — from GET `/api/workspaces/:id/audit-log`. The endpoint merges both sets server-side so the client never needs a second request or a client-side merge.
- Optional filter params: `action_type`, `date_from`, `date_to`, `cursor`

**Data Flow:**
1. Screen mounts → fetches the first page of entries, most recent first.
2. Changing a filter (action type or date range) re-fetches from the first page with the new params; it does not filter client-side against an unbounded local set.
3. Opening an entry's detail reveals `payload_sent_to_model` already present on that entry — no additional round trip, since the field is part of the AuditLogEntry itself.
4. "Load more" appends the next cursor page to the existing list without resetting scroll position.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Initial fetch in progress | Skeleton rows/cards |
| **Loaded** | Entries present | Full filtered list, most recent first |
| **Empty** | No entries yet (brand-new workspace) | Shared Empty State: "No activity yet — actions will appear here as you use this workspace" |
| **Filtered Empty** | Filters applied, no entries match | "No matching activity" with a "Clear filters" `Link`, distinct from the true-empty state so it's not mistaken for a broken log |
| **Error** | Fetch failed | Shared Error State with retry |

**Edge Cases:**
- Read-only shared member ([[features]] FEAT-045) navigating to this route directly → screen is hidden/inaccessible entirely, not just read-only, consistent with the same non-owner treatment as [[13-workspace-settings|Workspace Settings]]'s Danger Zone — a read-only member should not see what the model was sent on the owner's behalf.
- `action_type` other than `model_call` (e.g., `query_executed`, `export`, `deletion`, `login`) → `payload_sent_to_model` is an empty string per [[schema]]; the detail view shows "No model payload for this action" rather than a blank panel.
- `login` entries have an empty `workspace_id` per [[schema]] and are shown with an "Account" Scope badge rather than the workspace's name; only the current user's own logins are ever merged in, never another member's, even for the workspace owner — a login is proof of one account's access, not something a workspace grants visibility into for anyone else.
- Very large `payload_sent_to_model` (a wide table's schema/profiling context) → truncated in the row summary, shown in full (scrollable) in the detail drawer/expansion — never silently cut without an affordance to see the rest.
- High-volume workspace with thousands of entries → cursor-based pagination only; no attempt to load the full history into memory at once.
- Workspace deleted while an audit log tab/page was open elsewhere → per [[features]] FEAT-043, the audit log itself is destroyed in the deletion cascade; a stale open tab shows the shared Error State on its next fetch rather than an empty log (the workspace no longer exists to query).
