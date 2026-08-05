# Features

**Last Updated:** 2026-08-04
**Source:** [[README]], [[user_personas]], [[branding]]

## Overview

Groundwork turns uploaded data into AI-narrated, chart-illustrated, verifiably-grounded answers via a two-stage architecture (model plans SQL, a disposable engine returns facts) wrapped in a layered verification system and an observable, interruptible pipeline. This feature set covers the full loop from file upload through ingest confirmation, question-answering, verification, report generation, export, and the privacy/deletion controls that make the trust story real — serving [[user_personas|Chidinma, Tunde, and Ngozi]] as co-equal, first-class users of the same product.

## Priority Definitions

| Priority | Label | Meaning |
|---|---|---|
| **P0** | Must Have | Critical for launch. The product cannot ship without these. |
| **P1** | Should Have | Important but not launch-blocking. Ship soon after launch. |
| **P2** | Could Have | Nice to have. Adds polish or delight. Ship when capacity allows. |
| **P3** | Won't Have Now | Acknowledged but deferred. Revisit in a future planning cycle. |

## Account & Access

### FEAT-001: Account Creation & Authentication

**Priority:** P0
**Effort:** M
**Depends On:** None

**User Story:**
As a new user (Chidinma, Tunde, or Ngozi), I want to create an account and sign in, so that my workspaces and data are private to me.

**Description:**
Standard account creation and login so a user's workspaces, reports, and semantic layer are scoped to their identity, and so the audit/egress log ([[#FEAT-042 Visible Model-Egress Audit Log|FEAT-042]]) can attribute actions to a real user.

**Acceptance Criteria:**
- [ ] User can create an account with email and password (or equivalent standard auth method)
- [ ] User can sign in and sign out
- [ ] Unauthenticated users cannot access any workspace, report, or account data
- [ ] Password reset flow is available

## Ingest & Workspace

### FEAT-002: Workspace Creation

**Priority:** P0
**Effort:** S
**Depends On:** FEAT-001

**User Story:**
As any user, I want to create a workspace, so that I have a durable container for my data, semantic layer, and reports that I fully own and can delete.

**Description:**
A workspace is the top-level, durable, user-managed object — the unit of deletion and privacy. Users create one before uploading anything.

