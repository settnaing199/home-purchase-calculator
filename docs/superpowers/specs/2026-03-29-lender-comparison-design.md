# Lender Comparison Feature Design

**Date:** 2026-03-29
**Feature:** Lender Comparison Tab for NYC Home Calculator
**Status:** Approved

## Overview

Add a new "Lenders" tab to the NYC Home Calculator that allows users to compare multiple mortgage lenders side-by-side. Users can input lender-specific rates, discount points, and origination fees, then see calculated metrics including monthly payment, total interest, upfront costs, effective APR, and total cost.

## Requirements

### Functional Requirements
- New navigation tab labeled "Lenders" (5th tab after Buying Guide)
- Auto-sync loan amount and term from Mortgage tab
- Support 1-6 lenders for comparison
- User can add/remove lenders dynamically
- User inputs per lender: name, APR, discount points, origination fee
- Display metrics: monthly payment, total interest, upfront costs, effective APR, total cost
- Highlight the best deal (lowest total cost)
- Persist all lender data to localStorage
- "Sync from Mortgage tab" button to refresh loan parameters

### Non-Functional Requirements
- Match existing design system (dark theme, card-based layout, accent colors)
- Responsive layout (2-column desktop, 1-column mobile)
- Real-time calculation updates on input change
- Maintain single-file architecture (all code in index.html)

## Architecture

### Integration Points

**Navigation (lines ~455):**
```html
<button class="nav-tab" onclick="switchPage('page-lender',this)"
        role="tab" aria-selected="false">Lenders</button>
```

**Page Container (after page-guide):**
```html
<div class="page" id="page-lender" role="tabpanel" aria-hidden="true">
  <!-- Lender comparison content -->
</div>
```

**JavaScript Functions:**
- `updateLenders()` - Main calculation function
- `addLender()` - Add new lender to array and render
- `removeLender(id)` - Remove lender and re-render
- `renderLenders()` - Rebuild DOM for all lender cards
- `syncLendersFromMortgage()` - Pull loan/term from mortgage inputs

**State Management:**
- Global `lenders` array stores all lender objects
- Synced values: loan amount and term from mortgage tab
- All data persists to existing localStorage key `'nyc-home-calc-v2'`

## Data Model

### Lender Object
```javascript
{
  id: 'lender-1',           // Unique ID (lender-1, lender-2, etc.)
  name: 'Lender A',         // User-editable name
  rate: 6.5,                // APR as decimal percentage
  points: 0,                // Discount points as percentage
  origFee: 1500            // Origination fee in dollars
}
```

### Global State
```javascript
let lenders = [
  {id: 'lender-1', name: 'Lender A', rate: 6.5, points: 0, origFee: 1500},
  {id: 'lender-2', name: 'Lender B', rate: 6.3, points: 0.5, origFee: 1800}
];
```

### Computed Metrics (per lender)
- **Monthly Payment:** `mthly(loan, lender.rate, term)`
- **Total Interest:** Sum of interest over full loan term via amortization
- **Upfront Costs:** `(lender.points / 100) * loan + lender.origFee`
- **Effective APR:** True APR including fees amortized over loan term
- **Total Cost:** `totalInterest + upfrontCosts`

