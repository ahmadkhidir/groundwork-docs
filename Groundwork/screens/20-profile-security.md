# Screen 20: Profile & Security

**Route:** `/settings/profile`
**Platform:** Both (simple stacked-card layout that works identically on desktop and a mobile viewport; no dense controls)
**Purpose:** Account self-management — sign out, change password, and delete the account ([[features]] FEAT-052). Account-level, deliberately distinct from the per-workspace [[13-workspace-settings|Workspace Settings]] screen.
**Entry Points:** Navigation Shell → account menu → "Profile & Security."
**Exit Points:** [[03-workspace-list|Workspace List]] (via Navigation Shell), [[01-landing-page|Landing Page]] (after sign out), [[01-landing-page|Landing Page]] with `?deleted=1` (after account deletion).

### Layout

**Mobile layout:**
- Navigation Shell (account menu is where the user came from)
- Vertical stack of `Card` sections: "Account" (email + display name, read-only), "Password" (change-password form), "Danger Zone" (delete account)
- Sections expand/collapse (`Accordion`) to manage length on a small screen

**Web layout:**
- Navigation Shell
- Same vertical stack of `Card` sections, wider cards: "Account," "Password," "Danger Zone"

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Email display | Text | Both | Read-only account identifier — there is no email-change flow | Account state |
| Display name display | Text | Both | Read-only display name, shown as stored per [[schema]] `User.display_name` | Account state |
| Change password form | Form (3× TextInput masked) | Both | Current password, new password, confirm new password | POST `/v1/auth/password-change` |
| Sign out | Button (Outline) | Both | The product's single sign-out affordance — calls the existing Auth.js `signOut` (`web/src/auth.ts`); no second path | — |
| Delete account | Button (Destructive) | Both | Opens the account-scope Delete Confirmation Modal (shared component, type-to-confirm the account email) | DELETE `/v1/account` |
| Account-deletion scope list | List | Both | Explicit enumeration of what account deletion destroys — all workspaces and data, billing cancelled immediately, no automatic refund | Static text |

### Data Requirements

**Data Needed:**
- Signed-in email and display name — from the Auth.js session (`web/src/auth.ts`), which carries the user's identity; the display name is also the value stored per [[schema]] `User.display_name`
- On submit (change password): current password, new password, confirm → POST `/v1/auth/password-change` with `{ "current_password", "new_password" }`
- On confirm (delete account): DELETE `/v1/account`
- No data needed on load for the password or danger sections — the email/display name come from the session

**Data Flow:**
1. Screen mounts → renders the three sections; no fetch is needed (identity comes from the session).
2. Change password: user fills the three fields, taps Submit → POST `/v1/auth/password-change`. On success, the form clears and an inline confirmation ("Password updated") shows; on failure, an inline error names the reason.
3. Sign out: user taps Sign out → Auth.js `signOut` clears the session cookie → redirect to [[01-landing-page|Landing Page]].
4. Delete account: user opens the Delete Confirmation Modal, types their account email to match, confirms → DELETE `/v1/account` → on success the browser is signed out and redirected to [[01-landing-page|Landing Page]] with `?deleted=1` showing the confirmation banner; on partial failure the modal shows the failed step and a retry affordance, and the account remains signed in and intact.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loaded** | Initial load | Account card, Password form, Danger Zone |
| **Password Submitting** | Change-password form submitted | Submit button spinner, fields disabled |
| **Password Updated** | Change succeeded | Inline confirmation, form cleared |
| **Password Error** | Current password wrong (400 `invalid_current_password`) or validation failure | Inline error, form remains editable — never routed to the login gate (a 401 would be misread as session expiry) |
| **Confirming (delete)** | Modal open, email not yet matched | Delete button disabled |
| **Ready to Confirm (delete)** | Typed email matches the account email exactly | Delete button enabled |
| **Deleting** | Account cascade in progress | Modal shows an honest in-progress state per the shared honest-status principle — not a silent wait |
| **Deleted** | Cascade completed | Browser signed out and redirected to the Landing Page with the `?deleted=1` confirmation banner |
| **Deletion Partial/Failed** | Cascade did not complete | Explicit state naming the failed step and that the account is intact with a retry in progress — never a false success, per [[user_flow]] Flow 6 Alternate Path B |
| **Offline** | No network connectivity | Submit disabled, banner: "You're offline — check your connection" |

**Edge Cases:**
- Wrong current password → 400 `invalid_current_password` shown inline; the user stays on the screen (deliberately not 401 — see [[features]] FEAT-052).
- Change password does not revoke existing sessions (Auth.js JWT strategy); the screen's copy states that changing the password does not sign out other devices.
- A stale session cookie from a deleted account (e.g., an old tab) decrypts but every API request fails closed with 401/404 — the optimistic web proxy is not an authorization boundary, per [[architecture]].
- User closes the browser/app during the Deleting state → the cascade continues server-side; on the next visit the account is either gone (landing page) or, if the cascade genuinely failed, still present and able to retry the deletion from this screen.
