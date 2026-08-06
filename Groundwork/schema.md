# Schema

**Last Updated:** 2026-08-06
**Source:** [[screens]] (per-screen Data Requirements), [[features]], [[architecture]], [[monetization]], [[user_flow]]

## Overview

This is Groundwork's data model, expressed as JSON Schema 2020-12 entities. It follows the storage split established in [[architecture]]: every entity here lives in the application database (Neon Postgres) as durable, queryable state — accounts, password reset tokens, workspaces, the semantic layer, orchestration state, reports, sharing, notifications, the audit/egress log, usage, and deletion records. Raw ingested data itself is **not** modeled here as a schema entity — it lives as normalized Parquet in object storage (R2), referenced by `Dataset.parquet_location`, with the disposable session DuckDB engine as a compute layer over it, never a system of record.

Two design choices apply throughout:
- **Every relationship between entities uses `$ref`** — a `*_id` property plus a documented relationship in the Relationships section below, never an inlined copy of another entity's fields.
- **`ChartSpec` and `ReconciliationSummary` are inlined as value objects inside `Report`, and `type_fit_rates` is inlined inside `Column`.** None of the three has independent identity, is queried on its own, or is referenced from any entity other than its parent — each exists only as structured detail attached to one row, which is exactly the exception the flat-object rule allows. `type_fit_rates`'s six properties are fixed by the `inferred_type` enum, not an open-ended map, so it doesn't reintroduce the union problem `Step.input`/`Step.output` avoid by staying string-typed.
- **`Step.input` and `Step.output`** are modeled as JSON-encoded strings rather than typed objects. Per [[architecture]], step payload shape genuinely varies by `step_name` (a `generate_sql` step's output is SQL text; a `reconcile` step's output is row counts; an `adversarial_review` step's output is a verdict) — typing this as one flat object would either be dishonestly narrow or require a union the flat-object rule doesn't support. The string is a deliberate, documented boundary, not an oversight; per-`step_name` payload shapes are an implementation concern for the `Implementor` agent to type per step, not a schema entity.

## Entity Definitions

### User

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/user.schema.json",
  "title": "User",
  "description": "An account holder. Owns workspaces, holds a subscription tier, and is the identity behind every workspace membership, run, report, notification, and audit log entry.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the user." },
    "email": { "type": "string", "description": "The user's email address, used for authentication and notifications." },
    "display_name": { "type": "string", "description": "The user's display name shown in workspace member lists (per [[features]] FEAT-045) and audit entries. Collected as an optional field on [[screens/02-sign-up-login|Sign Up]]; if left blank, defaults to the local-part of the user's email (e.g., 'chidinma' from 'chidinma@example.com') at account creation. Optional rather than required at signup because it is only ever seen by others once a workspace is shared (FEAT-045) or a report is exported with a viewer attached — not a decision either persona should be forced to make before their first upload." },
    "tier": { "type": "string", "enum": ["free", "pro", "team"], "description": "The user's current subscription tier, per [[monetization]]. Determines feature gating (export formats, sharing, usage detail) across the product." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the account was created." }
  },
  "required": ["id", "email", "display_name", "tier", "created_at"]
}
```

### Workspace

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/workspace.schema.json",
  "title": "Workspace",
  "description": "The durable, user-managed container that owns a set of datasets, the semantic layer, runs, and reports. The unit of deletion and privacy per [[README]] and [[architecture]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the workspace." },
    "name": { "type": "string", "description": "User-assigned workspace name." },
    "owner_id": { "type": "string", "description": "References the User who owns this workspace." },
    "engine_status": { "type": "string", "enum": ["sleeping", "waking", "ready"], "description": "The current live status of the workspace's session-scoped DuckDB engine, shown honestly in the UI per [[branding]]." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the workspace was created." },
    "updated_at": { "type": "string", "format": "date-time", "description": "When the workspace was last modified (e.g., new dataset ingested, semantic layer changed)." }
  },
  "required": ["id", "name", "owner_id", "engine_status", "created_at", "updated_at"]
}
```

