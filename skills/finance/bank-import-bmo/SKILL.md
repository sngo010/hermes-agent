---
name: bank-import-bmo
description: Import and categorize BMO credit card and bank account CSV statements into Google Sheets Expenses tab. Handles BMO-specific column format, item numbers, and YYYYMMDD date format.
version: 1.0.0
metadata:
  hermes:
    tags: [finance, bmo, bank-import, csv, canada]
    category: personal-finance
---

# BMO Statement Import

## When to Use
- Sophie uploads a CSV file from BMO online banking
- Says "import my BMO statement" / "here is my BMO CSV"
- File contains columns: Item #, Card No., Transaction Date, Posting Date, Transaction Amount, Description

## BMO CSV Format

### Credit Card CSV columns:
```
Item # | Card No. | Transaction Date | Posting Date | Transaction Amount | Description
```

### Chequing/Savings CSV columns:
```
Date | Description | Withdrawals | Deposits | Balance
```

### Key differences from Amex and RBC:
| Property | BMO Credit Card | BMO Chequing | Amex |
|---|---|---|---|
| Date format | YYYYMMDD (e.g. 20260105) | YYYY-MM-DD | MM/DD/YYYY |
| Debit amount | NEGATIVE number | Withdrawals column | POSITIVE |
| Payment | POSITIVE amount | Deposits column | POSITIVE with keyword |
| Description | Single Description column | Single Description | Description + Extended Details |
| Extra columns | Item #, Card No., Posting Date | Balance column | Reference, Address |

---

## Column Translation — BMO Credit Card to Expenses Tab

```python
def translate_bmo_credit_row(bmo_row):
    # Parse date — BMO credit card uses YYYYMMDD
    raw_date = bmo_row.get('Transaction Date', '')
    date = reformat_date(raw_date, from_fmt='%Y%m%d', to_fmt='%Y-%m-%d')

    description = bmo_row.get('Description', '').strip()

    # BMO credit card: debits are NEGATIVE
    amount_raw = bmo_row.get('Transaction Amount', '0').replace(',', '')
    amount = float(amount_raw)
    is_debit = amount < 0
    expense_amount = abs(amount)

    return {
        'Date':              date,
        'Description':       description,
        'Category':          infer_category(description),
        'CAD Amount':        round(expense_amount, 2),
        'Original Amount':   round(expense_amount, 2),
        'Original Currency': 'CAD',
        'Who':               'Sophie',
        'Trip':              '',
        'Month':             date[:7],
        'Notes':             f"BMO Card: {bmo_row.get('Card No.', '')}",
        'Dedup_Hash':        make_hash(date, expense_amount, description),
        'Source':            'bmo_csv',
        'Status':            'pending',
        '_is_debit':         is_debit
    }
```

## Column Translation — BMO Chequing to Expenses Tab

```python
def translate_bmo_chequing_row(bmo_row):
    # BMO chequing uses YYYY-MM-DD
    date = bmo_row.get('Date', '')
    description = bmo_row.get('Description', '').strip()

    # Separate Withdrawals and Deposits columns
    withdrawal = bmo_row.get('Withdrawals', '').replace(',', '').replace('-', '')
    deposit = bmo_row.get('Deposits', '').replace(',', '')

    is_withdrawal = bool(withdrawal and float(withdrawal) > 0)
    is_deposit = bool(deposit and float(deposit) > 0)
    amount = float(withdrawal) if is_withdrawal else float(deposit) if is_deposit else 0

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
        'Source':            'bmo_csv',
        'Status':            'pending',
        '_is_withdrawal':    is_withdrawal,
        '_is_deposit':       is_deposit
    }
```

---

## How to Detect BMO File Type

```python
def detect_bmo_file_type(headers):
    credit_headers = {'Item #', 'Card No.', 'Transaction Date', 'Transaction Amount'}
    chequing_headers = {'Withdrawals', 'Deposits', 'Balance'}

    header_set = set(headers)
    if credit_headers.issubset(header_set):
        return 'credit_card'
    elif chequing_headers.issubset(header_set):
        return 'chequing'
    else:
        return 'unknown'
```