### Default Values
- Start with 2 lender placeholders
- Default rate: 6.5%, points: 0, origination fee: $1,500
- Max lenders: 6 (UI constraint)
- Min lenders: 1 (can't delete last one)

## UI Design

### Page Structure

```
┌─────────────────────────────────────────────┐
│ COMPARE LENDERS                             │
│ Find the Best Mortgage Deal                 │
│ Compare rates, fees, and total costs        │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ Loan Details (synced from Mortgage)        │
│ • Loan Amount: $800,000                     │
│ • Loan Term: 30 years                       │
│ [Sync from Mortgage Tab]                    │
└─────────────────────────────────────────────┘

┌───────────────────┐  ┌───────────────────┐
│ Lender A       [×]│  │ Lender B       [×]│
│                   │  │                   │
│ Interest Rate     │  │ Interest Rate     │
│ 6.5%         [━●━]│  │ 6.3%         [━●━]│
│                   │  │                   │
│ Discount Points   │  │ Discount Points   │
│ 0%           [━●━]│  │ 0.5%         [━●━]│
│                   │  │                   │
│ Origination Fee   │  │ Origination Fee   │
│ $1,500       [━●━]│  │ $1,800       [━●━]│
│                   │  │                   │
│ ───────────────── │  │ ───────────────── │
│                   │  │                   │
│ Monthly Payment   │  │ Monthly Payment   │
│ $4,770            │  │ $4,652            │
│                   │  │                   │
│ Total Interest    │  │ Total Interest    │
│ $917,000          │  │ $874,000          │
│                   │  │                   │
│ Upfront Costs     │  │ Upfront Costs     │
│ $1,500            │  │ $5,800            │
│                   │  │                   │
│ Effective APR     │  │ Effective APR     │
│ 6.51%             │  │ 6.38%             │
│                   │  │                   │
│ TOTAL COST        │  │ TOTAL COST ✓ Best │
│ $918,500          │  │ $879,800          │
└───────────────────┘  └───────────────────┘

        [+ Add Lender]
```

### Component Breakdown

**1. Page Header**
- Eyebrow: "COMPARE LENDERS" (`.eyebrow`)
- H1: "Find the Best Mortgage Deal" (`.ph h1`)
- Description: "Compare rates, fees, and total costs across multiple lenders"

**2. Sync Info Card**
- Display synced loan amount (formatted with `fmt()`)
- Display loan term
- "Sync from Mortgage Tab" button
- Uses `.card` styling

**3. Lender Cards Grid**
- Grid layout: `.g2` (2 columns desktop, 1 column mobile)
- Each card is a `.card` with lender-specific content

**4. Lender Card Components**
- **Header:** Text input for lender name + remove button
- **Input Rows (`.ir`):**
  - Interest Rate: range slider 3-12%, step 0.1%, displays as `rate.toFixed(1)%`
  - Discount Points: range slider 0-3%, step 0.125%, displays as `points.toFixed(3)%`
  - Origination Fee: range slider $0-$5000, step $100, displays with `fmt()`
- **Metrics Display:**
  - Monthly Payment: Large text, accent color
  - Total Interest: Standard formatting
  - Upfront Costs: Standard formatting
  - Effective APR: Percentage formatting
  - Total Cost: Highlighted if lowest, green border/badge

**5. Add Lender Button**
- Bottom of page, centered
- Disabled when 6 lenders reached
- Uses standard button styling

### Visual Indicators

**Best Deal Highlighting:**
- Card with lowest total cost gets:
  - Green border: `border: 1px solid var(--green)`
  - Badge: "✓ Best Deal" in green
  - Subtle green background tint

**Color Coding:**
- Monthly payment: Accent color (`var(--accent)`)
- Total cost: Green if best (`var(--green)`), normal otherwise
- Remove buttons: Danger color (`var(--danger)`)

## Calculations

### Monthly Payment
```javascript
const monthly = mthly(loan, lender.rate, term);
```
Uses existing `mthly()` function (standard amortization formula).

### Total Interest
```javascript
function calculateTotalInterest(loan, rate, term) {
  let balance = loan;
  let totalInterest = 0;
  const monthlyRate = rate / 100 / 12;
  const monthlyPayment = mthly(loan, rate, term);
  const totalMonths = term * 12;

  for (let i = 0; i < totalMonths; i++) {
    const interest = balance * monthlyRate;
    const principal = monthlyPayment - interest;
    totalInterest += interest;
    balance -= principal;
    if (balance <= 0) break;
  }

  return totalInterest;
}
```

### Upfront Costs
```javascript
const upfrontCosts = (lender.points / 100) * loan + lender.origFee;
```

### Effective APR
```javascript
function calculateEffectiveAPR(loan, rate, term, upfrontCosts) {
  // Net loan amount after upfront costs
  const netLoan = loan - upfrontCosts;
  const monthlyPayment = mthly(loan, rate, term);
  const totalMonths = term * 12;

  // Solve for rate where NPV of payments equals net loan
  // Use Newton's method or binary search
  let low = 0, high = 20;
  let effectiveAPR = rate;

  for (let iter = 0; iter < 20; iter++) {
    const mid = (low + high) / 2;
    const npv = calculateNPV(monthlyPayment, mid, totalMonths);

    if (Math.abs(npv - netLoan) < 1) {
      effectiveAPR = mid;
      break;
    }

    if (npv > netLoan) {
      low = mid;
    } else {
      high = mid;
    }
  }

  return effectiveAPR;
}

function calculateNPV(payment, apr, months) {
  const r = apr / 100 / 12;
  if (r === 0) return payment * months;
  return payment * ((1 - Math.pow(1 + r, -months)) / r);
}
```

### Total Cost
```javascript
const totalCost = totalInterest + upfrontCosts;
```

### Best Deal Detection
```javascript
function findBestLender(lenders, loan, term) {
  let bestId = null;
  let lowestCost = Infinity;

  lenders.forEach(lender => {
    const totalCost = calculateTotalInterest(loan, lender.rate, term)
                    + (lender.points / 100) * loan
                    + lender.origFee;

    if (totalCost < lowestCost) {
      lowestCost = totalCost;
      bestId = lender.id;
    }
  });

  return bestId;
}
```

## Data Persistence

### Storage Integration

**Extend allInputIds:**
```javascript
const allInputIds = [
  'm_price', 'm_down', 'm_rate', ...,
  'lenders_data'  // Add lenders array
];
```

**Save to localStorage:**
```javascript
function saveLenders() {
  localStorage.setItem('nyc-home-calc-v2', JSON.stringify({
    ...existingData,
    lenders: lenders
  }));
}
```

**Load from localStorage:**
```javascript
function loadLenders() {
  const data = JSON.parse(localStorage.getItem('nyc-home-calc-v2'));
  if (data && data.lenders) {
    lenders = data.lenders;
  } else {
    // Initialize with defaults
    lenders = [
      {id: 'lender-1', name: 'Lender A', rate: 6.5, points: 0, origFee: 1500},
      {id: 'lender-2', name: 'Lender B', rate: 6.3, points: 0.5, origFee: 1800}
    ];
  }
}
```

### Sync Mechanism

**Sync from Mortgage Tab:**
```javascript
function syncLendersFromMortgage() {
  // Pull loan amount and term from mortgage tab
  const price = parseFloat(document.getElementById('m_price').value);
  const down = parseFloat(document.getElementById('m_down').value);
  const term = parseFloat(document.getElementById('m_term').value);

  const loan = price - down;

  // Update display
  document.getElementById('lender-loan-display').textContent = fmt(loan);
  document.getElementById('lender-term-display').textContent = term + ' years';

  // Recalculate all lenders
  updateLenders();
}
```

## Implementation Details

### HTML Structure

```html
<div class="page" id="page-lender" role="tabpanel" aria-hidden="true">
  <!-- Page Header -->
  <div class="ph">
    <div class="eyebrow">COMPARE LENDERS</div>
    <h1>Find the Best <em>Mortgage Deal</em></h1>
    <p>Compare rates, fees, and total costs across multiple lenders</p>
  </div>

  <!-- Sync Info Card -->
  <div class="section">
    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center;">
        <div>
          <div style="font-size:11px;color:var(--text3);margin-bottom:var(--sp-2);">LOAN DETAILS</div>
          <div style="font-size:13px;">
            Loan Amount: <span id="lender-loan-display" style="color:var(--accent);">$0</span>
          </div>
          <div style="font-size:13px;">
            Term: <span id="lender-term-display" style="color:var(--accent);">30 years</span>
          </div>
        </div>
        <button onclick="syncLendersFromMortgage()"
                style="padding:var(--sp-3) var(--sp-4);background:var(--bg3);border:1px solid var(--border);border-radius:var(--radius);color:var(--text2);cursor:pointer;font-size:11px;font-family:var(--fm);">
          Sync from Mortgage Tab
        </button>
      </div>
    </div>
  </div>

  <!-- Lender Cards Grid -->
  <div class="section">
    <div class="sl">LENDERS</div>
    <div id="lenders-grid" class="g2" style="gap:var(--sp-5);"></div>
  </div>

  <!-- Add Lender Button -->
  <div style="text-align:center;margin-top:var(--sp-6);">
    <button id="add-lender-btn" onclick="addLender()"
            style="padding:var(--sp-3) var(--sp-5);background:var(--bg3);border:1px solid var(--border);border-radius:var(--radius);color:var(--accent);cursor:pointer;font-size:12px;font-family:var(--fm);">
      + Add Lender
    </button>
  </div>
</div>
```

### JavaScript Functions

**1. updateLenders()**
```javascript
function updateLenders() {
  const price = parseFloat(document.getElementById('m_price').value);
  const down = parseFloat(document.getElementById('m_down').value);
  const term = parseFloat(document.getElementById('m_term').value);
  const loan = price - down;

  let bestId = null;
  let lowestCost = Infinity;

  // Calculate metrics for each lender
  lenders.forEach(lender => {
    const monthly = mthly(loan, lender.rate, term);
    const totalInterest = calculateTotalInterest(loan, lender.rate, term);
    const upfrontCosts = (lender.points / 100) * loan + lender.origFee;
    const effectiveAPR = calculateEffectiveAPR(loan, lender.rate, term, upfrontCosts);
    const totalCost = totalInterest + upfrontCosts;

    // Track best deal
    if (totalCost < lowestCost) {
      lowestCost = totalCost;
      bestId = lender.id;
    }

    // Update DOM
    document.getElementById(`${lender.id}-monthly`).textContent = fmt(monthly);
    document.getElementById(`${lender.id}-interest`).textContent = fmt(totalInterest);
    document.getElementById(`${lender.id}-upfront`).textContent = fmt(upfrontCosts);
    document.getElementById(`${lender.id}-effective-apr`).textContent = effectiveAPR.toFixed(2) + '%';
    document.getElementById(`${lender.id}-total`).textContent = fmt(totalCost);
  });

  // Highlight best deal
  lenders.forEach(lender => {
    const card = document.getElementById(`${lender.id}-card`);
    const badge = document.getElementById(`${lender.id}-badge`);
    if (lender.id === bestId) {
      card.style.borderColor = 'var(--green)';
      badge.style.display = 'block';
    } else {
      card.style.borderColor = 'var(--border)';
      badge.style.display = 'none';
    }
  });

  saveLenders();
}
```

**2. renderLenders()**
```javascript
function renderLenders() {
  const grid = document.getElementById('lenders-grid');
  grid.innerHTML = '';

  lenders.forEach(lender => {
    const card = createLenderCard(lender);
    grid.appendChild(card);
  });

  updateLenders();

  // Enable/disable add button
  document.getElementById('add-lender-btn').disabled = lenders.length >= 6;
}
```

**3. createLenderCard(lender)**
```javascript
function createLenderCard(lender) {
  const card = document.createElement('div');
  card.className = 'card';
  card.id = `${lender.id}-card`;

  card.innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:var(--sp-4);">
      <input type="text" value="${lender.name}"
             onchange="updateLenderName('${lender.id}', this.value)"
             style="background:transparent;border:none;color:var(--text);font-size:14px;font-family:var(--fm);font-weight:400;padding:0;">
      ${lenders.length > 1 ? `<button onclick="removeLender('${lender.id}')" style="background:none;border:none;color:var(--danger);cursor:pointer;font-size:16px;">×</button>` : ''}
    </div>

    <div id="${lender.id}-badge" style="display:none;font-size:10px;color:var(--green);margin-bottom:var(--sp-3);">✓ BEST DEAL</div>

    <!-- Interest Rate -->
    <div class="ir">
      <label class="il">Interest Rate (APR)</label>
      <input type="range" id="${lender.id}-rate" min="3" max="12" step="0.1" value="${lender.rate}"
             oninput="updateLenderRate('${lender.id}', this.value)">
      <div class="iv" id="${lender.id}-rate-val">${lender.rate.toFixed(1)}%</div>
    </div>

    <!-- Discount Points -->
    <div class="ir">
      <label class="il">Discount Points</label>
      <input type="range" id="${lender.id}-points" min="0" max="3" step="0.125" value="${lender.points}"
             oninput="updateLenderPoints('${lender.id}', this.value)">
      <div class="iv" id="${lender.id}-points-val">${lender.points.toFixed(3)}%</div>
    </div>

    <!-- Origination Fee -->
    <div class="ir">
      <label class="il">Origination Fee</label>
      <input type="range" id="${lender.id}-origfee" min="0" max="5000" step="100" value="${lender.origFee}"
             oninput="updateLenderOrigFee('${lender.id}', this.value)">
      <div class="iv" id="${lender.id}-origfee-val">${fmt(lender.origFee)}</div>
    </div>

    <div style="border-top:1px solid var(--border);margin:var(--sp-5) 0;"></div>

    <!-- Metrics -->
    <div style="display:grid;gap:var(--sp-4);">
      <div>
        <div style="font-size:10px;color:var(--text3);margin-bottom:var(--sp-1);">MONTHLY PAYMENT</div>
        <div id="${lender.id}-monthly" style="font-size:20px;color:var(--accent);">$0</div>
      </div>

      <div>
        <div style="font-size:10px;color:var(--text3);margin-bottom:var(--sp-1);">TOTAL INTEREST</div>
        <div id="${lender.id}-interest" style="font-size:14px;">$0</div>
      </div>

      <div>
        <div style="font-size:10px;color:var(--text3);margin-bottom:var(--sp-1);">UPFRONT COSTS</div>
        <div id="${lender.id}-upfront" style="font-size:14px;">$0</div>
      </div>

      <div>
        <div style="font-size:10px;color:var(--text3);margin-bottom:var(--sp-1);">EFFECTIVE APR</div>
        <div id="${lender.id}-effective-apr" style="font-size:14px;">0%</div>
      </div>

      <div style="background:var(--bg3);padding:var(--sp-3);border-radius:var(--radius);">
        <div style="font-size:10px;color:var(--text3);margin-bottom:var(--sp-1);">TOTAL COST</div>
        <div id="${lender.id}-total" style="font-size:16px;font-weight:400;">$0</div>
      </div>
    </div>
  `;

  return card;
}
```

**4. Input Update Handlers**
```javascript
function updateLenderName(id, name) {
  const lender = lenders.find(l => l.id === id);
  lender.name = name;
  saveLenders();
}

