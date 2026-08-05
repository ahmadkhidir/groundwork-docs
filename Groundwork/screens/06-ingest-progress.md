# Screen 6: Ingest Progress

**Route:** `/workspaces/:id/ingest/progress`
**Platform:** Both
**Purpose:** Show the user that their uploaded file(s) are being provisioned, ingested, and profiled ([[features]] FEAT-004, FEAT-005).
**Entry Points:** [[05-file-upload|File Upload]] (automatic, on successful upload).
**Exit Points:** [[07-ingest-confirmation|Ingest Confirmation]] (on success), [[05-file-upload|File Upload]] (on failure, to retry).

### Layout

**Mobile & Web layout (same structure, centered content):**
- Navigation Shell
- Centered step indicator (`Stepper` or simple ordered list): "Provisioning engine" → "Loading data" → "Profiling columns" → (if 2+ files) "Finding relationships"
- Current step highlighted with a `LinearProgressIndicator`/`ProgressBar`; completed steps show a checkmark
- No user action available during this screen except an implicit background wait — this is intentionally non-interactive but never a bare unlabeled spinner, consistent with [[branding]]'s honest-status principle

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Step indicator | Stepper | Both | Shows which ingest stage is active | Streamed ingest status events |
| Progress bar | ProgressBar / LinearProgressIndicator | Both | Visual progress within the current step where determinable | Streamed ingest status events |

### Data Requirements

**Data Needed:**
- Streamed ingest step status (provisioning, loading, profiling, relationship-inference if applicable) — from the same live-event mechanism used by [[08-run-viewer|Run Viewer]], scoped to the ingest pipeline rather than a query run

**Data Flow:**
1. Screen mounts, opens a live status stream for the in-progress ingest.
2. Steps update in place as events arrive.
3. On final success → auto-navigate to [[07-ingest-confirmation|Ingest Confirmation]].
4. On failure at any step → show the specific failure reason inline, with a "Back to Upload" action.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Provisioning** | Engine being started for this workspace | "Provisioning engine" step active |
| **Loading** | File(s) being loaded/normalized | "Loading data" step active |
| **Profiling** | Column statistics being computed | "Profiling columns" step active |
| **Relationship Inference** | 2+ files uploaded together | "Finding relationships" step active |
| **Success** | All steps complete | Brief success state before auto-navigating to Ingest Confirmation |
| **Ingest Error** | Any step fails (malformed row, unsupported encoding, etc.) | Specific error message tied to the failed step + "Back to Upload" button |

**Edge Cases:**
- Very large file taking noticeably longer at the "Loading data" step → progress bar reflects genuine sub-progress if available; otherwise an indeterminate bar is shown rather than a stalled-looking determinate one.
- User closes the tab mid-ingest → ingest continues server-side; returning to [[04-workspace-home|Workspace Home]] later shows the correct post-ingest state (Ingest Confirmation pending, or already resolved) rather than losing progress.
