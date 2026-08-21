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

### What is deliberately not changing

The aesthetic. DM Serif Display against DM Mono, the near-black ground, the restrained gold accent, hairline section rules, the numbered step-card rhythm. This is a distinctive point of view and it stays. Every change below is information architecture, input model, or content — not palette or voice.

## Requirements

### Functional

1. **One deal object.** A single `deal` holds every fact about the transaction. All pages read from it. No page owns its own copy of a shared value.
2. **Deal bar.** A persistent, collapsible strip on every page showing price, down payment, rate, and monthly payment; expands to edit. Both "Sync from Mortgage Tab" buttons and `syncFrom()` are deleted.
3. **Input model matched to the kind of number.** Transcribed facts become typed fields; explored assumptions stay sliders and gain a live delta readout; tested decisions become pinned side-by-side scenarios.
4. **Verdict-first pages.** Every page opens with a plain-language sentence answering its question. Inputs move below the answer and collapse.
5. **Stage model.** `deal.stage` ∈ `exploring | contract | closing | owned`, inferred from data already collected (a contract date implies `contract`) and overridable. Navigation reorders and de-emphasizes by stage.
6. **One progress spine.** The guide's 8-step checklist and the commitment tracker's 18 items become a single sequence; the commitment period is a zoomed view of one segment, not a parallel tracker.
7. **Affordability.** Collect gross income, liquid assets, and existing monthly debt. Derive DTI against a 43% threshold, months of reserves remaining after closing, and monthly cost as a share of gross.
8. **Reading.** Introduce a prose typeface at ~15px. DM Mono is retained strictly for numerals, labels, and eyebrows. DM Serif Display is untouched.
9. **Light theme.** Full token parity, honoring `prefers-color-scheme` with an explicit override.
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