### WorkspaceMember

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/workspace-member.schema.json",
  "title": "WorkspaceMember",
  "description": "Grants a User access to a Workspace beyond ownership. Read-only sharing per [[features]] FEAT-045; concurrent semantic-layer editing is explicitly out of scope, so only owner/read_only roles exist.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for this membership record." },
    "workspace_id": { "type": "string", "description": "References the Workspace this membership grants access to." },
    "user_id": { "type": "string", "description": "References the User granted access." },
    "role": { "type": "string", "enum": ["owner", "read_only"], "description": "The member's access level. Read-only members can view reports, charts, and provenance but cannot mutate the semantic layer, upload data, or run queries." },
    "invited_at": { "type": "string", "format": "date-time", "description": "When the invitation was sent." },
    "accepted_at": { "type": "string", "format": "date-time", "description": "When the invitation was accepted. Empty string if not yet accepted." }
  },
  "required": ["id", "workspace_id", "user_id", "role", "invited_at"]
}
```

### Dataset

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/dataset.schema.json",
  "title": "Dataset",
  "description": "One ingested and normalized file within a workspace. The canonical stored form is normalized Parquet in object storage, not the original upload, per [[architecture]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the dataset." },
    "workspace_id": { "type": "string", "description": "References the Workspace this dataset belongs to." },
    "original_filename": { "type": "string", "description": "The filename as originally uploaded, kept for display purposes only." },
    "format": { "type": "string", "enum": ["csv", "excel", "parquet", "sqlite"], "description": "The format of the originally uploaded file." },
    "parquet_location": { "type": "string", "description": "URI of the normalized Parquet object in object storage (R2), the canonical stored form of this dataset." },
    "ingest_status": { "type": "string", "enum": ["pending", "provisioning", "profiling", "complete", "failed"], "description": "Current stage of the ingest pipeline for this dataset, shown on [[screens/06-ingest-progress|Ingest Progress]]." },
    "row_count": { "type": "integer", "description": "Total number of rows in the normalized dataset." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the dataset was uploaded." }
  },
  "required": ["id", "workspace_id", "original_filename", "format", "ingest_status", "created_at"]
}
```

