# Product Idea — Working Spec

**Status:** Raw spec. Input for `product-shaper`, not a finished product doc.
**Last updated:** 2026-08-04
**Working name:** _(none chosen — placeholder needed)_

This captures a design conversation about an AI-driven analytics platform. It records decisions
made, the reasoning behind them, findings verified against live sources, and open questions. It is
deliberately not structured as a product README — it exists to be fed into a product-shaping pass.

---

## 1. Concept

A finance/analytics platform where anyone with data — a data analyst, or a non-technical person
with a spreadsheet — uploads it and gets AI-driven analysis grounded in that data.

The user brings data in whatever shape they have it. The platform stands up a disposable OLAP
engine, loads the data, profiles it, and an AI generates reports and answers questions by querying
that engine. Output is narrative text plus charts, exportable to PDF and other targets.

### Core loop

1. User uploads data — CSV, Excel, Parquet, or a file-based OLTP database (SQLite and similar)
2. Platform provisions a **session-scoped OLAP engine** (DuckDB), ingests and profiles the data
3. Schema, profiling stats, and semantic-layer context are supplied to the model as grounding
4. User asks a question, or requests a report
5. Model plans and writes SQL; the engine executes it and returns **facts**
6. Model narrates over the returned facts and emits a **chart spec**
7. First-party chart components render the spec; text renders alongside it
8. Conversation is **compositional** — ask a question about a report to produce another report or an
   immediate answer
9. Reports export to PDF and other structured targets (§7.10)
10. On completion, the user tears the workspace down and all derived state is destroyed

---

## 2. Decisions Made

### 2.1 Two-stage AI, not data-in-context

**The model does not answer from data placed in its context. It writes SQL, and the engine returns
the facts it narrates over.**

Metadata in context exists to *plan the query*, not to *be the data*. This is the decision the rest
of the architecture hangs on:

- It is what makes "grounded" literally true rather than aspirational
- It lets the product work on 50M rows rather than 5,000
- It bounds what data leaves the trust boundary (see §6)

### 2.2 Charts are specs, not images

The model emits a **parsable chart spec** — chart type, encodings, which result columns map to which
axes — and first-party components render it. The model never emits chart images or raw markup.

This is what makes charts consistent, themeable, interactive, and PDF-exportable.

### 2.3 No live production OLTP connections

Rejected deliberately. Production databases sit behind security groups, and the credential-handling,
network-access, schema-discovery, and incremental-sync burden is a different and much larger product
than file ingestion.

**File-based OLTP only** — SQLite and similar. That is a file upload with extra parsing, not a
networking and security problem.

### 2.4 Three-tier data lifecycle

| Tier | Role | Lifetime |
|---|---|---|
| **Object store (Parquet)** | Canonical source of truth | Lifetime of the workspace |
| **DuckDB instance** | Derived, disposable, rebuildable | Session, killed on idle |
| **Reports & conversation** | Independent of both | Lifetime of the workspace |

**Engine teardown is a cost event, not a data event.** Because the engine rebuilds from the object
store, aggressive idle timeouts are cheap — kill after ~10 minutes idle, rehydrate on next query.

Store the **ingested and normalized Parquet**, not the original CSV. Re-parsing and re-typing a
messy CSV is the slow part of rebuild, and re-inferring types risks inferring them *differently* on
rebuild — a correctness bug, not just a performance one.

Consider surfacing engine state in the UI ("engine sleeping") rather than hiding it. It turns a cost
optimization into a visible trust signal.

### 2.5 Sessions

A session spans many request/response cycles. The OLAP engine must be available for its duration.
Automatic teardown on idle is required — users will forget to close sessions, and some operations
(re-reading a generated report) don't touch the engine at all.

**Open:** exact idle threshold, and whether teardown is signalled to the user or silent.

### 2.6 Observable, interruptible pipeline

Every pipeline step — profiling, query planning, SQL generation, execution, chart spec, narration —
emits events the user watches live. The user can halt any step, amend it (edit the SQL), and resume
from that point.

This is a primary differentiator against black-box "ask a question, get an answer" competitors, and
it doubles as the front line of verification (§5).

Two design constraints that must be honored from the start:

- **Each step's input and output must be serializable state**, so the pipeline is resumable from
  arbitrary points. Retrofitting this is painful.
- **User edits must persist as preference.** If a user corrects the same join twice, the system
  should stop making the mistake.

### 2.7 SQL is visible and editable

Always. Not hidden behind an affordance.

### 2.8 Distributed system with notifications

Work is distributed. A notification system surfaces what every part of the system is doing. Users
can stop a particular operation, send input back into it (e.g. an SQL correction), and resume.

### 2.9 Multi-file joins are in scope

Nobody's real question lives in one file. This is where value concentrates and where difficulty
concentrates.

Approach: a **relationship-inference pass at ingest** — name similarity, type compatibility, value
overlap sampling, cardinality checks — producing **candidate joins with confidence scores, surfaced
for user confirmation rather than silently assumed.**

---

## 3. Storage Format — Verified Findings

