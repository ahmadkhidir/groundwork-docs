# Branding & Design System

**Last Updated:** 2026-08-04
**Source:** [[README]], [[user_personas]]

## Brand Identity

**Product Name:** Groundwork
**Experience Tier:** `Standard` — utility-first, trust-forward. Users come to verify a number or get an honest answer, not to be delighted by motion. Sections 5-8 (Creative Direction, Motion Design, Asset Style Guide, Spatial & Sound Design) are intentionally absent.
**Target Platform:** `Web`, with a responsive mobile view. Desktop is the primary workspace (SQL editing, ingest confirmation, export) per [[user_personas]] (Chidinma works desktop-only); mobile is a first-class *viewing* surface for checking a report or a number before a meeting (Tunde, Ngozi).
**Tagline:** "Every number, traced back to the data."
**Brand Personality:** Precise, calm, transparent, unpretentious, quietly confident.
**Tone of Voice:** Groundwork writes like a careful analyst, not a hype machine. It states what it knows and flags what it doesn't ("this excludes 12% of rows where region was null" rather than silence). Plain declarative sentences over exclamation points. No "revolutionary," "seamless," or "magic." Confident but not boastful — the tone earns trust by being checkable, not by asserting authority. It never talks down to Tunde and never over-simplifies for Chidinma; the same voice explains a filtered row count to both, just with the SQL visible or hidden depending on who's looking.

## Logo & Iconography

**Logo Concept:** A wordmark-led lockup: "Groundwork" set in the display typeface, paired with a small mark — a single grounded horizontal baseline beneath a data point (a filled circle sitting exactly on a line), suggesting "a claim resting on solid ground." The mark works alone as an app icon/favicon at small sizes; the baseline motif also reappears subtly as an underline treatment on verified/provenance-linked numbers in-product (see Component Tokens below), tying the logo concept to a real UI mechanism rather than being decorative only.
**Logo Usage:** Full lockup (mark + wordmark) in the marketing header and light-background contexts. Mark alone as app icon, favicon, and browser tab icon. Mark alone, reduced to single color (`text-primary`), in the in-app header at a small size so it doesn't compete with report content.
**Icon Style:** Outlined, 1.5px stroke weight, rounded line caps, 2px corner radius on containing shapes. Outlined (not filled) reads as more neutral/precise and pairs better with dense data UI than heavier filled icons; a filled variant is reserved only for active/selected navigation states.
**Icon Library:** Lucide — outlined by default, consistent stroke weight, MIT-licensed, and covers the data/finance-adjacent icon needs (database, table, filter, git-branch for forks, shield for privacy) without requiring custom icon work.

## Design System

### Color Palette

Groundwork avoids the generic "AI startup" purple/violet register on purpose (see [[README]] — Weaknesses/positioning). The primary hue is a restrained, desaturated blue — closer to a financial-terminal or engineering-tool blue than a consumer-SaaS gradient blue — paired with a warm, muted amber accent used sparingly for anything that needs a human, non-clinical touch (onboarding empty states, encouraging copy) so the product doesn't read as cold to a non-technical user like Tunde.

#### Primary Colors
| Token | Light | Dark | Usage |
|---|---|---|---|
| `primary` | `#1D4ED8` | `#5B8DEF` | Primary buttons, links, active nav, focus states |
| `primary-hover` | `#1839A8` | `#7DA4F5` | Button hover (pointer platforms) / pressed (touch) |
| `primary-light` | `#E8EEFC` | `#16223D` | Selected backgrounds, badges, active row highlight |

#### Neutral Colors
| Token | Light | Dark | Usage |
|---|---|---|---|
| `surface` | `#FFFFFF` | `#15171A` | Page backgrounds, card backgrounds |
| `surface-secondary` | `#F4F5F7` | `#1F2226` | Secondary panels, input backgrounds, code block background |
| `text-primary` | `#111418` | `#F2F3F5` | Headings, body text, narrative report text |
| `text-secondary` | `#5B6270` | `#9AA1AC` | Captions, helper text, timestamps, row-count annotations |
| `border` | `#DFE1E6` | `#2E3238` | Dividers, input borders, card borders, table gridlines |