### Column

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/column.schema.json",
  "title": "Column",
  "description": "A single profiled column within a Dataset, carrying inferred type, PII flag, and model-visibility state reviewed on [[screens/07-ingest-confirmation|Ingest Confirmation]] and managed on [[screens/13-workspace-settings|Workspace Settings]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the column." },
    "dataset_id": { "type": "string", "description": "References the Dataset this column belongs to." },
    "name": { "type": "string", "description": "Column name as it appears in the dataset." },
    "inferred_type": { "type": "string", "enum": ["string", "integer", "number", "boolean", "date", "timestamp"], "description": "The type inferred at ingest, reviewable and overridable by the user per [[features]] FEAT-006." },
    "type_fit_rates": {
      "type": "object",
      "description": "A fit rate (0-1) for every candidate type, computed once during profiling as additional FILTER clauses in the same combined aggregate query profiling already runs — not a per-type extra round trip, per [[features]] FEAT-005. For boolean/date/timestamp/integer/number, the fraction of sampled non-null values that coerce to that type without loss. For string (the fallback type), the fraction of sampled non-null values that do not also match a stricter type pattern tested and rejected. Each rate is weakened by the column's null rate, with a floor. This is the durable source of truth `type_confidence` reads from; it is computed once and never recomputed when the user changes the override, per [[features]] FEAT-005's cost note.",
      "properties": {
        "string": { "type": "number", "description": "Fit rate (0-1) if this column's active type were `string`." },
        "integer": { "type": "number", "description": "Fit rate (0-1) if this column's active type were `integer`." },
        "number": { "type": "number", "description": "Fit rate (0-1) if this column's active type were `number`." },
        "boolean": { "type": "number", "description": "Fit rate (0-1) if this column's active type were `boolean`." },
        "date": { "type": "number", "description": "Fit rate (0-1) if this column's active type were `date`." },
        "timestamp": { "type": "number", "description": "Fit rate (0-1) if this column's active type were `timestamp`." }
      },
      "required": ["string", "integer", "number", "boolean", "date", "timestamp"]
    },
    "type_confidence": { "type": "number", "description": "The fit rate (0-1) for this column's currently ACTIVE type — `inferred_type` unless the user has set an override, in which case the override — presented to the user via the HIGH (≥0.85) / MEDIUM (0.6-0.84) / LOW (<0.6) bands specified in [[features]] FEAT-005, labeled 'N% of values fit this type.' Not an independently stored value: it is a read-time lookup into `type_fit_rates` keyed by whichever type is currently active, so it always describes the type actually shown next to it and updates instantly (no recomputation, no engine wake) when the user changes the override. The bands, the label, and 'always describes the currently active type' are the specified contract; sample size, exact coercion rules, and the floor value used to compute `type_fit_rates` are implementation-defined." },
    "pii_flag": { "type": "boolean", "description": "Whether this column was automatically flagged as likely containing sensitive personal data at ingest, per [[features]] FEAT-007." },
    "pii_pattern": { "type": "string", "description": "The detected pattern type if pii_flag is true (e.g., 'email', 'national_id', 'card_number', 'name'). Empty string if not flagged." },
    "pii_reviewed_at": { "type": "string", "format": "date-time", "description": "When a user last made an explicit reviewed decision about this column's pii_flag — clearing it, or explicitly confirming it as correct — during ingest confirmation or later in [[screens/13-workspace-settings|Workspace Settings]], per [[features]] FEAT-007. Empty string if no user has ever explicitly reviewed this column's flag, which is what distinguishes a reviewed-and-cleared column from one that was simply never flagged. Written by FEAT-007's override action, alongside a corresponding `pii_override` [[#AuditLogEntry|AuditLogEntry]]." },
    "excluded_from_model": { "type": "boolean", "description": "Whether the user has explicitly excluded this column from ever being sent to the model, per [[features]] FEAT-010. The column remains queryable by the engine regardless." },
    "null_rate": { "type": "number", "description": "Fraction (0-1) of values in this column that are null, from profiling." },
    "distinct_count": { "type": "integer", "description": "Number of distinct values in this column, from profiling." }
  },
  "required": ["id", "dataset_id", "name", "inferred_type", "type_fit_rates", "type_confidence", "pii_flag", "excluded_from_model", "null_rate", "distinct_count"]
}
```

### CandidateJoin

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/candidate-join.schema.json",
  "title": "CandidateJoin",
  "description": "A relationship between two columns proposed at ingest via relationship inference, awaiting explicit user confirmation before it can ever be used. Never auto-applied, per [[features]] FEAT-008 and FEAT-009.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the candidate join." },
    "workspace_id": { "type": "string", "description": "References the Workspace this candidate join was inferred within." },
    "source_column_id": { "type": "string", "description": "References the source Column of the proposed join." },
    "target_column_id": { "type": "string", "description": "References the target Column of the proposed join." },
    "confidence_score": { "type": "number", "description": "Confidence score (0-1) for this candidate join, based on name similarity, value overlap, and cardinality checks, presented to the user via the same HIGH (≥0.85) / MEDIUM (0.6-0.84) / LOW (<0.6) bands as `Column.type_confidence`. Per [[features]] FEAT-008, a join whose only meaningful evidence is value overlap on a non-identifier-shaped numeric column is capped at MEDIUM and can never score HIGH; a measure-shaped column name (`total`, `amount`, `price`, etc.) defeats the near-unique exemption regardless of actual distinctness." },
    "basis": { "type": "string", "description": "Plain-language explanation of why this join was suggested (e.g., 'matched column names, 94% value overlap')." },
    "status": { "type": "string", "enum": ["pending", "accepted", "rejected"], "description": "Whether the user has reviewed and accepted, rejected, or not yet reviewed this candidate join." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the candidate join was inferred." }
  },
  "required": ["id", "workspace_id", "source_column_id", "target_column_id", "confidence_score", "basis", "status", "created_at"]
}
```

