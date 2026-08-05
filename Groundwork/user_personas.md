# User Personas

**Last Updated:** 2026-08-04
**Source:** [[README]] — Target Audience & SWOT

## Overview

Groundwork serves two structurally different users on the same product, deliberately without a default-surface bias (see [[README]]): people who want to see and control the SQL behind an answer, and people who never want to know SQL exists. The key segmentation dimension is **SQL comfort and desired control over the query**, not company size or industry — a solo analyst and an FP&A lead at a mid-size company can both be the "trusts the narrative" persona, and a self-taught spreadsheet power user can behave like the "wants the SQL" persona. A third, hybrid persona sits between them: someone who currently does the job of both by hand — pulling numbers in spreadsheets and separately waiting on an analyst — because no single tool serves both needs today.

## Persona 1: Chidinma, the Data Analyst

**Tagline:** "I don't want an answer I can't check — I want the query, and I want to fix it when it's wrong."
**Demographic:** 29, based in Chicago, BS in Economics, 5 years in analytics roles
**Role:** Senior Data Analyst at a mid-size logistics company (~400 employees)
**Tech Comfort:** High

### Goals
- Get from "raw files dumped in a shared drive" to a working, joinable dataset without hand-writing every join herself
- See and edit the exact SQL behind any number before she'll put it in a deck for leadership
- Reuse and correct persistent business logic (e.g., what "revenue" means net of refunds) instead of re-deriving it every session
- Produce reports she can hand off to non-technical stakeholders without a follow-up meeting to explain the numbers

### Frustrations
- Currently spends a disproportionate amount of time on multi-file joins — matching `orders.cust_id` to `customers.id`, guessing cardinality, re-discovering the same relationships every time a new export lands
- Distrusts general chatbot-style "chat with your data" tools because they've quietly given her a plausible-sounding wrong number before, with no way to check it
- Has to re-explain the same business rule (a status code, a null convention) to a junior analyst or a tool every time, because nothing remembers the correction
- Worried about pasting anything with real customer or transaction data into a tool she doesn't control the egress of

### Behaviors
- Lives in SQL editors and BI tools (Metabase, dbt, occasionally raw psql) most of the workday
- Pulls raw exports (CSV/Excel/SQLite dumps) from source systems by hand and stages them before building anything
- Reviews query output row-by-row before trusting a summary number, especially before a filter is applied
- Desktop-only during work hours; would not use a mobile surface for this kind of work

### Scenario
Chidinma gets three new CSV exports — orders, customers, and refunds — ahead of a Friday leadership review. In her current workflow she'd spend an hour guessing at the join keys and writing SQL from scratch. In Groundwork, she uploads all three files, reviews the candidate joins the platform infers (with confidence scores) and confirms the two it got right, then asks a natural-language question. She reads the generated SQL before trusting the output, notices it excludes refunds from one but not the other, edits the query inline, and re-runs from that step. The corrected join logic is saved so it doesn't have to be re-taught next session, and she exports the resulting report as a PDF for the review.

## Persona 2: Tunde, the Small-Business Owner

**Tagline:** "I have a spreadsheet and a question. I don't want to learn SQL to get an answer."
**Demographic:** 44, based in Austin, no formal data/technical background, runs a 12-person specialty retail business
**Role:** Owner/operator, does his own bookkeeping and reporting
**Tech Comfort:** Low

### Goals
- Upload the spreadsheet he already has (sales export from his POS system, or a bank statement CSV) and just ask a question in plain English
- Get a clear narrative answer with a chart he can screenshot and send to his accountant or business partner
- Trust that a number Groundwork gives him is actually correct before he repeats it to a lender or investor
- Never be shown SQL, a schema, or anything that looks like a database tool

### Frustrations
- Has tried general AI chatbots with a spreadsheet attached and gotten answers that sounded right but turned out to be wrong when he checked by hand — now he double-checks everything, which defeats the purpose
- Doesn't know what a "join" or a "schema" is and shuts down when a tool asks him to configure one
- Worried, in a vague but real way, about uploading his sales and customer data somewhere and not knowing where it goes
- Has been burned by a BI tool that required a dashboard-building tutorial before he could get a single number out of it

### Behaviors
- Works from a laptop for anything involving a spreadsheet, but would check a report on his phone if a link was texted to him
- Currently keeps most business questions in his head or asks his bookkeeper by email and waits days for an answer
- Exports things to PDF to print or attach to email; has never used Excel/CSV export from a BI tool because he doesn't need the underlying rows, just the answer
- Trusts a number more when it's explained in a sentence than when it's shown as a raw table

### Scenario
Tunde wants to know whether his slow month was actually slow, or just felt that way. He uploads his POS sales export CSV to Groundwork and asks, "Was this month slower than last month, and why?" He never sees SQL — he sees a short narrative explaining that revenue was down 8%, driven mostly by a drop in weekday afternoon traffic, next to a bar chart comparing the two months. Because he's been burned before, he clicks the number in the narrative and sees exactly which transactions it's based on before he mentions it to his business partner. Satisfied, he exports the report as a PDF to attach to an email.

