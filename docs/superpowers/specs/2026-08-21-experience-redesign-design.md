# Experience Redesign Design

**Date:** 2026-08-21
**Feature:** Restructure the calculator around one deal and the buyer's stage
**Status:** Approved

## Overview

The app is organized around calculations. The user is living through a transaction. This redesign reorganizes it around the second thing without changing its visual identity.

Eight workstreams, ordered by dependency. Each lands as its own commit and is independently verifiable, so the sequence can be stopped or reverted at any point without leaving the app broken.

## Diagnosis

Measured from the current `index.html` (2,670 lines, 6 pages):

| Symptom | Evidence |
|---|---|
| One deal exists as three copies | `m_price`/`s_price`/`r_price`, same for down/rate/term/tax/ins |
| Divergence is patched by hand | `syncFrom()` plus two "Sync from Mortgage Tab" buttons in the UI |
| Property type asked four times | `propertyTypes = {mortgage, sale, rent, commitment}` |
| Every number is a slider | 45 `input[type=range]`, including transcribed facts like a lender's $1,800 origination fee |
| The form is the hero | Landing page opens with 7 sliders; subhead reads "Adjust the sliders"; results sit below the fold |
| Two unrelated progress trackers | Guide's 8-item checklist offers "Under contract" while the Commitment page assumes you already signed |
| Affordability is never computed | Cash to close is shown as $431,838; income, assets, and existing debt are never collected, so the 43% DTI the guide cites cannot be evaluated |
| Prose set in monospace | Guide and Commitment pages are long-form, set in DM Mono 13px/300 |
| Dark only | `color-scheme: dark`, no light tokens |
| Nothing leaves the browser | State lives only in `localStorage`; no print, no share |

### The visual identity is being replaced

An earlier draft of this spec kept the existing aesthetic. That was wrong. Read as a set of tokens rather than as an overall impression, the current design sits in a recognizable machine-generated cluster:

| Tell | Where |
|---|---|
| `10px` / `.18em` / uppercase micro-eyebrows | `.sl`, `.eyebrow`, `.ct`, `.cmt-stat-label` — 15+ instances |
| Hairline rule trailing every section label | `.sl::after { flex:1; height:1px }` |
| Fluid serif hero with an italic accent-coloured second line | all six `.ph h1` blocks |
| 135° whisper-gradient on cards | `.card.bc`, `.guide-checklist`, `.cmt-hero` |
| Monospace as body copy | `body { font-family: var(--fm) }` at 13px/300 |
| `--sp-1`…`--sp-10` + `--radius-sm/md/lg` token ladder | `:root` |
| Everything a bordered card at `rgba(255,255,255,.07)` | `--border` |

Any one is defensible. Together they are a fingerprint.

### Direction: The Instrument

Drawn from the artifacts that actually govern this transaction — the Blumberg contract of sale, its riders, and the recorded deed as it appears in ACRIS.

**The hero is a caption block, not a headline.** Legal instruments have no display typography: they open with a ruled caption naming the instrument and the parties. Removing the fluid `clamp()` serif hero eliminates the single strongest tell, and it is also more honest — the page is a document about a deal, not a landing page.

**Signature: the form is printed, your entries are typed onto it.** Static and computed content sets in the document serif. Anything the user enters renders in typewriter on a ruled fill-in line. This is authentic to the source — riders and amendments are typed onto pre-printed forms — and it doubles as the input affordance for the facts/assumptions split: a typed value *looks* enterable.

**Palette.** Cool bone ground rather than warm cream, deliberately avoiding the cream-serif-terracotta cluster. Structure carries NY legal "blueback" navy, the colour of the backing sheet on New York court filings. One oxblood rubber-stamp accent, rotated, used exactly once per page.

| Token | Value | Role |
|---|---|---|
| `--paper` | `#EDEAE3` | ground — cool bone, not cream |
| `--ink` | `#14161A` | body text |
| `--blueback` | `#1F2B40` | structural: caption blocks, index, rules |
| `--stamp` | `#8C2F27` | the one stamp per page; nothing else |
| `--manila` | `#D9C9A3` | filed tabs, schedule banding |
| `--rule` | `#C9C3B6` | ruled lines and fill-in underscores |

**Type.** Source Serif 4 for prose and tabular schedule figures; Archivo Narrow for the condensed labels that forms use; Courier Prime for typed entries only. No display face, and no monospace for prose.

**Structure encodes the document.** Numbering becomes real rather than decorative: `¶` paragraph references for the guide's genuine sequence, lettered `(a) (b) (c)` items for schedule entries, roman numerals for the index. Cards, borders, and radii are replaced by ruled sections, hanging indents, and marginal references set in the left margin the way a contract cites its own clauses.

**Motion, spent once.** The stamp presses in on load — 200ms, scale and opacity, once per page — and nothing else animates. Suppressed under `prefers-reduced-motion`.

**Navigation.** The tab bar becomes a document index: `I. Purchase · II. Commitment · III. Lenders …`, ruled, with the current section marked. Calmer than tabs and true to the source.

## Requirements

### Functional

