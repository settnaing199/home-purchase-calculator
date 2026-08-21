# Experience Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Restructure `index.html` around one deal object and the buyer's stage in the transaction, and replace its visual identity with the Instrument direction.

**Execution order:** Task 6 runs first. It is independent of the data work and it is the change that was actually asked for; the remaining tasks then write markup into a settled visual system rather than being converted twice.

**Architecture:** A single `deal` object becomes the source of truth for every fact about the transaction. Task 1 introduces it as a write-through layer over the existing DOM inputs, so the three duplicated input sets stay in sync automatically while all existing calculation code keeps working unchanged. Later tasks progressively move inputs, verdicts, and navigation onto that foundation. Each task leaves the app fully working.

**Tech Stack:** Single-file HTML/CSS/vanilla JS. No build step, no backend, no dependencies beyond the existing Google Fonts link and Chart.js CDN.

**Spec:** `docs/superpowers/specs/2026-08-21-experience-redesign-design.md`

## Global Constraints

- Single file. All work lands in `index.html`. No build step, no backend, no accounts, no new runtime dependencies.
- Visual identity is being replaced, per the spec's **Direction: The Instrument**. Task 6 carries the whole visual system; every other task writes markup that assumes it. Until Task 6 lands, new markup uses the existing classes and is converted with the rest.
- No display serif hero, no `10px`/`.18em` uppercase micro-eyebrows, no 135° card gradients, no `--radius-*` ladder, no monospace prose. These are the tells being removed; do not reintroduce them in new markup.
- Typed entries render in Courier Prime on a ruled fill-in line. Computed and static content sets in Source Serif 4. This is the signature and it is load-bearing for the input model — a typed value must look enterable.
- One stamp per page, in `--stamp` oxblood, rotated. Never two.
- Existing saved state must survive. Users have real data in `nyc-home-calc-v2` (commitment dates, two checklists, lender rows). Write v3 alongside v2; never delete v2.
- The app must run correctly at every commit boundary. No task may leave a page broken.
- Verdict copy states arithmetic, never advice. "38% of gross, above the 43% ceiling lenders use" — not "you can afford this."
- Accessibility floor: visible keyboard focus, `prefers-reduced-motion` respected, no horizontal overflow at 390px.
- Facts get typed fields. Assumptions get sliders. This rule is mechanical — see the spec's facts/assumptions split.

## Verification Cycle

This repo has no test runner. Every task uses the same three-part cycle in place of one:

1. **Syntax:** extract the inline script and check it parses.
   ```bash
   cd /Users/setthein/home-purchase && python3 -c "
   import io,re
   s=io.open('index.html',encoding='utf-8').read()
   m=re.findall(r'<script(?![^>]*\bsrc=)[^>]*>(.*?)</script>',s,re.S)
   io.open('/tmp/check.js','w',encoding='utf-8').write('\n'.join(m))
   " && node --check /tmp/check.js && echo "JS OK"
   ```
2. **Behavior:** the browser assertions listed in that task, run against `file:///Users/setthein/home-purchase/index.html`. Every assertion must print its actual value, not a boolean, so a wrong-but-truthy result is visible.
3. **Regression:** all six pages still switch and compute; browser console has zero errors; no horizontal overflow at 390px.

---

### Task 1: The deal object and v2 migration

**Files:**
- Modify: `index.html` — add deal state above `const propertyTypes` (~line 1509); replace `saveToStorage`/`loadFromStorage` (~line 2556 onward)

**Interfaces:**
- Consumes: nothing
- Produces:
  - `deal` — the object described below
  - `dealSet(key, value)` — writes a fact, pushes it to every bound DOM input, recalculates affected pages, persists
  - `dealGet(key)` — reads a fact
  - `pushDealToInputs()` — writes every deal fact into its bound DOM inputs and fires the existing slider-display updates
  - `migrateV2ToDeal(v2data)` → deal-shaped object
  - `DEAL_INPUT_MAP` — the fact → DOM-id binding table
  - `STORAGE_KEY` is `'nyc-home-calc-v3'`; `STORAGE_KEY_V2` is `'nyc-home-calc-v2'`

**The binding table.** These are the duplicated inputs the deal replaces. Note `annualUpkeep` (a house's annual maintenance, `r_maint`) is distinct from `monthlyMaintenance` (a co-op's monthly charge, `r_coopMaint`) — they are different quantities with different periods, and conflating them is a bug.