### Relationship

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/relationship.schema.json",
  "title": "Relationship",
  "description": "A confirmed join relationship persisted in the semantic layer, viewable and editable on [[screens/15-semantic-layer|Semantic Layer / Corrections]]. Created when a CandidateJoin is accepted, or added manually.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the relationship." },
    "workspace_id": { "type": "string", "description": "References the Workspace this relationship belongs to." },
    "source_column_id": { "type": "string", "description": "References the source Column of the confirmed join." },
    "target_column_id": { "type": "string", "description": "References the target Column of the confirmed join." },
    "cardinality": { "type": "string", "enum": ["one_to_one", "one_to_many", "many_to_one", "many_to_many"], "description": "The relationship's cardinality, used by static SQL analysis to detect join-grain fan-out per [[features]] FEAT-028." },
    "grain_notes": { "type": "string", "description": "Free-text notes on the grain implications of this relationship (e.g., 'joining introduces one row per order line, not per order')." },
    "confirmed_at": { "type": "string", "format": "date-time", "description": "When this relationship was confirmed." }
  },
  "required": ["id", "workspace_id", "source_column_id", "target_column_id", "cardinality", "confirmed_at"]
}
```

### MetricDefinition

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/metric-definition.schema.json",
  "title": "MetricDefinition",
  "description": "A named, persisted business metric definition in the semantic layer (e.g., what 'revenue' means for this workspace), viewable and editable on [[screens/15-semantic-layer|Semantic Layer / Corrections]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the metric definition." },
    "workspace_id": { "type": "string", "description": "References the Workspace this metric definition belongs to." },
    "name": { "type": "string", "description": "The metric's name (e.g., 'revenue', 'active_customer')." },
    "sql_expression": { "type": "string", "description": "The SQL expression that computes this metric, used as grounding when the model plans a query." },
    "description": { "type": "string", "description": "Plain-language description of what this metric means and why it's defined this way." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the metric definition was created." },
    "updated_at": { "type": "string", "format": "date-time", "description": "When the metric definition was last modified." }
  },
  "required": ["id", "workspace_id", "name", "sql_expression", "description", "created_at", "updated_at"]
}
```

### Correction

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/correction.schema.json",
  "title": "Correction",
  "description": "A persisted record of a user correcting generated SQL or semantic-layer logic during a run, so the same mistake is not repeated, per [[features]] FEAT-020. Viewable on [[screens/15-semantic-layer|Semantic Layer / Corrections]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the correction." },
    "workspace_id": { "type": "string", "description": "References the Workspace this correction applies to." },
    "description": { "type": "string", "description": "Plain-language summary of what was corrected (e.g., 'refunds join now excludes cancelled orders')." },
    "original_logic": { "type": "string", "description": "The original SQL or logic before correction." },
    "corrected_logic": { "type": "string", "description": "The corrected SQL or logic, now used going forward." },
    "originating_run_id": { "type": "string", "description": "References the Run during which this correction was made." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the correction was made." }
  },
  "required": ["id", "workspace_id", "description", "original_logic", "corrected_logic", "originating_run_id", "created_at"]
}
```

### Run

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/run.schema.json",
  "title": "Run",
  "description": "A single execution of the resumable DAG pipeline answering one question, per [[architecture]]. May fork from another Run via parent_run_id when a completed step is amended, per [[features]] FEAT-025.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the run." },
    "workspace_id": { "type": "string", "description": "References the Workspace this run belongs to." },
    "question_text": { "type": "string", "description": "The natural-language question that initiated this run." },
    "status": { "type": "string", "enum": ["queued", "waking", "running", "halted", "failed", "complete"], "description": "The run's overall status, shown on [[screens/08-run-viewer|Run Viewer]]." },
    "parent_run_id": { "type": "string", "description": "References the Run this run was forked from, if it originated from amending a completed step. Empty string if this is an original run." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the run was created." },
    "completed_at": { "type": "string", "format": "date-time", "description": "When the run reached a terminal status. Empty string while still in progress." }
  },
  "required": ["id", "workspace_id", "question_text", "status", "created_at"]
}
```

