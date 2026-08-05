# Screen 16: Notifications

**Route:** `/notifications`
**Platform:** Both
**Purpose:** Show durable, persisted notifications — export ready, engine slept, ingest failed, workspace invite — deliberately separate from the high-frequency live run event stream ([[features]] FEAT-026).
**Entry Points:** Navigation Shell (notifications bell icon, all authenticated screens).
**Exit Points:** Deep links into the relevant screen per notification type — e.g. an "export ready" notification links to [[09-report-view|Report View]]; an "ingest failed" notification links to [[05-file-upload|File Upload]]; a workspace invite links to [[03-workspace-list|Workspace List]].

### Layout

**Mobile layout:**
- Navigation Shell
- Stacked `ListView` of notification `Card`s, unread ones visually distinguished (left accent bar in `primary` token, bold title)
- Pull-to-refresh to check for new notifications

**Web layout:**
- Compact `Popover`/`Panel` triggered from the Navbar bell icon for quick recent-notification glance, plus a full `/notifications` page for the complete history
- Full page: `ListView` of notification rows, unread distinguished the same way, "Mark all as read" `Button` (Ghost) top-right

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Notification bell | IconButton (with unread badge) | Both | Entry point from the Navigation Shell | GET `/api/notifications?unread_count=true` |
| Notification row/card | List item | Both | Shows notification type, message, timestamp, read state | GET `/api/notifications` |
| Mark as read | IconButton / auto-on-open | Both | Marks a notification read | PATCH `/api/notifications/:id` |

### Data Requirements

**Data Needed:**
- Notification list: id, type (export_ready, engine_slept, ingest_failed, workspace_invite, etc.), message, related entity link (report id, workspace id, etc.), read/unread state, created_at — from GET `/api/notifications`
- Unread count — from the same endpoint or a lightweight count-only variant, polled or pushed for the Navbar badge

**Data Flow:**
1. Navbar bell shows unread count on every authenticated screen, refreshed periodically or pushed.
2. Opening the panel/page fetches the full list.
3. Clicking a notification marks it read and navigates to its linked screen.
4. "Mark all as read" (web) clears the unread state without navigating anywhere.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | List being fetched | Skeleton rows |
| **Empty** | No notifications yet | Shared Empty State: "No notifications yet" |
| **Loaded** | Notifications present | Full list, unread visually distinguished |
| **Error** | Fetch failed | Shared Error State with retry |

**Edge Cases:**
- A notification's linked entity has since been deleted (e.g., "export ready" for a report that was later removed) → notification remains in the list (it's a historical record) but the link is disabled with a tooltip explaining it's no longer available, rather than a broken navigation.
- High-volume notification periods (e.g., several exports queued at once) → list is not deduplicated or collapsed automatically; each event remains individually visible, consistent with this being a durable, low-noise-by-design channel per [[architecture]] (distinct from the live event stream, which is where high-frequency noise is expected instead).
- Workspace-invite notification for a workspace the user has since lost access to (invite revoked before acceptance) → clicking it shows an explanatory state ("This invite is no longer valid") rather than a generic error.
