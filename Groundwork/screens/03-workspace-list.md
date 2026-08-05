# Screen 3: Workspace List

**Route:** `/workspaces`
**Platform:** Both
**Purpose:** Show all workspaces the user owns or has been shared into, and let them create or open one ([[features]] FEAT-002).
**Entry Points:** [[02-sign-up-login|Sign Up / Login]] (post-auth), Navigation Shell (workspace switcher), [[14-deletion-confirmation-proof|Deletion Confirmation & Proof]] (after deleting a workspace).
**Exit Points:** [[04-workspace-home|Workspace Home]] (open or newly created workspace).

### Layout

**Mobile layout:**
- Navigation Shell (see [[screens|shared components]])
- Stacked `ListView` of workspace `Card`s (name, engine status pill, last-activity timestamp)
- `FloatingActionButton` ("+") to create a workspace

**Web layout:**
- Navigation Shell
- `Grid` of workspace `Card`s (3 columns on wide viewports)
- "Create Workspace" `Button` (Primary) top-right of the content area

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Workspace card | Card | Both | Shows workspace name, engine status pill, member count if shared, last activity | GET `/api/workspaces` |
| Create Workspace | Button / FAB | Both | Opens workspace-name prompt, then navigates to the new [[04-workspace-home\|Workspace Home]] | — |
| Shared badge | Badge | Both | Marks workspaces the user has read-only access to, not ownership ([[features]] FEAT-045) | Workspace membership role |

### Data Requirements

**Data Needed:**
- List of workspaces: id, name, owner/role, engine status (`sleeping`/`waking`/`ready`), last activity timestamp — from GET `/api/workspaces`

**Data Flow:**
1. Screen mounts → Loading state.
2. Fetch workspace list.
3. On success with 0 workspaces → Empty state (new-account case).
4. On success with 1+ workspaces → Loaded, cards rendered.
5. On failure → Error state with retry.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loading** | Initial fetch in progress | Skeleton cards |
| **Empty** | Account has zero workspaces | Shared Empty State: "No workspaces yet" + "Create your first workspace" CTA |
| **Loaded** | 1+ workspaces returned | Grid/list of workspace cards |
| **Error** | Fetch failed | Shared Error State with retry |

**Edge Cases:**
- Workspace with a very long name → truncate with ellipsis in the card, full name in a tooltip (web) or on tap (mobile).
- A read-only shared workspace ([[features]] FEAT-045) is visually marked distinct from owned workspaces (badge) so the user doesn't attempt to upload/delete from the list view.
- Workspace deleted by its owner while this user had shared access → card disappears on next fetch; no explicit notification required on this screen (handled by [[16-notifications|Notifications]] if the owner chooses to notify).
