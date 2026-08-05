# Screen 14: Deletion Confirmation & Proof

**Route:** Modal on [[13-workspace-settings|Workspace Settings]] → `/workspaces/:id/deleted` (post-deletion proof page)
**Platform:** Both
**Purpose:** Make the scope of workspace deletion explicit before it happens, execute the cascade, and show provable evidence of what was deleted and when ([[features]] FEAT-043).
**Entry Points:** [[13-workspace-settings|Workspace Settings]] ("Delete Workspace" in the Danger Zone).
**Exit Points:** [[03-workspace-list|Workspace List]] (after viewing the deletion proof).

### Layout

**Confirmation Modal (both platforms):**
- Uses the shared Delete Confirmation Modal pattern, extended with explicit scope: a bullet list naming exactly what will be destroyed — "object store contents, semantic layer, reports and conversation history, query caches, and logs" — not a generic "Are you sure?"
- Workspace name must be typed to confirm (`TextInput` matching the workspace name exactly) before "Delete" (Destructive) is enabled — a deliberate high-friction confirmation given irreversibility
- "Cancel" (Ghost)

**Deletion Proof page (`/workspaces/:id/deleted`, both platforms):**
- Centered confirmation `Card`: success icon, "Workspace deleted" heading
- Itemized list of what was deleted, each with a checkmark (object store contents, semantic layer, reports/conversation history, query caches, logs)
- Timestamp of deletion completion
- "Back to Workspaces" `Button` (Primary), routing to [[03-workspace-list|Workspace List]]

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Scope list (confirmation) | List | Both | Explicit enumeration of what will be destroyed | Static text, workspace-scoped |
| Confirm-by-typing input | TextInput | Both | Requires typing the workspace name to enable Delete | Workspace name (for match validation) |
| Delete button | Button (Destructive) | Both | Executes the cascade | — |
| Deletion proof list | List (with checkmarks) | Both | Confirms each category was actually destroyed | Deletion job result |
| Deletion timestamp | Text | Both | Exact time the cascade completed | Deletion job result |

### Data Requirements

**Data Needed:**
- On confirm: workspace id → DELETE `/api/workspaces/:id`
- Deletion result: per-category completion status (object store, semantic layer, reports, caches, logs), completed_at timestamp — from the DELETE response or a follow-up GET `/api/deletions/:deletionId`

**Data Flow:**
1. User opens the confirmation modal from [[13-workspace-settings|Workspace Settings]].
2. User types the workspace name to match, enabling Delete.
3. User confirms → DELETE request sent, modal shows an executing state.
4. On success → navigate to `/workspaces/:id/deleted`, showing the itemized Deletion Proof.
5. User taps "Back to Workspaces" → navigates to [[03-workspace-list|Workspace List]], where the deleted workspace no longer appears.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Confirming** | Modal open, name not yet matched | Delete button disabled |
| **Ready to Confirm** | Typed name matches workspace name exactly | Delete button enabled |
| **Executing** | Cascade in progress | Modal shows a determinate/indeterminate progress state per the shared honest-status principle — not a silent wait |
| **Deletion Proof (Success)** | All categories confirmed destroyed | Full itemized proof list with checkmarks and timestamp |
| **Deletion Partial/Failed** | Cascade did not complete fully | Explicit state naming which category failed and that a retry is in progress — never a false success shown, per [[user_flow]] Flow 5 Alternate Path B |

**Edge Cases:**
- Typed name doesn't match (case-sensitive or typo) → inline validation message, Delete remains disabled; no vague "doesn't match" without showing what's expected.
- User closes the browser/app during the Executing state → the cascade continues server-side; navigating back to the workspace list later either shows the workspace gone (success) or, if genuinely failed, the workspace still present with a durable notification ([[16-notifications|Notifications]]) explaining the deletion needs retry.
- Read-only shared members never see this screen at all — deletion is owner-only, enforced at the [[13-workspace-settings|Workspace Settings]] level per that screen's edge cases.