```js
const DEAL_INPUT_MAP = {
  price:              ['m_price', 's_price', 'r_price'],
  downPayment:        ['m_down', 's_down', 'r_down'],
  rate:               ['m_rate', 's_rate', 'r_rate'],
  term:               ['m_term', 's_term'],
  annualTax:          ['m_tax', 's_tax', 'r_tax'],
  annualInsurance:    ['m_ins', 's_ins'],
  monthlyCommon:      ['m_common', 'r_common'],
  monthlyMaintenance: ['m_maintenance', 'r_coopMaint'],
  annualUpkeep:       ['r_maint']
};
```

- [ ] **Step 1: Write the browser assertion that fails**

Save as a Playwright block to run after the change. It must fail now, because setting the price on one page does not currently move it on the others.

```js
// Setting a fact once must move every page that shows it
await page.evaluate(() => dealSet('price', 1234000));
return {
  m: document.getElementById('m_price').value,
  s: document.getElementById('s_price').value,
  r: document.getElementById('r_price').value,
  deal: dealGet('price')
};
// Expected after Task 1: all four report 1234000
```

- [ ] **Step 2: Run it to confirm it fails**

Expected: `ReferenceError: dealSet is not defined`.

- [ ] **Step 3: Add the deal object and defaults**

Insert above `const propertyTypes`. Defaults match the current DOM defaults so behavior is unchanged on a fresh load.

```js
// ── Deal State ──
// One deal, read by every page. Facts live here; assumptions live in
// deal.assumptions because they belong to the question a lens is asking,
// not to the transaction itself.
const dealDefaults = {
  price: 950000, downPayment: 400000, rate: 6.5, term: 15,
  propertyType: 'single',
  annualTax: 5400, annualInsurance: 2000,
  monthlyCommon: 800, monthlyMaintenance: 1500, annualUpkeep: 9500,
  grossIncome: 0, liquidAssets: 0, monthlyDebt: 0,
  stage: 'exploring', contractDate: '', contingencyDays: 45,
  assumptions: {
    appreciation: 3, sellYear: 5, commission: 6, flipTax: 2,
    rentToday: 3000, rentIncrease: 3, investReturn: 7, compareYears: 5,
    extraPayment: 0
  }
};
let deal = JSON.parse(JSON.stringify(dealDefaults));
```

- [ ] **Step 4: Add the binding table, `dealSet`, `dealGet`, and `pushDealToInputs`**

`dealSet` is the only way a fact changes. It writes through to every bound input so no page can hold a stale copy — this is what makes the sync buttons unnecessary.

```js
const DEAL_INPUT_MAP = { /* table above */ };

function dealGet(key) { return deal[key]; }

function dealSet(key, value) {
  deal[key] = value;
  const ids = DEAL_INPUT_MAP[key] || [];
  ids.forEach(id => {
    const el = document.getElementById(id);
    if (!el) return;
    el.value = value;
    // keep the paired .iv text display in step with the slider
    const out = document.querySelector('[data-slider="' + id + '"]');
    if (out) syncSliderDisplay(id);
  });
  recalcAll();
  saveToStorage();
}

function pushDealToInputs() {
  Object.keys(DEAL_INPUT_MAP).forEach(key => {
    DEAL_INPUT_MAP[key].forEach(id => {
      const el = document.getElementById(id);
      if (el && deal[key] !== undefined) {
        el.value = deal[key];
        syncSliderDisplay(id);
      }
    });
  });
}

function recalcAll() {
  updateMortgage(); updateSale(); updateRent();
  if (typeof renderCommitment === 'function') renderCommitment();
  if (typeof updateLenders === 'function') syncLendersFromMortgage();
}
```

Locate the existing function that formats a slider's paired `.iv` display and reuse it as `syncSliderDisplay(id)`; if it is inline in a listener, extract it to a named function first without changing its body.

- [ ] **Step 5: Make the existing sliders write into the deal**

Add one listener that reverses the binding, so dragging any bound slider updates the deal and therefore its twins on other pages.

```js
Object.keys(DEAL_INPUT_MAP).forEach(key => {
  DEAL_INPUT_MAP[key].forEach(id => {
    const el = document.getElementById(id);
    if (!el) return;
    el.addEventListener('input', () => {
      const v = parseFloat(el.value);
      if (Number.isFinite(v) && v !== deal[key]) dealSet(key, v);
    });
  });
});
```