Researched against live sources on 2026-08-04 rather than assumed.

### 3.1 Backward compatibility is solved

Since **v0.10.0** (Feb 2024), newer DuckDB always reads database files written by older versions.
This is a committed direction and has held in production use.

### 3.2 Forward compatibility is best-effort — and is the real hazard

An *older* engine reading a *newer* file is not guaranteed. This is the failure mode that would
actually bite: the engine auto-upgrades, writes a snapshot, and an older-pinned worker cannot open
it.

### 3.3 `STORAGE_VERSION` removes that risk

**v1.2.0** introduced a `STORAGE_VERSION` option on `ATTACH`, pinning the format written:

```sql
ATTACH 'session_snapshot.db' (STORAGE_VERSION 'v1.0.0');
```

Files written this way open in that version and every newer version. Pin the written storage version
to a conservative floor, upgrade engines freely, and forward compatibility stops being a risk class.
`duckdb_databases()` exposes storage-version tags, so a file can be inspected before opening.

**Do this from day one.** It costs nothing.

### 3.4 Remote attach is read-only — this is the decisive constraint

A `.duckdb` file **can** be attached directly over HTTPS/S3 without downloading. But DuckDB creates
**read-only** connections for remote endpoints by design. The documented partial-read optimization
(file metadata + HTTP range requests to fetch only needed bytes) is specified for **Parquet**, not
for remote `.duckdb` attach.

Sessions *write* — derived tables, cleaned data, materialized joins. A read-only remote attach
cannot serve a live session, so a `.duckdb` snapshot must be pulled whole to be useful. Parquet gets
genuine lazy partial reads over the network; the database file does not.

### 3.5 Resulting decision

- **Parquet in object store is canonical.** Only path with true streaming partial reads,
  engine-portable, and it preserves types so rebuilds are deterministic.
- **`.duckdb` snapshot is an optional warm cache** — written with a pinned `STORAGE_VERSION`, pulled
  whole on resume, with fallback to Parquet views if it is missing or unopenable. **Measure before
  building it.** For many sessions, views over remote Parquet will resume fast enough that the
  snapshot earns nothing.
- **Semantic layer lives in the application database**, not inside a DuckDB file. It is the state
  that cannot be lost, and it should be portable, versionable, and re-appliable to a rebuilt engine.
  This keeps the snapshot purely a performance optimization, discardable at any time without loss.

**Sources:**
- https://duckdb.org/2024/02/13/announcing-duckdb-0100
- https://duckdb.org/2025/02/05/announcing-duckdb-120
- https://duckdb.org/docs/current/internals/storage
- https://duckdb.org/docs/current/guides/network_cloud_storage/duckdb_over_https_or_s3
- https://duckdb.org/docs/current/sql/statements/attach
- https://duckdb.org/docs/current/core_extensions/httpfs/overview

---

## 4. The Semantic Layer

Everything true about the data that is **not** recoverable from column names and types alone.
It sits between raw schemas and the model.

Given `orders.csv`, `customers.csv`, `refunds.csv`, a raw schema gives the model names and types.
The semantic layer adds what it cannot know:

- **Relationships** — `orders.cust_id` → `customers.id`, many-to-one, 98% match rate
- **Grain** — one row per order *line*, not per order. Naive sums double-count.
- **Metric definitions** — "revenue" means `sum(qty * unit_price)` net of refunds, not `sum(amount)`
- **Business meaning** — `status = 'C'` means cancelled, not completed; `region` is null for online
  orders by design, not by error
- **Accumulated corrections** — user-supplied fixes that must persist across sessions

Without it, every session starts cold and the model re-guesses. With it, the model reads a curated
description and writes correct SQL.

Mechanically it is rows in the application database — relationships, metrics, annotations — keyed to
the workspace. It is where user edits from the interruptible pipeline (§2.6) accumulate.

**It is derived from the data and is therefore itself sensitive.** See §6.

---

## 5. Verification — The Hardest Problem

The model will occasionally write SQL that runs clean and answers the wrong question. Wrong join
grain, silent null exclusion, a filter that drops 30% of rows without saying so. No error is raised —
just a confident, well-formatted, wrong report.

**Reframe:** you cannot make the model always right. You can make wrongness *visible*. Layered
defense:

### Layer 1 — Determinism
Same question, same data, same SQL. Cache by question + schema hash. A report that silently changes
on re-run destroys trust faster than one that is wrong once.

### Layer 2 — Static SQL analysis before execution
Pure code, no model call. Catches a real fraction:
- **Join grain** — joining on a key non-unique on both sides causes fan-out, row multiplication, and
  inflated sums. Detectable from profiling stats.
- Aggregate-over-join without dedup
- Filters on nullable columns silently dropping NULLs
- Comparison of columns with incompatible units or scales
- Date ranges excluding recent data — usually a timezone or partial-period bug

### Layer 3 — Post-execution reconciliation *(highest leverage)*
Every query reports what it touched: 1.2M rows in, 340K after filters, 12 groups out. If a filter
drops more than a threshold, surface it unprompted: *"excluded 31% of rows where region was null."*

