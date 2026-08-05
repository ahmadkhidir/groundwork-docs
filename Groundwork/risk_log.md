# Risk Log

**Last Updated:** 2026-08-04
**Source:** [[README]] (SWOT — Weaknesses & Threats), [[architecture]], [[monetization]], [[features]]

Groundwork is built and operated by a single, pre-revenue owner ([[README]]). Every risk below is owned by that same role — **Founder / Solo Builder** — since there is no separate Product Manager, Tech Lead, or Legal function to delegate to yet. Owner is recorded as a role per the standard convention, and should be re-assigned to a named function the moment the team grows past one person.

## Risk Assessment Scale

### Likelihood

| Score | Label | Definition |
|---|---|---|
| 1 | Rare | <5% chance in the project timeline |
| 2 | Unlikely | 5-20% chance |
| 3 | Possible | 20-50% chance |
| 4 | Likely | 50-80% chance |
| 5 | Almost Certain | >80% chance |

### Impact

| Score | Label | Definition |
|---|---|---|
| 1 | Negligible | Minor inconvenience, easily absorbed |
| 2 | Minor | Some disruption, manageable with existing resources |
| 3 | Moderate | Significant disruption, requires dedicated effort to resolve |
| 4 | Major | Threatens timeline or budget materially |
| 5 | Critical | Threatens product viability or company reputation |

### Risk Score

**Risk Score = Likelihood × Impact**

| Score Range | Severity | Action |
|---|---|---|
| 1-4 | Low | Monitor |
| 5-9 | Medium | Plan mitigation, monitor regularly |
| 10-19 | High | Active mitigation required, escalate to stakeholders |
| 20-25 | Critical | Immediate action required, may require product pivot |

## Risk Register

### Market Risks

Risks related to target audience, competition, and product-market fit. Sourced from [[README]] Threats and Weaknesses.

| ID | Risk | L | I | Score | Mitigation | Contingency | Owner |
|---|---|---|---|---|---|---|---|
| R-001 | Competitors market a simpler "just chat with your data" positioning that undercuts Groundwork's more careful, more complex pitch on perceived simplicity, before a buyer has been burned by a wrong answer. | 4 | 3 | 12 | Lead marketing and onboarding with a concrete, visible demonstration of claim-level provenance ([[features]] FEAT-030) rather than an abstract trust claim — let a prospect click a number and see it verified in the first five minutes. | If a specific competitor's simpler positioning is winning deals, add a side-by-side comparison surfacing a real wrong-answer example (with permission) or a live "verify this" demo to sales/marketing materials. | Founder / Solo Builder |
| R-002 | Foundation model providers ship native "connect your data" agentic features directly into general chat products, commoditizing the core two-stage interaction pattern from underneath a standalone product. | 3 | 4 | 12 | Treat the verification layer stack ([[features]] FEAT-027-033) and claim-level provenance as the durable moat, not the two-stage architecture alone — a general chat product bolting on data connection is unlikely to also ship reconciliation, shadow queries, and adversarial review. Track major model providers' product announcements quarterly. | If a provider ships a directly competing feature, accelerate differentiation on verification depth and the observable/interruptible pipeline ([[features]] FEAT-022-024) in messaging and roadmap priority. | Founder / Solo Builder |
| R-003 | A single publicized incident of a confidently wrong AI-generated number — even from a competitor, not Groundwork itself — raises category-wide buyer skepticism and increases the sales burden of proving verification claims are real. | 3 | 3 | 9 | Keep the egress/audit log ([[features]] FEAT-042) and provenance drill-down easy to demo unprompted in any sales or onboarding conversation, so "prove it" has a 30-second answer ready before it's asked. | If a category-damaging incident occurs, proactively publish how Groundwork's specific verification layers would have caught or flagged the same failure mode. | Founder / Solo Builder |
| R-004 | Serving two structurally different default surfaces (Chidinma's SQL-visible, control-oriented mode and Tunde's narrative-only, SQL-hidden mode) on one product without a default-surface bias risks a diluted experience if both aren't executed to the same quality bar. | 3 | 4 | 12 | Explicit persona-level acceptance criteria per surface in [[features]] and [[user_personas]]; usability-test each surface independently against its own persona rather than testing once and assuming coverage of both. | If usage data or feedback shows one surface (e.g., the analyst SQL view) is clearly under-serving its persona, prioritize a dedicated hardening pass for that surface in the next [[roadmap]] phase rather than continuing to split effort evenly. | Founder / Solo Builder |

### Technical Risks