- [ ] **Step 6: Write the migration**

v2 stored flat `m_`/`s_`/`r_` string values. Mortgage-page values win for shared facts because that page is the current landing page and its values are the ones users have curated. Assumptions with no mortgage counterpart come from their own page.

```js
const STORAGE_KEY = 'nyc-home-calc-v3';
const STORAGE_KEY_V2 = 'nyc-home-calc-v2';

function migrateV2ToDeal(v2) {
  const d = JSON.parse(JSON.stringify(dealDefaults));
  const num = (v, fallback) => {
    const n = parseFloat(v);
    return Number.isFinite(n) ? n : fallback;
  };
  d.price = num(v2.m_price, d.price);
  d.downPayment = num(v2.m_down, d.downPayment);
  d.rate = num(v2.m_rate, d.rate);
  d.term = num(v2.m_term, d.term);
  d.annualTax = num(v2.m_tax, d.annualTax);
  d.annualInsurance = num(v2.m_ins, d.annualInsurance);
  d.monthlyCommon = num(v2.m_common, d.monthlyCommon);
  d.monthlyMaintenance = num(v2.m_maintenance, d.monthlyMaintenance);
  d.annualUpkeep = num(v2.r_maint, d.annualUpkeep);
  if (v2.propertyTypes && v2.propertyTypes.mortgage) d.propertyType = v2.propertyTypes.mortgage;
  d.contractDate = v2.c_signed || d.contractDate;
  d.contingencyDays = num(v2.c_days, d.contingencyDays);
  d.assumptions.appreciation = num(v2.s_appRate, d.assumptions.appreciation);
  d.assumptions.sellYear = num(v2.s_sellYear, d.assumptions.sellYear);
  d.assumptions.commission = num(v2.s_commission, d.assumptions.commission);
  d.assumptions.flipTax = num(v2.s_flipTax, d.assumptions.flipTax);
  d.assumptions.rentToday = num(v2.r_rent, d.assumptions.rentToday);
  d.assumptions.rentIncrease = num(v2.r_rentInc, d.assumptions.rentIncrease);
  d.assumptions.investReturn = num(v2.r_invest, d.assumptions.investReturn);
  d.assumptions.compareYears = num(v2.r_years, d.assumptions.compareYears);
  d.assumptions.extraPayment = num(v2.m_extra, d.assumptions.extraPayment);
  return d;
}
```

- [ ] **Step 7: Rewrite `saveToStorage` / `loadFromStorage` for v3**

v3 persists `deal` plus the three collections that carry over unchanged. v2 is read only when no v3 record exists, and is never deleted.

```js
function saveToStorage() {
  const data = { version: 3, deal: deal,
    guideChecklist: {...guideChecklist},
    commitChecklist: {...commitChecklist},
    lenders: lenders };
  try { localStorage.setItem(STORAGE_KEY, JSON.stringify(data)); } catch(e) {}
}

function loadFromStorage() {
  try {
    const rawV3 = localStorage.getItem(STORAGE_KEY);
    if (rawV3) {
      const data = JSON.parse(rawV3);
      if (data.deal) deal = { ...dealDefaults, ...data.deal,
        assumptions: { ...dealDefaults.assumptions, ...(data.deal.assumptions || {}) } };
      if (data.guideChecklist) guideChecklist = { ...defaultGuideChecklist, ...data.guideChecklist };
      if (data.commitChecklist) commitChecklist = { ...defaultCommitChecklist, ...data.commitChecklist };
      if (data.lenders) lenders = data.lenders;
    } else {
      const rawV2 = localStorage.getItem(STORAGE_KEY_V2);
      if (rawV2) {
        const v2 = JSON.parse(rawV2);
        deal = migrateV2ToDeal(v2);
        if (v2.guideChecklist) guideChecklist = { ...defaultGuideChecklist, ...v2.guideChecklist };
        if (v2.commitChecklist) commitChecklist = { ...defaultCommitChecklist, ...v2.commitChecklist };
        if (v2.lenders) lenders = v2.lenders;
        saveToStorage();  // write v3; leave v2 in place as a fallback
      }
    }
    pushDealToInputs();
    setPropertyTypeEverywhere(deal.propertyType);
    pushAssumptionsToInputs();
    recalcAll();
  } catch(e) {}
}
```

