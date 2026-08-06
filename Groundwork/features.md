# Features

**Last Updated:** 2026-08-06
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

**Ordering convention:** within this document, `### FEAT-NNN` headings run in strict ascending ID order from top to bottom, full stop — `##` sections group them thematically, but grouping never overrides ID order. A feature scoped after the sections below were first written (as FEAT-048 and FEAT-049 were) is appended at the end of the document, in its own thematically-named section if it doesn't fit an existing one, rather than inserted into an earlier section out of ID order. Depending on an earlier feature by ID is always fine regardless of section distance; what's not fine is a heading appearing out of numeric sequence. See Dataset Lifecycle Management below for the section this produced.

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
After ingest, the platform computes per-column statistics — distinct value counts, cardinality, null rates, inferred type — that ground every downstream ingest-confirmation step. Min/max are deliberately not computed or stored in Phase 1 — no acceptance criterion here or in [[schema]]'s `Column` needs them yet; add them as a field on `Column` (owned by this feature) if and when a later feature's acceptance criteria actually requires range statistics, rather than computing them speculatively now.

`type_confidence` (per [[schema]] `Column`) is shown to the user as one of three bands — **HIGH** (≥0.85), **MEDIUM** (0.6–0.84), **LOW** (<0.6) — and those bands, plus what each means to the user ("HIGH: this type fits the data well," "MEDIUM/LOW: a meaningful share of values won't cleanly fit — review before trusting"), are the specified, user-facing contract.

**What the number measures.** `type_confidence` always describes the **currently selected type for the column** — the user's override if one is set on [[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]]'s review step, otherwise the inferred type. It is never a description of the original inference once an override exists; a badge frozen to the inference the user just rejected is exactly the silent-wrongness failure [[README]] names as the product's differentiator, on the one screen built to prevent it. Because the number can describe any of the six candidate types depending on what's currently selected, and switching the selection must not trigger a fresh engine query (see below), profiling computes a **fit rate for every candidate type, not just the winning one** — stored per [[schema]] `Column` as `type_fit_rates`. `type_confidence` is then a cheap read-time lookup into `type_fit_rates` for whichever type is currently active; it is not an independently-set value and never goes stale, because there is nothing to go stale — there's no second write to forget.

The heuristic behind each per-type fit rate is specified, not left implementation-defined, because a heuristic that always lands HIGH regardless of wording defeats the point of showing a confidence signal at all: it must measure **agreement between the sampled non-null values and that specific candidate type**, not simply which type the sniffer picked.
- For **boolean, date, timestamp, integer, number**: the fraction of sampled non-null values that successfully coerce to that type without loss (e.g. `"12.5"` coerces cleanly to `number`; `"12.5kg"` does not). A column that's mostly clean numbers with a handful of malformed values scores below HIGH on `number` — correctly, since that's exactly the case worth a second look, and exactly the case that must stay visible after the user overrides `text` → `number` rather than reverting to a reassuring number computed against the type they no longer want.
- For **string** (the fallback type, chosen when no stricter type fits confidently, and always selectable as an override): the fraction of sampled non-null values that do **not** also match one of the stricter type patterns (boolean/integer/number/date/timestamp) tested and rejected. A column of genuinely free-text values scores HIGH, because none of those patterns match anything. A column that's mostly numeric-looking with a handful of stray text values scores lower on `string`, because the sniffer had real, unresolved ambiguity about whether it should have been a stricter type — that ambiguity is exactly what a reviewer needs surfaced. This is a structurally different kind of "fit" than the coercion rates above (absence of a stricter match, not success of a cast) and the badge does not distinguish the two mechanisms in its wording — "% of values fit this type" reads correctly for both.

Each per-type coercion/agreement rate is then weakened by the column's null rate as before (a mostly-empty column gave the sniffer little to work with), with a floor. Everything else — sample size, exact coercion rules, floor value — remains implementation-defined and tunable without a docs change; only the definitions above (what each per-type number measures), the fact that `type_confidence` always reflects the currently selected type, and the band thresholds/meanings are the fixed, specified contract.

