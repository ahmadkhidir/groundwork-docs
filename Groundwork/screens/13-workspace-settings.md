# Screen 13: Workspace Settings

**Route:** `/workspaces/:id/settings`
**Platform:** Both (desktop-optimized; mobile supports viewing and the most common actions, dense controls like per-column exclusion are easier on desktop)
**Purpose:** Manage column-level exclusion, workspace/report sharing, usage visibility, the entry point to the visible model-egress/audit log, and the entry point to workspace deletion ([[features]] FEAT-010, FEAT-042, FEAT-045, FEAT-046, FEAT-047).
**Entry Points:** [[04-workspace-home|Workspace Home]] (Sidebar/Navigation Shell).
**Exit Points:** [[14-deletion-confirmation-proof|Deletion Confirmation & Proof]] ("Delete workspace"), [[17-egress-audit-log|Egress / Audit Log]] ("Audit Log" tab), [[04-workspace-home|Workspace Home]] (back navigation).

### Layout

**Mobile layout:**
- Navigation Shell
- Vertical stack of settings sections, each a `Card`: "Column Visibility," "Sharing," "Usage," "Audit Log" (link out to [[17-egress-audit-log|Egress / Audit Log]]), "Danger Zone" (delete)
- Sections expand/collapse (`Accordion`) to manage length on a small screen

**Web layout:**
- Navigation Shell with workspace `Sidebar`
- `Tabs`: "Column Visibility," "Sharing," "Usage," "Audit Log," "Danger Zone"
- Column Visibility tab: `Table` of all ingested columns (workspace-wide, across files) with an exclusion `Toggle` per column, plus the automatic PII flag shown for context
- Sharing tab: two sub-sections — "Workspace members" (Team tier — invite by email, role shown as read-only, remove action) and "Shared report links" (list of active share links with revoke action)
- Usage tab: current-period engine-seconds, storage, and model-token usage, shown as simple stat rows (Free tier: totals only; Pro/Team: detailed breakdown, per [[monetization]])
- Audit Log tab: not inline content — a short explanatory line plus a `Button` that navigates to the full [[17-egress-audit-log|Egress / Audit Log]] screen, kept as its own route rather than a tab body since the log itself is a filterable, paginated surface, not a settings form
- Danger Zone tab: single "Delete Workspace" `Button` (Destructive), isolated visually from other settings

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Column exclusion toggle | Toggle | Both | Marks a column as never leaving the trust boundary to the model | GET/PATCH `/api/workspaces/:id/columns/:columnId/exclusion` |
| Member list row | List item | Both | Shows a workspace member and role, with remove action | GET `/api/workspaces/:id/members` |
| Shared link row | List item | Both | Shows an active report share link with revoke action | GET `/api/workspaces/:id/report-shares` |
| Usage stat row | Stat display | Both | Engine-seconds / storage / tokens for the current period | GET `/api/workspaces/:id/usage` |
| Delete Workspace | Button (Destructive) | Both | Opens [[14-deletion-confirmation-proof\|Deletion Confirmation & Proof]] | — |
| View Audit Log | Button | Both | Navigates to [[17-egress-audit-log\|Egress / Audit Log]] | — |
| Upgrade Prompt | Shared component | Both | Shown on Sharing and detailed-Usage sections for Free-tier accounts | [[monetization]] Feature Gate Mapping |

### Data Requirements

**Data Needed:**
- All ingested columns workspace-wide, with current exclusion state and PII flag — from GET `/api/workspaces/:id/columns`
- Workspace members and their roles (Team tier) — from GET `/api/workspaces/:id/members`
- Active report share links (Pro/Team tier) — from GET `/api/workspaces/:id/report-shares`
- Current-period usage: engine-seconds, storage bytes, model tokens — from GET `/api/workspaces/:id/usage`
- User's current tier — from account/subscription state

**Data Flow:**
1. Screen mounts → each tab's data loads independently (lazy-loaded per tab on web to avoid fetching sharing/usage data before the user opens those tabs).
2. Toggling column exclusion → optimistic UI update, PATCH request, rollback with an inline error if it fails.
3. Free-tier users see Sharing and detailed Usage sections replaced with an Upgrade Prompt state rather than a fetch attempt.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Tab's data being fetched | Skeleton per tab |
| **Loaded** | Data fetched successfully | Full tab content |
| **Locked (Free tier)** | Sharing tab or detailed Usage accessed on Free tier | Upgrade Prompt in place of the tab's normal content, per [[monetization]] |
| **Error** | Fetch or update failed | Shared Error State; for a failed toggle update specifically, an inline error with the toggle reverted to its prior state |

**Edge Cases:**
- Excluding a column that's referenced by an existing semantic-layer metric definition → a warning is shown before confirming ("This column is used in 2 metric definitions — excluding it may affect future answers"), not a silent block.
- Read-only shared member (FEAT-045) viewing this screen → sees Usage and their own access level only; Column Visibility, Sharing management, Audit Log, and Danger Zone are hidden entirely for non-owners, not just disabled, since a read-only member should not know the workspace deletion control exists as an actionable item, nor see what data left to the model on the owner's behalf.
- Revoking a report share link that someone currently has open → the link stops working on next load for that viewer; no forced disconnect of an already-open page is implied.