### Step

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/step.schema.json",
  "title": "Step",
  "description": "One node in a Run's DAG, persisted at every boundary per [[architecture]]. This is what makes interruption, amendment, provenance, and determinism caching all fall out of one schema.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the step." },
    "run_id": { "type": "string", "description": "References the Run this step belongs to." },
    "parent_step_id": { "type": "string", "description": "References the Step this step forked from, used to build fork trees within a run's amendment history. Empty string if not forked." },
    "step_name": { "type": "string", "enum": ["profile", "plan", "generate_sql", "static_check", "execute", "reconcile", "shadow_check", "adversarial_review", "chart_spec", "narrate", "assemble"], "description": "Which stage of the pipeline this step represents, per [[architecture]]'s DAG." },
    "status": { "type": "string", "enum": ["pending", "running", "complete", "failed", "halted"], "description": "The step's current status." },
    "input": { "type": "string", "description": "JSON-encoded input to this step. Shape varies by step_name (e.g., generate_sql's input includes schema and semantic layer context); steps are pure functions of this persisted input per [[architecture]]." },
    "output": { "type": "string", "description": "JSON-encoded output of this step. Shape varies by step_name (e.g., reconcile's output is row counts per stage; generate_sql's output is SQL text)." },
    "idempotency_key": { "type": "string", "description": "Unique key ensuring this step cannot be double-executed on retry, per [[architecture]]'s orchestration correctness requirements." },
    "started_at": { "type": "string", "format": "date-time", "description": "When execution of this step began. Empty string if not yet started." },
    "completed_at": { "type": "string", "format": "date-time", "description": "When this step reached a terminal status. Empty string while still in progress." }
  },
  "required": ["id", "run_id", "step_name", "status", "idempotency_key"]
}
```

### Report

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/report.schema.json",
  "title": "Report",
  "description": "A first-class, versioned answer object per [[features]] FEAT-034 — not a chat message. Fully self-contained and viewable without the session engine running, per [[architecture]]'s Report Store.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the report." },
    "workspace_id": { "type": "string", "description": "References the Workspace this report belongs to." },
    "run_id": { "type": "string", "description": "References the Run that produced this report." },
    "title": { "type": "string", "description": "A short, human-readable title for the report, shown on [[screens/04-workspace-home|Workspace Home]] and [[screens/09-report-view|Report View]]." },
    "narrative": { "type": "string", "description": "The plain-language narrative text answering the user's question, generated only from the run's result set." },
    "chart_spec": {
      "type": "object",
      "description": "The parsable chart specification rendered by first-party components, per [[features]] FEAT-016/FEAT-017. Inlined here as it has no independent identity outside its parent Report.",
      "properties": {
        "chart_type": { "type": "string", "enum": ["bar", "line", "pie", "scatter", "table"], "description": "The type of chart to render." },
        "x_axis_column": { "type": "string", "description": "Name of the result column mapped to the x-axis." },
        "y_axis_column": { "type": "string", "description": "Name of the result column mapped to the y-axis." },
        "series_column": { "type": "string", "description": "Name of the result column used to split the data into multiple series, using the categorical chart tokens from [[branding]]. Empty string if the chart has a single series." },
        "title": { "type": "string", "description": "The chart's display title." }
      },
      "required": ["chart_type", "x_axis_column", "y_axis_column", "title"]
    },
    "provenance_links": {
      "type": "array",
      "description": "The set of claim-level provenance links backing numbers in this report's narrative, per [[features]] FEAT-030.",
      "items": { "type": "string", "description": "References a ProvenanceLink entity by id." }
    },
    "reconciliation_summary": {
      "type": "object",
      "description": "Row-level reconciliation data surfaced per [[features]] FEAT-029. Inlined here as it has no independent identity outside its parent Report.",
      "properties": {
        "rows_in": { "type": "integer", "description": "Total rows read before any filtering." },
        "rows_out": { "type": "integer", "description": "Total rows remaining after all filtering and grouping." },
        "exclusion_warnings": {
          "type": "array",
          "description": "Plain-language unprompted warnings for any filter that excluded a significant share of rows (e.g., 'excluded 31% of rows where region was null').",
          "items": { "type": "string", "description": "A single exclusion warning message." }
        }
      },
      "required": ["rows_in", "rows_out", "exclusion_warnings"]
    },
    "parent_report_id": { "type": "string", "description": "References the Report this report was forked from via a compositional follow-up question, per [[features]] FEAT-021/FEAT-025. Empty string if this report has no parent." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the report was created." }
  },
  "required": ["id", "workspace_id", "run_id", "title", "narrative", "chart_spec", "provenance_links", "reconciliation_summary", "created_at"]
}
```

