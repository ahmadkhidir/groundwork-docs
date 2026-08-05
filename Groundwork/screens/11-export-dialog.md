# Screen 11: Export Dialog

**Route:** Modal on [[09-report-view|Report View]] (`?export=true`)
**Platform:** Both
**Purpose:** Let the user export a report in one of several formats, with tier-gated formats shown but clearly marked rather than hidden ([[features]] FEAT-035–040; gating per [[monetization]]).
**Entry Points:** [[09-report-view|Report View]] ("Export" action).
**Exit Points:** Closes back to [[09-report-view|Report View]]; successful export triggers a file download or a durable "export ready" notification ([[16-notifications|Notifications]]) for larger/slower formats.

### Layout

**Mobile & Web layout (same structure, `Modal`/full-screen sheet on mobile, centered `Modal` on web):**
- Title: "Export report"
- List of format options, each as a selectable row: icon, format name, one-line description
- Available formats (per the user's tier, from [[monetization]]) are fully selectable
- Locked formats use the shared Upgrade Prompt component: reduced-emphasis row, small lock icon, "Upgrade to Pro/Team" `Link` — never hidden entirely, per [[monetization]]'s "always show what you're missing" principle
- "Export" `Button` (Primary), disabled until a format is selected, enabled only for available formats

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Format option row | List item / Radio | Both | Selects PDF, Excel/CSV, PowerPoint, Markdown/HTML, JSON, or Image | User's current tier (from account/subscription state) |
| Upgrade Prompt | Shared component | Both | Shown on locked format rows | [[monetization]] Feature Gate Mapping |
| Export button | Button (Primary) | Both | Triggers the export job | — |

### Data Requirements

**Data Needed:**
- User's current tier (Free/Pro/Team) — from account/subscription state, to determine which format rows are locked
- Report id (to export) — from the current [[09-report-view|Report View]] context

**Data Flow:**
1. Dialog opens, tier state already available from account context (no separate fetch needed in the common case).
2. User selects an available format and taps Export.
3. For fast formats (PDF, JSON, image) → export completes inline, file downloads directly, dialog closes.
4. For slower formats (PowerPoint, large Excel) → dialog shows "Preparing your export..." briefly, then either completes inline or, if it exceeds a short threshold, closes with a durable notification promised ("We'll notify you when it's ready") rather than blocking the dialog open indefinitely.
5. User clicking a locked format's "Upgrade to Pro/Team" link routes to a billing/upgrade flow (out of this screen's scope — see [[monetization]]).

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Default** | Dialog just opened | All format options listed, available ones selectable, locked ones showing Upgrade Prompt |
| **Exporting** | Export in progress | Selected format row shows a spinner, Export button disabled |
| **Export Ready (inline)** | Fast export completed | File download triggers automatically, dialog shows a brief success state then closes |
| **Export Queued** | Slower export exceeds the inline-wait threshold | Dialog shows "We'll notify you when it's ready," closes; completion surfaces via [[16-notifications|Notifications]] |
| **Export Error** | Export generation failed | Inline error on the selected format row with a "Try Again" action; dialog remains open |

**Edge Cases:**
- Free-tier user attempts to select a locked format row directly (not just the Upgrade link) → row is not selectable at all (Export button stays disabled for that row), Upgrade Prompt is the only actionable element on it.
- Report contains no chart (narrative-only) → Image export format is omitted from the list entirely for that specific report, rather than shown and then failing.
- User closes the dialog while an export is queued → the queued export still completes server-side and the durable notification still fires; closing the dialog does not cancel the job.