function updateLenderRate(id, rate) {
  const lender = lenders.find(l => l.id === id);
  lender.rate = parseFloat(rate);
  document.getElementById(`${id}-rate-val`).textContent = parseFloat(rate).toFixed(1) + '%';
  updateLenders();
}

function updateLenderPoints(id, points) {
  const lender = lenders.find(l => l.id === id);
  lender.points = parseFloat(points);
  document.getElementById(`${id}-points-val`).textContent = parseFloat(points).toFixed(3) + '%';
  updateLenders();
}

function updateLenderOrigFee(id, fee) {
  const lender = lenders.find(l => l.id === id);
  lender.origFee = parseFloat(fee);
  document.getElementById(`${id}-origfee-val`).textContent = fmt(parseFloat(fee));
  updateLenders();
}
```

**5. Add/Remove Lenders**
```javascript
function addLender() {
  if (lenders.length >= 6) return;

  const newId = 'lender-' + Date.now();
  lenders.push({
    id: newId,
    name: 'Lender ' + String.fromCharCode(65 + lenders.length),
    rate: 6.5,
    points: 0,
    origFee: 1500
  });

  renderLenders();
}

function removeLender(id) {
  if (lenders.length <= 1) return;
  lenders = lenders.filter(l => l.id !== id);
  renderLenders();
}
```

## Edge Cases & Error Handling

1. **No lenders:** Always maintain at least 1 lender (disable remove button)
2. **Max lenders:** Disable "Add Lender" button at 6 lenders
3. **Invalid inputs:** Range sliders prevent invalid values
4. **Zero loan amount:** Show $0 for all metrics (no division by zero)
5. **Sync before mortgage tab populated:** Show placeholders, recalculate after sync
6. **localStorage quota exceeded:** Graceful degradation (no persistence)

## Testing Strategy

### Manual Testing Checklist
1. Navigate to Lenders tab - page displays with 2 default lenders
2. Sync from mortgage tab - loan amount and term update correctly
3. Adjust interest rate slider - monthly payment recalculates
4. Adjust discount points - upfront costs and total cost update
5. Adjust origination fee - upfront costs and total cost update
6. Add lender - new card appears, limit enforced at 6
7. Remove lender - card disappears, minimum 1 enforced
8. Edit lender name - name persists
9. Best deal highlighting - green border on lowest total cost
10. Refresh page - all lender data loads from localStorage
11. Mobile responsive - cards stack vertically
12. Compare with different rates - calculations match expectations

### Calculation Validation
- Verify monthly payment matches standard amortization formula
- Verify total interest sums all interest payments over loan term
- Verify upfront costs = points + origination fee
- Verify effective APR accounts for fees
- Verify total cost = total interest + upfront costs
- Cross-check against external mortgage calculators

## Success Metrics

- Users can compare at least 2 lenders side-by-side
- All calculations are accurate to within $1 of industry-standard formulas
- Page loads and responds to input changes within 100ms
- Mobile layout works on screens down to 320px width
- Data persists across page refreshes
- Best deal is clearly highlighted

## Future Enhancements (Out of Scope)

- Export comparison to PDF
- Pre-populated lender rate database (requires API/maintenance)
- Amortization schedule per lender
- Chart visualization of cost comparison
- Email comparison results
- APR vs interest rate explainer tooltip