### ProvenanceLink

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/provenance-link.schema.json",
  "title": "ProvenanceLink",
  "description": "Ties a specific number stated in a Report's narrative to the exact query and rows that produced it, per [[features]] FEAT-030. Shown on [[screens/10-provenance-panel|Provenance Panel]] when a user clicks a claim.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the provenance link." },
    "report_id": { "type": "string", "description": "References the Report this provenance link belongs to." },
    "claim_text": { "type": "string", "description": "The exact text span in the narrative that this link corresponds to." },
    "stated_value": { "type": "string", "description": "The value as stated in the narrative (kept as a string to preserve exact formatting, e.g. currency or percentage display)." },
    "source_query": { "type": "string", "description": "The exact SQL query that produced this value." },
    "source_row_ids": {
      "type": "array",
      "description": "Identifiers of the specific result rows/cells backing this value, shown highlighted on [[screens/10-provenance-panel|Provenance Panel]].",
      "items": { "type": "string", "description": "A single source row identifier within the query's result set." }
    }
  },
  "required": ["id", "report_id", "claim_text", "stated_value", "source_query", "source_row_ids"]
}
```

### ReportShare

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/report-share.schema.json",
  "title": "ReportShare",
  "description": "A read-only shareable link to a single Report, per [[features]] FEAT-046. Access is scoped to possession of the token, not account identity, per [[user_flow]]'s Authentication Gate.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the share record." },
    "report_id": { "type": "string", "description": "References the Report being shared." },
    "share_token": { "type": "string", "description": "The unguessable token used in the shareable URL." },
    "revoked": { "type": "boolean", "description": "Whether the report owner has revoked this share link, per [[screens/13-workspace-settings|Workspace Settings]]." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the share link was created." }
  },
  "required": ["id", "report_id", "share_token", "revoked", "created_at"]
}
```