1. **One deal object.** A single `deal` holds every fact about the transaction. All pages read from it. No page owns its own copy of a shared value.
2. **Deal bar.** A persistent, collapsible strip on every page showing price, down payment, rate, and monthly payment; expands to edit. Both "Sync from Mortgage Tab" buttons and `syncFrom()` are deleted.
3. **Input model matched to the kind of number.** Transcribed facts become typed fields; explored assumptions stay sliders and gain a live delta readout; tested decisions become pinned side-by-side scenarios.
4. **Verdict-first pages.** Every page opens with a plain-language sentence answering its question. Inputs move below the answer and collapse.
5. **Stage model.** `deal.stage` ∈ `exploring | contract | closing | owned`, inferred from data already collected (a contract date implies `contract`) and overridable. Navigation reorders and de-emphasizes by stage.
6. **One progress spine.** The guide's 8-step checklist and the commitment tracker's 18 items become a single sequence; the commitment period is a zoomed view of one segment, not a parallel tracker.
7. **Affordability.** Collect gross income, liquid assets, and existing monthly debt. Derive DTI against a 43% threshold, months of reserves remaining after closing, and monthly cost as a share of gross.
8. **The Instrument visual system.** Replace the token set, typography, and structural devices per the direction above. No display serif hero, no micro-eyebrows, no card gradients, no radius ladder. Prose sets at ~16px in Source Serif 4.
9. **Dark mode as a night reading mode.** The paper ground is the primary theme. Dark inverts to an ink-on-charcoal reading mode — not the old near-black-with-gold. Full token parity, honoring `prefers-color-scheme` with an explicit override.
10. **Export.** A print stylesheet producing a one-page deal summary, and a share URL encoding the deal in the location hash.
11. **Open-house mode.** A compact single-screen view for standing in an apartment: enter an asking price, get monthly, cash to close, and whether it clears the ceiling the deal establishes.

### Non-functional

- Single file. No build step, no backend, no accounts, no new runtime dependencies.
- Existing saved state must survive. Users have real data in `nyc-home-calc-v2` today (commitment dates, checklists, lender rows).
- Keyboard focus visible, reduced motion respected, no horizontal overflow at 390px.
- Every workstream verified in a real browser before its commit.

## Architecture

### The deal object

```js
const deal = {
  // Facts — transcribed, typed fields
  price, downPayment, rate, term, propertyType,
  annualTax, annualInsurance, monthlyCommon, monthlyMaintenance,

  // Affordability facts
  grossIncome, liquidAssets, monthlyDebt,

  // Transaction state
  stage, contractDate, contingencyDays,

  // Assumptions — explored, sliders. Per-lens, not facts about the deal.
  assumptions: {
    appreciation, sellYear, commission, flipTax,
    rentToday, rentIncrease, investReturn, compareYears,
    extraPayment
  }
};
```

The facts/assumptions split is the load-bearing distinction. A fact has one correct value and belongs to the deal; an assumption is a question you are asking and belongs to the lens asking it. This is what makes the input-model rules in workstream 3 mechanical rather than case-by-case.

### Storage migration

Bump `STORAGE_KEY` to `nyc-home-calc-v3` and migrate on first read rather than discarding:

- `m_*` values are canonical for shared facts (the mortgage page is the current landing page, so its values are the ones users have actually curated).
- `s_*` and `r_*` values that have no `m_` counterpart (`s_sellYear`, `s_appRate`, `s_commission`, `s_flipTax`, `r_rent`, `r_rentInc`, `r_invest`, `r_years`) become `deal.assumptions`.
- `propertyTypes.mortgage` becomes `deal.propertyType`.
- `guideChecklist`, `commitChecklist`, `lenders`, `c_signed`, `c_days` carry over unchanged.
- v2 data is left in place, not deleted, so a failed migration is recoverable.

Divergent duplicates are the one place data is necessarily lost — three prices collapse to one. The migration takes the mortgage value silently; this is correct because the alternative (prompting) asks the user to adjudicate an inconsistency the app created.

### Typography

`DM Sans` for prose, from the same superfamily as the existing DM Serif Display and DM Mono. Chosen over introducing a fourth voice: the superfamily was designed to work together, and the goal is to stop misusing mono for paragraphs, not to re-open the type direction. Mono stays wherever its tabular alignment does real work — figures, labels, eyebrows, the checklist due-date column.

### Light theme

The current `:root` block already defines every color as a token, so light mode is a token swap rather than a restyle. Structure: complete light palette on bare `:root`, dark overrides under `@media (prefers-color-scheme: dark)` guarded by `:root:not([data-theme="light"])`, and again under `:root[data-theme="dark"]` so an explicit toggle wins in both directions.

### Workstream order and dependencies

| # | Workstream | Depends on | Why here |
|---|---|---|---|
| 1 | Deal object + migration | — | Everything else reads from it |
| 2 | Deal bar; delete sync buttons | 1 | Makes the single source of truth visible |
| 3 | Input model: typed facts, slider assumptions | 1, 2 | Mostly deletion once state is unified |
| 4 | Affordability inputs and derivations | 1 | New facts on the deal object |
| 5 | Verdict-first page headers | 3, 4 | Verdicts need affordability to say anything useful |
| 6 | Prose typeface + light theme | — | Independent; can land any time after 1 |
| 7 | Stage model + merged progress spine | 1, 4 | Needs stage inference and the unified checklist |
| 8 | Print, share URL, open-house mode | 1–5 | Exports whatever the deal model settles into |

Workstream 6 touches only CSS and markup classes, so it shares no dependency with the data work and can be pulled to the front if the visual change is wanted early.

## Risks

- **The migration is the only irreversible step.** Mitigated by writing v3 alongside v2 rather than over it, and by verifying a v2 fixture restores correctly before the commit.
- **Scope.** Eight workstreams against one 2,670-line file. Mitigated by independent commits; the app must run correctly at every commit boundary.
- **Verdict copy can overreach.** A sentence like "you can afford this" is a financial claim. Verdicts state arithmetic ("38% of gross, above the 43% limit lenders use") and never advice.
- **Open-house mode risks becoming a second app.** It reads the same deal object and adds no state of its own; if it needs its own model, it is cut.

## Out of scope

Accounts, sync across devices, live rate feeds, address lookup, amortization export to CSV, multi-property comparison, anything requiring a server.