Add `pushAssumptionsToInputs()` mapping `deal.assumptions` onto `s_appRate`, `s_sellYear`, `s_commission`, `s_flipTax`, `r_rent`, `r_rentInc`, `r_invest`, `r_years`, `m_extra`, with a reverse listener on each, mirroring Steps 4–5. Add `setPropertyTypeEverywhere(type)` that sets `deal.propertyType`, activates the matching `.type-btn` in all four selectors, and calls `updateVisibility` on all four pages.

- [ ] **Step 8: Run the verification cycle**

Syntax check, then these assertions:

```js
// 1. write-through
await page.evaluate(() => dealSet('price', 1234000));
// expect m_price, s_price, r_price, dealGet('price') all 1234000

// 2. reverse binding
await page.fill('#s_down', '250000');
await page.dispatchEvent('#s_down', 'input');
// expect dealGet('downPayment') === 250000 and m_down === '250000'

// 3. property type is global
await page.evaluate(() => setPropertyTypeEverywhere('coop'));
// expect all four selectors show coop active, and deal.propertyType === 'coop'

// 4. v2 migration
await page.evaluate(() => {
  localStorage.clear();
  localStorage.setItem('nyc-home-calc-v2', JSON.stringify({
    m_price: '777000', m_down: '155400', m_rate: '5.75', m_term: '30',
    r_rent: '4200', s_sellYear: '7',
    propertyTypes: { mortgage: 'condo' },
    c_signed: '2026-08-06', c_days: '45',
    commitChecklist: { exactDate: true },
    guideChecklist: { financesReady: true },
    lenders: [{ id: 'lender-1', name: 'Migrated', rate: 6.1, points: 0, origFee: 0,
                underwritingFee: 0, appFee: 0, processingFee: 0, lockDays: 45 }]
  }));
});
await page.reload();
// expect deal.price 777000, deal.rate 5.75, deal.propertyType 'condo',
//   deal.assumptions.rentToday 4200, deal.assumptions.sellYear 7,
//   commitChecklist.exactDate true, lenders[0].name 'Migrated',
//   localStorage still holds nyc-home-calc-v2, and now also v3
```

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Introduce one deal object as the source of truth for shared facts"
```

---

### Task 2: The deal bar, and deleting the sync buttons

**Files:**
- Modify: `index.html` — new markup after `<nav class="nav">`; new CSS after the `.nav` block (~line 69); remove both sync buttons and `syncFrom()`

**Interfaces:**
- Consumes: `deal`, `dealSet`, `dealGet` (Task 1)
- Produces: `renderDealBar()`, `toggleDealBar()`

- [ ] **Step 1: Assertion that fails**

```js
// The deal bar reflects the deal on every page, and editing it writes through
await page.click('button.nav-tab:has-text("Sale proceeds")');
await page.evaluate(() => dealSet('price', 880000));
return {
  barPrice: await page.textContent('#dealbar-price'),
  syncButtons: await page.locator('button:has-text("Sync from Mortgage")').count()
};
// Expected after Task 2: barPrice contains "880,000"; syncButtons === 0
```

- [ ] **Step 2: Run it, confirm failure**

Expected: `#dealbar-price` not found; `syncButtons` is 2.

- [ ] **Step 3: Add the deal bar markup**

Directly after the closing `</nav>`. Sticky beneath the nav, four figures, one edit toggle. Mono for the numerals, matching the existing figure treatment.

```html
<div class="dealbar" id="dealbar">
  <button class="dealbar-toggle" onclick="toggleDealBar()" aria-expanded="false"
          aria-controls="dealbar-edit">
    <span class="dealbar-item"><span class="dealbar-label">Price</span>
      <span class="dealbar-val" id="dealbar-price">—</span></span>
    <span class="dealbar-item"><span class="dealbar-label">Down</span>
      <span class="dealbar-val" id="dealbar-down">—</span></span>
    <span class="dealbar-item"><span class="dealbar-label">Rate</span>
      <span class="dealbar-val" id="dealbar-rate">—</span></span>
    <span class="dealbar-item"><span class="dealbar-label">Monthly</span>
      <span class="dealbar-val accent" id="dealbar-monthly">—</span></span>
    <span class="dealbar-edit-hint">Edit</span>
  </button>
  <div class="dealbar-edit" id="dealbar-edit" hidden></div>
</div>
```

- [ ] **Step 4: Add `renderDealBar()` and call it from `recalcAll()`**