### Notification

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/notification.schema.json",
  "title": "Notification",
  "description": "A durable, persisted notification, deliberately separate from the high-frequency live run event stream per [[features]] FEAT-026. Shown on [[screens/16-notifications|Notifications]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the notification." },
    "user_id": { "type": "string", "description": "References the User this notification is for." },
    "type": { "type": "string", "enum": ["export_ready", "engine_slept", "ingest_failed", "workspace_invite"], "description": "The category of event this notification represents." },
    "message": { "type": "string", "description": "The human-readable notification message." },
    "related_entity_type": { "type": "string", "enum": ["report", "workspace", "dataset"], "description": "The type of entity this notification links to, used to route the deep link." },
    "related_entity_id": { "type": "string", "description": "The id of the related entity (a Report, Workspace, or Dataset id, depending on related_entity_type)." },
    "read": { "type": "boolean", "description": "Whether the user has marked this notification read." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the notification was generated." }
  },
  "required": ["id", "user_id", "type", "message", "related_entity_type", "related_entity_id", "read", "created_at"]
}
```

### AuditLogEntry

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/audit-log-entry.schema.json",
  "title": "AuditLogEntry",
  "description": "A single entry in the combined audit and model-egress log, per [[features]] FEAT-042. The source spec is explicit this is one object, not two — it records queries, model payloads, exports, deletions, and logins in one place. Most action types are workspace-scoped; `login` is account-scoped and carries no workspace, per `workspace_id` below and the Relationships note.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the audit log entry." },
    "workspace_id": { "type": "string", "description": "References the Workspace this action occurred within. Empty string for account-scoped entries — currently only `action_type: \"login\"`, since authentication happens before any workspace is selected and a login is never assigned a placeholder workspace to satisfy this field." },
    "user_id": { "type": "string", "description": "References the User who performed the action." },
    "action_type": { "type": "string", "enum": ["query_executed", "model_call", "export", "deletion", "login", "pii_override"], "description": "The category of action this entry records. `query_executed`, `model_call`, `export`, `deletion`, and `pii_override` are workspace-scoped (`workspace_id` set). `login` is account-scoped (`workspace_id` empty). `pii_override` records a user explicitly clearing or confirming a column's `pii_flag` (per [[features]] FEAT-007) — a trust-surface decision worth its own action type since it governs what the model is allowed to see, not just internal bookkeeping." },
    "payload_sent_to_model": { "type": "string", "description": "JSON-encoded payload sent to the model provider, if action_type is model_call. Empty string for action types that don't involve a model call." },
    "timestamp": { "type": "string", "format": "date-time", "description": "When the action occurred." }
  },
  "required": ["id", "user_id", "action_type", "timestamp"]
}
```

### UsageRecord

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/usage-record.schema.json",
  "title": "UsageRecord",
  "description": "Metered usage for a workspace over a period, recorded from day one regardless of billing status per [[architecture]] and [[features]] FEAT-047. Backs both the user-facing Usage view on [[screens/13-workspace-settings|Workspace Settings]] and the overage billing calculation in [[monetization]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the usage record." },
    "workspace_id": { "type": "string", "description": "References the Workspace this usage record measures." },
    "period_start": { "type": "string", "format": "date-time", "description": "Start of the metering period this record covers." },
    "period_end": { "type": "string", "format": "date-time", "description": "End of the metering period this record covers." },
    "engine_seconds": { "type": "number", "description": "Total engine compute time consumed in this period, in seconds." },
    "storage_bytes": { "type": "integer", "description": "Total object storage consumed by this workspace's data, in bytes, as of period_end." },
    "model_tokens": { "type": "integer", "description": "Total model tokens (input plus output) consumed in this period." }
  },
  "required": ["id", "workspace_id", "period_start", "period_end", "engine_seconds", "storage_bytes", "model_tokens"]
}
```

### DeletionRecord

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/deletion-record.schema.json",
  "title": "DeletionRecord",
  "description": "Proof of a completed workspace deletion cascade, per [[features]] FEAT-043. Shown on [[screens/14-deletion-confirmation-proof|Deletion Confirmation & Proof]] as the user's evidence of what was deleted and when.",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the deletion record." },
    "workspace_id": { "type": "string", "description": "References the Workspace that was deleted. The Workspace record itself is removed; this id is retained here as a historical reference only." },
    "initiated_by": { "type": "string", "description": "References the User who initiated the deletion." },
    "deleted_categories": {
      "type": "array",
      "description": "The categories of derived state confirmed destroyed by this cascade.",
      "items": { "type": "string", "enum": ["object_store", "semantic_layer", "reports_and_conversation", "query_caches", "logs"], "description": "A single category of derived state included in the deletion cascade." }
    },
    "completed_at": { "type": "string", "format": "date-time", "description": "When the deletion cascade finished." }
  },
  "required": ["id", "workspace_id", "initiated_by", "deleted_categories", "completed_at"]
}
```

### PasswordResetToken

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://groundwork.app/schema/password-reset-token.schema.json",
  "title": "PasswordResetToken",
  "description": "A durable, single-use password reset token per [[features]] FEAT-001, requested on [[screens/18-forgot-password|Forgot Password]] and consumed on [[screens/19-reset-password|Reset Password]]. Persisted here (rather than held in process memory) so a reset link keeps working across a restart or across multiple running instances, per [[architecture]].",
  "type": "object",
  "properties": {
    "id": { "type": "string", "description": "Unique identifier for the reset token record." },
    "user_id": { "type": "string", "description": "References the User this reset token was issued for." },
    "token_hash": { "type": "string", "description": "A one-way hash of the reset token. The raw token is included only in the delivered email link and is never itself persisted." },
    "expires_at": { "type": "string", "format": "date-time", "description": "When this token expires — 30 minutes after issuance." },
    "consumed_at": { "type": "string", "format": "date-time", "description": "When this token was used to complete a password reset. Empty string if not yet consumed. A consumed token cannot be used again." },
    "created_at": { "type": "string", "format": "date-time", "description": "When the reset token was issued." }
  },
  "required": ["id", "user_id", "token_hash", "expires_at", "created_at"]
}
```

