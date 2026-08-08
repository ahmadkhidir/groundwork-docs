# Monetization

**Last Updated:** 2026-08-08
**Source:** [[README]], [[user_personas]], [[features]], [[architecture]]

## Revenue Model

**Model Type:** Hybrid — Freemium with usage-based overage.

**Summary:**
Groundwork's real infrastructure cost is genuinely usage-variable per [[architecture]] — engine-seconds on Fly.io, object-store bytes on R2, and model tokens via the Anthropic API are metered per workspace from day one regardless of billing ([[features]] FEAT-047). A flat seat-based price would either overcharge a light Tunde-style user or undercharge a heavy Chidinma/Ngozi-style user running large multi-file joins. Instead, each tier includes a fixed monthly price plus an included usage allotment (workspaces, questions/runs per month, engine-minutes, storage, model tokens); usage beyond the included allotment is billed at cost-plus-margin rates. The free tier exists to let a price-sensitive small-business user like Tunde experience the full core loop at real but low volume, not as a crippled trial.

**Design decision — trust and verification are never paywalled.** Every verification layer ([[features]] FEAT-027 through FEAT-033, including claim-level provenance FEAT-030) and every privacy/governance feature (PII flagging FEAT-007, ingest join confirmation FEAT-009, the egress/audit log FEAT-042, the deletion cascade FEAT-043, the zero-retention notice FEAT-044) is available identically on every tier, including Free. [[README]]'s core positioning is that Groundwork is verifiably honest rather than merely trusted — gating the mechanisms that make that claim checkable would directly contradict the product's differentiator and turn "you can verify this" into "you can verify this if you pay." What tiers gate instead is *usage volume* and *convenience/collaboration features* layered on top of a trust floor that is the same for everyone.

**Design decision — discoverability is never paywalled either.** Question suggestions ([[features]] FEAT-050) are on every tier: they cost nothing to produce (deterministic, no model call, no engine), and gating the exact feature that de-risks a Tunde-style Free user's first session would contradict the Free tier's stated purpose as a real, uncrippled experience. The runs a suggestion triggers are metered like any other question, so usage volume is still gated per tier.

**Design decision — convenience is paywalled; the discoverability floor is not.** Model-assisted smart question generation ([[features]] FEAT-051) is a convenience/capability feature layered on top of the FEAT-050 strip: it costs a model call and engine-seconds per click, so it is gated Pro/Team — and never Team-only, because a single Pro analyst is its primary beneficiary and nothing about it is collaboration-specific, so reserving it for Team would over-price it against the Pro tier's stated purpose as the core revenue tier. The deterministic strip stays on every tier, unchanged, and on Free the smart affordance appears as the standard locked Upgrade Prompt ([[screens]] shared component) — visible with an "Upgrade to Pro/Team" CTA, never silently absent. Each generation is metered and egress-logged like any model usage, so usage volume stays gated per tier while the trust floor remains identical everywhere.

## Pricing Tiers

### Free Tier

**Price:** $0 / month, plus pay-as-you-go overage beyond the included monthly allotment
**Target:** [[user_personas|Tunde]] — high price sensitivity, occasional/monthly usage, needs the full core loop to actually work at his scale, not a stripped-down demo.
**Purpose:** Acquisition and genuine usability for low-volume individual users. This is not a crippled trial — it is a real, permanently free plan for people who ask a handful of questions a month.
**Includes:**
- 1 workspace
- Up to 20 questions/runs per month, included engine-seconds, storage, and model-token allotment sized to typical small-file usage
- Full core grounded-answer loop (natural-language question → SQL → narrated, charted answer)
- All verification and trust layers, unrestricted (determinism caching, static analysis, reconciliation, shadow queries, adversarial review, claim-level provenance, honest uncertainty)
- Full privacy stack (PII flagging, join confirmation, visible egress/audit log, one-workspace-one-delete, zero-retention notice)
- SQL visibility (never hidden, regardless of tier)
- PDF export
- Mobile-responsive report viewing