#### Semantic Colors
| Token | Light | Dark | Usage |
|---|---|---|---|
| `success` | `#15803D` | `#4ADE80` | Successful ingest, export ready, reconciliation passed |
| `warning` | `#B45309` | `#FBBF24` | Row-exclusion notices, low-confidence join, ambiguous question prompts |
| `error` | `#B91C1C` | `#F87171` | Failed run, unopenable snapshot, destructive delete confirmation |
| `info` | `#0369A1` | `#67C3F3` | Engine `waking` status, informational tooltips |

#### Trust & Provenance Colors
Groundwork's core differentiator is that every number is traceable ([[README]] §Strengths — claim-level provenance). These tokens exist so "this number is a clickable, verified claim" is a distinct, consistent visual signal across the product, not implied by ordinary link styling.

| Token | Light | Dark | Usage |
|---|---|---|---|
| `provenance` | `#0F766E` | `#5EEAD4` | Underline/marker on any number in narrative text that traces to a query result; provenance panel accents |
| `provenance-light` | `#E6F5F3` | `#0E2926` | Background highlight when a provenance-linked number is hovered/focused |
| `code-accent` | `#334155` | `#CBD5E1` | Syntax accents in the SQL/code viewer (keywords), distinct from narrative text color |

#### Chart Colors

First-party rendered chart specs ([[README]] — charts are specs, never images) need a disciplined, brand-consistent palette independent of the UI palette above, so financial data reads clearly and consistently across every report.

**Categorical** (up to 8 series — distinct hues at matched lightness/saturation so no single series dominates by default):

| Token | Light | Dark | Usage |
|---|---|---|---|
| `chart-cat-1` | `#1D4ED8` | `#5B8DEF` | Series 1 (matches `primary` for brand continuity) |
| `chart-cat-2` | `#B45309` | `#F4A93B` | Series 2 |
| `chart-cat-3` | `#0F766E` | `#5EEAD4` | Series 3 (matches `provenance` for continuity) |
| `chart-cat-4` | `#7C3AED` | `#B794F6` | Series 4 |
| `chart-cat-5` | `#BE185D` | `#F472B6` | Series 5 |
| `chart-cat-6` | `#4D7C0F` | `#A3D65C` | Series 6 |
| `chart-cat-7` | `#0369A1` | `#67C3F3` | Series 7 |
| `chart-cat-8` | `#78716C` | `#A8A29E` | Series 8 / "other" bucket |

**Sequential** (single-hue ramp, for magnitude — e.g. a heatmap of revenue by region):

| Token | Light | Dark | Usage |
|---|---|---|---|
| `chart-seq-1` (lightest) | `#E8EEFC` | `#16223D` | Lowest magnitude |
| `chart-seq-3` (mid) | `#6690E8` | `#3D5FA8` | Mid magnitude |
| `chart-seq-5` (darkest) | `#1D4ED8` | `#5B8DEF` | Highest magnitude |

**Diverging** (for values relative to a baseline — e.g. variance vs. budget, period-over-period change):

| Token | Light | Dark | Usage |
|---|---|---|---|
| `chart-div-negative` | `#B91C1C` | `#F87171` | Below baseline (reuses `error` for consistency with the rest of the system) |
| `chart-div-neutral` | `#DFE1E6` | `#2E3238` | At baseline (reuses `border`) |
| `chart-div-positive` | `#15803D` | `#4ADE80` | Above baseline (reuses `success`) |

Guidelines:
- Categorical tokens are assigned to series in fixed order (`chart-cat-1` first, etc.) so the same metric renders in the same color across reports within a session — this matters for a product whose pitch is consistency and trust, not just aesthetics.
- Never use `chart-div-negative`/`chart-div-positive` for categorical (non-directional) series — reserve red/green-coded meaning strictly for above/below-baseline contexts so it isn't overloaded.
- Full contrast/colorblind-safe validation of chart colors happens at implementation time against the actual rendering library; this palette is the brand-consistent starting point, not a substitute for that check.

### Typography