Risks related to implementation, infrastructure, and technical dependencies. Sourced from [[README]] Weaknesses and [[architecture]].

| ID | Risk | L | I | Score | Mitigation | Contingency | Owner |
|---|---|---|---|---|---|---|---|
| R-005 | The hand-rolled Postgres-based durable orchestrator ([[architecture]] — chosen over Temporal) hits known failure modes of self-built job systems: lost updates, stuck jobs, double execution. | 3 | 4 | 12 | Implement leases with heartbeats, idempotency keys per step, and retry counts/backoff on the `steps` row from day one, as specified in [[architecture]] — not retrofitted later. Write dedicated tests for lease expiry, retry idempotency, and fork semantics before any feature depends on the orchestrator, per the source spec's explicit build-order guidance. | If stuck jobs or double-execution are observed in production, add a manual step-recovery admin action and instrument lease-expiry metrics to catch recurrence before users notice. | Founder / Solo Builder |
| R-006 | Fly.io cold-start latency ("a few seconds," platform-level only per public docs) has not been measured end-to-end including application boot, so the wake-on-intent UX ([[features]] FEAT-012) is sized against an unverified number. | 3 | 3 | 9 | Measure real end-to-end wake latency (platform cold start + application boot) before finalizing the wake-on-intent trigger point; instrument this as a tracked metric from first deployment, not assumed. | If measured latency is materially worse than assumed, move the wake trigger earlier (e.g., workspace-open instead of input-focus) and communicate honest wake status more prominently per [[features]] FEAT-011. | Founder / Solo Builder |
| R-007 | The `LISTEN`/`NOTIFY`-plus-safety-net-poll design for claiming orchestration work ([[architecture]]) is unproven at real concurrency; a misconfigured poll cadence either burns Neon CU-hours (defeating scale-to-zero) or misses notifications, stalling runs. | 3 | 3 | 9 | Load-test the claim mechanism under simulated concurrent runs before launch; set and document the safety-net poll interval explicitly with the CU-hour cost of that interval calculated up front, not guessed. | If stalled runs are observed in production, tighten the safety-net poll interval as an immediate fix while investigating the root `NOTIFY` delivery gap. | Founder / Solo Builder |
| R-008 | Single-region deployment ([[architecture]]) has no disaster-recovery or multi-region story; a regional outage on Fly.io, Neon, or R2 takes the entire product down with no failover. | 2 | 4 | 8 | Rely on each provider's own regional durability guarantees for now; document the single point of failure explicitly rather than implying resilience that doesn't exist; monitor each provider's status page. | If a regional outage occurs, communicate honestly and immediately via the durable notification system ([[features]] FEAT-026) rather than a silent outage, consistent with the product's honesty principle. | Founder / Solo Builder |
| R-009 | Core infrastructure dependencies (DuckDB, Fly.io, Neon, R2) could change pricing, deprecate a relied-upon feature (e.g., `STORAGE_VERSION`, scale-to-zero billing), or degrade service, forcing an unplanned architecture or cost-model change. | 2 | 4 | 8 | The Engine Provider seam ([[architecture]]) already isolates the Fly.io-specific implementation from the rest of the system, containing the blast radius of a provider change to one component. Track each provider's changelog/pricing page on a recurring basis. | If a dependency becomes untenable, the isolated seam (Engine Provider for compute; `httpfs`-compatible object storage for R2) defines the specific component to replace rather than requiring a full rewrite. | Founder / Solo Builder |
| R-010 | The DuckDB Go driver's cgo dependency rules out fully static binary builds and adds container/build complexity. | 2 | 2 | 4 | Document the `CGO_ENABLED=1` and C-compiler build requirement explicitly in deployment tooling; keep the container build simple and reproducible rather than optimizing away the constraint. | If cgo build complexity causes a deployment failure, keep a known-good pinned build image as a fallback rather than debugging the toolchain live during an incident. | Founder / Solo Builder |

### Operational Risks

Risks related to the team, processes, and running the product. Sourced from [[README]] Weaknesses.