```js
function renderDealBar() {
  const set = (id, txt) => { const el = document.getElementById(id); if (el) el.textContent = txt; };
  set('dealbar-price', fmt(deal.price));
  set('dealbar-down', fmt(deal.downPayment) + ' · ' +
      Math.round((deal.downPayment / deal.price) * 100) + '%');
  set('dealbar-rate', deal.rate + '% · ' + deal.term + 'y');
  const pi = monthlyPI(deal.price - deal.downPayment, deal.rate, deal.term);
  set('dealbar-monthly', fmt(Math.round(pi)));
}
```

Reuse the existing currency formatter and the existing monthly-payment routine rather than writing new ones; if the payment math is inline inside `updateMortgage`, extract it to `monthlyPI(loan, rate, years)` first, with no change to its arithmetic, and have `updateMortgage` call it.

- [ ] **Step 5: Build the expand-to-edit panel**

`toggleDealBar()` flips `hidden` on `#dealbar-edit` and the `aria-expanded` state. The panel holds typed fields for price, down payment, rate, term, and the property-type selector, each calling `dealSet` on `change`. This panel is the canonical editor the later tasks reuse.

- [ ] **Step 6: Delete both sync buttons and `syncFrom()`**

Remove the `Sync from mortgage tab` button on the rent page, the `Sync from Mortgage Tab` button on the lender page, the `syncFrom()` function, and `syncLendersFromMortgage`'s button-triggered path. Keep `syncLendersFromMortgage` itself — it is now called from `recalcAll()`, driven by the deal.

- [ ] **Step 7: Verification cycle**

Assertions: the bar reads the same four figures on all six pages; editing price in the panel moves `m_price`, `s_price`, `r_price`; zero elements match `text=Sync from`; the bar does not overflow at 390px; `grep -c syncFrom index.html` returns 0.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Add persistent deal bar and remove the manual sync buttons"
```

---

### Task 3: Typed fields for facts, sliders only for assumptions

**Files:**
- Modify: `index.html` — the `.ir` input rows on the mortgage, sale, rent, and lender pages; `.fact-field` CSS

**Interfaces:**
- Consumes: `deal`, `dealSet` (Task 1); the deal bar edit panel (Task 2)
- Produces: `factField(id, label, value, opts)` markup convention; no new JS API

- [ ] **Step 1: Assertion that fails**

```js
return {
  rangeCount: await page.locator('input[type=range]').count(),
  lenderRanges: await page.locator('#page-lender input[type=range]').count()
};
// Now: 45 and 14. After Task 3: assumptions only — lenderRanges === 0
```

- [ ] **Step 2: Run it, record the baseline**

- [ ] **Step 3: Convert the mortgage page facts**

`m_price`, `m_down`, `m_rate`, `m_term`, `m_tax`, `m_ins`, `m_common`, `m_maintenance` become typed fields. `m_extra` stays a slider — an extra payment is a decision being explored, not a transcribed fact. Each field keeps its existing id so `DEAL_INPUT_MAP` and every calculation continue to work untouched.

```html
<div class="ir">
  <label class="il" for="m_price">Purchase price</label>
  <input class="fact-field" type="text" inputmode="decimal" id="m_price" value="950000">