**Limitations:**
- Single workspace only
- No Excel/CSV or other export formats beyond PDF
- No column-level exclusion (the fine-grained, user-controlled override sits above the automatic PII defaults, which apply on every tier)
- No workspace or report sharing
- No smart question generation (the deterministic suggestions strip remains available on every tier)
- Usage beyond the included monthly allotment is billed at pay-as-you-go overage rates rather than blocked outright, so a good month doesn't hard-stop mid-analysis — but sustained heavy use makes Pro cheaper

### Pro Tier

**Price:** $29 / month (or $290 / year), plus overage beyond a larger included allotment
**Target:** [[user_personas|Chidinma]] — regular, several-times-a-week individual use, expensed as a work tool, needs higher usage ceilings and the export/workflow features an analyst actually relies on.
**Purpose:** Core revenue tier — the plan a working analyst or a serious individual user should land on.
**Includes:**
- Everything in Free, plus:
- Up to 5 workspaces
- Substantially higher included monthly allotment of questions/runs, engine-seconds, storage, and model tokens (sized for regular weekly analyst use)
- Excel/CSV export
- Column-level exclusion (fine-grained control over what a model can ever see)
- Compositional follow-up questions and fork-on-amendment (explore alternative analyses without losing prior work)
- Durable notifications
- Markdown/HTML and JSON export
- Single chart image export
- Detailed usage view (engine-seconds, storage, tokens breakdown)
- Model-assisted smart question suggestions (one-click, data-informed question generation — [[features]] FEAT-051)
- Read-only report sharing by ID

**Limitations:**
- Workspace sharing (inviting a read-only collaborator into a full workspace, not just a single report) is not included
- PowerPoint export is not included

### Team Tier

**Price:** $79 / month base (includes up to 3 members), plus $15/month per additional member, plus overage beyond a pooled team allotment
**Target:** [[user_personas|Ngozi]] — departmental FP&A use, needs to reconcile across sources and hand off both board-level and team-level output, with multiple people touching the same workspace.
**Purpose:** Serves teams that need shared workspace access and higher pooled usage, without building toward a full enterprise/self-host tier that [[README]] explicitly rules out.
**Includes:**
- Everything in Pro, plus:
- Unlimited workspaces
- Read-only workspace sharing (invite colleagues into a full workspace, not just a single report)
- Pooled usage allotment shared across team members, sized for departmental-cycle usage (month-end, board cycles)
- PowerPoint export
- Priority (discounted) overage rates

**Limitations:**
- Workspace collaboration remains read-only per [[README]] v1 scope — concurrent semantic-layer editing is out of scope on every tier, not just gated by price
- No SSO, custom contracts, or self-hosted deployment — these are explicit non-goals in [[README]], not a future Enterprise tier waiting to be built

## Feature Gate Mapping

Verification, trust, and privacy features are marked **Yes** on every tier deliberately — see the Design Decision above. Core-loop features required for the product to function at all are likewise available on every tier, gated only by the usage allotment, not by feature presence.

