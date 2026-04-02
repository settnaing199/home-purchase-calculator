# Lender Comparison Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a "Lenders" tab to the NYC Home Calculator that allows users to compare multiple mortgage lenders side-by-side based on rates, points, and fees.

**Architecture:** Single-file HTML architecture maintained. New page section added to existing index.html. JavaScript functions follow existing patterns (global state, real-time updates, localStorage persistence). All calculations use existing `mthly()` helper and follow the same amortization logic as the mortgage calculator.

**Tech Stack:** Vanilla JavaScript, HTML5, CSS3 (embedded), Chart.js (already included)

---

## File Structure

**Modified:**
- `index.html` - Add navigation button (line ~456), new page section (after line 754), JavaScript functions for lender calculations (after line 1065), storage integration (modify existing save/load functions around line 1528)

**No new files created** - maintaining single-file architecture per spec.

---

## Task 1: Add Navigation Tab

**Files:**
- Modify: `index.html:456`

- [ ] **Step 1: Add Lenders tab button to navigation**

Add this line after the "Buying guide" button on line 456:

```html
  <button class="nav-tab" onclick="switchPage('lender',this)" role="tab" aria-selected="false">Lenders</button>
```

- [ ] **Step 2: Test navigation tab appears**

Open `index.html` in browser. Verify:
- 5th tab labeled "Lenders" appears in navigation
- Tab is clickable (will show empty page for now)
- Tab styling matches other tabs

- [ ] **Step 3: Commit navigation change**

```bash
git add index.html
git commit -m "feat: add Lenders navigation tab"
```

---

## Task 2: Add Lenders Page HTML Structure

**Files:**
- Modify: `index.html:~1050` (after page-guide closing tag)

- [ ] **Step 1: Find insertion point for new page**

Locate the closing `</div>` tag for `page-guide` (around line 1050). The new page will be inserted immediately after.

- [ ] **Step 2: Add complete Lenders page HTML**

Insert this HTML block after the `page-guide` closing tag:

```html
<!-- ══ PAGE 5 — LENDER COMPARISON ══ -->
<div class="page" id="page-lender" role="tabpanel" aria-hidden="true">
  <div class="ph">
    <div class="eyebrow">COMPARE LENDERS</div>
    <h1>Find the Best <em>Mortgage Deal</em></h1>
    <p>Compare rates, fees, and total costs across multiple lenders</p>
  </div>

  <!-- Sync Info Card -->
  <div class="section">
    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center;flex-wrap:wrap;gap:var(--sp-4);">
        <div>
          <div style="font-size:11px;color:var(--text3);margin-bottom:var(--sp-2);">LOAN DETAILS</div>
          <div style="font-size:13px;">
            Loan Amount: <span id="lender-loan-display" style="color:var(--accent);">$0</span>
          </div>
          <div style="font-size:13px;">
            Term: <span id="lender-term-display" style="color:var(--accent);">30 years</span>
          </div>
        </div>
        <button onclick="syncLendersFromMortgage()" style="padding:var(--sp-3) var(--sp-4);background:var(--bg3);border:1px solid var(--border);border-radius:var(--radius);color:var(--text2);cursor:pointer;font-size:11px;font-family:var(--fm);transition:all var(--duration) var(--ease);">
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
    <button id="add-lender-btn" onclick="addLender()" style="padding:var(--sp-3) var(--sp-5);background:var(--bg3);border:1px solid var(--border);border-radius:var(--radius);color:var(--accent);cursor:pointer;font-size:12px;font-family:var(--fm);transition:all var(--duration) var(--ease);">
      + Add Lender
    </button>
  </div>
</div>
```

- [ ] **Step 3: Test page structure**

Open in browser and click "Lenders" tab. Verify:
- Page header appears with "Find the Best Mortgage Deal"
- Loan Details card shows (with $0 values)
- Empty grid area for lender cards
- "Add Lender" button appears

- [ ] **Step 4: Commit page structure**