**Most wrong answers are wrong by omission, and omission is mechanically detectable.**

### Layer 4 — Shadow queries
For any aggregate, cheaply compute an independent cross-check — totals summed a second way, count
reconciliation against the unfiltered source. Disagreement means the grain is wrong. Cheap in DuckDB.

### Layer 5 — Adversarial second pass
A separate model call sees the question and the SQL — **not the results** — and asks "does this SQL
answer this question?" Catches semantic mismatch that static analysis cannot: the user asked for
*active* customers and the SQL omitted the status filter.

### Layer 6 — Claim-level provenance
Every number in the narrative traces to a specific cell in a specific result set. Click it, see the
query and the rows. **Non-negotiable** — it is also what prevents the model inventing numbers in
prose that appeared in no result set, which it will otherwise do.

### Layer 7 — Honest uncertainty
When a question is ambiguous — "top customers" by revenue or by count? over what period? — ask
rather than guess. The instinct is to always answer; resisting it is a feature.

**Note:** layers 2, 3, and 6 are deterministic code and deliver most of the value. Layer 5 is the
only one requiring an additional model call.

---

## 6. Privacy & Data Sensitivity

Users' data will often belong to their employer, or be personal data they cannot risk leaking. This
is a primary adoption barrier, not a compliance afterthought.

### 6.1 Derived state is sensitive state

The semantic layer is derived from the data and is **itself sensitive**. Relationship inference
requires sampling values. Profiling records distinct values, min/max, and top categories. If a column
holds patient IDs or salaries, the profile contains real values.

**It cannot be treated as "just metadata" and exempted from deletion.** The same applies to query
result caches, chart specs, generated reports, and conversation history — reports contain the actual
numbers.

### 6.2 One workspace, one delete

Deleting a workspace must destroy: object store contents, DuckDB snapshots, semantic layer rows,
profiling statistics, query caches, generated reports, conversation history, and logs.

Requirements:
- Every piece of derived state carries a workspace ID
- Deletion is a single cascade
- Deletion is **provable** — show the user what was deleted and when

If deletion is partial, or eventually-consistent in a way that cannot be described precisely, users
in regulated industries will not adopt. Design for this now; retrofitting deletion across a
distributed system with caches and queues is genuinely painful.

### 6.3 The model provider is the sharper problem

Deletion controls first-party storage. But schema, column names, sample values, and query results all
go to the LLM. That is the exposure users will actually ask about, and it cannot be fixed with a
delete button.

Mitigations that genuinely help:

- **Never send raw rows by default.** The two-stage design (§2.1) already provides this — the model
  plans from metadata, the engine returns facts. Aggregates crossing the boundary is materially
  different from rows crossing it.
- **Profile statistically, not by example.** "12,400 distinct values, high cardinality, string"
  rather than five sample values. Sample values improve inference — make them **opt-in per column**,
  not default.
- **PII detection at ingest.** Flag likely-sensitive columns (emails, national IDs, card patterns,
  names) and default them to name-and-type only. User can override.
- **Column-level exclusion.** A user marks a column as never leaving the boundary. It stays
  queryable in the engine but is invisible to the model.
- **Visible egress log.** Every payload sent to the model, viewable by the user. This is the
  observable-pipeline principle (§2.6) applied to privacy, and it converts nervous enterprise buyers
  because it turns a promise into something auditable.
- **Zero-retention provider terms**, stated plainly on the upload screen.

### 6.4 The honest framing

> Your data stays in our infrastructure and is deleted on command. *Descriptions* of your data and
> *aggregate results* go to the model under zero-retention terms — and you can see and restrict
> exactly what that includes.

This is defensible **and true**, which matters: "we never send your data to AI" is what competitors
claim and it is usually false.

---

## 7. Application Layer

### 7.1 Object model — three levels

Confirmed. Conflating any two of these causes downstream problems.

| Level | What it is | Lifetime |
|---|---|---|
| **Workspace** | Durable container. Owns the data (object store), the semantic layer, and the members. **The unit of deletion and the unit of privacy.** | Long-lived, user-managed |
| **Session** | A live engine attached to a workspace. Cost-bearing, killable, rebuildable. | Ephemeral; zero or many per workspace over time |
| **Conversation** | A thread of questions and reports within a workspace. The actual work product. | Survives session death entirely |

**The session is an implementation detail, not a user-facing object.** What users believe they own
is the workspace — their data plus everything built on it. If sessions are objects users must reason
about, the cost model has leaked into their mental model. Users create a workspace, upload, and
work; the engine spins up and down invisibly behind an honest status indicator.

**Conversation must outlive session**, or the compositional loop ("ask a question about a report")
breaks the moment the engine idles out.

### 7.2 Runs as resumable DAGs

A *run* is a directed graph of typed steps. Each step declares inputs and outputs as serializable
state, **persisted at every boundary**. Nothing lives only in memory.

A representative run:

```
profile → plan → generate_sql → static_check → execute
                                                  ↓
                              reconcile → shadow_check → adversarial_review
                                                              ↓
                                              chart_spec → narrate → assemble
```

Because every boundary is persisted, four capabilities come free rather than being built separately:

- **Interruption** — halt at step N; state before N is already on disk
- **Amendment** — mutate step N's input and re-run from N. Editing SQL means writing to
  `generate_sql`'s output and resuming at `static_check` — the same code path as a normal run, not a
  special case
- **Provenance** — claim-level provenance (verification layer 6, §5) is a byproduct of reading the
  persisted graph, not a separate feature
- **Determinism caching** — hash step inputs (verification layer 1, §5)

**Two rules to hold firmly:**

1. **Steps are pure functions of their persisted input.** No hidden reads of live engine state. If a
   step needs schema, the schema is *in its input*, versioned. Otherwise resume-after-edit produces
   different results than the original run, and nothing users report can be reproduced.
2. **Every step is individually retryable and idempotent.** Model calls fail; engines get killed
   mid-query. If the engine died between `execute` and `reconcile`, rehydrate and resume — do not
   restart the run.

**Runs are server-side state; the UI is a viewer attached to them.** Runs continue when the user
disconnects — closed browser, shut laptop, backgrounded mobile. Getting this wrong turns every long
report into a "don't close this tab" experience.

### 7.3 Fork on amendment

When a user edits step N on a run that has already completed step N+3, the run **forks**. The
original stays intact; the amendment creates a branch.

This makes report history a **tree, not a line**, and it means *"ask a question about a report to
generate another report"* is the **same mechanism** as amendment — one primitive, both features. It
also lets analysts explore alternatives without destroying prior work, which is most of the job.

### 7.4 Reports as first-class objects

Not chat messages. A report is a versioned DAG node that knows:

- Its run and the queries that produced it
- Its provenance links (claim → cell → query → rows)
- Its chart spec
- Its parent report, if derived

Consequences: re-runnable against updated data, shareable by ID, exportable standalone. If reports
exist only as chat scrollback, both export and the compositional loop become awkward.

### 7.5 Notifications — two systems, deliberately separate

| | Live run events | Durable notifications |
|---|---|---|
| **Frequency** | High | Low |
| **Scope** | Whoever is watching a run | The user, across sessions |
| **Storage** | Streamed, not stored | Persisted |
| **Examples** | Step started, SQL generated, rows returned | Export ready, engine slept, ingest failed, workspace invite |

The common failure is building the live stream first and bolting durable notifications on as "unread
events" — which produces a notification store full of noise.

### 7.6 Ingest — the highest-risk UX in the product

Everything downstream is grounded in it, and it is where a non-technical user is most likely to
abandon.

```
upload → parse & type-infer → profile → PII flag → relationship inference → CONFIRMATION
```

**The confirmation step is not optional.** The user confirms inferred types, reviews flagged PII
columns (§6.3), and accepts or rejects candidate joins (§2.9). This is simultaneously where the
semantic layer gets seeded correctly and where privacy defaults get set — and it largely determines
the model's later accuracy.

Make it skippable with sane defaults, but never silent. Surface the cost of getting it wrong at that
moment, not later.

### 7.7 Collaboration — read-only sharing for v1

Share a workspace, or share a single report. Viewers see reports and provenance; they do not mutate
the semantic layer.

Concurrent semantic-layer editing is a CRDT-shaped concurrency problem and is not where the value
is. Report sharing by ID is likely the more valuable half regardless — it is how analysis spreads
through an organization.

### 7.8 Metering from day one, billing later

Three real costs, recorded per workspace whether or not anyone is being charged:

- **Engine-seconds**
- **Object-store bytes**
- **Model tokens** (in and out)

Without these numbers it is impossible to set an idle-teardown threshold (§2.5), price a tier, or
identify unprofitable usage. Cheap to add now; archaeology later.

### 7.9 Audit log = egress log

Who queried what, what payload left to the model, who exported, who deleted. **This is the same
object as the model-egress log in §6.3 — build one thing.** It is what makes the privacy claim
auditable rather than merely promised.

### 7.10 Export — structured, multi-target

PDF is the first target, **not the only one**. Because reports are first-class structured objects
(§7.4) rather than rendered blobs, export is a rendering concern over a stable internal
representation, and additional targets are incremental.

Plausible targets:

| Target | Notes |
|---|---|
| **PDF** | First target. Narrative + rendered charts, print-oriented |
| **Share link** | Live, provenance-inspectable, re-runnable. Near-free given §7.4 |
| **Excel / CSV** | Underlying result sets, not the narrative. Analysts will expect this |
| **PowerPoint / slides** | Charts + narrative as decks. High demand in finance contexts |
| **Markdown / HTML** | Embedding into wikis, docs, internal reports |
| **JSON** | Structured report + provenance, for programmatic consumers |
| **Image** | Single chart export for pasting elsewhere |

**Design implication:** do not let PDF generation become the canonical rendering path. Export targets
consume a shared structured report representation; PDF is one renderer among several. Scheduled and
re-run exports also fall out of this nearly free.