</div>
```

Add a shared `input` handler that strips `$` and `,`, parses, and calls `dealSet`; and a `blur` handler that reformats to `$950,000`. Reuse the existing `.iv` parse/format helpers rather than writing new ones.

- [ ] **Step 4: Convert the lender page**

All 14 sliders become typed fields. These are figures transcribed off a Loan Estimate — rate, points, origination, underwriting, application, processing, lock days. Lender name stays the text input it already correctly is.

- [ ] **Step 5: Convert the sale and rent page facts, keep their assumptions as sliders**

Typed: `s_price`, `s_down`, `s_rate`, `s_term`, `s_tax`, `s_ins`, `r_price`, `r_down`, `r_rate`, `r_tax`, `r_maint`, `r_common`, `r_coopMaint`.
Sliders retained, because each is a question rather than a fact: `s_sellYear`, `s_appRate`, `s_commission`, `s_flipTax`, `r_rent`, `r_rentInc`, `r_invest`, `r_years`.

- [ ] **Step 6: Give the retained sliders a live delta**

Each assumption slider gains a readout of what moving it does to that page's headline figure, so dragging shows consequence rather than only a value. Compute against the pre-drag value captured on `pointerdown`/`focus`.

- [ ] **Step 7: Verification cycle**

Assertions: typing `847,500` into `#m_price` yields `deal.price === 847500`; blur reformats to `$847,500`; typing garbage leaves the previous value intact rather than producing `NaN`; `#page-lender input[type=range]` is 0; every retained slider is one of the eight named assumptions; all headline figures still match their pre-change values for the default deal.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "Type the transcribed facts; keep sliders for explored assumptions"
```

---

### Task 4: Affordability

**Files:**
- Modify: `index.html` — new fields in the deal bar edit panel; new "Can you carry this" section on the mortgage page

**Interfaces:**
- Consumes: `deal` (Task 1)
- Produces: `affordability()` → `{ dti, frontEnd, monthsReserves, cashAfterClosing, monthlyTotal, verdictParts }`

- [ ] **Step 1: Assertion that fails**

```js
await page.evaluate(() => { dealSet('grossIncome', 240000); dealSet('liquidAssets', 500000); dealSet('monthlyDebt', 800); });
return await page.evaluate(() => affordability());
// Expected: dti and monthsReserves are finite numbers, not undefined
```

- [ ] **Step 2: Run it, confirm `affordability is not defined`**

- [ ] **Step 3: Add the three fact fields**

`grossIncome`, `liquidAssets`, `monthlyDebt` as typed fields in the deal bar edit panel, all optional. Every derived figure must degrade to `—` when income is 0, never to `NaN` or `Infinity`.

- [ ] **Step 4: Implement `affordability()`**

```js
function affordability() {
  const loan = deal.price - deal.downPayment;
  const pi = monthlyPI(loan, deal.rate, deal.term);
  const housing = pi + deal.annualTax / 12 + deal.annualInsurance / 12
    + (deal.propertyType === 'coop' ? deal.monthlyMaintenance
       : deal.propertyType === 'condo' ? deal.monthlyCommon : 0);
  const grossMonthly = deal.grossIncome / 12;
  const cashToClose = deal.downPayment + closingCosts();
  const cashAfter = deal.liquidAssets - cashToClose;
  return {
    monthlyTotal: housing,
    frontEnd: grossMonthly > 0 ? (housing / grossMonthly) * 100 : null,
    dti: grossMonthly > 0 ? ((housing + deal.monthlyDebt) / grossMonthly) * 100 : null,
    cashAfterClosing: cashAfter,
    monthsReserves: housing > 0 ? cashAfter / housing : null
  };
}
```

Reuse the existing closing-cost routine as `closingCosts()`; extract it from `updateMortgage` unchanged if it is inline.

- [ ] **Step 5: Render the section**

Three figures — DTI against the 43% ceiling, cash left after closing, months of reserves — each with an `ok`/`warn`/`late` tone reusing the classes Task 6 keeps. Copy states arithmetic: "38% of gross — under the 43% ceiling lenders use", "4.2 months of payments left after closing". Negative cash after closing is the one case that gets an explicit warning, because it means the deal cannot close.

- [ ] **Step 6: Verification cycle**

Assertions: income 0 yields `—` everywhere and no `NaN` in the DOM; income 240000 / assets 500000 / debt 800 on the default deal yields a DTI matching a hand calculation stated in the test; `liquidAssets` below `cashToClose` shows the negative-cash warning; co-op vs condo vs single-family each pick the right monthly charge.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add affordability: DTI, cash after closing, months of reserves"
```

---

### Task 5: Verdict-first page headers

**Files:**
- Modify: `index.html` — the `.ph` header block on the mortgage, sale, rent, and lender pages; move each page's inputs below its verdict

**Interfaces:**
- Consumes: `affordability()` (Task 4), `deal` (Task 1)
- Produces: `renderVerdicts()`, called from `recalcAll()`

- [ ] **Step 1: Assertion that fails**

```js
return { subhead: await page.textContent('#page-mortgage .ph p') };
// Now: "Adjust the sliders — all figures update in real time."
// After: a sentence about this deal, containing a figure
```

- [ ] **Step 2: Run it, record the current copy**

- [ ] **Step 3: Add a verdict line to each of the four calculator pages**

One sentence, arithmetic only. Mortgage: monthly total, share of gross, cash left after closing. Sale: net proceeds at the chosen year and whether it exceeds cash invested. Rent: which option is ahead and by how much — the existing "Renting wins" treatment, kept and promoted. Lenders: the cheaper lender and the total-cost gap.

- [ ] **Step 4: Delete "Adjust the sliders"**

