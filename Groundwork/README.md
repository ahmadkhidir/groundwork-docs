# Groundwork

**One-liner:** Groundwork lets anyone upload their data and get AI-narrated, chart-illustrated answers where every number is provably traced back to a real query result, not guessed.
**Status:** In Documentation
**Last Updated:** 2026-08-04

## Overview

Groundwork is an AI-driven finance/analytics platform. A user — a data analyst or someone with a spreadsheet and no SQL knowledge — uploads data (CSV, Excel, Parquet, or a file-based database like SQLite). The platform provisions a disposable, session-scoped DuckDB engine, ingests and profiles the data, and an AI plans and writes SQL against it to answer questions or generate reports. The model never answers from data stuffed into its context — it writes SQL, the engine returns facts, and the model narrates over those facts alongside a rendered chart. Conversation is compositional: asking a question about a report produces another report. Reports are exportable to PDF and other structured formats, and a workspace can be torn down on command, destroying all derived state.

## The Problem

Getting a trustworthy answer out of your own data currently forces a choice between three bad options: learn SQL and BI tooling yourself, wait on a data analyst's time for every question, or hand the data to a general-purpose AI chatbot that stuffs it into context, can't handle real data volumes, and offers no way to check whether a number it states is actually real. All three cost time, money, or trust — and the third one fails silently, producing confident, wrong answers with no error raised.

**How it's solved today:** Traditional BI tools (Tableau, Power BI, Looker) require SQL or dashboard-building skill and offer no natural-language narration. General-purpose "chat with your data" AI tools load a file into model context, which breaks down past small datasets and can't verify its own output. Hiring or tasking a data analyst is accurate but doesn't scale to every question a business asks. Emerging AI-analytics competitors mostly ship black-box question-answering with no verification or provenance layer.

## The Solution

Groundwork separates *planning* from *answering*: the model writes SQL against a real disposable OLAP engine and narrates only over what that engine actually returns, so answers scale to tens of millions of rows and stay literally grounded rather than aspirationally grounded. A layered verification system (determinism caching, static SQL analysis, post-execution reconciliation, shadow queries, adversarial review, and claim-level provenance) makes wrongness visible instead of hiding it, and every pipeline step is observable and user-interruptible — a user can watch, halt, and amend a step (like editing generated SQL) rather than accepting a black-box answer.

## Target Audience

**Primary:** Data analysts who want to see and edit the SQL behind an answer, and non-technical business users (e.g., someone with a spreadsheet and a question) who want a narrated answer without ever seeing SQL. Both are first-class from day one — no default-surface bias.
**Secondary:** Finance-adjacent teams (FP&A, ops analysts) who currently split time between spreadsheet wrangling and asking analysts for reports.
**Not for:** Enterprises needing live production-OLTP BI connections, teams needing real-time collaborative dashboard editing, or organizations requiring self-hosted/on-prem deployment.

These two primary segments have opposite instincts — one wants control and transparency over the query, the other wants to never think about it — and Groundwork's observable/interruptible pipeline and provenance model are built to serve both without forcing either into the other's mode.

## SWOT Analysis

### Strengths *(internal, positive)*
- Two-stage architecture (model plans SQL, engine returns facts) is a structural guarantee against context-stuffing hallucination, not a prompting technique — a real technical moat competitors using data-in-context can't easily retrofit.
- Layered verification system (determinism caching, static analysis, reconciliation, shadow queries, adversarial review, claim-level provenance) is deterministic-code-heavy — most of its value ships without extra model calls, keeping it cheap to run on every query.
- Persisted, resumable DAG run model (§7.2) makes interruption, amendment, provenance, and caching fall out of one mechanism instead of four separately-built features — lower long-term maintenance surface.
- Fork-on-amendment unifies "edit a step" and "ask a follow-up question" into one primitive, giving report history a branching tree structure other chat-based tools don't have.
- Infrastructure plan (Fly.io + Neon + R2, process-per-session) is scale-to-zero end to end, letting the product run at near-$0/month idle cost pre-revenue without a later platform migration.

### Weaknesses *(internal, negative)*
- The durable orchestrator behind the DAG run model is hand-built on Postgres rather than an off-the-shelf platform like Temporal, and hand-rolled job systems have known failure modes (lost updates, stuck jobs, double execution) that must be engineered around explicitly (leases, heartbeats, idempotency keys).
- Even with mitigations (aggregates-only by default, PII flagging, column exclusion, egress log), schema, sample values, and query results still go to a third-party model provider — the sharper privacy exposure isn't fully closed by first-party deletion controls alone.
- Solo builder with a pre-revenue cost ceiling (target <$100/month) constrains infrastructure and staffing choices, and there is no backup owner if the single builder is unavailable.
- The DuckDB Go driver dependency requires cgo, which rules out fully static binary builds and adds container/build complexity.
- Serving two default-audience surfaces (analyst SQL-visible mode and business-user narrative-only mode) without picking a bias risks a diluted default experience if both aren't executed to the same quality bar.
- Fly.io cold-start latency ("a few seconds," platform-level only) has not yet been measured end-to-end including application boot, so the wake-on-intent UX design is sized against an unverified number.

