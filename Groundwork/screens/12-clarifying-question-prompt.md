# Screen 12: Clarifying Question Prompt

**Route:** Inline on [[08-run-viewer|Run Viewer]] (no separate route — a state of that screen)
**Platform:** Both
**Purpose:** Ask the user a specific clarifying question instead of guessing, when their question is genuinely ambiguous ([[features]] FEAT-033).
**Entry Points:** [[08-run-viewer|Run Viewer]] (automatic, when the `plan` step detects ambiguity).
**Exit Points:** [[08-run-viewer|Run Viewer]] (resumes with the clarification applied).

### Layout

**Mobile & Web layout (same structure, inline within the Run Viewer's step feed):**
- Replaces the next-step placeholder in the step feed with a distinct `Card`: an `info`-toned icon, the specific clarifying question text, and a small set of response options if the ambiguity has discrete resolutions (e.g., "By revenue" / "By count"), or a free-text `TextInput` if it doesn't
- "Submit" `Button` (Primary), disabled until a response is given

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Clarifying question card | Card | Both | Presents the specific ambiguity and asks for resolution | Streamed `plan` step output (ambiguity detected) |
| Response options | Button group / Radio | Both | Discrete resolution choices, when the ambiguity has a small enumerable set of answers | Streamed `plan` step output |
| Free-text response | TextInput | Both | Used when the ambiguity doesn't map to discrete options | User input |

### Data Requirements

**Data Needed:**
- The specific ambiguity description and, if applicable, a small set of candidate resolutions — from the `plan` step's output when it flags ambiguity rather than proceeding

**Data Flow:**
1. During a run, the `plan` step determines the question is ambiguous instead of guessing.
2. The Run Viewer's step feed shows this card instead of proceeding to `generate_sql`.
3. User selects an option or types a free-text clarification and submits.
4. The run resumes from `plan` with the clarification incorporated, continuing through `generate_sql` onward as normal.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Awaiting Response** | Ambiguity detected, run paused | Question card with response controls, Submit disabled until answered |
| **Submitting** | Response submitted | Submit button spinner, controls disabled |
| **Resolved** | Run resumed | Card collapses to a compact summary ("Clarified: revenue, not count") within the step feed, run continues below it |

**Edge Cases:**
- User navigates away while a clarification is pending → the run remains paused server-side; returning to [[08-run-viewer|Run Viewer]] later shows the same pending clarification, not a lost or auto-resolved state.
- The same ambiguity recurs on a functionally identical follow-up question in the same context → per [[features]] FEAT-033's acceptance criteria, the system does not re-ask; this screen simply does not appear for that follow-up, and the run proceeds using the previously given clarification.
