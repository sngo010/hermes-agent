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

---

## Amex CSV Import

### When to Trigger
Any time Sophie uploads a CSV file or says:
- "Here is my Amex statement"
- "Import my credit card CSV"
- "Categorize my transactions"
- Sends any file with .csv extension

### Critical: Ignore Amex's Own Categories
Amex CSV contains a "Category" column that shows values like:
- "Flexible" / "Other" — Membership Rewards eligibility
- "Merchandise & Supplies" — Amex's own broad taxonomy

**NEVER use Amex's Category column.** Always re-categorize
from scratch using merchant name matching below.

### Amex CSV Column Mapping

| Amex Column | Use? | Notes |
|---|---|---|
| Date | ✅ Yes | Transaction date |
| Description | ✅ Yes | Primary source for merchant matching |
| Amount | ✅ Yes | Amex shows debits as POSITIVE numbers |
| Extended Details | ✅ Optional | Extra merchant info if Description unclear |
| Appears On Your Statement As | ✅ Optional | Fallback for merchant matching |
| Address / City / Country | ✅ Optional | Helps with ambiguous merchants |
| Category | ❌ IGNORE | Amex internal — not useful |
| Type | ❌ IGNORE | Amex internal classification |
| Reference | ❌ IGNORE | Transaction ID only |

### Rows to Skip Entirely

Skip and do NOT log these row types:
```
Description contains: "PAYMENT", "THANK YOU", "AUTOPAY"
→ These are bill payments, not expenses

Amount is negative
→ These are credits or refunds — log separately to Credits column

Description contains: "ANNUAL FEE"
→ Log to Utilities & Subscriptions as "Amex Annual Fee"
```

### Amex-Specific Merchant Mapping

Add these to the standard category mapping:

```
Merchant Pattern          → Category
─────────────────────────────────────────────────────
WHOLEFDS / WHOLE FOODS    → Groceries
LOBLAWS / LOBLAW          → Groceries
SUPERSTORE / RCSS         → Groceries
SAVE-ON-FOODS / SAVEON    → Groceries
T&T SUPERMARKET           → Groceries
FRESHCO                   → Groceries
COSTCO WHSE               → Groceries (default; may be Shopping)

MCDONALD / MCDONALDS      → Dining
TIM HORTONS / TIMHORTON   → Dining
STARBUCKS                 → Dining
EARLS / BROWNS / CACTUS   → Dining
UBER* EATS / UBEREATS     → Dining
DOORDASH                  → Dining
SKIP THE DISHES / SKIP    → Dining
A&W / SUBWAY / CHIPOTLE   → Dining

AMZN / AMAZON.CA          → Shopping
AMAZON PRIME              → Utilities & Subscriptions
APPLE.COM/BILL            → Utilities & Subscriptions
NETFLIX.COM               → Utilities & Subscriptions
SPOTIFY                   → Utilities & Subscriptions
GOOGLE *                  → Utilities & Subscriptions
TELUS                     → Utilities & Subscriptions
ROGERS                    → Utilities & Subscriptions
SHAW / ROGERS CABLE       → Utilities & Subscriptions
HYDRO / BC HYDRO          → Utilities & Subscriptions

ESSO / CHEVRON / PETRO    → Transport
UBER* (non-EATS)          → Transport
LYFT                      → Transport
TRANSLINK / COMPASS       → Transport
IMPARK / ORCA / PARK      → Transport

SHOPPERS / LONDON DRUG    → Health
REXALL / PHARMASAVE       → Health
LULULEMON / ARITZIA       → Shopping
SPORT CHEK / MEC          → Shopping
WINNERS / HOMESENSE       → Shopping
IKEA / STRUCTUBE          → Shopping

AIRBNB                    → Travel
EXPEDIA / BOOKING.COM     → Travel
WESTJET / AIR CANADA      → Travel
DELTA / UNITED / CATHAY   → Travel

VANCOUVER COLLEGE         → Kids
CHESS / KUMON / SYLVAN    → Kids
REGISTRATION / LESSONS    → Kids

AMEX TRAVEL               → Travel
FOREIGN TRANSACTION FEE   → Misc
```