### Opportunities *(external, positive)*
- Most "AI analytics" competitors ship black-box Q&A with no verification or provenance layer — a documented, auditable trust story is a differentiator that's hard for a text-in-context competitor to bolt on retroactively.
- Growing enterprise wariness of AI data-handling claims creates demand for a provably-scoped, auditable egress model ("you can see exactly what left") rather than a marketing promise of "we never send your data."
- File-based-only ingestion (no live production DB connections) sidesteps a huge, security-sensitive integration burden that slows down BI-tool competitors, letting Groundwork ship faster in its chosen scope.
- DuckDB's storage-version pinning and forward-compatibility guarantees (since v1.2.0) remove a class of infra risk that would otherwise complicate the disposable-engine model, letting the architecture rely on a fast-maturing OSS engine instead of building one.
- Finance/analytics teams are a well-funded, spend-tolerant segment with a recurring, ongoing need (not a one-off purchase), supporting a subscription monetization model.

### Threats *(external, negative)*
- Competitors can market a simpler "just chat with your data" positioning that sounds equally trustworthy to a buyer who hasn't been burned by a wrong answer yet, undercutting Groundwork's more careful (and more complex) pitch on perceived simplicity.
- Foundation model providers could ship native "connect your data" agentic features directly into general chat products, commoditizing the core interaction pattern from underneath a standalone product.
- Regulatory or compliance requirements in target verticals (finance) could demand data-residency or on-prem guarantees that the cloud-SaaS-only deployment model explicitly does not offer, closing off segments of the target market.
- DuckDB or a core infrastructure dependency (Fly.io, Neon, R2) changing pricing, deprecating a relied-upon feature (e.g., `STORAGE_VERSION`, scale-to-zero billing), or degrading service could force an unplanned architecture or cost-model change.
- A single publicized incident of a confidently wrong AI-generated number — even from a competitor — could raise buyer skepticism toward the entire AI-analytics category, increasing the sales burden of proving verification claims are real.

## Scope

**In scope:**
- File-based data ingestion: CSV, Excel, Parquet, and file-based OLTP databases (SQLite and similar)
- Session-scoped, disposable DuckDB OLAP engine per workspace, rebuildable from canonical Parquet
- Two-stage AI: model plans and writes SQL, engine executes, model narrates over returned facts
- First-party parsable chart specs rendered by product-owned components
- Observable, interruptible, resumable run pipeline with user amendment and fork-on-amendment
- Layered verification (determinism caching, static SQL analysis, reconciliation, shadow queries, adversarial review, claim-level provenance, honest uncertainty)
- Semantic layer: relationships, grain, metric definitions, business meaning, accumulated user corrections
- Multi-file join inference with confidence scores, surfaced for user confirmation
- Privacy controls: PII detection at ingest, column-level exclusion, visible model-egress/audit log, one-workspace-one-delete cascade with provable deletion
- Reports as first-class, versioned, shareable, re-runnable objects with provenance
- Export to PDF (first target), with Excel/CSV, PowerPoint, Markdown/HTML, JSON, and image as incremental targets
- Read-only workspace and report sharing
- Usage metering (engine-seconds, object-store bytes, model tokens) from day one
- Cloud SaaS deployment on a scale-to-zero infrastructure model

**Out of scope (for now):**
- Warm pools of pre-started engines — deferred until a single machine can no longer hold concurrent sessions or provisioning latency is measured and found to matter
- `.duckdb` snapshot caching for fast-resume of expensive workspaces — deferred until metering shows rebuild time actually justifies it
- Concurrent, real-time collaborative editing of the semantic layer — read-only sharing ships first; live co-editing is a CRDT-shaped problem not where v1 value concentrates
- Export targets beyond PDF — Excel/CSV, slides, etc. are planned but sequenced after audience priorities are confirmed
- Self-host deployment planning of any kind (not merely deferred — see non-goals)

**Explicit non-goals:**
- Live connections to production OLTP databases behind security groups — file-based ingestion only, permanently
- Answering questions from data placed directly in the model's context window — the two-stage SQL-over-engine architecture is foundational and will not be bypassed for convenience
- Self-hosted or on-premises deployment
- Drag-and-drop dashboard building as the core interaction model — Groundwork is conversational and report-driven, not a dashboard builder
- Model-emitted chart images or freeform markup — charts are always structured specs rendered by first-party components

## Documentation

| Document                | Purpose                                                          |
| ----------------------- | ----------------------------------------------------------------- |
| README                  | Product concept, problem/solution, target audience, SWOT, scope   |
| [[user_personas]]       | Who this product is for                                           |
| [[branding]]             | Brand identity, design system, experience tier, and target platform |
| [[features]]             | What gets built, prioritized, with acceptance criteria |
| [[roadmap]]              | When it gets built, in phases |
| [[architecture]]         | How the system is structured |
| [[monetization]]         | How it makes money |
| [[risk_log]]             | What could go wrong and the plan for it |
| [[user_flow]]            | How users move through the product |
| [[screens]]               | Every screen, its states, and its components |
| [[schema]]                | The data model |