## Persona 3: Ngozi, the FP&A Generalist

**Tagline:** "I'm the one who builds the spreadsheet AND waits on the analyst — I need something that does both jobs."
**Demographic:** 35, based in Denver, MBA, 8 years in finance/operations roles
**Role:** FP&A Manager at a ~150-person company, reports to the CFO
**Tech Comfort:** Medium

### Goals
- Get from "several exports from different systems" (ERP, CRM, a manual tracking spreadsheet) to a combined report without opening a ticket with the data team
- Occasionally look at the underlying query or logic when a number looks surprising, without needing to write SQL herself
- Produce board-ready and finance-team-ready outputs from the same underlying analysis — a deck for the board, a detailed Excel breakdown for her own team
- Build up a reusable, correct definition of business metrics (e.g., what counts as "active" revenue) that doesn't have to be re-argued every quarter

### Frustrations
- Spends real time each month manually reconciling numbers across a spreadsheet she maintains and the ad hoc reports the data team sends her — they don't always agree, and she can't tell why
- Currently the bottleneck between "I have a finance question" and "someone with SQL access answers it," which means simple questions take days
- Has been given wrong or misleading numbers from spreadsheet formulas before (double-counted rows from a bad join/lookup) and only caught it because a total looked implausible — she's aware this kind of silent error is her biggest risk
- Needs to send different levels of detail to different audiences (board summary vs. team-level detail) and currently rebuilds the same analysis twice by hand for that

### Behaviors
- Splits time between Excel/Google Sheets, a BI dashboard she didn't build and only partially trusts, and email threads chasing the data team
- Works primarily on desktop during business hours, but checks dashboards on mobile before a meeting
- Cross-checks big numbers against a second source before presenting them — an informal version of the reconciliation Groundwork automates
- Exports to both PDF (for the board) and Excel/CSV (for her own team's working files) from whatever tool she uses

### Scenario
Ahead of a board meeting, Ngozi needs to reconcile revenue across her ERP export and a manual adjustments spreadsheet her team maintains. She uploads both files to Groundwork; the platform flags a likely join between them and shows its confidence, which she confirms after a quick sanity check. She asks for a revenue trend report, and because the platform surfaces that a filter excluded 12% of rows with a null region, she catches an issue she would otherwise have missed. She doesn't touch SQL herself but does open the "what happened" view once to understand why the number moved from her last report. She exports a clean summary as PDF for the board and the full result set as Excel for her own team's records.

## Persona Comparison

| Need / Behavior | Chidinma (Analyst) | Tunde (Small-Business Owner) | Ngozi (FP&A Generalist) |
|---|---|---|---|
| Primary goal | Verify and control the exact query behind a number | Get a trustworthy plain-English answer, fast | Reconcile multiple sources into one correct, reusable report |
| SQL visibility desired | Always visible, wants to edit it | Never shown | Available on demand, rarely opened |
| Device preference | Desktop only | Desktop primarily, mobile for viewing | Desktop primarily, mobile for viewing |
| Usage frequency | Several times a week | Monthly / as-needed | Weekly, spikes around month-end and board cycles |
| Price sensitivity | Medium (expensed as a work tool) | High (small-business budget) | Low-medium (departmental budget) |
| Trust mechanism | Reads the SQL and row counts directly | Clicks a number to see the source, then trusts the narrative | Uses reconciliation/omission surfacing as an early-warning check |
| Export needs | PDF for leadership, occasional Excel for handoff | PDF for email/print | PDF for board, Excel/CSV for team detail |
| Key frustration today | Manual, repeated multi-file join work | Can't trust "chat with your data" tools that have been wrong before | Manually reconciling disagreeing numbers across tools |
| Value of interruptible/observable pipeline | High — this is a primary reason she'd choose Groundwork | Low to none — she never wants to see a pipeline step | Medium — mainly valued as a trust/verification signal, not a workflow she actively drives |

## Anti-Personas

**The Enterprise Data Platform Buyer** — an IT/data-platform leader evaluating Groundwork as a replacement for a full enterprise BI stack with live production-database connectivity, on-prem deployment, and SSO/role-based governance across thousands of seats. Groundwork's file-based-only ingestion and cloud-SaaS-only deployment ([[README]] — Scope) are explicit non-goals for this buyer's requirements, and chasing this persona would pull the product toward the live-OLTP and self-host work the README rules out.

**The Real-Time Collaborative Team Analyst** — someone who wants multiple people simultaneously editing the same semantic layer or dashboard live, the way a team co-edits a shared spreadsheet or Figma file. [[README]] scopes collaboration to read-only sharing for v1 explicitly because concurrent semantic-layer editing is a CRDT-shaped problem outside where the product's value concentrates; building for this persona now would misallocate early effort away from the core two-stage grounding and verification differentiators.