**Acceptance Criteria:**
- [ ] User can create a named workspace
- [ ] User can view a list of their workspaces
- [ ] Workspace is scoped to the creating user (and any invited members, once [[#FEAT-045 Read-Only Workspace Sharing|FEAT-045]] exists)
- [ ] Creating a workspace does not provision an engine (engine provisioning is deferred to first upload)

### FEAT-003: File Upload (CSV, Excel, Parquet, File-Based OLTP)

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-002

**User Story:**
As Tunde, I want to upload the spreadsheet I already have, so that I don't need to reformat or learn a new tool before asking a question.

**Description:**
Users upload one or more files into a workspace. Supported formats: CSV, Excel, Parquet, and file-based OLTP databases (SQLite and similar). No live database connections are supported.

**Acceptance Criteria:**
- [ ] User can upload CSV, Excel, Parquet, and SQLite files via drag-and-drop or file picker
- [ ] User can upload multiple files into the same workspace in one session
- [ ] Unsupported file types are rejected with a clear, specific error message
- [ ] Upload progress is visible for large files

### FEAT-004: Session-Scoped Engine Provisioning & Ingest

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-003

**User Story:**
As any user, I want my uploaded files loaded into a working analytical engine automatically, so that I can start asking questions without configuring infrastructure.

**Description:**
On upload, the platform provisions a disposable, session-scoped OLAP engine, ingests the uploaded files, and normalizes them into the canonical stored format. This is invisible infrastructure to the user — see [[branding]] for the honest-status principle applied to this in [[#FEAT-011 Engine Sleep/Wake Lifecycle & Honest Status|FEAT-011]].

**Acceptance Criteria:**
- [ ] Uploaded files are ingested into a queryable engine without user configuration
- [ ] Ingest handles files up to a stated size ceiling with a clear error if exceeded
- [ ] Ingested data is normalized and stored durably, independent of the engine's lifecycle
- [ ] Ingest failure surfaces a specific, actionable error (e.g., malformed CSV row, unsupported encoding)

### FEAT-005: Data Profiling

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-004

**User Story:**
As Chidinma, I want the platform to profile my uploaded data automatically, so that type inference, PII flagging, and join inference all have real statistics to work from.

**Description:**
After ingest, the platform computes per-column statistics — distinct value counts, cardinality, min/max, null rates, inferred type — that ground every downstream ingest-confirmation step.

**Acceptance Criteria:**
- [ ] Every ingested column has computed distinct-value count, null rate, and inferred type
- [ ] Profiling completes and is reviewable before the user asks their first question
- [ ] Profiling statistics are stored and retrievable, not recomputed on every use

### FEAT-006: Type Inference Review & Confirmation

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-005

**User Story:**
As Chidinma, I want to review and correct inferred column types before I start querying, so that a wrong type inference doesn't silently produce wrong answers later.

**Description:**
Part of the ingest confirmation step ([[README]] — highest-risk UX). The user reviews inferred types per column and can override them before the semantic layer is seeded.

**Acceptance Criteria:**
- [ ] User sees inferred type for every column with the option to change it
- [ ] Confirmation step has sane defaults and can be skipped without leaving it in an undefined state
- [ ] Type corrections are stored and used for all subsequent ingest of the same schema shape

### FEAT-007: PII Detection & Flagging at Ingest

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-005

**User Story:**
As Tunde, I want the platform to flag columns that might contain sensitive personal data, so that I don't have to know what "PII" means to be protected by default.

**Description:**
At ingest, likely-sensitive columns (emails, national IDs, card patterns, names) are automatically flagged and default to name-and-type-only visibility to the model, per [[README]] privacy commitments.

**Acceptance Criteria:**
- [ ] Columns matching common PII patterns are flagged automatically at ingest
- [ ] Flagged columns default to name-and-type-only exposure to the model
- [ ] User can review flagged columns during ingest confirmation and override the flag
- [ ] Flagging happens without requiring the user to understand the underlying detection method

### FEAT-008: Multi-File Relationship Inference

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-005

**User Story:**
As Chidinma, I want the platform to infer likely joins between my uploaded files, so that I don't have to manually rediscover the same relationships every time a new export lands.

**Description:**
At ingest, when multiple files are uploaded, the platform runs a relationship-inference pass — name similarity, type compatibility, value overlap sampling, cardinality checks — and produces candidate joins with confidence scores.

**Acceptance Criteria:**
- [ ] Candidate joins are generated automatically when 2+ files are uploaded to the same workspace
- [ ] Each candidate join shows a confidence score and the basis for it (e.g., matched column names, value overlap %)
- [ ] No join is silently assumed — every candidate requires explicit confirmation before use in queries

### FEAT-009: Candidate Join Review, Accept, Reject

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-008

**User Story:**
As Ngozi, I want to review and confirm which candidate joins are correct before the platform uses them, so that I catch a wrong relationship before it silently corrupts a reconciliation.

**Description:**
The user-facing half of [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]] — part of the mandatory ingest confirmation flow.

**Acceptance Criteria:**
- [ ] User can accept, reject, or manually adjust each candidate join
- [ ] Accepted joins are seeded into the semantic layer ([[#FEAT-019 Semantic Layer Storage|FEAT-019]])
- [ ] Rejected joins are never silently reintroduced on a later session with the same files

### FEAT-010: Column-Level Exclusion

**Priority:** P1
**Effort:** S
**Depends On:** FEAT-007

**User Story:**
As Chidinma, I want to mark a specific column as never leaving the trust boundary, so that a sensitive field stays queryable in the engine but invisible to the model.

**Description:**
A user-controlled override on top of automatic PII flagging — the column remains usable in SQL execution but its name/values are never sent to the model.

**Acceptance Criteria:**
- [ ] User can mark any column as excluded from model visibility
- [ ] Excluded columns remain queryable by the engine (e.g., usable in a WHERE clause) but never appear in payloads sent to the model
- [ ] Exclusion status is visible wherever the column appears in ingest review or the semantic layer

### FEAT-011: Engine Sleep/Wake Lifecycle & Honest Status

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-004

**User Story:**
As any user, I want to see whether the engine behind my workspace is sleeping, waking, or ready, so that I understand why a question might take a moment instead of assuming the product is broken.

**Description:**
The disposable engine is torn down after idle and rehydrated on demand. Status (`sleeping`/`waking`/`ready`) is always visible, never hidden, per the honest-status principle in [[branding]].

**Acceptance Criteria:**
- [ ] Engine status is visible in the UI at all times using the `sleeping`/`waking`/`ready` status pill defined in [[branding]]
- [ ] Idle engines tear down automatically after a configurable threshold without losing any workspace data
- [ ] A new question submitted while sleeping queues immediately and shows `waking` as the first visible step, never blocking or erroring
- [ ] No "your session expired" warning is ever shown to the user

### FEAT-012: Wake-on-Intent Trigger

**Priority:** P1
**Effort:** S
**Depends On:** FEAT-011

**User Story:**
As Chidinma, I want the engine to start waking as soon as I show intent to ask a question, so that it's likely ready by the time I finish typing.

**Description:**
Rehydration triggers on a signal of intent (e.g., focusing the question input) rather than waiting for submission, minimizing perceived wait time.

**Acceptance Criteria:**
- [ ] Focusing the question input on a sleeping workspace triggers engine wake
- [ ] An unused wake (user doesn't end up asking anything) does not error or leave the workspace in a bad state
- [ ] Wake trigger is tuned and documented against measured cold-start latency

## Core Query & Answer Loop

### FEAT-013: Natural-Language Question → SQL Generation

**Priority:** P0
**Effort:** XL
**Depends On:** FEAT-006, FEAT-009

**User Story:**
As Tunde, I want to ask a question in plain English and get an answer grounded in my actual data, so that I don't have to know SQL or trust an AI that might be making numbers up.

**Description:**
The foundational two-stage mechanism: the model does not answer from data placed in its context. It plans a query from schema/profiling/semantic-layer metadata and writes SQL, which [[#FEAT-014 SQL Execution & Fact Return|FEAT-014]] executes.

**Acceptance Criteria:**
- [ ] User can type a natural-language question against their ingested data
- [ ] The model's response is always backed by generated SQL — there is no code path where the model answers directly from a data dump in context
- [ ] Generated SQL references confirmed types, confirmed joins, and the semantic layer, not raw guesses
- [ ] Ambiguous or unanswerable questions are handled via [[#FEAT-033 Honest Uncertainty|FEAT-033]] rather than a best-effort guess

### FEAT-014: SQL Execution & Fact Return

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-013

**User Story:**
As any user, I want the generated SQL to actually run against my data and return real results, so that every answer is grounded in what's actually in my files.

**Description:**
The engine executes the generated SQL and returns facts (rows, aggregates) that [[#FEAT-015 Narrative Generation|FEAT-015]] and [[#FEAT-016 Chart Spec Generation|FEAT-016]] consume.

**Acceptance Criteria:**
- [ ] Generated SQL executes against the session engine and returns a result set
- [ ] Execution errors (e.g., malformed SQL) are caught and surfaced without crashing the run
- [ ] Query execution scales to the product's stated row-count ceiling without failing

### FEAT-015: Narrative Generation

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-014

**User Story:**
As Tunde, I want a clear written explanation of what the data shows, so that I get an answer I can act on without reading a raw table.

**Description:**
The model narrates only over the facts returned by [[#FEAT-014 SQL Execution & Fact Return|FEAT-014]] — never over data not present in the result set.

**Acceptance Criteria:**
- [ ] Narrative text is generated for every completed query and only references values present in the query's result set
- [ ] Narrative is written in plain language with no SQL or schema terminology required to understand it
- [ ] Every stated number in the narrative is eligible for provenance linking ([[#FEAT-030 Claim-Level Provenance|FEAT-030]])

### FEAT-016: Chart Spec Generation

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-014

**User Story:**
As any user, I want a chart generated automatically when it helps explain the data, so that I can see a trend or comparison at a glance.

**Description:**
The model emits a parsable chart spec (type, encodings, column-to-axis mapping) — never a chart image or freeform markup.

**Acceptance Criteria:**
- [ ] A structured chart spec is produced for query results where a chart adds clarity
- [ ] Chart spec output never includes raw image data or arbitrary markup
- [ ] Chart spec uses the categorical/sequential/diverging color tokens defined in [[branding]]

### FEAT-017: First-Party Chart Rendering

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-016

**User Story:**
As any user, I want charts rendered consistently and interactively, so that they match the rest of the product and I can inspect them.

**Description:**
First-party chart components render the spec from [[#FEAT-016 Chart Spec Generation|FEAT-016]]. This is what makes charts themeable, interactive, and exportable.

**Acceptance Criteria:**
- [ ] Every valid chart spec renders correctly using first-party components
- [ ] Rendered charts support light and dark mode per [[branding]] tokens
- [ ] Rendered charts include an accessible tabular data equivalent per [[branding]] accessibility requirements

### FEAT-018: SQL Visibility & Inline Editing

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-013

**User Story:**
As Chidinma, I want to see and edit the exact SQL behind any answer, so that I can verify it or correct it before I trust the result.

**Description:**
SQL is always visible, never hidden behind an affordance, and directly editable. This is core scope per [[README]], not an advanced/optional mode — Tunde and Ngozi may never open it, but it must always be present for Chidinma.

**Acceptance Criteria:**
- [ ] Generated SQL is visible alongside every answer without requiring an extra click to reveal it
- [ ] User can edit the SQL directly using the code display styling defined in [[branding]]
- [ ] Editing SQL and resuming produces a result using the exact edited query, following the same run mechanics as [[#FEAT-024 Halt, Amend, and Resume Any Step|FEAT-024]]

### FEAT-019: Semantic Layer Storage

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-009

**User Story:**
As Chidinma, I want the platform to remember relationships, grain, and metric definitions across sessions, so that I don't have to re-teach the same business logic every time.

**Description:**
Rows in the application database — relationships, grain, metric definitions, business meaning, accumulated corrections — keyed to the workspace, sitting between raw schema and the model.

**Acceptance Criteria:**
- [ ] Confirmed joins, types, and metric definitions persist in the semantic layer beyond a single session
- [ ] The semantic layer is used as grounding context for every subsequent question in the workspace
- [ ] Semantic layer content is included in workspace deletion ([[#FEAT-043 One-Workspace-One-Delete Cascade|FEAT-043]])

### FEAT-020: Persistent User Corrections

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-019, FEAT-018

**User Story:**
As Chidinma, I want a correction I make to a join or a metric definition to stick, so that I never have to fix the same mistake twice.

**Description:**
When a user edits generated SQL or a semantic-layer definition, that correction is captured as a durable preference rather than a one-off edit.

**Acceptance Criteria:**
- [ ] Editing the same join or filter logic twice in a workspace results in the system applying the corrected version by default on the third occurrence
- [ ] Corrections are attributable to a specific semantic-layer entry, not buried in unstructured history
- [ ] User can view and remove a stored correction

### FEAT-021: Compositional Follow-Up Questions

**Priority:** P1
**Effort:** L
**Depends On:** FEAT-015, FEAT-017

**User Story:**
As Ngozi, I want to ask a follow-up question about a report I just got, so that I can drill deeper without starting over.

**Description:**
Asking a question about an existing report produces a new report, using the same fork-on-amendment mechanism as [[#FEAT-025 Fork-on-Amendment|FEAT-025]].

**Acceptance Criteria:**
- [ ] User can ask a question that references a prior report's context and get a new, linked report
- [ ] The new report records its parent report per [[#FEAT-034 Reports as First-Class Versioned Objects|FEAT-034]]
- [ ] Compositional questions do not require the engine to still hold the original session state in memory

## Observable & Interruptible Pipeline

### FEAT-022: Resumable DAG Run Engine

**Priority:** P0
**Effort:** XL
**Depends On:** FEAT-013

**User Story:**
As Chidinma, I want every step of generating an answer to be resumable, so that I can interrupt, fix, and continue instead of starting over.

**Description:**
Every run (profile → plan → generate_sql → static_check → execute → reconcile → shadow_check → adversarial_review → chart_spec → narrate → assemble) is a persisted, resumable DAG. Nothing lives only in memory.

**Acceptance Criteria:**
- [ ] Every pipeline step's input and output is persisted before the next step begins
- [ ] A run can be resumed from any completed step after a process restart or engine loss
- [ ] Each step is individually retryable without duplicating effects (idempotent)

### FEAT-023: Live Run Event Stream

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-022

**User Story:**
As Chidinma, I want to watch each pipeline step happen live, so that I understand what the system is doing instead of staring at a spinner.

**Description:**
Every pipeline step emits an event the user watches in real time — the front line of both transparency and verification.

**Acceptance Criteria:**
- [ ] User sees each pipeline step (planning, SQL generation, execution, reconciliation, etc.) as it happens, in order
- [ ] Live events are streamed, not polled, and do not require a page refresh
- [ ] Live event stream is visually distinct from durable notifications ([[#FEAT-026 Durable Notifications|FEAT-026]])

### FEAT-024: Halt, Amend, and Resume Any Step

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-022, FEAT-023

**User Story:**
As Chidinma, I want to stop a run mid-way, correct a step, and resume from there, so that I don't have to restart the whole question when only one part was wrong.

**Description:**
Users can halt any in-flight step, amend its input (most commonly, edit the generated SQL), and resume from that point using the same persisted-DAG mechanics.

**Acceptance Criteria:**
- [ ] User can halt a running pipeline at any step
- [ ] User can amend a halted step's input and resume execution from that step forward
- [ ] Resuming after an amendment never re-runs already-completed, unaffected steps

### FEAT-025: Fork-on-Amendment

**Priority:** P1
**Effort:** L
**Depends On:** FEAT-024

**User Story:**
As Chidinma, I want editing a completed report's step to create a new branch instead of overwriting the original, so that I can explore alternatives without losing prior work.

**Description:**
Amending a step on an already-completed run forks it — the original stays intact, the amendment creates a branch, turning report history into a tree.

**Acceptance Criteria:**
- [ ] Amending a completed run creates a new, linked run rather than mutating the original
- [ ] The original run and its report remain fully intact and accessible after a fork
- [ ] Forked runs are visually presented as a branch/tree, not a flat list

### FEAT-026: Durable Notifications

**Priority:** P2
**Effort:** M
**Depends On:** FEAT-023

**User Story:**
As any user, I want to be notified when something finishes outside the current view (like an export or a sleeping engine), so that I don't have to keep a tab open watching it.

**Description:**
A low-frequency, persisted notification system, deliberately separate from the high-frequency live run stream, for events like export-ready, engine-slept, ingest-failed, or workspace-invite.

**Acceptance Criteria:**
- [ ] Durable notifications persist across sessions until acknowledged
- [ ] Durable notifications are not populated from every live run event (no noise from step-level activity)
- [ ] User can view a history of past durable notifications

## Verification & Trust Layers

### FEAT-027: Determinism Caching

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-022

**User Story:**
As Ngozi, I want the same question against the same data to always give the same answer, so that a report doesn't silently change and undermine my trust in it.

**Description:**
Verification Layer 1. Cache by question + schema hash so identical questions against unchanged data return identical results.

**Acceptance Criteria:**
- [ ] Re-asking an identical question against unchanged data returns the identical cached result
- [ ] Cache is invalidated automatically when the underlying schema or data changes
- [ ] Cache hits are distinguishable from fresh runs in the live event stream

### FEAT-028: Static SQL Analysis

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-018

**User Story:**
As Chidinma, I want obviously risky SQL patterns caught before execution, so that I catch a fan-out join or a silent null exclusion before it produces a wrong number.

**Description:**
Verification Layer 2. Pure static analysis, no model call — checks join-grain fan-out, aggregate-over-join without dedup, filters silently dropping nulls, unit/scale mismatches, and suspicious date-range exclusions.

**Acceptance Criteria:**
- [ ] Generated SQL is checked against known risky patterns before execution
- [ ] Detected issues (e.g., non-unique join key causing fan-out) are surfaced to the user before results are shown, not buried after the fact
- [ ] Static checks run without an additional model call

### FEAT-029: Post-Execution Reconciliation

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-014

**User Story:**
As Ngozi, I want to see how many rows were touched, filtered, and grouped at each step, so that I catch a filter silently dropping a chunk of my data.

**Description:**
Verification Layer 3 — the highest-leverage layer, since most wrong answers are wrong by omission rather than by an outright bad query, and omission is mechanically detectable. Every query reports rows in, rows after each filter, and groups out, surfacing significant drops unprompted.

**Acceptance Criteria:**
- [ ] Every executed query reports row counts in and out at each filtering/grouping stage
- [ ] A filter dropping more than a defined threshold of rows triggers an unprompted, explicit callout in the narrative (e.g., "excluded 31% of rows where region was null")
- [ ] Reconciliation data is available for provenance drill-down ([[#FEAT-030 Claim-Level Provenance|FEAT-030]])

### FEAT-030: Claim-Level Provenance

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-015, FEAT-029

**User Story:**
As Tunde, I want to click any number in the narrative and see exactly which data it came from, so that I can verify it before repeating it to someone else.

**Description:**
Verification Layer 6 — non-negotiable, since it is also what prevents the model from inventing numbers in prose that appeared in no result set, which it will otherwise do. Every number in the narrative traces to a specific cell in a specific result set, using the styling defined in [[branding]].

**Acceptance Criteria:**
- [ ] Every numeric claim in generated narrative text is clickable and links to its source query and rows
- [ ] Clicking a claim shows the underlying query and the specific rows/cells that produced the number
- [ ] No number appears in narrative text that cannot be traced to an actual result set (the model cannot state a figure with no backing query)

### FEAT-031: Shadow Queries

**Priority:** P1
**Effort:** M
**Depends On:** FEAT-014

**User Story:**
As Chidinma, I want the platform to independently double-check an aggregate a second way, so that a wrong join grain gets caught even if the SQL runs clean.

**Description:**
Verification Layer 4. For any aggregate, cheaply compute an independent cross-check (e.g., totals summed a second way) and flag disagreement.

**Acceptance Criteria:**
- [ ] Aggregate results are cross-checked against an independently computed shadow query where feasible
- [ ] A disagreement between the primary and shadow result is surfaced to the user, not silently discarded
- [ ] Shadow query computation does not require an additional model call

### FEAT-032: Adversarial Second-Pass Review

**Priority:** P1
**Effort:** M
**Depends On:** FEAT-013

**User Story:**
As Chidinma, I want a second check on whether the generated SQL actually answers my question, so that a semantic mismatch (like a missing status filter) gets caught before I see the result.

**Description:**
Verification Layer 5. A separate model call sees the question and the SQL — not the results — and evaluates whether the SQL answers the question asked.

**Acceptance Criteria:**
- [ ] Generated SQL is reviewed by a separate model pass that does not see query results, only the question and the SQL
- [ ] A detected semantic mismatch (e.g., a missing filter implied by the question) blocks or flags the run before results are shown
- [ ] This layer is clearly the only verification layer requiring an extra model call, and is monitored as a cost/latency line item

### FEAT-033: Honest Uncertainty

**Priority:** P1
**Effort:** M
**Depends On:** FEAT-013

**User Story:**
As any user, I want the platform to ask me for clarification on an ambiguous question instead of guessing, so that I don't get a confidently wrong answer to a question I didn't actually ask.

**Description:**
Verification Layer 7. When a question is ambiguous ("top customers" by revenue or count? over what period?), the system asks rather than assumes.

**Acceptance Criteria:**
- [ ] Questions with a clear ambiguity (e.g., undefined metric or time period) trigger a clarifying question instead of a best-guess answer
- [ ] Clarifying questions are specific to the actual ambiguity, not generic
- [ ] Once clarified, the same ambiguity is not re-asked for functionally identical follow-up questions in the same context

## Reports & Export

### FEAT-034: Reports as First-Class Versioned Objects

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-015, FEAT-017, FEAT-030

**User Story:**
As Ngozi, I want a report to be a durable, shareable, re-runnable thing rather than a chat message, so that I can hand it off, export it, and re-run it against updated data.

**Description:**
A report is a versioned object that knows its run, provenance links, chart spec, and parent report if derived — not chat scrollback.

**Acceptance Criteria:**
- [ ] Every completed answer is stored as a report with a stable ID
- [ ] A report retains its provenance links, chart spec, and parent-report reference (if any) independent of the engine's lifecycle
- [ ] A report is viewable, in full, without the session engine running ([[#FEAT-011 Engine Sleep/Wake Lifecycle & Honest Status|FEAT-011]])

### FEAT-035: PDF Export

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-034

**User Story:**
As Tunde, I want to export a report as a PDF, so that I can email or print it for my accountant or business partner.

**Description:**
The first export target — narrative plus rendered charts, print-oriented.

**Acceptance Criteria:**
- [ ] User can export any report as a PDF including its narrative text and rendered charts
- [ ] Exported PDF preserves chart appearance consistent with the in-app rendering
- [ ] PDF export is available without requiring the session engine to be awake

### FEAT-036: Excel/CSV Export

**Priority:** P1
**Effort:** S
**Depends On:** FEAT-034

**User Story:**
As Ngozi, I want to export the underlying result set as Excel or CSV, so that my team can work with the detailed numbers directly.

**Description:**
Exports the underlying result set(s), not the narrative — analysts and finance teams expect access to the raw rows behind a report.

**Acceptance Criteria:**
- [ ] User can export a report's underlying result set as Excel or CSV
- [ ] Exported data matches exactly what was shown/summarized in the report
- [ ] Export is available per-report, not only for the full workspace

### FEAT-037: PowerPoint Export

**Priority:** P2
**Effort:** M
**Depends On:** FEAT-034

**User Story:**
As Ngozi, I want to export a report as a slide deck, so that I can present it directly to the board without rebuilding it.

**Description:**
Charts and narrative rendered as a deck — high-demand in finance contexts per [[README]].

**Acceptance Criteria:**
- [ ] User can export a report as a slide deck with charts and narrative arranged per slide
- [ ] Exported deck is editable in standard presentation software after export

### FEAT-038: Markdown/HTML Export

**Priority:** P2
**Effort:** S
**Depends On:** FEAT-034

**User Story:**
As Chidinma, I want to export a report as Markdown or HTML, so that I can embed it in an internal wiki or doc.

**Description:**
A lightweight structured export target for embedding into internal documentation systems.

**Acceptance Criteria:**
- [ ] User can export a report as Markdown or HTML
- [ ] Exported format preserves narrative structure and includes chart images or embeddable chart data

### FEAT-039: JSON Export

**Priority:** P2
**Effort:** S
**Depends On:** FEAT-034

**User Story:**
As Chidinma, I want to export a report as structured JSON, so that I can consume it programmatically in another system.

**Description:**
Structured report plus provenance, for programmatic consumers.

**Acceptance Criteria:**
- [ ] User can export a report as JSON including narrative, chart spec, and provenance links
- [ ] JSON export follows a documented, stable schema

### FEAT-040: Single Chart Image Export

**Priority:** P2
**Effort:** S
**Depends On:** FEAT-017

**User Story:**
As Tunde, I want to export just one chart as an image, so that I can paste it directly into an email or message.

**Description:**
A single-chart image export, separate from full-report export.

**Acceptance Criteria:**
- [ ] User can export any individual rendered chart as a standalone image file
- [ ] Exported image matches the in-app chart's appearance and current theme

### FEAT-041: Mobile-Responsive Report Viewing

**Priority:** P1
**Effort:** M
**Depends On:** FEAT-034

**User Story:**
As Tunde and Ngozi, I want to open and read a report on my phone, so that I can check a number right before a meeting without needing my laptop.

**Description:**
Per [[branding]], the mobile responsive view is a first-class viewing surface even though the primary workspace (SQL editing, ingest) is desktop-first.

**Acceptance Criteria:**
- [ ] Reports, including narrative text, charts, and provenance links, render legibly and are fully interactive on a mobile viewport
- [ ] Touch targets on mobile meet the 44×44px minimum defined in [[branding]]
- [ ] Ingest and SQL-editing flows may remain desktop-oriented, but are not broken or inaccessible on mobile

## Privacy & Data Governance

### FEAT-042: Visible Model-Egress / Audit Log

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-001

**User Story:**
As Chidinma, I want to see every payload that was sent to the model, so that the platform's privacy claims are something I can verify, not just trust.

**Description:**
A single object serving as both the model-egress log and the general audit log, named together as "visible model-egress/audit log" in [[README]]'s scope — implemented as one system, not two: who queried what, what left to the model, who exported, who deleted. The dependency is on [[#FEAT-001 Account Creation & Authentication|FEAT-001]], not on question-answering ([[#FEAT-013 Natural-Language Question → SQL Generation|FEAT-013]]): every entry needs a real user identity to attribute the action to (per FEAT-001's own description), but the log itself is the underlying infrastructure that later-arriving action types (`model_call` from FEAT-013 in Phase 2, `export` from FEAT-035 in Phase 4) write into once they exist — it does not need those features built first. `login`, `query_executed`, and `deletion` entries are already meaningful in Phase 1 without FEAT-013.

**Acceptance Criteria:**
- [ ] Every payload sent to the model is logged and viewable by the workspace owner
- [ ] The log records who queried what, what data left to the model, who exported, and who deleted
- [ ] The egress log and general audit log are implemented as a single object, not two separate systems

### FEAT-043: One-Workspace-One-Delete Cascade

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-002, FEAT-019, FEAT-034, FEAT-042

**User Story:**
As any user, I want deleting my workspace to destroy everything derived from it, so that I can trust that "delete" actually means gone.

**Description:**
Deleting a workspace destroys object store contents, DuckDB snapshots, semantic layer rows, profiling statistics, query caches, reports, conversation history, and logs — in a single, provable cascade.

**Acceptance Criteria:**
- [ ] Deleting a workspace removes all associated data, semantic layer content, reports, caches, and logs
- [ ] Deletion is a single cascading operation, not a series of independently-triggered deletions
- [ ] User is shown confirmation of what was deleted and when, as proof of deletion

### FEAT-044: Zero-Retention Provider Terms Notice

**Priority:** P0
**Effort:** S
**Depends On:** FEAT-003

**User Story:**
As Tunde, I want to see plainly, before I upload anything, what happens to my data and what goes to the AI model, so that I can decide whether to trust the platform with my business data.

**Description:**
Stated plainly on the upload screen, per the honest framing in [[README]]: data stays in Groundwork's infrastructure and is deleted on command; descriptions and aggregates go to the model under zero-retention terms.

**Acceptance Criteria:**
- [ ] The upload screen states, in plain language, what stays in Groundwork's infrastructure vs. what may be sent to the model
- [ ] Zero-retention provider terms are stated explicitly, not implied
- [ ] This notice links to the fuller privacy/egress explanation (e.g., linking toward [[#FEAT-042 Visible Model-Egress Audit Log|FEAT-042]])

## Collaboration & Usage

### FEAT-045: Read-Only Workspace Sharing

**Priority:** P1
**Effort:** M
**Depends On:** FEAT-002, FEAT-001

**User Story:**
As Ngozi, I want to share my workspace with a colleague in view-only mode, so that they can see reports and provenance without being able to change my semantic layer.

**Description:**
V1 collaboration scope per [[README]]: viewers see reports and provenance; they cannot mutate the semantic layer. Concurrent semantic-layer editing is explicitly out of scope.

**Acceptance Criteria:**
- [ ] Workspace owner can invite another user with read-only access
- [ ] Read-only members can view reports, charts, and provenance but cannot edit the semantic layer, upload files, or run new queries that mutate state
- [ ] Read-only access can be revoked by the workspace owner at any time

### FEAT-046: Read-Only Report Sharing by ID

**Priority:** P1
**Effort:** S
**Depends On:** FEAT-034

**User Story:**
As Ngozi, I want to share a single report by link, so that a stakeholder outside my workspace membership can see one specific analysis without full workspace access.

**Description:**
Sharing a single report by ID — likely the more valuable half of v1 collaboration per [[README]], since it's how analysis spreads through an organization.

**Acceptance Criteria:**
- [ ] Any report can be shared via a stable link/ID
- [ ] Recipients can view the report, its charts, and its provenance without needing a full workspace invite
- [ ] Report owner can revoke a share link

### FEAT-047: User-Facing Usage View

**Priority:** P2
**Effort:** M
**Depends On:** FEAT-001

**User Story:**
As Ngozi, I want to see how much engine time, storage, and model usage my workspace has consumed, so that I understand what drives cost before it becomes a billing conversation.

**Description:**
A user-facing view over the metering data (engine-seconds, object-store bytes, model tokens) collected from day one per [[README]], surfaced for transparency even before formal billing exists.

**Acceptance Criteria:**
- [ ] User can view engine-seconds, storage bytes, and model token usage for their workspace(s)
- [ ] Usage data updates on a reasonable, stated cadence (does not need to be real-time)
- [ ] Usage view requires no configuration to access — available by default to any account holder

## Priority Summary

| Priority | Count | Feature IDs |
|---|---|---|
| P0 | 30 | FEAT-001, FEAT-002, FEAT-003, FEAT-004, FEAT-005, FEAT-006, FEAT-007, FEAT-008, FEAT-009, FEAT-011, FEAT-013, FEAT-014, FEAT-015, FEAT-016, FEAT-017, FEAT-018, FEAT-019, FEAT-020, FEAT-022, FEAT-023, FEAT-024, FEAT-027, FEAT-028, FEAT-029, FEAT-030, FEAT-034, FEAT-035, FEAT-042, FEAT-043, FEAT-044 |
| P1 | 11 | FEAT-010, FEAT-012, FEAT-021, FEAT-025, FEAT-031, FEAT-032, FEAT-033, FEAT-036, FEAT-041, FEAT-045, FEAT-046 |
| P2 | 6 | FEAT-026, FEAT-037, FEAT-038, FEAT-039, FEAT-040, FEAT-047 |
| P3 | 0 | None |

**Integrity checks:** All `Depends On:` IDs resolve · dependency graph is acyclic (verified by walking each chain: ingest → core loop → pipeline/verification → reports → privacy/collaboration, with no back-references) · no P0 depends on a lower priority (FEAT-012, FEAT-021, FEAT-025, FEAT-041, FEAT-045 are P1 features depending on P0 features, which is valid; no P0 feature lists a P1/P2/P3 dependency) · every feature traces to a persona goal or frustration in [[user_personas]] (including shared-infrastructure features with no single named persona in their user story — FEAT-001, FEAT-002, FEAT-004, FEAT-011, FEAT-014, FEAT-016, FEAT-017, FEAT-026, FEAT-033, FEAT-043 — each still traces to a goal shared by all three personas, e.g. FEAT-033's honest-uncertainty behavior is a stated goal of every persona in [[user_personas]], not just whoever's user story happened to be written against it) · every persona (Chidinma, Tunde, Ngozi) is served by 5+ features each, verified against the literal "As `<persona>`" line in each feature above — Chidinma: FEAT-005, FEAT-006, FEAT-008, FEAT-010, FEAT-012, FEAT-018, FEAT-019, FEAT-020, FEAT-022, FEAT-023, FEAT-024, FEAT-025, FEAT-028, FEAT-031, FEAT-032, FEAT-038, FEAT-039, FEAT-042; Tunde: FEAT-003, FEAT-007, FEAT-013, FEAT-015, FEAT-030, FEAT-035, FEAT-040, FEAT-041, FEAT-044; Ngozi: FEAT-009, FEAT-021, FEAT-027, FEAT-029, FEAT-034, FEAT-036, FEAT-037, FEAT-041, FEAT-045, FEAT-046, FEAT-047