| ID | Risk | L | I | Score | Mitigation | Contingency | Owner |
|---|---|---|---|---|---|---|---|
| R-011 | Groundwork is built and operated by a single person with a pre-revenue cost ceiling and no backup owner — if the solo builder is unavailable for an extended period, all development and incident response halts. | 3 | 5 | 15 | Keep [[architecture]], [[features]], and this risk log current enough that a contractor or collaborator could onboard from documentation alone; avoid undocumented tribal-knowledge dependencies, especially in the orchestration engine (R-005). | If the solo builder becomes unavailable, the up-to-date doc set is the recovery path — a paused product (no new development) is preferable to an undocumented one, since the architecture is explicitly designed so idle infrastructure costs near-zero while paused. | Founder / Solo Builder |
| R-012 | Under time or resource pressure, verification layers ([[features]] FEAT-027-033) could be silently skipped or weakened to ship faster — directly undermining the core trust differentiator the product is positioned on. | 2 | 5 | 10 | Treat verification layers 1, 2, 3, and 6 (determinism caching, static analysis, reconciliation, claim-level provenance) as P0 and non-negotiable for any release that reaches real users, exactly as scoped in [[features]] and [[roadmap]] Phase 3 — never descoped under deadline pressure. | If a release deadline genuinely cannot accommodate full verification, delay the release rather than ship an ungrounded answer loop — per [[monetization]], verification is explicitly never a paywalled or optional feature, and shipping without it breaks that promise for every tier at once. | Founder / Solo Builder |

### Legal & Compliance Risks

Risks related to regulations, data handling, and legal exposure. Sourced from [[README]] Weaknesses and Threats.

| ID | Risk | L | I | Score | Mitigation | Contingency | Owner |
|---|---|---|---|---|---|---|---|
| R-013 | Even with mitigations (aggregates-only by default, PII flagging, column exclusion, visible egress log), schema, sample values, and query results still reach the third-party model provider — this is the sharper, unresolved half of the privacy story that first-party deletion controls alone don't close. | 3 | 5 | 15 | Keep the egress/audit log ([[features]] FEAT-042) complete and genuinely auditable; state zero-retention provider terms plainly at upload ([[features]] FEAT-044); default sample-value exposure to opt-in per column rather than default-on, per [[README]]'s privacy mitigations. | If a user or regulator raises a specific concern about provider-side exposure, the egress log provides a precise, per-payload answer to "what left and when" rather than a general reassurance — use it directly in the response. | Founder / Solo Builder |
| R-014 | Regulatory or compliance requirements in target finance verticals could demand data-residency or on-premises guarantees that the cloud-SaaS-only deployment model ([[README]], [[architecture]]) explicitly does not offer, closing off segments of the target market. | 2 | 4 | 8 | State the cloud-SaaS-only deployment model plainly in sales and onboarding materials so this is never discovered late in a buying process; do not pursue enterprise/regulated accounts whose stated requirements conflict with this model, per [[README]]'s explicit non-goals. | If a large-enough segment of qualified demand consistently requires data residency or self-hosting, treat it as a signal to revisit the deployment-model decision in [[README]] deliberately — not to quietly build toward it feature by feature. | Founder / Solo Builder |

### Financial Risks

Risks related to budget, pricing, and revenue. Sourced from [[monetization]].

| ID | Risk | L | I | Score | Mitigation | Contingency | Owner |
|---|---|---|---|---|---|---|---|
| R-015 | Free-tier usage patterns may not match the assumed monthly allotment sizing ([[monetization]]); if price-sensitive Tunde-like users consume more engine-seconds, storage, or tokens than modeled, free-tier cost could scale faster than Free-to-Paid conversion revenue, and price-sensitive users may churn before ever upgrading rather than absorb overage. | 3 | 4 | 12 | Usage is metered per workspace from day one regardless of billing status (per [[architecture]] and [[features]] FEAT-047), so real free-tier cost data exists before the allotment needs to be finalized or defended. Track free-tier gross margin as a first-class metric per [[monetization]] from the first cohort onward. | If free-tier costs consistently exceed the modeled allotment, tighten the included allotment or introduce a soft usage cap with a clear upgrade prompt — rather than removing free-tier access to verification/trust features, which [[monetization]] explicitly rules out as a lever. | Founder / Solo Builder |

## Risk Trends

| Date | Change | Risk ID | Description |
|---|---|---|---|
| 2026-08-04 | Added | R-001–R-015 | Initial risk log generated from [[README]] SWOT (Weaknesses/Threats), [[architecture]], and [[monetization]] at first documentation pass. |

## Review Cadence

- **Weekly:** Review all High and Critical risks (score ≥10) — currently R-001, R-002, R-004, R-005, R-011, R-012, R-013, R-015.
- **Monthly:** Full risk register review; re-score any risk where new information (measured latency, real usage data, competitor activity) has changed likelihood or impact.
- **Per Milestone:** Re-assess all risks before each [[roadmap]] phase begins, since new features (e.g., Team-tier sharing, additional export formats) can introduce new risk categories not yet captured here.