| Feature ID | Feature | Free | Pro | Team |
|---|---|---|---|---|
| FEAT-001 | Account Creation & Authentication | Yes | Yes | Yes |
| FEAT-002 | Workspace Creation | 1 workspace | Up to 5 | Unlimited |
| FEAT-003 | File Upload | Yes | Yes | Yes |
| FEAT-004 | Session-Scoped Engine Provisioning & Ingest | Yes (metered) | Yes (metered) | Yes (metered) |
| FEAT-005 | Data Profiling | Yes | Yes | Yes |
| FEAT-006 | Type Inference Review & Confirmation | Yes | Yes | Yes |
| FEAT-007 | PII Detection & Flagging at Ingest | Yes | Yes | Yes |
| FEAT-008 | Multi-File Relationship Inference | Yes | Yes | Yes |
| FEAT-009 | Candidate Join Review, Accept, Reject | Yes | Yes | Yes |
| FEAT-010 | Column-Level Exclusion | No | Yes | Yes |
| FEAT-011 | Engine Sleep/Wake Lifecycle & Honest Status | Yes | Yes | Yes |
| FEAT-012 | Wake-on-Intent Trigger | Yes | Yes | Yes |
| FEAT-013 | Natural-Language Question → SQL Generation | Yes (usage-capped) | Yes (usage-capped, higher) | Yes (usage-capped, pooled) |
| FEAT-014 | SQL Execution & Fact Return | Yes | Yes | Yes |
| FEAT-015 | Narrative Generation | Yes | Yes | Yes |
| FEAT-016 | Chart Spec Generation | Yes | Yes | Yes |
| FEAT-017 | First-Party Chart Rendering | Yes | Yes | Yes |
| FEAT-018 | SQL Visibility & Inline Editing | Yes | Yes | Yes |
| FEAT-019 | Semantic Layer Storage | Yes | Yes | Yes |
| FEAT-020 | Persistent User Corrections | Yes | Yes | Yes |
| FEAT-021 | Compositional Follow-Up Questions | No | Yes | Yes |
| FEAT-022 | Resumable DAG Run Engine | Yes | Yes | Yes |
| FEAT-023 | Live Run Event Stream | Yes | Yes | Yes |
| FEAT-024 | Halt, Amend, and Resume Any Step | Yes | Yes | Yes |
| FEAT-025 | Fork-on-Amendment | No | Yes | Yes |
| FEAT-026 | Durable Notifications | No | Yes | Yes |
| FEAT-027 | Determinism Caching | Yes | Yes | Yes |
| FEAT-028 | Static SQL Analysis | Yes | Yes | Yes |
| FEAT-029 | Post-Execution Reconciliation | Yes | Yes | Yes |
| FEAT-030 | Claim-Level Provenance | Yes | Yes | Yes |
| FEAT-031 | Shadow Queries | Yes | Yes | Yes |
| FEAT-032 | Adversarial Second-Pass Review | Yes | Yes | Yes |
| FEAT-033 | Honest Uncertainty | Yes | Yes | Yes |
| FEAT-034 | Reports as First-Class Versioned Objects | Yes | Yes | Yes |
| FEAT-035 | PDF Export | Yes | Yes | Yes |
| FEAT-036 | Excel/CSV Export | No | Yes | Yes |
| FEAT-037 | PowerPoint Export | No | No | Yes |
| FEAT-038 | Markdown/HTML Export | No | Yes | Yes |
| FEAT-039 | JSON Export | No | Yes | Yes |
| FEAT-040 | Single Chart Image Export | No | Yes | Yes |
| FEAT-041 | Mobile-Responsive Report Viewing | Yes | Yes | Yes |
| FEAT-042 | Visible Model-Egress / Audit Log | Yes | Yes | Yes |
| FEAT-043 | One-Workspace-One-Delete Cascade | Yes | Yes | Yes |
| FEAT-044 | Zero-Retention Provider Terms Notice | Yes | Yes | Yes |
| FEAT-045 | Read-Only Workspace Sharing | No | No | Yes |
| FEAT-046 | Read-Only Report Sharing by ID | No | Yes | Yes |
| FEAT-047 | User-Facing Usage View | Basic (totals only) | Detailed breakdown | Detailed breakdown |
| FEAT-050 | Question Suggestions | Yes | Yes | Yes |
| FEAT-051 | Smart Question Generation (Model-Assisted) | No | Yes | Yes |

## Payment & Billing

**Payment Processor:** Stripe — chosen specifically for its native support of metered/usage-based billing alongside flat subscription tiers, which this hybrid model requires.
**Billing Cycle:** Monthly or annual (annual offers a discount, e.g. ~17% off, reflected in the Pro Tier annual price above). Usage overage is calculated and billed monthly regardless of the subscription cycle chosen, since usage variability is the entire reason overage exists.
**Trial Period:** No separate time-limited trial — the Free tier itself serves as the trial, since it runs the real, unrestricted core loop rather than a crippled preview. A user upgrades when their usage or workflow needs (sharing, exports, allotment) outgrow it, not when an artificial countdown ends.
**Currency Support:** USD only at launch, consistent with [[README]]'s pre-revenue, single-region deployment posture in [[architecture]].
**Tax Handling:** Handled by Stripe (Stripe Tax), avoiding a manual tax-compliance build given the solo-builder constraint.