| Role | Font | Fallback |
|---|---|---|
| Display / Headings | `IBM Plex Sans` | `sans-serif` |
| Body | `Inter` | `sans-serif` |
| Monospace (SQL, code, data values) | `IBM Plex Mono` | `monospace` |

IBM Plex Sans for headings gives a slightly more "engineered/financial-tool" character than a purely generic grotesk, without being a display font; Inter carries body text because of its exceptional legibility at small sizes in dense report/table layouts. IBM Plex Mono is used specifically (not a generic system mono) so SQL, code blocks, and raw data values share one consistent, highly-legible monospace family — this pairing (Plex Sans + Plex Mono) is a deliberate family relationship, not an arbitrary combination, and reinforces "narrative vs. query" as a visual system.

#### Type Scale
| Token | Size | Weight | Line Height | Usage |
|---|---|---|---|---|
| `display` | 36px / 2.25rem | 700 | 1.2 | Marketing/landing headline only |
| `h1` | 28px / 1.75rem | 700 | 1.3 | Page titles (workspace name, report title) |
| `h2` | 22px / 1.375rem | 600 | 1.35 | Section headings within a report |
| `h3` | 18px / 1.125rem | 600 | 1.4 | Card titles, subsections, chart titles |
| `body-large` | 17px / 1.0625rem | 400 | 1.55 | Narrative report text (the primary reading surface) |
| `body` | 15px / 0.9375rem | 400 | 1.5 | UI body text, form labels, table cells |
| `body-small` | 13px / 0.8125rem | 400 | 1.5 | Captions, helper text, timestamps, egress log entries |
| `caption` | 11px / 0.6875rem | 500 | 1.4 | Badges, overlines, status pills ("waking", "verified") |
| `code` | 14px / 0.875rem | 400 | 1.6 | SQL editor, code blocks, raw data values (uses monospace family) |

### Spacing

4px base grid.

| Token | Value | Usage |
|---|---|---|
| `space-xs` | 4px | Icon-to-label gaps, inline badges |
| `space-sm` | 8px | Compact — table cell padding, chip padding |
| `space-md` | 16px | Default — card padding, list item gaps, form field gaps |
| `space-lg` | 24px | Between report sections, dialog padding |
| `space-xl` | 32px | Page margins, panel padding |
| `space-2xl` | 48px | Major separation — between unrelated dashboard regions |
| `space-3xl` | 64px | Marketing/landing layout only |

### Sizing

| Token | Value | Usage |
|---|---|---|
| `radius-sm` | 4px | Badges, chips, small inputs, status pills |
| `radius-md` | 8px | Buttons, inputs, cards, code blocks |
| `radius-lg` | 12px | Modals, drawers (run viewer side panel), large cards |
| `radius-full` | 9999px | Avatars, engine-status dot |
| `button-height` | 40px | Standard buttons (desktop-primary product; 44px minimum still applied on touch targets in the mobile view) |
| `input-height` | 40px | Text inputs, selects |
| `icon-sm` | 16px | Inline icons, status indicators |
| `icon-md` | 20px | Navigation icons, button icons |
| `icon-lg` | 28px | Empty-state icons, feature callouts |

### Shadows

| Token | Light | Dark | Usage |
|---|---|---|---|
| `shadow-none` | `none` | `none` | Flat surfaces, table rows |
| `shadow-sm` | `0 1px 2px rgba(17,20,24,0.06)` | `0 1px 2px rgba(0,0,0,0.45)` | Cards, dropdown triggers |
| `shadow-md` | `0 4px 10px rgba(17,20,24,0.08)` | `0 4px 12px rgba(0,0,0,0.55)` | Dropdowns, tooltips, popovers, the run-event stream panel |
| `shadow-lg` | `0 12px 28px rgba(17,20,24,0.12)` | `0 12px 32px rgba(0,0,0,0.7)` | Modals, the SQL-edit drawer, deletion-confirmation dialog |

### Component Tokens