### 7.11 The engine-sleeping experience

A user returns to a workspace after two hours and asks a question. This is the most common cold path
in the product.

#### Where the time actually goes

| Stage | Cost | Notes |
|---|---|---|
| Provision compute | 200ms – 30s+ | Dominated by infrastructure choice |
| Process start + extension load | ~100ms | Trivial |
| Attach Parquet views | ~100ms – 1s | Metadata reads only — **independent of data size** |
| Materialize working set | 0 – minutes | Only if materialization happens at all |

**Key insight: the expensive stage is provisioning compute, not loading data.** Views over remote
Parquet are metadata-only, so a 5GB dataset attaches about as fast as a 50MB one. The intuition
"big data means slow wake" is wrong, and optimizing data loading optimizes the wrong stage.

**Corollary: do not materialize on wake.** Attach views lazily and let the first query pull only
what it touches. Materializing "to be ready" reintroduces the size-dependent cost just avoided.

#### Most of the product needs no engine

Reading past reports, inspecting provenance, viewing cached results, browsing the semantic layer,
exporting, and sharing all work without DuckDB — because reports are first-class objects with
persisted results (§7.4).

**A sleeping workspace must therefore be fully browsable.** The engine is needed only for *new*
queries. This reframes the problem from "the workspace is asleep" to "the next new question costs a
moment" — a much smaller problem.

#### Wake on intent, not on submit

Trigger rehydration when the user signals intent to query — opening the workspace, focusing the
question input, beginning to type. By the time a question is typed, the engine is ready. Most
perceived latency disappears here, and an unused wake is cheap.

Input-focus is nearly free and almost always correct; workspace-open is more aggressive and fires
more often unnecessarily.

#### Honest status, non-blocking

Show engine state (`sleeping` / `waking` / `ready`) rather than hiding it:

- A silent 3-second delay reads as *the product is slow*; a labeled 3-second wake reads as *the
  system is working*, and users forgive it
- It teaches the cost model, which supports the pricing story
- It makes the "compute is torn down when idle" claim visible rather than asserted

**Never block on it.** Accept the question immediately, queue it, and show `waking` as the first
step of the run. Because the pipeline is already step-wise and observable (§7.2), the wake is just
another visible step — no special-case loading UI.

#### Teardown policy

- Key the idle timer off **engine** idleness, not user presence — reading reports is not engine
  activity
- Start at ~10 minutes; tune from metering data (§7.8)
- **Never tear down mid-run**
- **No "your session expired" warnings.** Warnings imply required user action. Nothing is lost, so
  users must not be made to manage it — that is exactly the leak of the cost model into the user's
  mental model that §7.1 exists to prevent

#### Failure modes

- Snapshot unopenable (version mismatch, missing) → fall back to Parquet views **automatically and
  silently**
- Transient failure → retry transparently; the wake step is idempotent (§7.2)
- Genuine failure → fail the **run** with a retry affordance, not the **workspace**. The user's data
  and reports are untouched, and the UI must make that obvious

#### Warm pools — deferred, not skipped

At scale, the fix for provisioning cost is a **warm pool**: a small number of generic, empty engines
kept running and shared across the whole userbase. Waking a workspace becomes "take an idle engine,
attach views" — sub-second and independent of dataset size. The boot cost is paid in advance and
amortized across all users, so it improves with scale.

**Not needed initially** (see §7.13). Warm pooling solves the cost of provisioning *machines*; at
low scale, sessions are *processes*, which start in milliseconds. Deferred deliberately.

### 7.12 Infrastructure — start minimal, keep the seam

**Do not build engine orchestration yet.** DuckDB is an embedded library, not a server.

**Constraint:** pre-revenue. Infrastructure must **scale to zero** — an always-on VPS bills the same
whether one person or nobody uses it. Every component needs a scale-to-zero story, or a monthly
floor remains regardless.

**Starting configuration** *(pricing verified 2026-08-04)*:

| Concern | Choice | Cost at zero usage | Rationale |
|---|---|---|---|
| Compute / engines | **Fly.io Machines**, auto start/stop | Stopped machines bill **rootfs only, $0.15/GB-month** — no CPU or RAM | Per-second billing; a running shared-CPU 256MB machine is ~$1.94/mo |
| App database | **Neon** (serverless Postgres) | $0 on free tier | Scales to zero after 5 min idle. Free tier: 100 CU-hours/mo, 0.5GB/project, no card |
| Object storage | **Cloudflare R2** | Pennies, pay-per-use | S3-compatible (httpfs works unchanged) with **zero egress fees** |
| Engine unit | One OS process per active session | — | Millisecond starts; memory and crash isolation between tenants |

**Realistic total: $0–5/month idle**, single-digit dollars at light usage.

**Why Fly.io rather than a VPS:** it avoids committing to a phase. Fly Machines stop and start on
demand, and the *same primitive* serves both stages — today they run auto-stop with scale-to-zero;
later, a warm pool (§7.11) is simply "keep N machines started." No migration. A VPS forces a
platform move eventually; Fly does not.