**Cost.** All six per-type fit rates are computed together, once, during profiling — additional `FILTER` clauses inside the same combined aggregate query profiling already runs (per [[architecture]]), not additional round trips or a second query. Selecting a different type afterward (via override) is a lookup into already-computed data, not a recomputation, so it never wakes a sleeping engine — consistent with [[#FEAT-011 Engine Sleep/Wake Lifecycle & Honest Status|FEAT-011]]'s honest-idle behavior. This is a schema change (`Column` gains `type_fit_rates`) as well as a scoring-behavior change; see [[schema]] `Column`.

**Label.** The user-facing label for this badge reads along the lines of "**N% of values fit this type**," not "N% confident" — the number is a measured fit rate against a specific, currently-visible type, not an abstract confidence score, and the wording must make that unambiguous regardless of whether the type shown is the original inference or a user override.

**Acceptance Criteria:**
- [ ] Every ingested column has computed distinct-value count, null rate, inferred type, and a fit rate for every candidate type (`type_fit_rates`)
- [ ] Profiling completes and is reviewable before the user asks their first question
- [ ] Profiling statistics, including all per-type fit rates, are stored and retrievable, not recomputed on every use — computing a fit rate for a newly-selected type after an override is a lookup into already-stored data, never a fresh engine query
- [ ] `type_confidence` always reflects the fit rate of the column's currently selected type (override if set, otherwise inferred type), and is presented to the user using the HIGH/MEDIUM/LOW bands defined above with the "N% of values fit this type" label, not a raw decimal alone and not "N% confident"

### FEAT-006: Type Inference Review & Confirmation

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-005

**User Story:**
As Chidinma, I want to review and correct inferred column types before I start querying, so that a wrong type inference doesn't silently produce wrong answers later.

**Description:**
Part of the ingest confirmation step ([[README]] — highest-risk UX). The user reviews inferred types per column and can override them before the semantic layer is seeded. In Phase 1, a type correction is a per-`Column` override: it applies to that column for the life of that ingested [[schema|Dataset]], and is used by every subsequent question against that dataset. Automatically recognizing "the same schema shape" on a *later, separate* file upload (e.g. re-uploading the same monthly export) and replaying prior corrections onto the fresh columns is out of scope here — no schema-shape identity (what a "shape" hash covers: column names, order, and inferred types, at minimum) is defined anywhere yet, and inventing one is a real design decision, not a one-line fix. That capability is deferred to [[#FEAT-019 Semantic Layer Storage|FEAT-019]] (Phase 2), which is already the workspace-level, cross-session memory this behavior conceptually belongs in; see FEAT-019's description for the deferred note.

**Overriding a type updates the confidence badge, not just the dropdown.** The `type_confidence` badge shown beside the override control (per [[#FEAT-005 Data Profiling|FEAT-005]]) always describes the currently selected type, not the original inference. When the user changes the override dropdown, the badge updates immediately — in the same interaction, no loading state — to show the precomputed fit rate for the newly selected type (a lookup into [[schema]] `Column.type_fit_rates`, not a fresh computation, per FEAT-005). This is a deliberate safety requirement, not a display nicety: showing the inferred type's confidence next to an overridden type would read as reassurance about a choice it never measured. A low fit rate after an override is itself the warning — the product deliberately does not add a second, separate warning indicator for this case; the badge and its label are the whole mechanism.

**Acceptance Criteria:**
- [ ] User sees inferred type for every column with the option to change it
- [ ] Confirmation step has sane defaults and can be skipped without leaving it in an undefined state
- [ ] A type correction is stored on the corrected `Column` and used for every subsequent question against that same ingested dataset (not replayed onto a later, separately-uploaded file, even one with an identical schema — see description)
- [ ] Changing a column's type override updates its `type_confidence` badge in the same interaction, to the fit rate of the newly selected type — never left showing the fit rate of the previously selected or originally inferred type
- [ ] The badge's label reads "N% of values fit this type" (or equivalent unambiguous wording), never "N% confident," so the number cannot be misread as endorsing whichever type happens to be shown

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
- [ ] Flagged columns default to name-and-type-only exposure to the model — enforced by the payload-construction rule specified in [[#FEAT-013 Natural-Language Question → SQL Generation|FEAT-013]], since FEAT-013 is the first feature that builds a model payload
- [ ] User can review flagged columns during ingest confirmation and override the flag
- [ ] Overriding (or explicitly confirming) a PII flag is recorded as a timestamped, explicit user decision — `Column.pii_reviewed_at` per [[schema]] — and logged as a `pii_override` entry in the audit log, so a column that was reviewed-and-cleared is distinguishable later from one that was simply never flagged
- [ ] Flagging happens without requiring the user to understand the underlying detection method

### FEAT-008: Multi-File Relationship Inference

**Priority:** P0
**Effort:** L
**Depends On:** FEAT-005

**User Story:**
As Chidinma, I want the platform to infer likely joins between my uploaded files, so that I don't have to manually rediscover the same relationships every time a new export lands.

**Description:**
At ingest, when multiple files are uploaded, the platform runs a relationship-inference pass — name similarity, type compatibility, value overlap sampling, cardinality checks — and produces candidate joins with confidence scores.

**Confidence Scoring:** `confidence_score` (per [[schema]] `CandidateJoin`) is presented to the user using the same HIGH (≥0.85) / MEDIUM (0.6–0.84) / LOW (<0.6) bands as `type_confidence` in [[#FEAT-005 Data Profiling|FEAT-005]], and the score must be informed by at least these signals: column-name similarity, type compatibility, value-overlap sampling, and a cardinality/uniqueness check on the candidate key. The exact weighting between these signals is implementation-defined and may be tuned without a docs change — but one presentation-facing rule is a hard requirement, not implementation-defined: **a candidate join whose only meaningful evidence is value overlap on a column that is not identifier-shaped must be capped at MEDIUM, never reported as HIGH.**

"Identifier-shaped" is a **name-based** test, checked before uniqueness is ever consulted, and it has two independent branches that can each exempt a column from the cap:
- **Identifier-named:** the column name matches `id`/`key`/`code` as a whole name or a `_`-delimited affix (e.g. `customer_id`, `order_key`, `product_code` all match; `paid`, `void`, `residual` do not).
- **Near-unique:** the column's `distinct_count`/`row_count` ratio is ≥0.95, **and** the column's name does not match a measure-shaped name (below). Uniqueness alone is no longer sufficient — a column can be numerically near-unique by coincidence (e.g. every invoice happens to have a distinct total) without being an identifier.

**Measure-shaped names defeat the near-unique exemption, full stop.** A column whose name matches `total`, `amount`, `price`, `qty`, `quantity`, `balance`, `sum`, `cost`, `revenue`, `fee`, `tax`, or `discount` — as a whole name or a `_`-delimited affix, using the same whole-name/affix matching discipline as the identifier-named check above (so `paid`, `void`, `residual`, `taxonomy` do not match) — cannot claim the near-unique exemption regardless of its actual distinctness. If a measure-shaped-named column is the join's only meaningful evidence, the cap applies even at 100% value-overlap and 100% uniqueness on that side. This closes the gap where a distinct-valued money column (invoice totals, transaction amounts) scored HIGH on uniqueness alone despite being exactly the case the cap was written for.

Two unrelated numeric measure columns overlapping in value by chance is common and is not, on its own, evidence of a real relationship. When this cap is applied, `basis` must say so explicitly (e.g., "capped: value overlap only, not an identifier column" or, when the measure-name rule specifically is why the near-unique exemption didn't apply, "capped: measure-shaped name (`total`), near-unique exemption does not apply").

**Same-filename datasets are never compared.** Two datasets in the same workspace with the same `original_filename` are excluded from relationship inference entirely — no candidate join is generated between any column of one and any column of the other. This is not a confidence adjustment; the pair is skipped before scoring. Two datasets sharing a name are either literal copies of each other (the case [[#FEAT-049 Duplicate Filename Detection & Resolution at Upload|FEAT-049]]'s upload-time prompt exists to prevent from accumulating unprompted) or, in the deliberate "keep both" case FEAT-049 allows, two distinct files a user has already told the product are separate — either way, comparing a dataset against its own likely duplicate is never a meaningful join candidate, and was the direct mechanical cause of a real incident where 4 copies of one file produced 216 candidate joins among themselves. This rule is defense in depth: it holds regardless of whether duplicate accumulation is otherwise prevented upstream, and it is deliberately unconditional — it is not narrowed to skip only when the pair wasn't a "Keep both" choice, both because there is no reliable signal at inference time to distinguish that case from a Replace still mid-flight, and because narrowing it would reopen the quadratic-candidate-join explosion this rule exists to close.

**The skip is surfaced, not silent.** Because "Keep both" ([[#FEAT-049 Duplicate Filename Detection & Resolution at Upload|FEAT-049]]) exists precisely so two genuinely different, identically-named files can coexist, that pair is the one case this exclusion actually affects in practice — and leaving it unexplained would tell the user nothing about why two files they deliberately kept separate show no relationship candidates at all. When 2+ datasets in an ingest batch (or already present in the workspace and included in relationship inference) share an `original_filename`, [[screens/07-ingest-confirmation|Ingest Confirmation]]'s Relationships step shows an explicit notice naming the affected dataset pair(s) (by `original_filename` and disambiguating `created_at`) and stating that relationship inference was skipped between them because they share a filename — with a pointer to add a relationship manually on [[screens/15-semantic-layer|Semantic Layer / Corrections]], which [[schema]] `Relationship` already supports independent of `CandidateJoin` acceptance. Single-file uploads, or a workspace with no filename collisions, show no such notice.

**Acceptance Criteria:**
- [ ] Candidate joins are generated automatically when 2+ files are uploaded to the same workspace
- [ ] Each candidate join shows a confidence score — presented via the HIGH/MEDIUM/LOW bands above, not a raw decimal alone — and the basis for it (e.g., matched column names, value overlap %)
- [ ] A candidate join whose only evidence is value overlap on a non-identifier-shaped numeric column is capped at MEDIUM and its `basis` states the cap was applied
- [ ] A column whose name matches the measure-shaped name list (`total`, `amount`, `price`, `qty`, `quantity`, `balance`, `sum`, `cost`, `revenue`, `fee`, `tax`, `discount` — whole name or `_`-delimited affix) cannot claim the near-unique exemption, even at ≥0.95 uniqueness; a join relying on such a column as its only evidence is capped at MEDIUM
- [ ] No candidate join is ever generated between two datasets in the same workspace that share the same `original_filename`
- [ ] When 2+ datasets sharing an `original_filename` are part of the relationship-inference pass, [[screens/07-ingest-confirmation|Ingest Confirmation]]'s Relationships step shows an explicit notice identifying the pair and stating inference was skipped between them because they share a filename, rather than silently showing no candidates for that pair
- [ ] No join is silently assumed — every candidate requires explicit confirmation before use in queries

### FEAT-009: Candidate Join Review, Accept, Reject

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-008

**User Story:**
As Ngozi, I want to review and confirm which candidate joins are correct before the platform uses them, so that I catch a wrong relationship before it silently corrupts a reconciliation.

**Description:**
The user-facing half of [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]] — part of the mandatory ingest confirmation flow. "The same files," for the purpose of never silently reintroducing a rejected join, means the identity **`(workspace, Dataset.original_filename, Column.name)`** — a rejected candidate is suppressed on a later ingest by matching against prior `rejected` `CandidateJoin` rows in the same workspace whose source/target column names and parent dataset filenames match, with no new persisted entity required. This is a deliberate, documented heuristic, not a placeholder: it has two known, accepted limits — renaming a file loses the rejection (the user would see the candidate again and can reject it again), and two genuinely different files that happen to share both a filename and a column name would wrongly suppress a candidate that deserved fresh review. A content-hash-based identity (keyed to the dataset's actual bytes rather than its filename) would close both gaps and is a candidate future improvement if either limit is observed to matter in practice — it is not scoped as a Phase 1 acceptance criterion. This is confirmed as the Phase 1 behavior, not a placeholder awaiting a stronger design. It shares a root question with [[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]]'s deferred cross-ingest correction reuse — both ultimately want a notion of "the same file across ingests" stronger than filename matching. Neither invents one now; if a content-hash (or schema-shape) identity is ever built for one, it should be evaluated for reuse by the other rather than designed twice.

This "same filename" identity is also, separately, what [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]] now uses to skip generating candidate joins between two datasets that are likely copies of each other, and what [[#FEAT-049 Duplicate Filename Detection & Resolution at Upload|FEAT-049]] uses to prompt at upload time before a duplicate dataset is created at all. Those two features prevent the accumulation this feature's suppression heuristic was never meant to handle; this feature's scope remains exactly as described above — never re-suggesting a rejected join for the same file pair.

**Acceptance Criteria:**
- [ ] User can accept, reject, or manually adjust each candidate join
- [ ] Accepted joins are seeded into the semantic layer ([[#FEAT-019 Semantic Layer Storage|FEAT-019]])
- [ ] Rejected joins are never silently reintroduced on a later session with the same files, where "same files" is matched on `(workspace, original filename, column name)` per this feature's description, with its stated limits accepted as intended behavior

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

This is the first feature to construct any payload sent to the model, which makes it the enforcement point for [[#FEAT-007 PII Detection & Flagging at Ingest|FEAT-007]]'s exposure rule: **payload construction MUST read `Column.pii_flag` (and `excluded_from_model`, per [[#FEAT-010 Column-Level Exclusion|FEAT-010]]) and reduce any flagged or excluded column to name-and-type only before it can reach the model — never its values.** This MUST be implemented as one shared payload-construction function used by every model-calling step (`plan`, `generate_sql`, and later `adversarial_review`/`narrate`), not per-step ad hoc filtering, precisely so a future step can't forget to filter — see [[architecture]]'s note on this seam. See [[risk_log]] R-016 for the risk this closes.

**Acceptance Criteria:**
- [ ] User can type a natural-language question against their ingested data
- [ ] The model's response is always backed by generated SQL — there is no code path where the model answers directly from a data dump in context
- [ ] Generated SQL references confirmed types, confirmed joins, and the semantic layer, not raw guesses
- [ ] Every payload sent to the model is built through the single shared payload-construction function described above; no column with `pii_flag: true` or `excluded_from_model: true` ever has its values (as opposed to its name and type) included in a model payload
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
Rows in the application database — relationships, grain, metric definitions, business meaning, accumulated corrections — keyed to the workspace, sitting between raw schema and the model. A future extension of this store — not yet a scoped acceptance criterion, and not yet designed — is recognizing a re-uploaded file as "the same schema shape" as a prior ingest and automatically replaying its stored type corrections onto the fresh columns, deferred here from [[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]] (see FEAT-006's description). Scoping this properly requires defining a schema-shape identity (at minimum: column names, order, and inferred types) before it becomes a feature commitment, not an ingest-time speculation.

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

## Dataset Lifecycle Management

Both features below were scoped after a real incident during use of the shipped Phase 1 ingest flow (see each feature's description), later than every section above. They read most naturally beside Ingest & Workspace — deleting and de-duplicating datasets is squarely ingest-adjacent work — but per this document's ordering convention (see Priority Definitions above), a feature scoped after the document was first written is appended here, in its own section, rather than inserted earlier out of ID order.

### FEAT-048: Single-Dataset Deletion

**Priority:** P0
**Effort:** S
**Depends On:** FEAT-002, FEAT-042

**User Story:**
As Chidinma, I want to delete a single dataset from my workspace, so that I can remove a failed or unwanted upload without destroying everything else I've built in the workspace.

**Description:**
Before this feature, the product offered no deletion narrower than [[#FEAT-043 One-Workspace-One-Delete Cascade|FEAT-043]]'s full-workspace cascade — itself Phase 4. That gap was severe enough in practice to force a user whose workspace had accumulated duplicate datasets to have their entire workspace destroyed by hand, directly against the database and object storage, rather than remove the individual datasets responsible. This feature closes that gap at Phase 1, since it is a precondition for [[#FEAT-049 Duplicate Filename Detection & Resolution at Upload|FEAT-049]]'s "Replace" action and the everyday-use complement a data-privacy product cannot credibly ship without, per [[README]]'s deletion differentiator.

Deleting a single `Dataset` removes that dataset's row, its `Column` rows, and any `CandidateJoin`/`Relationship` rows that reference those columns. It also removes that dataset's objects from object storage — the original uploaded file and the normalized Parquet file — explicitly, in application code, not by relying on a foreign-key cascade: object storage has no foreign keys to the application database and does not cascade on its own (this is a confirmed operational fact, not a theoretical concern — see [[#FEAT-049 Duplicate Filename Detection & Resolution at Upload|FEAT-049]]'s description). A dataset deletion that only removes database rows and leaves orphaned objects in R2 would quietly break the same provable-deletion promise this feature exists to uphold at dataset scope, and that [[#FEAT-043 One-Workspace-One-Delete Cascade|FEAT-043]] makes at workspace scope.

This is deliberately narrower than FEAT-043: it does not touch reports, runs, or semantic-layer entries that don't reference the deleted dataset's columns (a `MetricDefinition` or `Correction` with no dependency on the deleted dataset is left alone). If a `Relationship` or `MetricDefinition` does depend on a column being deleted, the delete confirmation must say so before the user commits, rather than silently orphaning a semantic-layer entry.

**Deleting a dataset also discards its own columns' reviewed state, not just downstream dependents.** Removing a `Column` row removes whatever [[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]] type override and [[#FEAT-007 PII Detection & Flagging at Ingest|FEAT-007]] PII review decision (`pii_flag`, `pii_reviewed_at`) were recorded on it — a user's own review work on that dataset, not merely a semantic-layer entry that happened to reference it. That loss is inherent to deleting the row and is not a bug, but it must never be *silent*: the confirmation step (the shared Delete Confirmation Modal) must name it explicitly, alongside any dependent `Relationship`/`MetricDefinition`, before the user commits — the same disclosure discipline FEAT-043 applies at workspace scope. Phase 1 has no mechanism to carry those decisions forward onto a future re-upload of the same data; replaying them onto a schema-shape match is deferred to [[#FEAT-019 Semantic Layer Storage|FEAT-019]] (Phase 2), per FEAT-006's description — this feature's job is to make sure the user is told what they're losing, not to prevent the loss.

**Acceptance Criteria:**
- [ ] User can delete a single dataset from within the workspace (surfaced on [[screens/04-workspace-home|Workspace Home]])
- [ ] Deleting a dataset removes its `Column` rows and any `CandidateJoin`/`Relationship` rows referencing those columns
- [ ] Deleting a dataset removes both its original uploaded object and its normalized Parquet object from object storage, via an explicit delete call — never assumed to cascade automatically
- [ ] If any semantic-layer entry (e.g., a `Relationship` or `MetricDefinition`) depends on the dataset being deleted, the confirmation step states this before the user commits, using the shared Delete Confirmation Modal pattern
- [ ] If the dataset has any columns carrying a type override ([[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]]) or a reviewed PII decision (`pii_reviewed_at` set, per [[#FEAT-007 PII Detection & Flagging at Ingest|FEAT-007]]), the confirmation step states that this review work will be permanently lost and is not replayed onto any future re-upload, before the user commits
- [ ] The deletion is recorded as a `deletion` [[schema|AuditLogEntry]], consistent with [[#FEAT-042 Visible Model-Egress Audit Log|FEAT-042]]

### FEAT-049: Duplicate Filename Detection & Resolution at Upload

**Priority:** P0
**Effort:** M
**Depends On:** FEAT-003, FEAT-048

**User Story:**
As Tunde, I want the platform to tell me when I'm uploading a file with the same name as one already in my workspace, so that re-uploading a corrected spreadsheet doesn't silently multiply my data instead of fixing it.

**Description:**
Observed in real use: a bug (since fixed) made the ingest progress screen report failure for uploads that had actually succeeded, so the user retried, and each retry created a new `Dataset` row with the same `original_filename`. The end state was 5 `Dataset` rows named identically, 64 `Column` rows, and 216 `CandidateJoin` rows from [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]] comparing every copy's columns against every other copy's — [[screens/07-ingest-confirmation|Ingest Confirmation]]'s Types step listed the same columns four indistinguishable times over, and there was no way to remove any of it short of destroying the whole workspace by hand. The trigger was a bug, but the accumulation itself is not bug-dependent — uploading a corrected version of a file (a fixed spreadsheet, next month's export, a re-save after removing a bad row) is an entirely ordinary action that hits this exact path.

This is a narrower, and deliberately different, question than the one [[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]] and [[#FEAT-009 Candidate Join Review, Accept, Reject|FEAT-009]] already circled and declined to resolve — both of those defer *remembering decisions* ("is this the same schema shape," "is this the same file for join-rejection purposes") across separate ingests, which degrades gracefully when the heuristic misses: worst case, the user re-reviews something they'd already reviewed. This feature is about *data accumulating* in a way that is not graceful when it misses — it visibly breaks the ingest confirmation screen, multiplies relationship-inference work quadratically in the number of copies, and, before [[#FEAT-048 Single-Dataset Deletion|FEAT-048]], could not be undone by the user at all. That difference is why this feature does not defer again: it does not need a strong notion of "the same file" (content-hash identity, schema-shape identity) to be worth building — it only needs to notice an exact `original_filename` match within the same workspace, which is cheap, unambiguous, and already the identity FEAT-009 uses for its own, narrower purpose.

**Behavior.** When a file is uploaded whose `original_filename` exactly matches an existing `Dataset` in the same workspace — regardless of that dataset's `ingest_status`, including `failed` — the upload does not proceed straight to ingest. The user is shown an explicit choice before ingest begins:
- **Replace** (default selection): the prior dataset is deleted using [[#FEAT-048 Single-Dataset Deletion|FEAT-048]]'s mechanism (including its object-storage cleanup), then the new file is ingested as a fresh dataset. This is the expected path for the ordinary "I fixed the file and re-uploaded it" case, and for the retry-after-a-falsely-reported-failure case that caused the original incident — a `failed` dataset should never survive a retry silently. **This is also the ordinary path by which a user's prior type overrides ([[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]]) and PII review decisions ([[#FEAT-007 PII Detection & Flagging at Ingest|FEAT-007]]) on that dataset get destroyed** — deleting the prior dataset deletes its `Column` rows, and Phase 1 has no schema-shape identity to replay those decisions onto the new file's fresh columns (deferred to [[#FEAT-019 Semantic Layer Storage|FEAT-019]], per FEAT-006's description). Because Replace is the pre-selected option on the screen the source spec calls the product's highest-risk UX, this dialog must state that loss explicitly — using FEAT-048's confirmation disclosure — before the user commits, not just describe it as "the prior dataset will be deleted." The alternative of silently carrying decisions forward was considered and rejected: no schema-shape identity is defined anywhere in Phase 1, and inventing one here would quietly reverse a deferral decision [[#FEAT-006 Type Inference Review & Confirmation|FEAT-006]] and FEAT-019 have each already made deliberately. Naming the loss, rather than preventing it, is the Phase 1 answer.
- **Keep both**: the new file is ingested as a distinct `Dataset`. Both datasets keep their true `original_filename` unmodified (provenance is never rewritten), and are disambiguated in every review surface by their `created_at` timestamp, which [[schema]] `Dataset` already carries — no new schema field is needed. This path exists for the legitimate but rarer case of two genuinely different files that happen to share a name.

The choice always requires an explicit user action — there is no silent default that proceeds without the prompt being shown, even though "Replace" is the pre-selected option.

**Interaction with FEAT-008.** [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]] does not generate candidate joins between two datasets that share the same `original_filename` within a workspace (see FEAT-008's description) — this closes the 216-`CandidateJoin` explosion mechanism directly, as defense in depth independent of whether duplicate accumulation is otherwise prevented, since "Keep both" deliberately allows same-named datasets to coexist. That exclusion is unconditional — it is not narrowed to only apply when "Keep both" was chosen, because by the time relationship inference runs, a same-filename pair reaching it is always either two "Keep both" survivors or (in a race or retry edge case) a Replace that hasn't finished deleting yet; there is no reliable signal at inference time to tell those apart, and narrowing the check would reopen the exact explosion path it exists to close. Because "Keep both" exists specifically to let two genuinely different, identically-named files coexist, silently skipping their inference would leave the one case this option serves with no join candidates and no explanation. [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]]'s Ingest Confirmation Relationships step therefore surfaces the skip explicitly (see FEAT-008's acceptance criteria) rather than leaving it invisible; the user's recourse is the same manual relationship creation [[schema]] `Relationship` already supports on [[screens/15-semantic-layer|Semantic Layer / Corrections]].

**Acceptance Criteria:**
- [ ] Uploading a file whose `original_filename` matches an existing `Dataset` (any `ingest_status`, including `failed`) in the same workspace triggers a required resolution prompt before ingest begins — never silently creates an indistinguishable duplicate
- [ ] Choosing Replace deletes the prior dataset via [[#FEAT-048 Single-Dataset Deletion|FEAT-048]] (database rows and object storage) before the new file is ingested, and the confirmation states, before the user commits, that any type overrides and PII review decisions recorded on the prior dataset's columns will be lost and are not carried forward to the new file
- [ ] Choosing "Keep both" ingests the new file as a distinct dataset; both datasets retain their unmodified `original_filename` and are shown with their `created_at` timestamp everywhere they'd otherwise be indistinguishable, including [[screens/07-ingest-confirmation|Ingest Confirmation]]
- [ ] A dataset stuck at `ingest_status: failed` is handled by this same prompt on retry, not a special case
- [ ] No candidate join is generated between two datasets sharing the same `original_filename` (enforced in [[#FEAT-008 Multi-File Relationship Inference|FEAT-008]]), and when this applies to a "Keep both" pair, [[screens/07-ingest-confirmation|Ingest Confirmation]]'s Relationships step states that inference was skipped between them and why (enforced in FEAT-008's acceptance criteria)

## Priority Summary

| Priority | Count | Feature IDs |
|---|---|---|
| P0 | 32 | FEAT-001, FEAT-002, FEAT-003, FEAT-004, FEAT-005, FEAT-006, FEAT-007, FEAT-008, FEAT-009, FEAT-011, FEAT-013, FEAT-014, FEAT-015, FEAT-016, FEAT-017, FEAT-018, FEAT-019, FEAT-020, FEAT-022, FEAT-023, FEAT-024, FEAT-027, FEAT-028, FEAT-029, FEAT-030, FEAT-034, FEAT-035, FEAT-042, FEAT-043, FEAT-044, FEAT-048, FEAT-049 |
| P1 | 11 | FEAT-010, FEAT-012, FEAT-021, FEAT-025, FEAT-031, FEAT-032, FEAT-033, FEAT-036, FEAT-041, FEAT-045, FEAT-046 |
| P2 | 6 | FEAT-026, FEAT-037, FEAT-038, FEAT-039, FEAT-040, FEAT-047 |
| P3 | 0 | None |

**Integrity checks:** All `Depends On:` IDs resolve · `### FEAT-NNN` headings run in strict ascending ID order top to bottom across the whole document, with no gaps or duplicates (FEAT-048 and FEAT-049 were scoped after every section above and live in a dedicated Dataset Lifecycle Management section at the end, per this document's ordering convention, rather than left inserted mid-document out of order) · dependency graph is acyclic (verified by walking each chain: ingest → core loop → pipeline/verification → reports → privacy/collaboration → dataset lifecycle, with no back-references; FEAT-048 depends only on FEAT-002/FEAT-042, FEAT-049 depends on FEAT-003/FEAT-048 — both sit earlier in the document, so neither introduces a back-reference) · no P0 depends on a lower priority (FEAT-012, FEAT-021, FEAT-025, FEAT-041, FEAT-045 are P1 features depending on P0 features, which is valid; no P0 feature lists a P1/P2/P3 dependency) · every feature traces to a persona goal or frustration in [[user_personas]] (including shared-infrastructure features with no single named persona in their user story — FEAT-001, FEAT-002, FEAT-004, FEAT-011, FEAT-014, FEAT-016, FEAT-017, FEAT-026, FEAT-033, FEAT-043 — each still traces to a goal shared by all three personas, e.g. FEAT-033's honest-uncertainty behavior is a stated goal of every persona in [[user_personas]], not just whoever's user story happened to be written against it) · every persona (Chidinma, Tunde, Ngozi) is served by 5+ features each, verified against the literal "As `<persona>`" line in each feature above — Chidinma: FEAT-005, FEAT-006, FEAT-008, FEAT-010, FEAT-012, FEAT-018, FEAT-019, FEAT-020, FEAT-022, FEAT-023, FEAT-024, FEAT-025, FEAT-028, FEAT-031, FEAT-032, FEAT-038, FEAT-039, FEAT-042, FEAT-048; Tunde: FEAT-003, FEAT-007, FEAT-013, FEAT-015, FEAT-030, FEAT-035, FEAT-040, FEAT-041, FEAT-044, FEAT-049; Ngozi: FEAT-009, FEAT-021, FEAT-027, FEAT-029, FEAT-034, FEAT-036, FEAT-037, FEAT-041, FEAT-045, FEAT-046, FEAT-047
