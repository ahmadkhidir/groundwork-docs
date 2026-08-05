# Screen 5: File Upload

**Route:** `/workspaces/:id/upload`
**Platform:** Both (drag-and-drop enhanced on web; native file picker on mobile)
**Purpose:** Let the user bring one or more files into the workspace ([[features]] FEAT-003).
**Entry Points:** [[04-workspace-home|Workspace Home]] ("Upload data").
**Exit Points:** [[06-ingest-progress|Ingest Progress]] (on successful upload start), [[17-egress-audit-log|Egress / Audit Log]] (zero-retention notice link, opens in a new context without abandoning the in-progress upload).

### Layout

**Mobile layout:**
- Navigation Shell
- Large tappable upload area with icon + "Tap to select files" text
- Selected-files list below (filename, size, remove `IconButton`) once files are chosen
- "Start Ingest" `Button` (Primary), disabled until at least one file is selected

**Web layout:**
- Navigation Shell with workspace `Sidebar`
- Large dashed-border `FileUpload` drop zone, click-to-browse fallback
- Selected-files list (filename, size, type icon, remove `IconButton`)
- "Start Ingest" `Button` (Primary)
- Helper text linking to the zero-retention notice ([[features]] FEAT-044) — "See what happens to your data" `Link`, pointing to the fuller privacy/egress explanation on [[17-egress-audit-log|Egress / Audit Log]] (FEAT-042)

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Drop zone / picker | FileUpload | Both | Accepts CSV, Excel, Parquet, SQLite | User file selection |
| Selected file row | List item | Both | Shows filename, size, remove action | Client-side file list |
| Zero-retention notice link | Link | Both | Opens the FEAT-044 plain-language data-handling explanation, linking through to the fuller egress/audit log explanation on [[17-egress-audit-log\|Egress / Audit Log]] (FEAT-042) | Static content |
| Start Ingest button | Button (Primary) | Both | Begins upload + ingest | — |

### Data Requirements

**Data Needed:** None on load. On submit: file(s) → POST (multipart) to `/api/workspaces/:id/files`.

**Data Flow:**
1. User selects or drops file(s).
2. Client-side validation: file type and size ceiling checked before upload begins.
3. On "Start Ingest," files upload with progress; on completion, system navigates to [[06-ingest-progress|Ingest Progress]].

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Default** | No files selected | Empty drop zone / picker prompt |
| **Files Selected** | 1+ valid files chosen, not yet submitted | Selected-files list, Start Ingest enabled |
| **Uploading** | Upload in progress | Per-file progress bars, Start Ingest button shows spinner, drop zone disabled |
| **Upload Error** | Unsupported type, oversized file, or network failure mid-upload | Inline error per affected file with a specific reason; other valid files unaffected and still submittable |
| **Offline** | No network connectivity | Start Ingest disabled, banner: "You're offline" |

**Edge Cases:**
- Unsupported file type dropped (e.g., `.docx`) → immediately rejected client-side with a specific message ("Groundwork supports CSV, Excel, Parquet, and SQLite files"), never silently ignored.
- File exceeds the size ceiling → rejected with the specific ceiling stated, not a generic "file too large."
- Multiple files of mixed validity dropped at once → valid ones proceed to the selected list, invalid ones show inline errors individually.
- User navigates away mid-upload → confirmation prompt ("Upload in progress — leave anyway?") since partial uploads aren't resumed.