```bash
git add index.html
git commit -m "feat: add Lenders page HTML structure"
```

---

## Task 3: Initialize Lender State and Helper Functions

**Files:**
- Modify: `index.html:~1067` (after `propertyTypes` declaration)

- [ ] **Step 1: Add global lenders array**

Add this after the `propertyTypes` declaration (around line 1068):

```javascript
// ── Lender Comparison State ──
let lenders = [
  {id: 'lender-1', name: 'Lender A', rate: 6.5, points: 0, origFee: 1500},
  {id: 'lender-2', name: 'Lender B', rate: 6.3, points: 0.5, origFee: 1800}
];
```

- [ ] **Step 2: Add calculation helper functions**

Add these functions after the lenders array:

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

function calculateNPV(payment, apr, months) {
  const r = apr / 100 / 12;
  if (r === 0) return payment * months;
  return payment * ((1 - Math.pow(1 + r, -months)) / r);
}

function calculateEffectiveAPR(loan, rate, term, upfrontCosts) {
  if (upfrontCosts === 0) return rate;

  const netLoan = loan - upfrontCosts;
  const monthlyPayment = mthly(loan, rate, term);
  const totalMonths = term * 12;

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
```

- [ ] **Step 3: Test helper functions in console**

Open browser console and test:

```javascript
// Test calculateTotalInterest
calculateTotalInterest(800000, 6.5, 30); // Should return ~917,000
// Test calculateEffectiveAPR
calculateEffectiveAPR(800000, 6.5, 30, 5000); // Should return ~6.52
```

- [ ] **Step 4: Commit helper functions**

```bash
git add index.html
git commit -m "feat: add lender calculation helper functions"
```

---

## Task 4: Implement createLenderCard Function

**Files:**
- Modify: `index.html:~1120` (after calculation helpers)

- [ ] **Step 1: Add createLenderCard function**

```javascript
function createLenderCard(lender) {
  const card = document.createElement('div');
  card.className = 'card';
  card.id = `${lender.id}-card`;

  const removeBtn = lenders.length > 1
    ? `<button onclick="removeLender('${lender.id}')" style="background:none;border:none;color:var(--danger);cursor:pointer;font-size:18px;line-height:1;padding:var(--sp-1);" aria-label="Remove ${lender.name}">×</button>`
    : '';

  card.innerHTML = `
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:var(--sp-4);">
      <input type="text" value="${lender.name}" onchange="updateLenderName('${lender.id}', this.value)" style="background:transparent;border:none;color:var(--text);font-size:14px;font-family:var(--fm);font-weight:400;padding:0;flex:1;min-width:0;" aria-label="Lender name">
      ${removeBtn}
    </div>

    <div id="${lender.id}-badge" style="display:none;font-size:10px;color:var(--green);margin-bottom:var(--sp-3);letter-spacing:0.1em;">✓ BEST DEAL</div>

    <div class="ir">
      <label class="il">Interest Rate (APR)</label>
      <input type="range" id="${lender.id}-rate" min="3" max="12" step="0.1" value="${lender.rate}" oninput="updateLenderRate('${lender.id}', this.value)" aria-label="Interest rate">
      <div class="iv" id="${lender.id}-rate-val">${lender.rate.toFixed(1)}%</div>
    </div>

    <div class="ir">
      <label class="il">Discount Points</label>
      <input type="range" id="${lender.id}-points" min="0" max="3" step="0.125" value="${lender.points}" oninput="updateLenderPoints('${lender.id}', this.value)" aria-label="Discount points">
      <div class="iv" id="${lender.id}-points-val">${lender.points.toFixed(3)}%</div>
    </div>

    <div class="ir">
      <label class="il">Origination Fee</label>
      <input type="range" id="${lender.id}-origfee" min="0" max="5000" step="100" value="${lender.origFee}" oninput="updateLenderOrigFee('${lender.id}', this.value)" aria-label="Origination fee">
      <div class="iv" id="${lender.id}-origfee-val">${fmt(lender.origFee)}</div>
    </div>

    <div style="border-top:1px solid var(--border);margin:var(--sp-5) 0;"></div>

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

- [ ] **Step 2: Test card creation in console**

Open console and test:

```javascript
// Test card creation
const testCard = createLenderCard(lenders[0]);
console.log(testCard); // Should show a DOM element
```

- [ ] **Step 3: Commit createLenderCard function**

```bash
git add index.html
git commit -m "feat: add createLenderCard function"
```

---

## Task 5: Implement Input Update Handlers

**Files:**
- Modify: `index.html:~1200` (after createLenderCard)

- [ ] **Step 1: Add input update handler functions**

```javascript
function updateLenderName(id, name) {
  const lender = lenders.find(l => l.id === id);
  if (lender) {
    lender.name = name;
    saveToStorage();
  }
}

function updateLenderRate(id, rate) {
  const lender = lenders.find(l => l.id === id);
  if (lender) {
    lender.rate = parseFloat(rate);
    document.getElementById(`${id}-rate-val`).textContent = parseFloat(rate).toFixed(1) + '%';
    updateLenders();
  }
}

function updateLenderPoints(id, points) {
  const lender = lenders.find(l => l.id === id);
  if (lender) {
    lender.points = parseFloat(points);
    document.getElementById(`${id}-points-val`).textContent = parseFloat(points).toFixed(3) + '%';
    updateLenders();
  }
}

function updateLenderOrigFee(id, fee) {
  const lender = lenders.find(l => l.id === id);
  if (lender) {
    lender.origFee = parseFloat(fee);
    document.getElementById(`${id}-origfee-val`).textContent = fmt(parseFloat(fee));
    updateLenders();
  }
}
```

- [ ] **Step 2: Commit input handlers**

```bash
git add index.html
git commit -m "feat: add lender input update handlers"
```

---

## Task 6: Implement updateLenders Calculation Function

**Files:**
- Modify: `index.html:~1230` (after input handlers)

- [ ] **Step 1: Add updateLenders function**

```javascript
function updateLenders() {
  const price = parseFloat(document.getElementById('m_price').value) || 0;
  const down = parseFloat(document.getElementById('m_down').value) || 0;
  const term = parseFloat(document.getElementById('m_term').value) || 30;
  const loan = price - down;

  if (loan <= 0) {
    // Show zeros if no loan amount
    lenders.forEach(lender => {
      document.getElementById(`${lender.id}-monthly`).textContent = '$0';
      document.getElementById(`${lender.id}-interest`).textContent = '$0';
      document.getElementById(`${lender.id}-upfront`).textContent = '$0';
      document.getElementById(`${lender.id}-effective-apr`).textContent = '0%';
      document.getElementById(`${lender.id}-total`).textContent = '$0';
    });
    return;
  }

  let bestId = null;
  let lowestCost = Infinity;

  lenders.forEach(lender => {
    const monthly = mthly(loan, lender.rate, term);
    const totalInterest = calculateTotalInterest(loan, lender.rate, term);
    const upfrontCosts = (lender.points / 100) * loan + lender.origFee;
    const effectiveAPR = calculateEffectiveAPR(loan, lender.rate, term, upfrontCosts);
    const totalCost = totalInterest + upfrontCosts;

    if (totalCost < lowestCost) {
      lowestCost = totalCost;
      bestId = lender.id;
    }

    document.getElementById(`${lender.id}-monthly`).textContent = fmt(monthly);
    document.getElementById(`${lender.id}-interest`).textContent = fmt(totalInterest);
    document.getElementById(`${lender.id}-upfront`).textContent = fmt(upfrontCosts);
    document.getElementById(`${lender.id}-effective-apr`).textContent = effectiveAPR.toFixed(2) + '%';
    document.getElementById(`${lender.id}-total`).textContent = fmt(totalCost);
  });

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

  saveToStorage();
}
```

- [ ] **Step 2: Commit updateLenders function**

```bash
git add index.html
git commit -m "feat: add updateLenders calculation function"
```

---

## Task 7: Implement renderLenders Function

**Files:**
- Modify: `index.html:~1290` (after updateLenders)

- [ ] **Step 1: Add renderLenders function**

```javascript
function renderLenders() {
  const grid = document.getElementById('lenders-grid');
  if (!grid) return;

  grid.innerHTML = '';

  lenders.forEach(lender => {
    const card = createLenderCard(lender);
    grid.appendChild(card);
  });

  updateLenders();

  const addBtn = document.getElementById('add-lender-btn');
  if (addBtn) {
    addBtn.disabled = lenders.length >= 6;
    addBtn.style.opacity = lenders.length >= 6 ? '0.5' : '1';
    addBtn.style.cursor = lenders.length >= 6 ? 'not-allowed' : 'pointer';
  }
}
```

- [ ] **Step 2: Commit renderLenders function**

```bash
git add index.html
git commit -m "feat: add renderLenders function"
```

---

## Task 8: Implement Add/Remove Lender Functions

**Files:**
- Modify: `index.html:~1310` (after renderLenders)

- [ ] **Step 1: Add addLender and removeLender functions**

```javascript
function addLender() {
  if (lenders.length >= 6) return;

  const newId = 'lender-' + Date.now();
  const newName = 'Lender ' + String.fromCharCode(65 + lenders.length);

  lenders.push({
    id: newId,
    name: newName,
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

- [ ] **Step 2: Test add/remove in browser**

Navigate to Lenders tab and verify:
- Click "Add Lender" - new card appears
- Can add up to 6 lenders, button disables at 6
- Click × on a lender card - card is removed
- Cannot remove last lender (no × button when only 1 remains)

- [ ] **Step 3: Commit add/remove functions**

```bash
git add index.html
git commit -m "feat: add addLender and removeLender functions"
```

---

## Task 9: Implement Sync Function

**Files:**
- Modify: `index.html:~1335` (after add/remove functions)

- [ ] **Step 1: Add syncLendersFromMortgage function**

```javascript
function syncLendersFromMortgage() {
  const price = parseFloat(document.getElementById('m_price').value) || 0;
  const down = parseFloat(document.getElementById('m_down').value) || 0;
  const term = parseFloat(document.getElementById('m_term').value) || 30;

  const loan = price - down;

  const loanDisplay = document.getElementById('lender-loan-display');
  const termDisplay = document.getElementById('lender-term-display');

  if (loanDisplay) loanDisplay.textContent = fmt(loan);
  if (termDisplay) termDisplay.textContent = term + ' years';

  updateLenders();
}
```

- [ ] **Step 2: Test sync functionality**

In browser:
1. Go to Mortgage tab
2. Set price: $1,000,000, down: $200,000, term: 30 years
3. Click Lenders tab
4. Click "Sync from Mortgage Tab"
5. Verify loan amount shows $800,000 and term shows 30 years

- [ ] **Step 3: Commit sync function**

```bash
git add index.html
git commit -m "feat: add syncLendersFromMortgage function"
```

---

## Task 10: Integrate with localStorage

**Files:**
- Modify: `index.html:~1528` (existing saveToStorage function)
- Modify: `index.html:~1534` (existing loadFromStorage function)

- [ ] **Step 1: Find saveToStorage function**

Locate the existing `saveToStorage()` function around line 1528.

- [ ] **Step 2: Add lenders to data object being saved**

Inside the `data` object construction (before `localStorage.setItem`), add:

```javascript
    lenders: lenders,
```

The full data object should include this new line along with existing properties.

- [ ] **Step 3: Find loadFromStorage function**

Locate the existing `loadFromStorage()` function around line 1534.

- [ ] **Step 4: Add lenders loading logic**

After the existing input value restoration logic, add:

```javascript
    if (data.lenders) {
      lenders = data.lenders;
    }
```

- [ ] **Step 5: Test localStorage persistence**

In browser:
1. Navigate to Lenders tab
2. Add a 3rd lender, change rates/points
3. Refresh the page
4. Navigate back to Lenders tab
5. Verify all 3 lenders are still there with correct values

- [ ] **Step 6: Commit storage integration**

```bash
git add index.html
git commit -m "feat: integrate lenders with localStorage"
```

---

## Task 11: Initialize Lenders Page on Load

**Files:**
- Modify: `index.html:~1588` (end of loadFromStorage or init section)

- [ ] **Step 1: Add page initialization for lenders**

Find the page initialization code (likely at the end of the `<script>` section). Add a call to initialize the lenders page after the existing initialization:

```javascript
// Initialize lenders page
if (document.getElementById('lenders-grid')) {
  renderLenders();
  syncLendersFromMortgage();
}
```

This should run after `loadFromStorage()` is called.

- [ ] **Step 2: Test initial page load**

Refresh browser and verify:
- Click Lenders tab immediately after load
- Two default lenders appear
- Loan details are synced from Mortgage tab
- All calculations show correct values

- [ ] **Step 3: Commit initialization**

```bash
git add index.html
git commit -m "feat: initialize lenders page on load"
```

---

## Task 12: Manual Testing and Bug Fixes

**Files:**
- Modify: `index.html` (any discovered issues)

- [ ] **Step 1: Test complete user flow**

Complete workflow test:
1. Navigate to Mortgage tab, enter: $1M price, $200k down, 6.5% rate, 30yr term
2. Navigate to Lenders tab
3. Verify 2 default lenders appear
4. Click "Sync from Mortgage Tab" - verify $800k loan, 30 years
5. Adjust Lender A rate to 6.0% - verify monthly payment updates
6. Adjust Lender B points to 1.0% - verify upfront costs update
7. Verify "Best Deal" badge appears on lower total cost
8. Add 3rd lender - verify card appears
9. Remove Lender B - verify card disappears
10. Refresh page - verify all data persists
11. Test on mobile viewport (resize to 375px) - verify cards stack

- [ ] **Step 2: Test edge cases**

Edge case testing:
1. Try to add 7th lender - verify button is disabled at 6
2. Try to remove last remaining lender - verify no × button
3. Set mortgage price to $0 - verify lenders show $0 metrics
4. Set extreme values (rate 12%, points 3%, fee $5000) - verify calculations work

- [ ] **Step 3: Calculation validation**

Use external mortgage calculator (e.g., bankrate.com) to verify:
- Monthly payment for $800k loan at 6.5% for 30yr = ~$5,057
- Total interest over 30yr ≈ $1,020,000
- Effective APR with 1 point ($8000) ≈ 6.62%

Compare with lender comparison calculator output - should match within $10.

- [ ] **Step 4: Fix any discovered bugs**

Document and fix any issues found during testing. Common issues to watch for:
- NaN values when inputs are empty
- Layout breaking on mobile
- Best deal highlighting not updating
- localStorage not saving all fields

- [ ] **Step 5: Commit bug fixes**

```bash
git add index.html
git commit -m "fix: resolve issues found in manual testing"
```

---

## Task 13: Accessibility and Polish

**Files:**
- Modify: `index.html` (aria labels, keyboard navigation)

- [ ] **Step 1: Test keyboard navigation**

Using only keyboard (Tab, Enter, Arrow keys):
1. Tab through navigation - verify Lenders tab is reachable
2. Press Enter on Lenders tab - verify page switches
3. Tab through lender cards - verify all inputs are reachable
4. Use Arrow keys on range sliders - verify values update
5. Tab to "Add Lender" button - press Enter - verify lender added
6. Tab to × button - press Enter - verify lender removed

- [ ] **Step 2: Test with screen reader (optional)**

If screen reader is available (macOS VoiceOver, NVDA, JAWS):
- Verify navigation tabs are announced correctly
- Verify input labels are read properly
- Verify metric values are announced

- [ ] **Step 3: Add any missing aria labels**

If any inputs or buttons lack clear labels, add aria-label attributes. The createLenderCard function already includes some aria-labels, verify they're sufficient.

- [ ] **Step 4: Test focus indicators**

Verify:
- All interactive elements show focus outline when tabbed to
- Focus outline uses accent color (existing CSS should handle this)

- [ ] **Step 5: Commit accessibility improvements**

```bash
git add index.html
git commit -m "feat: improve accessibility for lender comparison"
```

---

## Task 14: Final Verification

**Files:**
- None (read-only verification)

- [ ] **Step 1: Run through complete feature checklist**

Verify all requirements from spec:
- ✓ New navigation tab labeled "Lenders"
- ✓ Auto-sync loan amount and term from Mortgage tab
- ✓ Support 1-6 lenders for comparison
- ✓ User can add/remove lenders dynamically
- ✓ User inputs: name, APR, discount points, origination fee
- ✓ Display metrics: monthly payment, total interest, upfront costs, effective APR, total cost
- ✓ Highlight best deal (lowest total cost)
- ✓ Persist all lender data to localStorage
- ✓ "Sync from Mortgage tab" button works
- ✓ Match existing design system
- ✓ Responsive layout (2-col desktop, 1-col mobile)
- ✓ Real-time calculation updates
- ✓ Single-file architecture maintained

- [ ] **Step 2: Cross-browser testing (if available)**

Test in multiple browsers:
- Chrome/Edge (Chromium)
- Firefox
- Safari

Verify no console errors and consistent rendering.

- [ ] **Step 3: Performance check**

Open DevTools Performance tab:
- Adjust a lender rate slider
- Verify updateLenders() completes in < 100ms
- Verify no layout thrashing or long tasks

- [ ] **Step 4: Create final commit if needed**

If any final tweaks were made:

```bash
git add index.html
git commit -m "chore: final polish for lender comparison feature"
```

- [ ] **Step 5: Verification complete**

Feature is ready for use. All tasks completed.

---

## Testing Summary

### Manual Test Cases

1. **Navigation:** Lenders tab appears and switches to lender page
2. **Sync:** Loan details pull from Mortgage tab correctly
3. **Add Lender:** Can add up to 6 lenders
4. **Remove Lender:** Can remove lenders (min 1 remains)
5. **Input Updates:** All sliders update values and recalculate
6. **Calculations:** Monthly payment, interest, APR, costs are accurate
7. **Best Deal:** Green border highlights lowest total cost
8. **Persistence:** Data survives page refresh
9. **Responsive:** Layout works on mobile and desktop
10. **Accessibility:** Keyboard navigation and screen reader support

### Calculation Validation

Use these test values to verify calculations:

**Test Case 1:**
- Loan: $800,000
- Rate: 6.5%
- Term: 30 years
- Points: 0
- Origination: $1,500

Expected:
- Monthly: ~$5,057
- Total Interest: ~$1,020,000
- Upfront: $1,500
- Total Cost: ~$1,021,500

**Test Case 2:**
- Loan: $800,000
- Rate: 6.3%
- Term: 30 years
- Points: 0.5% ($4,000)
- Origination: $1,800

Expected:
- Monthly: ~$4,949
- Total Interest: ~$981,000
- Upfront: $5,800
- Effective APR: ~6.38%
- Total Cost: ~$986,800

---

## Success Criteria

- ✓ All 14 tasks completed
- ✓ All manual test cases pass
- ✓ Calculations match external calculators (±$10)
- ✓ No console errors
- ✓ Responsive on mobile (tested at 375px width)
- ✓ Data persists across page refreshes
- ✓ Best deal highlighting works correctly
- ✓ Keyboard accessible
- ✓ Matches existing design system
