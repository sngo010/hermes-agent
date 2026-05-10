---
name: finance-sheets
description: Log expenses, income, debt, and travel budget to Google Sheets. Query budgets, cash flow, and net worth. Used by Finley the personal finance agent.
version: 2.0.0
metadata:
  hermes:
    tags: [finance, google-sheets, budget, income, debt, tracking]
    category: personal-finance
    config:
      - key: finance.sheets.spreadsheet_id
        description: "Google Sheets spreadsheet ID (from the URL)"
        prompt: "Enter your Google Sheets spreadsheet ID"
---

# Finance — Google Sheets Tracker

## When to Use
Load this skill whenever Sophie says anything related to:
- Logging expenses: "Spent $X on Y" / "Hai paid $X" / "X VND on Y"
- Logging income: "Salary deposited" / "Got paid" / "Dividend received"
- Logging debt: "Credit card balance is $X" / "Paid $X on Visa" / "Mortgage balance"
- Queries: "Budget status" / "Cash flow" / "Debt summary" / "Net worth"
- Travel: "Vietnam status" / "Trip summary"

## Prerequisites
- google-workspace skill must be authorized
- Spreadsheet ID saved in config: `finance.sheets.spreadsheet_id`

Set GAPI shorthand before all operations:
```bash
GAPI="python ${HERMES_HOME:-/opt/data}/skills/productivity/google-workspace/scripts/google_api.py"
SHEET_ID="YOUR_SPREADSHEET_ID"
```

---

## Google Sheet Structure

**Tab: Expenses**
`Date | Description | Category | CAD Amount | Original Amount | Original Currency | Who | Trip | Month | Notes`

**Tab: Income**
`Date | Description | Category | CAD Amount | Who | Month | Notes`

**Tab: Debt**
`Date | Debt Name | Type | Balance CAD | Rate % | Min Payment | Limit | Notes`

**Tab: Travel**
`Date | Description | Category | CAD Amount | VND Amount | Who | Trip Name | Day`

**Tab: NetWorth**
`Date | Account | Type | Balance CAD | Notes`

**Tab: Budget**
`Category | Monthly Budget CAD`

Pre-fill Budget tab:
```
Groceries | 600
Dining | 400
Housing | 3000
Transport | 300
Kids | 500
Health | 200
Shopping | 300
Utilities & Subscriptions | 200
Travel | 0
Professional | 150
Savings & Investments | 1500
Misc | 200
```

---

## Procedures

### 1. Log an Expense

Parse: description, amount, currency (CAD/VND), category, who (Sophie/Hai), is_travel.
Convert VND: `cad = vnd / vnd_rate` (from memory, default 17500).

```bash
$GAPI sheets append $SHEET_ID "Expenses!A:J" \
  --values "[[$DATE,$DESC,$CAT,$CAD,$ORIG,$CURR,$WHO,$TRIP,$MONTH,\"\"]]"

# If travel active, also log to Travel tab
$GAPI sheets append $SHEET_ID "Travel!A:H" \
  --values "[[$DATE,$DESC,$CAT,$CAD,$VND,$WHO,\"Vietnam 2026\",$DAY]]"
```

Read monthly total then reply:
```
✅ Logged: [emoji] [description] — $X CAD
📊 This month: $X spent / $X budget
💰 Remaining: $X ($X/day for X days left)
```

---

### 2. Log Income

Parse: description, amount CAD, category (Employment/Bonus/Investment/Rental/Other), who.

```bash
$GAPI sheets append $SHEET_ID "Income!A:G" \
  --values "[[$DATE,$DESC,$CAT,$AMOUNT,$WHO,$MONTH,\"\"]]"
```

Read month income total then reply:
```
✅ Income logged: [category] +$X ([who])
📅 [Month] income so far: $X
💰 Household surplus this month: $X (income $X − expenses $X)
```

---

### 3. Log / Update Debt

Parse: debt name, balance, type, rate %, minimum payment, limit (credit cards only).

```bash
$GAPI sheets append $SHEET_ID "Debt!A:H" \
  --values "[[$DATE,$NAME,$TYPE,$BALANCE,$RATE,$MIN,$LIMIT,\"\"]]"
```

Read most recent row per debt name (not sum — balance snapshots).

Reply:
```
✅ Debt updated: [name]
💳 [Name]: $X remaining (↓$X from last update)
📊 Total debt: $X | Debt-to-income: X%
```

Proactive checks:
- Credit utilization > 30% → ⚠️ flag
- Payment due within 5 days → 🚨 alert

---

### 4. Debt Summary Query

```bash
$GAPI sheets get $SHEET_ID "Debt!A:H"
```

Get latest row per debt. Sort by rate descending (avalanche order).

