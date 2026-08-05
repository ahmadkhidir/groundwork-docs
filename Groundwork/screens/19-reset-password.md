# Screen 19: Reset Password

**Route:** `/reset-password` (reset token read from the `token` query parameter of the emailed link)
**Platform:** Both
**Purpose:** Let a user set a new password using a valid, unconsumed, unexpired reset token ([[features]] FEAT-001).
**Entry Points:** The reset link emailed from [[18-forgot-password|Forgot Password]].
**Exit Points:** [[02-sign-up-login|Sign Up / Login]] (after a successful reset, with a confirmation banner, or via "Request a new link" on failure by way of [[18-forgot-password|Forgot Password]]).

### Layout

**Mobile layout:**
- Centered `Card`: logo, new-password `TextInput` (masked), confirm-password `TextInput` (masked), primary Submit `Button`
- Full-width form fields, safe-area padding

**Web layout:**
- Centered `Card` (max-width ~400px) on a `surface-secondary` background, same fields as mobile
- No sidebar/navbar chrome, consistent with [[02-sign-up-login|Sign Up / Login]] and [[18-forgot-password|Forgot Password]]

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| New password input | TextInput (masked) | Both | New credential | User input |
| Confirm password input | TextInput (masked) | Both | Confirms the new credential matches | User input |
| Submit button | Button (Primary) | Both | Triggers the reset confirmation | — |
| Request new link | Link | Both | Shown only in the Invalid/Expired Token state; routes to [[18-forgot-password\|Forgot Password]] | — |

### Data Requirements

**Data Needed:** `token` read from the URL query string on mount. On submit: `token`, new password → POST to a reset-confirm endpoint.

**Data Flow:**
1. Screen mounts, reads `token` from the URL.
2. If `token` is missing or malformed, the form is not shown at all — screen goes straight to the Invalid/Expired Token state without waiting for a submit attempt.
3. Otherwise, user enters and confirms a new password, taps Submit.
4. Button enters Submitting state, fields disabled.
5. Server validates the token (exists, unexpired, unconsumed) before applying the new password. On success, the `PasswordResetToken` is marked `consumed_at` (per [[schema]]), and any other unconsumed tokens for that user are invalidated.
6. On success → redirect to [[02-sign-up-login|Sign Up / Login]] with a "Password updated — sign in with your new password" confirmation.
7. On failure (expired, already consumed, or otherwise invalid token) → Invalid/Expired Token state, form replaced by an explanatory message and a link to request a new one.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Default** | Token present and well-formed, not yet submitted | Empty form |
| **Submitting** | Form submitted, awaiting response | Submit button spinner, fields disabled |
| **Validation Error** | Passwords don't match, or don't meet minimum password requirements | Inline field errors, form remains editable |
| **Invalid/Expired Token** | Token missing, malformed, expired (>30 minutes since issuance), or already consumed | Form is not shown; message: "This reset link is no longer valid," with a "Request a new link" `Link` to [[18-forgot-password\|Forgot Password]] |
| **Offline** | No network connectivity | Submit disabled, banner: "You're offline — check your connection" |

**Edge Cases:**
- Link opened a second time after the reset already completed (token already consumed) → Invalid/Expired Token state, since `consumed_at` is now set; no partial or misleading "already used" distinction is drawn beyond the standard message.
- Token expires at the exact moment of submit (30-minute boundary race) → the server-side check is authoritative; the user sees the same Invalid/Expired Token treatment rather than a partial success.
- User is already authenticated in another tab/session when opening the reset link → the reset still completes normally; it is not blocked or short-circuited by an existing session.
- Two reset links requested in succession, then the older one is opened → still honored as Default state as long as it is individually unexpired and unconsumed; requesting a new token does not invalidate prior ones (see [[18-forgot-password|Forgot Password]] edge cases).