### Processing Workflow

```python
def process_amex_csv(csv_content):
    results = {
        'logged': [],
        'skipped_payments': [],
        'skipped_credits': [],
        'unrecognized': [],
        'duplicates': []
    }

    for row in parse_csv(csv_content):
        desc = row['Description'].upper()
        amount = float(row['Amount'])

        # Step 1: Skip payments and credits
        if any(x in desc for x in ['PAYMENT', 'THANK YOU', 'AUTOPAY']):
            results['skipped_payments'].append(row)
            continue

        if amount < 0:
            results['skipped_credits'].append(row)
            continue

        # Step 2: Match category from Description
        # Use Amex-specific mapping FIRST, then standard mapping
        category = match_amex_merchant(desc)
        if not category:
            category = match_standard_keywords(desc)
        if not category:
            category = 'Misc'
            results['unrecognized'].append(row)

        # Step 3: Dedup check (date ± 2 days, amount ± $2)
        if is_duplicate(row, existing_expenses):
            results['duplicates'].append(row)
            continue

        # Step 4: Log to Expenses tab
        log_expense(
            date=row['Date'],
            description=row['Description'],
            category=category,
            amount_cad=amount,
            source='amex_csv',
            who='Sophie'  # default; Sophie can correct
        )
        results['logged'].append(row)

    return results
```

### Reply Format After Amex Import

```
📎 Amex Statement — [Month] [Year]

Processed:    X transactions
✅ Logged:    X categorized and added
⏭ Skipped:   X payments/credits (not expenses)
🔁 Duplicate: X already in Sheets
❓ Review:    X unrecognized merchants

Top spending:
🍽 Dining        $X  (X transactions)
🛒 Groceries     $X  (X transactions)
🛍 Shopping      $X  (X transactions)

Unrecognized merchants — how should I categorize these?
→ [MERCHANT NAME] $XX — [date]
→ [MERCHANT NAME] $XX — [date]
```

### After Import — Always Ask

After processing, always ask:
```
"I've logged X transactions. Here are X merchants I couldn't 
recognize — can you tell me the category for each?"
```

List each unrecognized merchant with amount and date.
Once Sophie confirms, update the row AND add to the
Amex merchant mapping in memory for future imports.

### Building Merchant Memory

After Sophie categorizes an unrecognized merchant, store it:
```
memory.set('amex_merchant_map', {
  'SQ *LITTLE RAYMOND': 'Kids',
  'YALETOWN BREW': 'Dining',
  ...
})
```

Check this memory FIRST on every future Amex import —
Sophie's confirmed mappings take priority over all defaults.

---

## Upcoming Expenses Tab

### Tab Structure
`Name | Amount CAD | Due Date | Frequency | Category | Who | Remind Days | Notes | Last Reminded | Status`

- `Frequency` — one-time | monthly | quarterly | annual
- `Who` — Sophie | Hai | Joint
- `Remind Days` — how many days before due date to start reminding
- `Last Reminded` — date Finley last sent a reminder (prevents spam)
- `Status` — upcoming | paid | overdue

### When to Use This Tab
- "Add upcoming expense: property tax $3,200 due July 1"
- "Remind me about car insurance renewal in September"
- "What bills are coming up this month?"
- "What's our total annual expense obligation?"
- Cron job runs daily at 9am to check for due reminders

---

### 9. Log an Upcoming Expense

When Sophie says "add upcoming expense" or "remind me about":

```bash
$GAPI sheets append $SHEET_ID "Upcoming!A:J" \
  --values "[[$NAME,$AMOUNT,$DUE_DATE,$FREQUENCY,$CATEGORY,$WHO,$REMIND_DAYS,$NOTES,'','upcoming']]"
```

