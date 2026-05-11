---
name: bank-import-cibc
description: Import and categorize CIBC credit card and bank account CSV statements into Google Sheets Expenses tab. Handles CIBC-specific split Debit/Credit columns and YYYY-MM-DD date format.
version: 1.0.0
metadata:
  hermes:
    tags: [finance, cibc, bank-import, csv, canada]
    category: personal-finance
---

# CIBC Statement Import

## When to Use
- Sophie uploads a CSV file from CIBC online banking
- Says "import my CIBC statement" / "here is my CIBC CSV"
- File contains columns: Date, Description, Debit, Credit, Balance (chequing)
- OR: Date, Description, Debit, Credit (credit card — no Balance column)

## CIBC CSV Format

### Credit Card CSV columns:
```
Date | Description | Debit | Credit
```

### Chequing/Savings CSV columns:
```
Date | Description | Debit | Credit | Balance
```

### Key differences from Amex, RBC, and BMO:
| Property | CIBC | RBC | BMO | Amex |
|---|---|---|---|---|
| Date format | YYYY-MM-DD | M/D/YYYY | YYYYMMDD (credit) | MM/DD/YYYY |
| Debit column | Separate `Debit` column | Negative CAD$ | Negative amount | Positive Amount |
| Credit column | Separate `Credit` column | Positive CAD$ | Positive amount | Positive with keyword |
| Amount sign | No negatives — split columns | Negative=debit | Negative=debit | Always positive |
| Balance | Yes (chequing) / No (credit card) | No | Yes (chequing) | No |

---

## Column Translation — CIBC to Expenses Tab

```python
def translate_cibc_row(cibc_row):
    date = cibc_row.get('Date', '').strip()  # already YYYY-MM-DD

    description = cibc_row.get('Description', '').strip()

    # CIBC uses SEPARATE Debit and Credit columns — no negatives
    debit_raw = cibc_row.get('Debit', '').replace(',', '').strip()
    credit_raw = cibc_row.get('Credit', '').replace(',', '').strip()

    debit = float(debit_raw) if debit_raw else 0.0
    credit = float(credit_raw) if credit_raw else 0.0

    is_expense = debit > 0
    is_payment_or_credit = credit > 0
    amount = debit if is_expense else credit

    return {
        'Date':              date,
        'Description':       description,
        'Category':          infer_category(description),
        'CAD Amount':        round(amount, 2),
        'Original Amount':   round(amount, 2),
        'Original Currency': 'CAD',
        'Who':               'Sophie',
        'Trip':              '',
        'Month':             date[:7],
        'Notes':             '',
        'Dedup_Hash':        make_hash(date, amount, description),
        'Source':            'cibc_csv',
        'Status':            'pending',
        '_is_expense':       is_expense,
        '_is_credit':        is_payment_or_credit
    }
```

---

## Rows to Skip

### Critical rule for CIBC:
**If Credit column has a value → it is a payment, refund, or deposit. Skip from Expenses.**
**If Debit column has a value → it is an expense. Log it.**

| Row type | How to detect | Action |
|---|---|---|
| Payment | Credit column populated + PAYMENT, THANK YOU in Description | Skip Expenses. Update Debt tab |
| Refund | Credit column populated + not payment keyword | Skip Expenses. Note as credit |
| Interest charge | Debit column + INTEREST, FIN CHG | Log → Expenses → Debt |
| Annual fee | Debit column + ANNUAL FEE | Log → Expenses → Utilities |
| Over-limit fee | Debit column + OVER LIMIT | Log → Expenses → Misc |
| NSF fee | Debit column + NSF, RETURNED | Log → Expenses → Misc |

### Chequing-specific skips:
```
Credit + PAYROLL, DIRECT DEPOSIT → Income tab
Credit + INTERAC e-Transfer → Income tab (money received)
Credit + GOVERNMENT, CANADA, CRA → Income tab
Debit + CIBC BILL PAYMENT → Skip (paying credit card — internal)
Debit + WIRE TRANSFER OUT → Review flag
Debit + ATM → Log → Misc (cash withdrawal)
```