```
💳 Debt Summary

🏠 Mortgage         $680,000  @ 4.5%   Min: $2,400/mo
🏦 LOC              $12,000   @ 7.2%   Min: $120/mo
💳 Visa             $3,200    @ 19.9%  Min: $64/mo  ⚠️ 42% utilization

Total debt: $695,200
Total min payments: $2,584/mo

🎯 Avalanche order:
1. Visa (19.9%)
2. LOC (7.2%)
3. Mortgage (4.5%)
```

---

### 5. Cash Flow Summary

```bash
$GAPI sheets get $SHEET_ID "Income!A:G"
$GAPI sheets get $SHEET_ID "Expenses!A:J"
$GAPI sheets get $SHEET_ID "Debt!A:H"
```

Filter all by current month. Calculate net cash flow and savings rate.

```
📊 [Month] Household Cash Flow

Income:
  Sophie employment    +$X
  Hai employment       +$X
  Other                +$X
  Total Income:        +$X

Expenses:             -$X
Debt payments:        -$X
Savings/investments:  -$X
──────────────────────────
NET CASH FLOW:        +/-$X
Savings rate:         X%
```

Flag if savings rate < 20%: `⚠️ Savings rate below 20% target`

---

### 6. Budget Status Query

```bash
$GAPI sheets get $SHEET_ID "Expenses!A:J"
$GAPI sheets get $SHEET_ID "Budget!A:B"
```

Filter expenses by current month. Group by category. Compare to targets.

```
📅 [Month] Budget — Day X of Y

✅ On track:
  🛒 Groceries      $420 / $600  (70%)

⚠️  Watch:
  🍽 Dining         $380 / $400  (95%)

🚨 Over:
  🛍 Shopping       $650 / $500  (130%)

Total: $X / $X | Projected: $X
```

---

### 7. Vietnam Trip Status

```bash
$GAPI sheets get $SHEET_ID "Travel!A:H"
```

Filter by Trip Name = "Vietnam 2026". Group by category.
Get trip_budget, trip_end from memory.

```
🇻🇳 Vietnam Trip Status

Spent:     $X of $X CAD (X%)
Remaining: $X | $X/day for X days left

🍜 Food & Drinks    $X
🛵 Transport        $X
🏨 Accommodation    $X
🎭 Activities       $X
🛍 Shopping         $X  ⚠️ approaching 25% limit
```

---

### 8. Net Worth Update / Summary

```bash
# Update
$GAPI sheets append $SHEET_ID "NetWorth!A:E" \
  --values "[[$DATE,$ACCOUNT,$TYPE,$BALANCE,$NOTES]]"

# Read
$GAPI sheets get $SHEET_ID "NetWorth!A:E"
```

Latest row per account. Assets - liabilities = net worth.

```
💰 Net Worth — [Date]

Assets:
  TFSA Sophie     $X
  TFSA Hai        $X
  RRSP Sophie     $X
  RRSP Hai        $X
  Wealthsimple    $X
  Total:          $X

Liabilities:
  Mortgage        $X
  LOC             $X
  Credit Cards    $X
  Total:          $X

NET WORTH: $X  (MoM: +/-$X)
```

---

## Category Mapping

| Keywords | Category |
|---|---|
| grocery, whole foods, superstore, save-on, costco, T&T | Groceries |
| restaurant, cafe, coffee, pho, sushi, dining, takeout, uber eats, doordash | Dining |
| rent, mortgage, strata, hydro, insurance, home | Housing |
| uber, lyft, grab, taxi, transit, compass, gas, parking | Transport |
| school, tuition, daycare, childcare, chess, swim, tennis | Kids |
| pharmacy, doctor, dentist, medical, gym | Health |
| amazon, shopping, clothing, zara, nordstrom | Shopping |
| netflix, spotify, phone, internet, subscription | Utilities & Subscriptions |
| hotel, flight, airbnb, tour, trip | Travel |
| course, conference, dues, books, software | Professional |
| tfsa, rrsp, wealthsimple, investment, savings | Savings & Investments |
| salary, paycheque, deposit, payroll | Employment (Income) |
| bonus, incentive, commission | Bonus (Income) |
| dividend, distribution, capital gain | Investment (Income) |
| tax refund, gst, ccb, rebate | Other (Income) |
| visa, mastercard, credit card, loc, mortgage, loan | Debt |

Default: **Misc**

---

## Pitfalls

- **Income ≠ Expense**: Salary deposit → Income tab, never Expenses tab
- **Debt balance**: Latest row per debt name only — not a running sum
- **Who attribution**: Always log Sophie vs Hai correctly for income tracking
- **VND rate**: Check memory, prompt update if >3 days old
- **Tab names**: Case-sensitive — must match exactly
- **GAPI path**: Confirm google-workspace skill is installed and authorized first

## Verification

After any write:
```bash
$GAPI sheets get $SHEET_ID "[Tab]!A:A"
# Confirm last row matches what was logged
```

