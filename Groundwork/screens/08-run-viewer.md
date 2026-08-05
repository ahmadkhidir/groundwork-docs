# Screen 8: Run Viewer

**Route:** `/workspaces/:id/runs/:runId`
**Platform:** Both (the SQL editing/halt/amend controls are desktop-primary — see Edge Cases for mobile behavior)
**Purpose:** Show a question's pipeline running live, let the user watch, inspect the generated SQL, and halt/amend/resume any step ([[features]] FEAT-018, FEAT-022, FEAT-023, FEAT-024, FEAT-027, FEAT-028).
**Entry Points:** [[04-workspace-home|Workspace Home]] (question submitted), [[09-report-view|Report View]] (compositional follow-up question).
**Exit Points:** [[09-report-view|Report View]] (on run completion), [[12-clarifying-question-prompt|Clarifying Question Prompt]] (on ambiguity), [[15-semantic-layer|Semantic Layer / Corrections]] (link to view a persisted correction after amending).

### Layout

**Mobile layout:**
- Navigation Shell
- Vertical live-event feed: each pipeline step (profile, plan, generate_sql, static_check, execute, reconcile, shadow_check, adversarial_review, chart_spec, narrate, assemble) as a `ListView` item with a status icon (pending/running/complete/failed)
- Generated SQL shown in a collapsible `Card` (SQL / Code Block shared component) — viewable, editing is view-only-hinted ("Open on desktop to edit") since inline SQL editing is desktop-primary
- No halt/amend controls on mobile — the run streams to completion; mobile is a watch-only surface for this screen

**Web layout:**
- Navigation Shell with workspace `Sidebar`
- Two-panel layout: left panel is the live-event step list (same steps as above, each expandable); right panel shows the generated SQL in an always-visible SQL / Code Block component, editable inline
- Per-step controls: "Halt" `IconButton` appears on the currently-running step; once halted, "Edit" becomes available on `generate_sql`'s output, with a "Resume from here" `Button` (Primary) after editing
- Reconciliation Notice (shared component) appears inline once the `reconcile` step completes, if applicable

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Step list item | List item | Both | One row per DAG step with live status | Streamed run events |
| SQL / Code Block | Shared component | Both (edit: web only) | Shows generated SQL, editable on web | Streamed `generate_sql` step output |
| Halt control | IconButton | Web only | Pauses the run at the current step | — |
| Resume from here | Button (Primary) | Web only | Resumes execution from the amended step | — |
| Reconciliation Notice | Shared component | Both | Surfaces significant row exclusions unprompted | Streamed `reconcile` step output |

### Data Requirements

**Data Needed:**
- Live step events (step name, status, timestamp, and — for `generate_sql` — the SQL text; for `reconcile` — row counts in/out) — streamed from the run's live event channel
- Run metadata: id, question text, workspace id, halted/amended state

**Data Flow:**
1. Screen mounts, opens a live stream scoped to `:runId`.
2. Steps update in place as events arrive, in order.
3. If the user halts a step (web only) and edits the SQL, the edit is submitted, and "Resume from here" re-opens the stream continuing from `static_check` forward — earlier steps are not re-run.
4. On the `assemble` step completing → auto-navigate to [[09-report-view|Report View]] for the resulting report.
5. If `plan`/`generate_sql` determines the question is ambiguous → the stream instead surfaces a clarification-needed event, and the screen shows [[12-clarifying-question-prompt|Clarifying Question Prompt]] inline rather than continuing the DAG.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Waking** | Engine was sleeping when the question was submitted | First step shows "Waking engine..." before `profile` begins; question is queued, not blocked, per [[user_flow]] Flow 4 |
| **Running** | Steps executing normally | Live-updating step list, current step highlighted |
| **Halted** | User halted a step (web) | Halted step marked, SQL editable, "Resume from here" available |
| **Reconnecting** | Transient engine loss mid-run | Brief "reconnecting" indicator on the affected step; resumes automatically once the engine is available (per [[architecture]]'s idempotent retry design) |
| **Clarification Needed** | Question judged ambiguous | [[12-clarifying-question-prompt\|Clarifying Question Prompt]] shown inline, DAG paused before `generate_sql` proceeds |
| **Failed (retryable)** | A step failed after exhausting automatic retries | That step shows a failed icon + "Retry" affordance scoped to the run; workspace and prior reports remain untouched |
| **Complete** | `assemble` step finished | Brief completion state before auto-navigating to [[09-report-view\|Report View]] |

**Edge Cases:**
- On mobile, if a user needs to edit SQL, the app shows a persistent hint on the SQL card ("SQL editing is available on desktop") rather than a broken partial-edit UI — this is a deliberate platform scope decision, not an oversight, consistent with [[branding]]'s desktop-primary workflow designation.
- User navigates away mid-run → run continues server-side (per [[architecture]] — "runs are server-side state; the UI is a viewer"); returning to this route later reconnects to the live stream at its current point rather than restarting the view.
- Multiple steps failing in sequence (e.g., cascading engine issue) → each failure is shown individually with its own retry, not collapsed into one generic "something went wrong" for the whole run.