---

## How to Detect CIBC File Type

```python
def detect_cibc_file_type(headers):
    has_balance = 'Balance' in headers
    has_debit_credit = 'Debit' in headers and 'Credit' in headers

    if has_debit_credit and has_balance:
        return 'chequing'
    elif has_debit_credit and not has_balance:
        return 'credit_card'
    else:
        return 'unknown'
```

---

## CIBC Merchant Mapping

```
CIBC Description Pattern     → Category
────────────────────────────────────────────────────
WHOLEFDS / WHOLE FOODS       → Groceries
LOBLAWS                      → Groceries
SAVE ON FOODS                → Groceries
REAL CDN SUPERSTORE          → Groceries
FAIRWAY MARKET               → Groceries
T&T SUPERMARKET              → Groceries
COSTCO WHOLESALE             → Groceries

TIM HORTONS                  → Dining
MCDONALD'S                   → Dining
STARBUCKS                    → Dining
EARLS KITCHEN                → Dining
CACTUS CLUB                  → Dining
UBER* EATS                   → Dining
DOORDASH                     → Dining
SKIP THE DISHES              → Dining
FRESHSLICE / PIZZA           → Dining

PETRO CANADA                 → Transport
ESSO                         → Transport
CHEVRON                      → Transport
HUSKY                        → Transport
TRANSLINK                    → Transport
MOBI BIKE / LIME             → Transport
PARKPLUS / HONK              → Transport
ICBC AUTOPLAN                → Transport

BC HYDRO                     → Utilities & Subscriptions
FORTISBC                     → Utilities & Subscriptions
TELUS                        → Utilities & Subscriptions
ROGERS / FIDO / CHATR        → Utilities & Subscriptions
SHAW COMMUNICATIONS          → Utilities & Subscriptions
NETFLIX.COM                  → Utilities & Subscriptions
SPOTIFY                      → Utilities & Subscriptions
AMAZON PRIME                 → Utilities & Subscriptions
APPLE.COM/BILL               → Utilities & Subscriptions
GOOGLE *STORAGE              → Utilities & Subscriptions

SHOPPERS DRUG MART           → Health
LONDON DRUGS                 → Health
REXALL                       → Health
PHARMASAVE                   → Health
MEDICENTRE / CLINIC          → Health

AMAZON.CA / AMZN             → Shopping
SPORT CHEK / FGL             → Shopping
LULULEMON                    → Shopping
WINNERS / HOMESENSE          → Shopping
IKEA CANADA                  → Shopping
STRUCTUBE                    → Shopping
INDIGO / CHAPTERS            → Shopping

VANCOUVER COLLEGE            → Kids
YMCA / RECREATION            → Kids
SWIM / TENNIS / CHESS        → Kids

CIBC FEE / SERVICE FEE       → Misc (bank fees)
CIBC VISA ANNUAL FEE         → Utilities & Subscriptions
INTERAC e-TRF                → Review
ATM WITHDRAWAL               → Misc
```

---

## Reply Format After CIBC Import

```
🏦 CIBC Statement — [Month] [Year]
Account: [Visa / Chequing / Savings]

Processed:    X transactions
✅ Expenses:  X logged (from Debit column)
💰 Income:    X deposits logged to Income tab
⏭ Skipped:   X payments/refunds (Credit column)
🔁 Duplicate: X already in Sheets
❓ Review:    X flagged

[same spending breakdown format]
```

---

## Pitfalls

- **Split columns are CIBC's biggest difference**: always check Debit vs Credit column, never assume sign from amount
- **Zero values**: CIBC sometimes exports 0.00 in both columns — skip these rows entirely
- **Date is clean**: YYYY-MM-DD already — no reformatting needed
- **Balance column**: ignore for expense tracking — only useful for reconciliation
- **CIBC Credit Card vs Visa**: both use same format — treat identically
- **Joint accounts**: if CIBC account is joint with Hai, ask Sophie which transactions are hers vs Hai's — default to Joint if unclear
- **Interac e-Transfer**: could be expense (sent to someone) or income (received) — always flag for Sophie to confirm category