Replaced by the verdict. The mechanism does not need narrating.

- [ ] **Step 5: Move inputs below the verdict and collapse them**

Each page opens with its answer; `Your inputs` becomes a collapsed `<details>`-style block, open by default only when the deal is still at its defaults (nothing entered yet).

- [ ] **Step 6: Verification cycle**

Assertions: each of the four pages has a verdict containing a digit; no page contains the string "Adjust the sliders"; verdicts update when `dealSet('price', …)` runs; with income unset, the mortgage verdict omits the share-of-gross clause rather than printing `null%`.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Lead each calculator page with its answer instead of its form"
```

---

### Task 6: The Instrument visual system

Runs first in practice — it is independent of the data work and it is the change the user actually asked for. Split into 6a–6d so each is separately reviewable; the app must render correctly after each.

**Files:**
- Modify: `index.html` — the Google Fonts `<link>` (~line 10); the entire `:root` token block (~lines 14-35); every structural selector; all six `.ph` header blocks; `<nav>`

**Interfaces:**
- Consumes: nothing — CSS, markup classes, and one attribute on `<html>`
- Produces: the token set below; `.caption`, `.schedule`, `.typed`, `.stamp`, `.marginal` class conventions; `toggleTheme()`; `data-theme` on `<html>`

**Token set** — replaces the existing `:root` wholesale:

```css
:root{
  --paper:#EDEAE3; --ink:#14161A; --ink2:#4A4E57; --ink3:#7B7F88;
  --blueback:#1F2B40; --stamp:#8C2F27; --manila:#D9C9A3; --rule:#C9C3B6;
  --ff-doc:'Source Serif 4',Georgia,serif;
  --ff-label:'Archivo Narrow',Arial Narrow,sans-serif;
  --ff-typed:'Courier Prime',Courier,monospace;
}
```

Every `--sp-*`, `--radius-*`, `--shadow-*`, `--accent`, `--bg*`, `--ff`, and `--fm` reference is removed or remapped. Grep for each token name after the swap; a surviving reference to a deleted token renders as an invalid value and fails silently.

- [ ] **6a: Tokens, fonts, and the caption block.** Swap the font link to `Source Serif 4`, `Archivo Narrow`, and `Courier Prime`. Replace the `:root` block. Convert all six `.ph` headers from fluid serif heroes to ruled caption blocks naming the instrument and the property. Delete every `.eyebrow`. Verify: no element computes to a font-size below 11px; `grep -c 'clamp(24px' index.html` is 0; `grep -c 'letter-spacing:.18em' index.html` is 0.
- [ ] **6b: Structure.** Replace `.card`, `.guide-checklist`, `.cmt-hero`, and `.cmt-item` with ruled `.schedule` rows and hanging indents. Remove every gradient and every `border-radius`. Convert the guide's step numbers to `¶` references and the commitment checklist to lettered `(a)`–`(t)` schedule items. Verify: `grep -cE 'linear-gradient|border-radius' index.html` is 0.
- [ ] **6c: The typed-entry signature.** Every user-editable field renders `--ff-typed` on a `.typed` fill-in line (`border-bottom:1px solid var(--rule)`, transparent background). Static and computed figures render `--ff-doc` with tabular figures. Verify: every `input` computes to Courier Prime; every stat value computes to Source Serif 4.
- [ ] **6d: Index nav, the stamp, and night mode.** Nav becomes a ruled roman-numeral index. One `.stamp` per page, rotated, in `--stamp`, pressing in over 200ms on load and suppressed under `prefers-reduced-motion`. Dark mode inverts to ink-on-charcoal with full token parity. Verify: exactly one `.stamp` visible per page; `data-theme="dark"` yields a dark ground with 4.5:1 on `--ink2`; the press animation does not run with reduced motion forced.

### Task 7: Stage model and one progress spine

**Files:**
- Modify: `index.html` — nav; the guide page checklist; the commitment page checklist; `switchPage`

**Interfaces:**
- Consumes: `deal.stage`, `deal.contractDate` (Task 1); `commitMilestones`, `defaultGuideChecklist`
- Produces: `inferStage()`, `setStage(stage)`, `renderNavForStage()`, `journeyMilestones`

- [ ] **Step 1: Assertion that fails**

```js
await page.evaluate(() => { dealSet('contractDate', '2026-08-06'); });
return await page.evaluate(() => ({ stage: deal.stage, inferred: inferStage() }));
// Expected after Task 7: both 'contract'
```

- [ ] **Step 2: Run it, confirm `inferStage is not defined`**

- [ ] **Step 3: Implement `inferStage()` and `setStage()`**

A contract date implies `contract`. All commitment milestones checked implies `closing`. An explicit user choice always wins over inference and is stored on `deal.stage`.

- [ ] **Step 4: Merge the two checklists into `journeyMilestones`**

The guide's 8 items become the `exploring` and `contract` segments; the commitment page's 20 become the `contract` segment's detail. One array, each entry tagged with its stage, so the guide shows the spine and the commitment page shows one segment zoomed in. Both existing storage keys keep working: `journeyMilestones` entries carry their original `guideChecklist` or `commitChecklist` key so no saved progress is lost.

- [ ] **Step 5: Make the nav stage-aware**

Tabs reorder so the current stage's page leads, and pages for stages not yet reached are de-emphasized but never hidden — hiding navigation would trap someone whose inferred stage is wrong.

- [ ] **Step 6: Verification cycle**

Assertions: setting a contract date moves the stage to `contract` and reorders the nav; an explicit `setStage('exploring')` survives a reload despite a contract date being present; a v2 fixture with `guideChecklist.financesReady` and `commitChecklist.exactDate` set shows both still checked after the merge; every tab remains reachable in every stage.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add stage model and merge the two checklists into one spine"
```