#### Buttons
| Variant | Background | Text | Border | Radius | Height | Padding |
|---|---|---|---|---|---|---|
| Primary | `primary` | `#FFFFFF` | none | `radius-md` | `button-height` | 10px 16px |
| Secondary | `primary-light` | `primary` | none | `radius-md` | `button-height` | 10px 16px |
| Outline | transparent | `text-primary` | 1px `border` | `radius-md` | `button-height` | 10px 16px |
| Ghost | transparent | `text-secondary` | none | `radius-md` | `button-height` | 10px 16px |
| Destructive | `error` | `#FFFFFF` | none | `radius-md` | `button-height` | 10px 16px |

#### Inputs
| State | Border | Background | Text |
|---|---|---|---|
| Default | `border` | `surface` | `text-primary` |
| Focus | `primary` | `surface` | `text-primary` |
| Error | `error` | `surface` | `text-primary` |
| Disabled | `border` | `surface-secondary` | `text-secondary` opacity 0.5 |

#### Provenance-Linked Claims
A number in narrative report text that traces back to a specific query result (the mechanism `README` calls "claim-level provenance") is styled distinctly from ordinary text so users learn to recognize and trust it as verifiable, and distinctly from an ordinary hyperlink so it isn't mistaken for navigation.

| State | Text Color | Decoration | Background |
|---|---|---|---|
| Default | `provenance` | Dotted underline, 1px, offset 3px | transparent |
| Hover / focus | `provenance` | Solid underline, 1px | `provenance-light` |
| Active (provenance panel open) | `provenance` | Solid underline, 1.5px | `provenance-light` |

#### SQL / Code Display
SQL is always visible and editable, never hidden behind an affordance ([[README]] — In scope). Code blocks use a visually distinct surface from narrative text so users always know which "mode" they're reading.

| Element | Background | Text | Border | Font |
|---|---|---|---|---|
| SQL/code block (read-only) | `surface-secondary` | `text-primary` | 1px `border` | `code` token, monospace family |
| SQL/code block (editable, focused) | `surface` | `text-primary` | 1px `primary` | `code` token, monospace family |
| Inline code / column names | `surface-secondary` | `code-accent` | none | `code` token, monospace family, `radius-sm` |

#### Status Pills
Used for engine state (`sleeping`/`waking`/`ready`), run state, and reconciliation results — surfacing system honesty is a core product principle ([[README]] — never hide engine state).

| Variant | Background | Text | Usage |
|---|---|---|---|
| Neutral | `surface-secondary` | `text-secondary` | `sleeping` |
| Info | `info` at 12% opacity | `info` | `waking`, `running` |
| Success | `success` at 12% opacity | `success` | `ready`, `verified`, `reconciled` |
| Warning | `warning` at 12% opacity | `warning` | `low-confidence join`, `rows excluded` |
| Error | `error` at 12% opacity | `error` | `failed`, `unresolvable` |

## Accessibility

- **Contrast ratios:** All text meets WCAG AA minimum (4.5:1 normal text, 3:1 large text) in both light and dark mode, independently verified — a pairing passing in light mode is re-checked in dark, not assumed.
- **Touch targets:** Minimum 44×44px on the mobile responsive view, even where the desktop equivalent uses the tighter 40px `button-height`.
- **Focus indicators:** Visible 2px `primary` outline, 2px offset, on every interactive element including provenance-linked claims and SQL editor regions — keyboard navigation through a report and its underlying query must be fully usable without a mouse, which matters for Chidinma's SQL-editing workflow specifically.
- **Color independence:** Status is never conveyed by color alone. Every status pill carries a text label (`sleeping`, `waking`, `verified`) alongside its color; row-exclusion warnings are always accompanied by explicit text ("excluded 31% of rows where region was null"), not just a warning-colored icon.
- **Screen reader support:** Chart specs render with accessible data-table equivalents alongside the visual chart, not as a decorative image with no text alternative — this is a requirement, not an enhancement, given charts are structured specs (README §2.2) with underlying tabular data available by construction. Provenance links are announced as "verified figure, opens source data" rather than a bare "link."
- **Dense-data legibility:** Because the product's primary reading surface is narrative text interleaved with tables and provenance markers, minimum body text size is fixed at `body` (15px) even in compact table views — density is achieved through spacing and hierarchy, not by shrinking text below the accessible floor.
