# Screen 5: File Upload

**Route:** `/workspaces/:id/upload`
**Platform:** Both (drag-and-drop enhanced on web; native file picker on mobile)
**Purpose:** Let the user bring one or more files into the workspace ([[features]] FEAT-003), and resolve any filename collision with an already-uploaded dataset before ingest begins ([[features]] FEAT-049).
**Entry Points:** [[04-workspace-home|Workspace Home]] ("Upload data" empty-state CTA; Datasets panel "Add data" once data exists).
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
| Duplicate Filename dialog | Modal | Both | Shown per colliding filename before ingest begins; offers Replace (default) or Keep Both per [[features]] FEAT-049. Reuses the shared Delete Confirmation Modal pattern for the Replace path, so it states what Replace deletes — including any type overrides ([[features]] FEAT-006) or reviewed PII decisions ([[features]] FEAT-007) on the prior dataset's columns, per [[features]] FEAT-048 | GET `/api/workspaces/:id/files?filename=` (existence + `created_at`/`ingest_status` check) |

### Data Requirements

**Data Needed:** None on load. On submit: file(s) → POST (multipart) to `/api/workspaces/:id/files`. Before that POST is treated as ingest-starting, each selected file's name is checked against existing `Dataset.original_filename` values in the workspace (any `ingest_status`, including `failed`) per [[features]] FEAT-049.

**Data Flow:**
1. User selects or drops file(s).
2. Client-side validation: file type and size ceiling checked before upload begins.
3. For each selected file whose name matches an existing dataset in the workspace, the Duplicate Filename dialog is shown before that file proceeds — Replace (deletes the prior dataset via [[features]] FEAT-048, pre-selected; if the prior dataset has any type overrides or reviewed PII decisions on its columns, the dialog states plainly that this review work will be lost and is not carried forward to the new file) or Keep Both (ingests as a distinct dataset, disambiguated by upload timestamp everywhere the two would otherwise look identical). Non-colliding files in the same batch are unaffected and proceed normally.
4. On "Start Ingest" (with any duplicate-filename choices resolved), files upload with progress; on completion, system navigates to [[06-ingest-progress|Ingest Progress]].

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Default** | No files selected | Empty drop zone / picker prompt |
| **Files Selected** | 1+ valid files chosen, not yet submitted | Selected-files list, Start Ingest enabled |
| **Duplicate Filename Detected** | 1+ selected files match an existing dataset's `original_filename` in this workspace | Duplicate Filename dialog per colliding file, Replace pre-selected; Start Ingest disabled until every collision is resolved |
| **Uploading** | Upload in progress | Per-file progress bars, Start Ingest button shows spinner, drop zone disabled |
| **Upload Error** | Unsupported type, oversized file, or network failure mid-upload | Inline error per affected file with a specific reason; other valid files unaffected and still submittable |
| **Offline** | No network connectivity | Start Ingest disabled, banner: "You're offline" |

**Edge Cases:**
- Unsupported file type dropped (e.g., `.docx`) → immediately rejected client-side with a specific message ("Groundwork supports CSV, Excel, Parquet, and SQLite files"), never silently ignored.
- File exceeds the size ceiling → rejected with the specific ceiling stated, not a generic "file too large."
- Multiple files of mixed validity dropped at once → valid ones proceed to the selected list, invalid ones show inline errors individually.
- User navigates away mid-upload → confirmation prompt ("Upload in progress — leave anyway?") since partial uploads aren't resumed.
- **A selected file's name matches an existing dataset in the workspace, including one stuck at `ingest_status: failed`** — the Duplicate Filename dialog is shown before ingest starts, never silently creating a second dataset with the same name. This is the exact path a real incident took: a falsely-reported ingest failure led to repeated retries, each creating a new, indistinguishable dataset, with no way to remove any of them short of deleting the whole workspace. Replace (default) supersedes the prior dataset; Keep Both ingests a distinct one, disambiguated by upload time. See [[features]] FEAT-049.
- **The existing dataset being replaced has a type override or a reviewed PII decision on one or more columns** — the ordinary case of "I fixed the spreadsheet and re-uploaded it," since a dataset a user has already worked through ingest confirmation on is exactly the one most likely to get re-uploaded later. The Duplicate Filename dialog states this explicitly before Replace can be confirmed (per [[features]] FEAT-048/FEAT-049); there is no way to carry those decisions forward onto the new file in Phase 1, so the dialog's job is disclosure, not prevention.
- **User chooses "Keep both"** → the two identically-named datasets are never compared against each other by relationship inference (per [[features]] FEAT-008); [[06-ingest-progress|Ingest Progress]] proceeds normally, and the skip itself — along with a pointer to add a relationship manually — is surfaced on [[07-ingest-confirmation|Ingest Confirmation]]'s Relationships step, not on this screen.
- Two or more selected files in the same upload batch share a name with each other (not with an existing dataset) → each is still checked against existing datasets independently; name collisions purely within the current batch are out of scope for this dialog and are handled as distinct uploads, since neither yet exists as a `Dataset` for the other to collide with.