---

### Task 8: Print, share, and open-house mode

**Files:**
- Modify: `index.html` — `@media print` block; hash routing; a `#quick` view

**Interfaces:**
- Consumes: `deal`, `affordability()`, `renderDealBar()`
- Produces: `encodeDeal()`, `decodeDeal(str)`, `renderQuick()`

- [ ] **Step 1: Assertion that fails**

```js
const url = await page.evaluate(() => encodeDeal());
// Expected after Task 8: a hash string that decodeDeal() round-trips to an equal deal
```

- [ ] **Step 2: Run it, confirm `encodeDeal is not defined`**

- [ ] **Step 3: Implement the share URL**

`encodeDeal()` JSON-stringifies the deal, `encodeURIComponent`s it, and base64s the result into `#d=…`. On load, a `#d=` hash decodes and applies it before `pushDealToInputs()`. A malformed hash is ignored silently and the stored deal is used — a bad link must never wipe saved state.

- [ ] **Step 4: Add the print stylesheet**

`@media print`: white ground, black text, nav / deal bar / theme toggle / charts hidden, one page containing the deal facts, the closing-cost breakdown, affordability, and the commitment deadline. Page-break rules keep each block whole.

- [ ] **Step 5: Build open-house mode**

Reached at `#quick`. One screen: a single price field, then monthly, cash to close, and how it compares to the deal's own price. Reads the deal; stores nothing of its own. If it needs its own state, cut it per the spec.

- [ ] **Step 6: Verification cycle**

Assertions: `decodeDeal(encodeDeal())` deep-equals `deal`; `#d=garbage` leaves the stored deal untouched and logs no error; `#quick` renders and computes at 390px; print emulation shows no nav, no charts, and content inside one page height; a shared link opened in a fresh context with empty `localStorage` reproduces the deal.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add print summary, shareable deal URL, and open-house mode"
```

---

## Self-Review

**Spec coverage.** All eleven functional requirements map to tasks: 1→T1, 2→T2, 3→T3, 4→T5, 5→T7, 6→T7, 7→T4, 8→T6, 9→T6, 10→T8, 11→T8. Non-functional constraints are in Global Constraints and re-checked in every task's regression step. The migration risk named in the spec is Task 1 Step 8 assertion 4.

**Type consistency.** `monthlyPI(loan, rate, years)` and `closingCosts()` are extracted in T2 and T4 respectively and reused by name in T4, T5, and T8. `affordability()` returns the same five keys everywhere it is consumed. `DEAL_INPUT_MAP` keys match `dealDefaults` keys exactly, including `annualUpkeep` versus `monthlyMaintenance`. `recalcAll()` is defined in T1 and extended, never redefined, in T2 and T5.

**Known ordering constraint.** T2 extracts `monthlyPI` and T4 extracts `closingCosts` from inside `updateMortgage`. Both extractions must preserve the arithmetic exactly; each task's regression step compares the mortgage page's headline figures against their pre-change values for the default deal to prove it.
