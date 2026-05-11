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

---

## Deduplication — Bank Statement Upload

### When to Run
Triggered when Sophie says:
- "Upload my bank statement"
- "Import transactions from CSV"
- "Reconcile my [bank] statement"
- Sends a CSV file attachment

### Updated Expenses Tab Structure
Add three columns to the right of existing columns:
`... Notes | Dedup_Hash | Source | Status`

- `Dedup_Hash` — 10-char fingerprint for duplicate detection
- `Source` — `telegram` | `csv_upload` | `email_import` | `receipt_photo`
- `Manual` entries get source tagged at time of logging
- `Status` — `confirmed` | `pending` | `flagged_review`

### Deduplication Script

```python
import hashlib
import csv
from datetime import datetime, timedelta

def make_hash(date, amount, description):
    """Generate dedup fingerprint — tolerates minor amount differences"""
    desc_clean = str(description).lower().strip()[:15]
    amt_rounded = round(float(amount))  # round to nearest dollar
    key = f"{date}|{amt_rounded}|{desc_clean}"
    return hashlib.md5(key.encode()).hexdigest()[:10]

def is_fuzzy_duplicate(new_tx, existing_rows):
    """
    Fuzzy match for near-duplicates:
    - Date within ±2 days (bank settlement lag)
    - Amount within ±$2.00 (tips, rounding, FX)
    Returns matching row or None
    """
    new_date = datetime.strptime(new_tx['date'], '%Y-%m-%d')
    new_amt = float(new_tx['amount'])

    for row in existing_rows:
        try:
            row_date = datetime.strptime(row['Date'], '%Y-%m-%d')
            row_amt = float(row['CAD Amount'])
            date_diff = abs((new_date - row_date).days)
            amt_diff = abs(new_amt - row_amt)

            if date_diff <= 2 and amt_diff <= 2.00:
                return row  # duplicate found
        except (ValueError, KeyError):
            continue
    return None

def process_csv_upload(csv_content, sheet_id, gapi):
    """Main dedup logic for bank statement CSV"""

    # 1. Load existing expenses from Sheets
    existing = gapi.sheets_get(sheet_id, "Expenses!A:M")
    existing_hashes = set(row.get('Dedup_Hash', '') for row in existing)

    results = {
        'new': [],
        'duplicates': [],
        'flagged': []
    }

    for tx in csv.DictReader(csv_content.splitlines()):
        # Normalize fields (bank CSVs vary — adapt column names as needed)
        date = tx.get('Date', tx.get('Transaction Date', ''))
        amount = tx.get('Amount', tx.get('Debit', '0'))
        description = tx.get('Description', tx.get('Merchant', ''))

        if not date or not amount:
            continue

        # Convert negative amounts (some banks use negative for debits)
        amount = abs(float(amount))

        # Layer 1: Hash match (exact duplicate)
        tx_hash = make_hash(date, amount, description)
        if tx_hash in existing_hashes:
            results['duplicates'].append(tx)
            continue

        # Layer 2: Fuzzy match (near-duplicate)
        fuzzy_match = is_fuzzy_duplicate(
            {'date': date, 'amount': amount},
            existing
        )

        if fuzzy_match:
            # Amount mismatch > $0.50 — flag for review
            row_amt = float(fuzzy_match['CAD Amount'])
            if abs(amount - row_amt) > 0.50:
                results['flagged'].append({
                    'bank': tx,
                    'manual': fuzzy_match,
                    'diff': abs(amount - row_amt)
                })
                # Update existing row status to flagged_review
                gapi.sheets_update(sheet_id, f"Expenses!M{fuzzy_match['_row']}", [['flagged_review']])
            else:
                # Minor rounding difference — confirm manual entry
                results['duplicates'].append(tx)
                gapi.sheets_update(sheet_id, f"Expenses!M{fuzzy_match['_row']}", [['confirmed']])
            continue

        # Layer 3: No match — insert as new transaction
        category = infer_category(description)  # use category mapping
        new_row = [
            date, description, category, amount, amount, 'CAD',
            'Auto', '', date[:7], '',  # Notes empty
            tx_hash, 'csv_upload', 'pending'
        ]
        gapi.sheets_append(sheet_id, "Expenses!A:M", [new_row])
        results['new'].append(tx)

    return results
```

### Reply Format After Upload

```
📎 [Bank] Statement — [Month] [Year]

Processed:   X transactions
✅ New:       X added
⚠️  Duplicates: X skipped (matched manual entries)
✅ Confirmed: X manual entries verified against bank
🔍 Review:   X flagged (amount mismatch)

[If flagged:]
Flagged for your review:
→ [Merchant] [Date]: you logged $X, bank shows $Y (Δ$Z)
→ [Merchant] [Date]: you logged $X, bank shows $Y (Δ$Z)

Reply "confirm all" to accept bank amounts, or correct individually.
```

### Handling Flagged Transactions

When Sophie says "confirm all":
- Accept bank amounts as truth
- Update CAD Amount in each flagged row to bank amount
- Update Status to "confirmed"
- Keep Sophie's original description (more readable than bank codes)

When Sophie corrects individually:
- "Keep my amount for Tim Hortons" → Status = confirmed, keep manual amount
- "Use bank amount for Grab" → update to bank amount, Status = confirmed

### Bank CSV Column Mapping

Different banks export different column names. Map on import:

| Bank | Date col | Amount col | Description col |
|---|---|---|---|
| RBC | `Transaction Date` | `CAD$` | `Description 1` |
| TD | `Date` | `Debit` | `Description` |
| CIBC | `Date` | `Debit` | `Transaction Name` |
| Scotiabank | `Date` | `Withdrawals` | `Description` |
| Wealthsimple | `Date` | `Amount` | `Merchant` |
| VPBank (VN) | `Ngày GD` | `Số tiền` | `Nội dung` |
| VietcomBank | `Ngày` | `Phát sinh Nợ` | `Diễn giải` |

If column names don't match, ask Sophie: "What bank is this statement from?"

### Pitfalls

- **Negative amounts**: Some banks use negative for debits — always use `abs(amount)`
- **Date format**: Parse flexibly — `%Y-%m-%d`, `%d/%m/%Y`, `%m/%d/%Y` all appear
- **VND statements**: Amount will be in thousands — divide by rate from memory for CAD
- **Credit card credits/refunds**: Skip rows where Amount is negative/credit — these are not expenses
- **Transfer rows**: Skip rows where Description contains "transfer", "payment", "e-transfer" — not expenses
- **Never insert income rows into Expenses tab** — if Amount is positive credit, route to Income tab instead