---

## Rows to Skip

### Credit Card:
| Row type | How to detect | Action |
|---|---|---|
| Payment | Amount POSITIVE AND description contains PAYMENT, THANK YOU | Skip Expenses. Update Debt tab |
| Refund | Amount POSITIVE AND not payment keyword | Skip Expenses. Note as credit |
| Interest | Description contains INTEREST, FINANCE CHARGE | Log → Expenses → Debt |
| Annual fee | Description contains ANNUAL FEE | Log → Expenses → Utilities |
| Cash advance | Description contains CASH ADVANCE | Log → Expenses → Misc |

### Chequing:
| Row type | How to detect | Action |
|---|---|---|
| Payroll deposit | Deposits column + PAYROLL, DIRECT DEP | Log to Income tab |
| Government | Deposits + CANADA, CRA, CCB, GST | Log to Income tab |
| Bill payment | Description contains BILL PAYMENT, BMO BILL | Skip — already in credit card |
| E-transfer sent | Description contains INTERAC e-TRF | Log as expense if to external |
| E-transfer received | Deposits + INTERAC e-TRF | Log to Income tab |
| NSF/overdraft fee | Description contains NSF, OD INTEREST | Log → Expenses → Misc |
| ATM withdrawal | Description contains ATM, CASH | Log → Expenses → Misc |

---

## BMO Merchant Mapping

```
BMO Description Pattern      → Category
────────────────────────────────────────────────────
LOBLAWS / LOBLAW             → Groceries
REAL CDN SUPERSTORE          → Groceries
SAVE ON FOODS                → Groceries
T&T SUPERMARKET              → Groceries
METRO / FOOD BASICS          → Groceries
SOBEYS                       → Groceries

TIM HORTONS                  → Dining
MCDONALD'S                   → Dining
STARBUCKS                    → Dining
EARLS                        → Dining
BROWNS SOCIALHOUSE           → Dining
UBER EATS                    → Dining
SKIP THE DISHES              → Dining

PETRO CANADA                 → Transport
ESSO                         → Transport
CHEVRON                      → Transport
TRANSLINK                    → Transport
PARKPLUS / IMPARK            → Transport
AUTOPLAN / ICBC              → Transport

BC HYDRO                     → Utilities & Subscriptions
FORTISBC                     → Utilities & Subscriptions
TELUS                        → Utilities & Subscriptions
ROGERS / FIDO                → Utilities & Subscriptions
SHAW / ROGERS CABLE          → Utilities & Subscriptions
NETFLIX                      → Utilities & Subscriptions
SPOTIFY                      → Utilities & Subscriptions

SHOPPERS DRUG MART           → Health
LONDON DRUGS                 → Health
REXALL                       → Health

AMAZON / AMZN                → Shopping
COSTCO                       → Groceries
SPORT CHEK                   → Shopping
LULULEMON                    → Shopping

BMO BANK CHARGE              → Misc (bank fees)
SERVICE CHARGE               → Misc
INTERAC e-TRF                → Review (could be anything)
```

---

## Reply Format After BMO Import

```
🏦 BMO Statement — [Month] [Year]
Account: [Credit Card / Chequing / Savings]

Processed:    X transactions
✅ Expenses:  X logged
💰 Income:    X deposits logged to Income tab
⏭ Skipped:   X payments/transfers
🔁 Duplicate: X already in Sheets
❓ Review:    X flagged

[same spending breakdown as Amex format]
```

---

## Pitfalls

- **Two date formats**: credit card uses YYYYMMDD (no dashes), chequing uses YYYY-MM-DD — detect before parsing
- **Detect file type first**: credit card and chequing have completely different columns
- **Item # column**: ignore — BMO internal sequence number
- **Card No.**: use only for Notes field to track which card
- **Multiple cards**: if multiple card numbers in one CSV, tag Who based on card number in memory
- **Posting Date vs Transaction Date**: always use Transaction Date for logging
- **BMO Savings**: same format as chequing — treat identically
