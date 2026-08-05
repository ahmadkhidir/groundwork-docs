# Screen 18: Forgot Password

**Route:** `/forgot-password`
**Platform:** Both
**Purpose:** Let a user request a password reset link by email, without ever revealing whether the entered address has an account ([[features]] FEAT-001).
**Entry Points:** [[02-sign-up-login|Sign Up / Login]] ("Forgot password?" link, Login mode).
**Exit Points:** [[02-sign-up-login|Sign Up / Login]] ("Back to login" link); user continues via the emailed link to [[19-reset-password|Reset Password]].

### Layout

**Mobile layout:**
- Centered `Card`: logo, single email `TextInput`, primary Submit `Button`, "Back to login" `Link`
- Full-width form field, safe-area padding

**Web layout:**
- Centered `Card` (max-width ~400px) on a `surface-secondary` background, same field as mobile
- No sidebar/navbar chrome beyond a minimal logo header — same distraction-free treatment as [[02-sign-up-login|Sign Up / Login]]

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Email input | TextInput | Both | Account identifier for the reset request | User input |
| Submit button | Button (Primary) | Both | Triggers the reset request | — |
| Back to login | Link | Both | Returns to [[02-sign-up-login\|Sign Up / Login]] | — |

### Data Requirements

**Data Needed:** None on load. On submit: email → POST to a reset-request endpoint.

**Data Flow:**
1. User enters email, taps Submit.
2. Button enters Submitting state, field disabled.
3. Server checks the email against existing accounts. If it matches one, a `PasswordResetToken` is created (per [[schema]]) and delivery is requested from the transactional email provider (per [[architecture]]). If it does not match, no token is created and nothing is sent.
4. Regardless of which happened, the response — and the UI state shown — is identical: a generic confirmation. This is a deliberate security requirement, not an oversight: the reset flow must never reveal whether an account exists for a given email.
5. Screen transitions in place to the Confirmation (Sent) state; it does not navigate away, since there is nothing further for the user to do here.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Default** | Initial load | Empty form |
| **Submitting** | Form submitted, awaiting response | Submit button spinner, field disabled |
| **Confirmation (Sent)** | Request completed (regardless of account existence) | "If an account exists for that email, we've sent a reset link. It expires in 30 minutes." — identical wording whether or not the account exists |
| **Validation Error** | Invalid email format or empty field | Inline field error, form remains editable |
| **Offline** | No network connectivity | Submit disabled, banner: "You're offline — check your connection" |
| **Error** | Request failed for a reason unrelated to account existence (e.g., network/server error) | Inline error banner with a "Try Again" affordance, distinct from Confirmation so a real failure is never mistaken for "sent" |

**Edge Cases:**
- Repeated submissions for the same email in a short window → each still returns the identical generic confirmation; a new token is issued each time, and any prior unconsumed tokens for that user remain valid until their own expiry or use (server-side detail, not surfaced to the user).
- Rapid double-tap on Submit → button disabled immediately after first tap to prevent duplicate requests.
- Email entered belongs to an account created via a future non-password auth method (out of scope for v1, noted for forward compatibility) → treated the same as a non-matching email; the generic confirmation is shown either way.