**Trade-off accepted:** per-second billing becomes *more* expensive than a VPS under steady load.
That is the correct problem to have — by then there is revenue and real usage data.

**R2 over S3 is close to a no-brainer for this architecture.** The design reads Parquet over the
network constantly and rebuilds engines from object storage repeatedly. On S3 that egress is a real
and unpredictable bill; on R2 it is free.

**Process-per-session** gives genuine tenant isolation — important for §6, since tenants must not
share an address space.

#### Fly.io vs. Cloud Run — decided, with reasons

Cloud Run was evaluated and **rejected for the engine layer**. Verified 2026-08-04.

| Requirement | Fly.io | Cloud Run |
|---|---|---|
| Route to a **specific** instance holding workspace state | Machines have stable IDs; `fly-replay` targets one | **Best-effort only.** Session affinity is a cookie-based routing *hint*; a request may be served by a different instance |
| Work continuing after the user disconnects (§7.2) | Normal | CPU throttled to ~zero between requests by default |
| Disk for DuckDB spilling | Persistent volumes | In-memory filesystem, counts against RAM |

**The disqualifying issue is addressability.** The engine holds live state — attached views, derived
tables, materialized joins. `EngineProvider.acquire()` must return a handle that reliably reaches
*the same process*. Cloud Run explicitly does not guarantee this: if the affinity target cannot
serve the request, it is served elsewhere and the cookie is rewritten. That produces silent cold
rebuilds or missing state.

**The CPU issue is fixable but self-defeating.** `--no-cpu-throttling` enables background work, but
then billing covers the **entire instance lifetime** rather than per-request — surrendering the
elasticity that motivated Cloud Run in the first place. (Always-allocated pricing is ~25% lower on
CPU and ~20% on memory, so cost alone does not rule it out; addressability does.)

**Where Cloud Run would win:** a split deployment — stateless API on Cloud Run, engines on Fly.
Google's free tier is more generous than Fly's, which no longer exists. Rejected as unnecessary
operational overhead for a solo builder; Fly serves the stateless portion adequately.

#### Caveats to track

- **Fly cold start is "a few seconds," not sub-second.** Application startup (framework boot,
  connection pools) adds to it. Mitigated by wake-on-intent (§7.11), but it must be measured, not
  assumed.
- **Fly's free tier is gone** (pay-as-you-go since Oct 2024). Costs are low but never zero.
- **Neon free-tier limits suspend compute until the next billing cycle** when hit — a hard stop, not
  a throttle. Monitor CU-hours, and treat the free tier as a starting point rather than a
  destination.

#### The seam that makes this safe

```
EngineProvider:
  acquire(workspace_id) → EngineHandle
  release(handle)
```

Local process spawning is implementation #1. A warm pool of processes is implementation #2. Remote
machines are #3. As long as application code only ever asks for "an engine for this workspace,"
changing strategy is contained rather than a rewrite.

**This is how warm pooling is deferred safely: not skipped, just not implemented.**

#### When to revisit

- One machine can no longer hold concurrent sessions
- Per-tenant isolation stronger than processes is required
- A large idle machine is being paid for overnight

Then move from auto-stop machines to a warm pool of started machines (§7.11) — the same Fly
primitive, a different policy. **Not before.** Nothing built in the meantime needs to be discarded.

### 7.15 Language and framework

#### Backend — Go

Good fit for the shape of this system: process supervision, long-running jobs, streaming events,
concurrent runs, and cancellation. The interruptible DAG orchestrator (§7.2) maps naturally onto
Go's concurrency and context-cancellation primitives.

**The DuckDB driver risk has decreased.** The Go driver moved under the official
`duckdb/duckdb-go` organization; `marcboeker/go-duckdb` is deprecated in favor of it. First-party
maintenance de-risks the single most important dependency in the system.

**Constraint:** the driver is cgo. Builds require `CGO_ENABLED=1` and a C compiler, and fully static
binaries are not available. Slightly heavier container builds; not a blocker.

Anthropic ships an official Go SDK, so the AI layer does not force Python.

**Do not split Go + Python.** Polyglot costs a solo builder more than the library gap saves, and
DuckDB handles most of the data work Python would otherwise be reached for — including reading
Excel.

#### Frontend — Next.js / React

**Decided.** Angular was the initial proposal; switched to Next.js/React.

The UI is chart-heavy, streaming, document-rendering, and export-heavy (§7.4, §7.10). React's
ecosystem for exactly those concerns — charting, markdown/report rendering, streaming UIs, PDF
generation — is substantially deeper, so less gets written from scratch. Angular's core strength is
imposing structure on large teams, which is not the constraint here.

#### Boundary rule: Go owns the API, Next.js owns presentation

**Go is the only backend.** Next.js must not accumulate a second backend in its route handlers.

This matters because Next.js makes it easy to drift: an API route here, a server action there, and
suddenly business logic is split across two languages with two deployment lifecycles. Every rule in
§7.2 — persisted step state, idempotent retries, server-side runs surviving disconnect — assumes a
single orchestrator. Splitting it silently breaks those guarantees.

