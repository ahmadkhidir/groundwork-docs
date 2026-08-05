# Screen 1: Landing Page

**Route:** `/`
**Platform:** Both (server-rendered per [[architecture]] — the one screen using SSR/SSG)
**Purpose:** Explain what Groundwork is and get a visitor to sign up.
**Entry Points:** External (search, direct link, shared marketing link).
**Exit Points:** [[02-sign-up-login|Sign Up / Login]].

### Layout

**Mobile layout:**
- `Navbar` (logo + "Sign In" / "Get Started" buttons, collapses extra links into a `Drawer`)
- Stacked hero section: headline, one-line explanation, primary CTA button
- Stacked sections below: how it works (3-step), a static example of a provenance-linked claim, pricing teaser linking to tiers from [[monetization]]
- Footer with links

**Web layout:**
- Sticky `Navbar` (logo, nav links, "Sign In" / "Get Started")
- Full-width hero with headline, subhead, primary CTA, and a static illustrative screenshot/mock of the Report View showing a provenance-linked claim
- 3-column "how it works" `Grid` section
- Pricing teaser `Grid` (3 cards mirroring [[monetization]] tiers) linking to Sign Up
- Footer with links

### Components

| Component | Type | Platform | Purpose | Data Source |
|---|---|---|---|---|
| Hero CTA | Button (Primary) | Both | Routes to Sign Up | Static |
| Pricing teaser cards | Card × 3 | Both | Summarizes Free/Pro/Team at a glance | Static content mirroring [[monetization]] |
| Example claim | Provenance-Linked Number (shared component) | Both | Demonstrates the trust mechanism before signup | Static example data |

### Data Requirements

**Data Needed:** None — fully static/marketing content, server-rendered.

**Data Flow:**
1. Page is statically generated/server-rendered per [[architecture]]; no client-side fetch on load.

### States

| State | Condition | What the User Sees |
|---|---|---|
| **Loaded** | Always (static content) | Full marketing page |

**Edge Cases:**
- Already-authenticated user visits `/` → still shown the landing page (not force-redirected), but the primary CTA changes to "Go to Workspaces" instead of "Get Started."
