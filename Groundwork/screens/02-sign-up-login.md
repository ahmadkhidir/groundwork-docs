# Screen 2: Sign Up / Login

**Route:** `/signup`, `/login`
**Platform:** Both
**Purpose:** Create an account or authenticate an existing one ([[features]] FEAT-001).
**Entry Points:** [[01-landing-page|Landing Page]], Authentication Gate redirect (per [[user_flow]] Global Flows) from any protected route while unauthenticated.
**Exit Points:** [[03-workspace-list|Workspace List]] (new users land on the empty state; returning users land on their list).

### Layout

**Mobile layout:**
- Centered `Card` containing: logo, form (email, password, and — Sign Up mode only — an optional display name), primary Submit `Button`, secondary link toggling between Sign Up and Login, "Forgot password?" `Link` (Login mode only, routes to [[18-forgot-password|Forgot Password]])
- Full-width form fields, safe-area padding

**Web layout:**
- Centered `Card` (max-width ~400px) on a `surface-secondary` background, same fields as mobile
- No sidebar/navbar chrome beyond a minimal logo header — deliberately distraction-free

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Email input | TextInput | Both | Account identifier | User input |
| Password input | TextInput (masked) | Both | Credential | User input |
| Display name input | TextInput | Both | Optional; shown only in Sign Up mode. When left blank, the stored value defaults to the email's local-part (derived server-side per [[schema]] `User.display_name`), so `display_name` is never stored empty. Shown in workspace member lists and audit entries per [[schema]] | User input |
| Submit button | Button (Primary) | Both | Triggers auth | — |
| Mode toggle | Link | Both | Switch between Sign Up / Login | — |
| Forgot password | Link | Both (Login mode only) | Routes to [[18-forgot-password\|Forgot Password]] | — |

### Data Requirements

**Data Needed:** None on load. On submit (Sign Up): email, password, display_name (optional) → POST to account-creation endpoint. The display name field is optional, and an empty submission is treated as "derive from the email's local-part": the blank default is that derived value, applied server-side per [[schema]]'s `User.display_name`, never stored as an empty string. On submit (Login): email, password → POST to auth endpoint.

**Data Flow:**
1. User fills form, taps Submit.
2. Button enters Submitting state, fields disabled.
3. On success → redirect to originally-requested protected route if the Authentication Gate sent the user here, otherwise to [[03-workspace-list|Workspace List]].
4. On failure → show inline error, re-enable form.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Default** | Initial load | Empty form |
| **Submitting** | Form submitted, awaiting response | Submit button spinner, all fields disabled |
| **Validation Error** | Invalid email format or empty required field | Inline field errors, form remains editable |
| **Auth Error** | Wrong credentials / account exists (sign up) | Inline error banner above the form, e.g. "Incorrect email or password" |
| **Offline** | No network connectivity | Submit disabled, banner: "You're offline — check your connection" |

**Edge Cases:**
- Rapid double-tap on Submit → button disabled immediately after first tap to prevent duplicate requests.
- Deep link into a protected route while unauthenticated → redirected here per the Authentication Gate; on success, redirected back to the originally requested route, not always to [[03-workspace-list|Workspace List]].
- Password field visibility toggle (show/hide) available on both platforms.
- Sign Up submitted with display name left blank → account is still created; `display_name` is derived from the email's local-part server-side per [[schema]], so the stored value is the derived local-part (the blank default), never an empty string. No error or extra prompt is shown for leaving it blank.