Permitted in the Next.js layer: rendering, routing, auth session handling, and BFF-style proxying to
Go. Not permitted: run orchestration, engine access, semantic-layer mutation, or anything touching
DuckDB.

#### Rendering model

The core surfaces — the run viewer, chart rendering, report exploration — are **client-heavy and
streaming**, not document-oriented. Server rendering earns little there.

- Use SSR/SSG for marketing, auth, and static shell
- Keep workspace, run viewer, and report surfaces client-side, streaming from Go
- Static export is viable if the Node server proves to be more operational cost than it is worth

#### Backend alternatives considered

Evaluated and rejected. Recorded so the decision is not reopened without new information.

| Stack | Argument for | Why rejected |
|---|---|---|
| **TypeScript everywhere** (Node/Bun) | One language; **shared types** for chart specs, report objects, provenance, step state — an unusual amount of structured data crosses this boundary. DuckDB's Node Neo client (`@duckdb/node-api`) is the official supported path. | Weaker at long-lived process supervision and cancellable CPU-bound work — the actual hard part of §7.2. Type duplication is tractable via generation; the supervision gap is not. **The closest competitor.** |
| **Python (FastAPI)** | Strongest AI/data ecosystem | The ecosystem argument does not apply — the system makes HTTP calls to a model API and lets DuckDB do the data work. Concurrency and process supervision are Python's weakest areas and this system's core need. |
| **Elixir/Phoenix** | Conceptually the best fit: supervision trees, per-session processes, native live event streaming, crash isolation. Would eliminate meaningful code. | Weakest DuckDB support of the candidates, and a steep curve unless already known. Worth revisiting only by someone already fluent. |

**Mitigation for Go's one real weakness:** generate TypeScript types from Go structs, or define shared
contracts (chart spec, report object, step state) in JSON Schema and generate both sides. Do not
maintain them by hand.

#### Durable orchestration — Postgres, not Temporal

**Decided.** The riskiest component is not the web framework but the durable orchestration layer
behind §7.2. Temporal was evaluated and rejected.

| Dimension | Temporal | Postgres (self-built) |
|---|---|---|
| Durable execution, retries, timers | Excellent, battle-tested | Must be built |
| Where step state lives | Temporal's datastore | **The app database, directly queryable** |
| Cost at current scale | Cloud is paid; self-hosting is a real cluster | **$0 — Neon already exists** |
| Workers | **Long-poll continuously → always-on compute** | Event-driven |
| Fork-on-amendment (§7.3) | Awkward — reset/continue-as-new are not branching | **Natural — a row with a parent pointer** |
| Deletion cascade (§6.2) | A second store to purge | **One store, one cascade** |
| Correctness risk | Low — solved by the platform | **Real — leases and idempotency must be correct** |

**Three decisive points:**

1. **Temporal workers break scale-to-zero.** Workers long-poll the Temporal service, requiring
   continuously running compute — precisely the always-on floor §7.12 was designed to avoid, plus
   Temporal Cloud cost on top, at zero users.
2. **Step state is product data, not workflow internals.** Provenance (§5, layer 6) means a user
   clicks a number and sees the query and rows behind it. Reports reference step outputs;
   determinism caching hashes step inputs. That data must join against workspaces, reports, and the
   audit log. In Temporal it would be mirrored into Postgres anyway — two systems storing one thing.
3. **Fork-on-amendment is the core primitive and Temporal does not model it.** Temporal workflows
   are linear executions with resets, not branchable DAGs.

**Accepted cost:** this means building a small workflow engine, and hand-rolled job systems fail in
known ways — lost updates, stuck jobs, double execution. Scope is bounded because the DAG is small
and known in advance. Required pieces:

- `runs` and `steps` tables, `parent_step_id` for forks, step state as JSONB
- Work claimed via `SELECT ... FOR UPDATE SKIP LOCKED`
- **Leases with heartbeats** — a worker holds a step and renews it; death reclaims it. This is the
  commonly skipped piece that produces stuck jobs.
- Idempotency keys per step so retries cannot double-execute
- Retry counts and backoff stored on the row

Provenance, determinism caching, forking, and workspace-scoped deletion then come nearly free,
because all of it is one schema.

**Implementation note:** do **not** tight-poll Postgres for work — it keeps Neon permanently awake
and burns CU-hours (§7.12). Use `LISTEN/NOTIFY` for immediate wakeup, plus a slow safety-net poll to
catch missed notifications.

**Revisit only if:** workflows span weeks, human-in-the-loop approvals span days, cross-service sagas
appear, or the team grows enough that owning an orchestrator is a real role. None apply to this
product.

#### Deployment

Next.js can deploy on Fly.io alongside Go (one platform, one bill, no cross-origin concerns), or on
Vercel (better DX, another vendor). **Fly is preferred** for consistency with §7.12, unless Vercel's
tooling proves decisive.

### 7.13 Snapshot policy for expensive workspaces — deferred