Reply:
```
✅ Added: [Name] $X due [Date]
📅 I'll remind you [X] days before — first reminder on [Date]
🔁 Frequency: [one-time/annual/etc]
```

---

### 10. Daily Reminder Check (Cron)

Runs at 9am daily. Query all rows where Status = "upcoming":

```python
def check_upcoming_reminders():
    rows = gapi.sheets_get(SHEET_ID, "Upcoming!A:J")
    today = date.today()

    for row in rows:
        if row['Status'] != 'upcoming':
            continue

        due_date = parse_date(row['Due Date'])
        days_until = (due_date - today).days
        remind_days = int(row['Remind Days'])
        last_reminded = parse_date(row['Last Reminded']) if row['Last Reminded'] else None

        # Already past due
        if days_until < 0:
            send_telegram(f"🔴 OVERDUE: {row['Name']} ${row['Amount CAD']} was due {abs(days_until)} days ago")
            update_status(row, 'overdue')
            continue

        # Within reminder window and not reminded today
        if days_until <= remind_days:
            if not last_reminded or last_reminded < today:
                send_reminder(row, days_until)
                update_last_reminded(row, today)

def send_reminder(row, days_until):
    urgency = "🔴" if days_until <= 3 else "⚠️" if days_until <= 7 else "📅"
    msg = f"""{urgency} Upcoming expense reminder:
{row['Name']} — ${row['Amount CAD']} CAD
Due: {row['Due Date']} ({days_until} days)
Category: {row['Category']} | {row['Who']}
{row['Notes'] if row['Notes'] else ''}"""
    telegram.send(msg)
```

---

### 11. Upcoming Expenses Query

When Sophie asks "what's coming up?" or "upcoming this month":

```bash
$GAPI sheets get $SHEET_ID "Upcoming!A:J"
```

Filter by Status = upcoming, sort by Due Date ascending.

Reply format:
```
📅 Upcoming Expenses

This month:
⚠️  Property tax        $3,200  due Jul 1   (Joint)   — 21 days
📅  Raymond school fees  $2,400  due Jul 15  (Sophie)  — 35 days

Next 3 months:
📅  Car insurance        $1,800  due Sep 15  (Joint)
📅  Costco membership    $130    due Aug 20  (Hai)

Annual total remaining: $X across X payments
Monthly cash flow impact: Jul $X | Aug $X | Sep $X
```

---

### 12. Mark as Paid

When Sophie says "paid [name]" or "mark [expense] as done":

```python
# Find matching row
# Update Status column to 'paid'
# If Frequency is annual/monthly/quarterly:
#   Add new row with next due date automatically
#   next_due = calculate_next_due(due_date, frequency)
```

Reply:
```
✅ Marked [Name] as paid — $X on [date]
📅 Next due: [next date] — added to Upcoming tab
```

---

### 13. Cash Flow Integration

When projecting cash flow, always include Upcoming tab:

```python
def get_upcoming_for_month(year, month):
    rows = gapi.sheets_get(SHEET_ID, "Upcoming!A:J")
    return [r for r in rows
            if r['Status'] == 'upcoming'
            and parse_date(r['Due Date']).year == year
            and parse_date(r['Due Date']).month == month]
```

Add upcoming expenses to the projected expenses line in cash flow:
```
July projection:
  Regular expenses (avg):   $7,800
  Upcoming — property tax:  $3,200
  Upcoming — school fees:   $2,400
  ─────────────────────────────────
  Total projected:          $13,400
  ⚠️ July is a heavy month — $5,600 above average
```

---

### Upcoming Tab Pitfalls

- **Annual expenses**: when marking paid, auto-create next year's row — don't lose track
- **Joint expenses**: always tag as Joint so both Sophie and Hai see it
- **Don't double-count**: if an upcoming expense is also in Expenses (already paid), Status must be 'paid'
- **Remind Days default**: 30 days for amounts >$500, 7 days for amounts <$500
- **Never remind more than once per day** — check Last Reminded before sending