## Relationships

- **Workspace** `owner_id` references **User** — each workspace has exactly one owner.
- **WorkspaceMember** `workspace_id` references **Workspace**, and `user_id` references **User** — each membership grants one user access to one workspace, with a role of `owner` or `read_only` (per [[features]] FEAT-045).
- **Dataset** `workspace_id` references **Workspace** — each dataset belongs to exactly one workspace.
- **Column** `dataset_id` references **Dataset** — each column belongs to exactly one dataset.
- **CandidateJoin** `workspace_id` references **Workspace**; `source_column_id` and `target_column_id` each reference **Column** — a candidate join proposes a relationship between two columns, which may belong to different datasets within the same workspace.
- **Relationship** `workspace_id` references **Workspace**; `source_column_id` and `target_column_id` each reference **Column** — a confirmed join, created when a CandidateJoin is accepted (or added manually) and persisted in the semantic layer.
- **MetricDefinition** `workspace_id` references **Workspace** — each metric definition belongs to exactly one workspace's semantic layer.
- **Correction** `workspace_id` references **Workspace**, and `originating_run_id` references **Run** — a correction is scoped to a workspace and traceable back to the run during which it was made.
- **Run** `workspace_id` references **Workspace**; `parent_run_id` references another **Run** — used for fork-on-amendment (per [[features]] FEAT-025), forming a tree of runs when a completed run's step is amended.
- **Step** `run_id` references **Run**; `parent_step_id` references another **Step** — steps belong to a run in sequence, with `parent_step_id` capturing forks within a run's amendment history.
- **Report** `workspace_id` references **Workspace**, `run_id` references **Run**, and `parent_report_id` references another **Report** — a report is produced by exactly one run and may be forked from a parent report via a compositional follow-up question (per [[features]] FEAT-021/FEAT-025). `chart_spec` and `reconciliation_summary` are inlined value objects, not references. `provenance_links` is an array of ids referencing **ProvenanceLink**.
- **ProvenanceLink** `report_id` references **Report** — each provenance link belongs to exactly one report.
- **ReportShare** `report_id` references **Report** — each share link grants read-only access to exactly one report.
- **Notification** `user_id` references **User**; `related_entity_id` references a **Report**, **Workspace**, or **Dataset** depending on `related_entity_type` — a polymorphic reference resolved by type at read time.
- **AuditLogEntry** `workspace_id` references **Workspace**, and `user_id` references **User**. `query_executed`, `model_call`, `export`, `deletion`, and `pii_override` entries are scoped to both the workspace they occurred in and the acting user. `login` entries are account-scoped: `workspace_id` is an empty string, since authentication happens before any workspace is in context, consistent with this schema's existing empty-string-means-absent convention (e.g., `Run.parent_run_id`). [[screens/17-egress-audit-log|Egress / Audit Log]] is a per-workspace screen; it additionally surfaces the current user's own account-scoped `login` entries alongside that workspace's entries, labeled by scope, rather than omitting them or inventing a workspace for them.
- **PasswordResetToken** `user_id` references **User** — a token is single-use (`consumed_at`) and time-bound (`expires_at`). Requesting a reset never reveals whether the referenced email matches an account; a token is only ever created when it does.
- **UsageRecord** `workspace_id` references **Workspace** — usage is always measured per workspace, per [[architecture]] and [[monetization]].
- **DeletionRecord** `workspace_id` references the id of a now-deleted **Workspace**, and `initiated_by` references **User** — this is the sole surviving record of a workspace after its deletion cascade completes, retained specifically to make deletion provable per [[features]] FEAT-043.