Most workspaces are cheap to wake (attach Parquet views). A workspace where the user spent
significant time building derived tables and materialized joins is not — that work lives *in the
engine*, and rebuilding it could take minutes.

**Agreed direction:** one simple uniform rule first, measure real rebuild times from metering
(§7.8), and add adaptive policy — keep expensive workspaces alive longer, or write a `.duckdb`
snapshot (§3.5) before teardown — only if the data shows it is needed.

This is the condition under which the `.duckdb` snapshot earns its complexity. Until measured, it
should not be built.

### 7.14 Auth

Standard. The only non-obvious intersection is §7.9 — the audit log ties identity to data access and
model egress, which is what makes the privacy posture verifiable.

---

## 8. Open Questions

Unresolved. These change downstream product decisions and should be settled during shaping.

### Product
1. **Product name.** Nothing chosen.
2. **Primary audience.** "Data analyst" and "anybody with data" want different products. The analyst
   wants to see and edit SQL; the business user wants to never know SQL exists. Both can be served,
   but the *default surface* differs. Note: the observable/interruptible pipeline (§2.6) mainly pays
   off for a user who understands what they are interrupting — which biases toward analyst-first.
3. **Deployment model.** Cloud SaaS, SaaS-now/self-host-later, or self-host from day one. This
   determines the entire privacy posture and the monetization model.

### Technical
4. **Idle-teardown threshold**, and whether teardown is signalled or silent.
5. **Expected data size ceiling.** 10MB spreadsheets and 5GB Parquet are different products.
6. ~~Is the `.duckdb` snapshot worth building?~~ **Resolved (§7.13):** deferred. Build only if
   measured rebuild times justify it, for expensive workspaces only.
7. **Concurrency model** — how many live engine processes per user and per host, and what happens
   when the host is saturated.
8. **Cost model per session** — drives teardown aggressiveness. Requires metering data (§7.8).

### Application layer
9. ~~Engine-sleeping experience~~ **Resolved (§7.11, §7.12).** Remaining sub-question: the exact
   wake-on-intent trigger (input-focus vs. workspace-open) should be tuned against real usage.
10. **Export target priority beyond PDF** (§7.10). Excel/CSV is the likely second for analysts;
    slides may matter more in finance contexts. Needs audience resolution (Q2) first.
11. **Run retention** — how long persisted DAG state is kept. It is both the provenance substrate
    and a storage cost, and it is workspace-scoped sensitive data subject to §6.2 deletion.
12. **Fork proliferation** (§7.3) — a tree of report branches needs pruning, naming, or collapsing
    at some point. Unaddressed.

### Stack
14. ~~Frontend framework~~ **Resolved (§7.15): Next.js / React.** Stack is settled end to end —
    Go backend, Next.js frontend, Fly.io, Neon, R2, DuckDB. Remaining sub-question: whether the
    Next.js Node server is worth its operational cost, or static export suffices.
15. **Fly.io cold-start latency measured end to end**, including application boot — the "a few
    seconds" figure is platform-level only, and wake-on-intent (§7.11) is sized against it.

### Scope
13. Remaining application concerns not yet discussed: onboarding, workspace templates, admin/team
    management, error and empty states, mobile support.

---

## 9. Notes for the Shaping Pass

- §2.1 (two-stage AI), §5 (verification), and §6 (privacy) are the **differentiators**. Verification
  and privacy are meaty enough to deserve standalone documents rather than sections buried inside
  `architecture.md`.
- Every product in this space either solves verification or quietly ships it. It is the defensibility
  argument.
- §3 findings are verified against live sources, not recalled. Re-verify before implementation if
  significant time has passed.
- §7 is a **first draft of the application layer**, discussed and agreed at the design level but not
  specified in detail. Screens, states, and flows are still open.
- Several §7 decisions are load-bearing and should not be quietly revised during shaping:
  the three-level object model (§7.1), persisted-DAG runs (§7.2), and fork-on-amendment (§7.3).
  Each was chosen because it collapses several features into one mechanism — undoing any of them
  re-fragments the design.
- **Some things are deferred by decision, not by oversight.** Warm pools (§7.11), the `.duckdb`
  snapshot (§7.13), and multi-machine orchestration (§7.12) are all deliberately unbuilt until
  measurement justifies them. A shaping pass should not list them as gaps or pull them into an early
  roadmap phase. The mitigation for all three is the `EngineProvider` seam (§7.12), which keeps them
  contained changes rather than rewrites.
- **The riskiest component is the Postgres-based orchestrator** (§7.15), not the UI or the AI layer.
  It is where a project of this shape most easily loses months. It should be built early, on its
  own, with tests for lease expiry, retry idempotency, and fork semantics — before any feature
  depends on it.
- **Cost discipline is an explicit constraint.** The user is pre-revenue and does not want
  infrastructure spend ahead of demand. Target is under ~$100/month until real usage exists. Any
  recommendation that violates this should be flagged rather than assumed.
- §7.9 and §6.3 describe **one object**, not two. The audit log and the model-egress log must not be
  specified separately.