**Billing Events:**
- **Upgrade (tier change):** Immediate access to the new tier's feature gates and allotment; charge is prorated for the remainder of the current billing period.
- **Downgrade (tier change):** Takes effect at the end of the current billing period, not immediately — a user doesn't lose access to a feature (e.g., workspace sharing) mid-cycle after already using it that month. If the new tier allows fewer workspaces than the user currently has, the excess workspaces become read-only (not deleted) when the downgrade takes effect — except the **most-recently-updated workspace stays writable** (most-recently-updated = `Workspace.updated_at`, per [[schema]]), so the user keeps working where they were last active and can consolidate or delete the excess at their own pace.
- **Allotment overage:** Usage beyond the included monthly allotment (engine-seconds, storage, tokens, or run count) is metered and billed automatically at the end of the billing period at the tier's stated overage rate — it does not block the user mid-session, consistent with [[architecture]]'s principle of never interrupting an in-flight run.
- **Cancellation:** Paid access continues until the end of the current billing period, then the account reverts to Free tier limits. Workspaces beyond the Free tier's single-workspace limit become read-only (not deleted) until the user either upgrades again or explicitly deletes them via the deletion cascade ([[features]] FEAT-043) — with the same most-recently-updated-workspace-stays-writable rule as Downgrade above.
- **Failed Payment:** Stripe's automatic retry schedule (up to 3 attempts over ~2 weeks). After final failure, the account downgrades to Free tier limits — no data is deleted; the one-workspace-one-delete cascade is a distinct, explicit user action, never an automatic consequence of a billing failure.
- **Refund:** Manual process handled via support request, issued through Stripe. Overage charges (reflecting real infrastructure cost already incurred) are refunded only in cases of demonstrable billing error, not buyer's remorse.

## Key Metrics

| Metric | Definition | Target |
|---|---|---|
| Free-to-Paid Conversion | % of Free tier users who upgrade to Pro or Team within 90 days | >5% |
| Monthly Recurring Revenue (MRR) | Sum of all active Pro/Team subscription base prices, excluding overage | Track from first paying cohort; no hard target pre-revenue |
| Overage Revenue Share | % of total monthly revenue coming from usage overage vs. flat subscription | Monitor as a signal of whether included allotments are sized correctly — a consistently high share suggests allotments are too low and should be rebalanced into tier pricing |
| Gross Margin per Workspace | (Subscription + overage revenue) − (engine-seconds + storage + token cost, per [[architecture]] metering) for a given workspace | Positive on Pro and Team tiers; Free tier may run at a small loss by design as an acquisition cost, bounded by the allotment ceiling |
| Churn Rate | % of paying (Pro/Team) users who cancel or downgrade to Free each month | <5% monthly |
| Customer Acquisition Cost (CAC) | Marketing/distribution spend divided by new paying users in the period | Track once any paid acquisition spend exists; currently near-zero given organic/pre-revenue positioning |
| Customer Lifetime Value (LTV) | Average revenue per paying customer over their subscription lifetime, including typical overage | >3x CAC once CAC is non-trivial |

## Monetization Roadmap

| Phase | Roadmap Phase ([[roadmap]]) | Monetization Milestone |
|---|---|---|
| 1-3 | Foundation, Core Grounded Answer Loop, Trust & Observability | No billing yet — product is usable but unmetered/unbilled while the core loop and trust differentiators are being validated. Usage metering (per [[features]] FEAT-047) is implemented early regardless, per [[architecture]], so real cost data exists before pricing is finalized. |
| 4 | Reports & Delivery | Introduce the Free and Pro tiers with Stripe billing once reports, PDF/Excel export, and the deletion cascade exist — these are the features that make a paid tier's value concrete (export, higher allotments). |
| 5 | Polish & Expansion | Introduce the Team tier alongside workspace sharing (FEAT-045) and PowerPoint export (FEAT-037), and tune overage rates and included allotments against real per-workspace cost data gathered since Phase 1. |
