---
name: bank-import-rbc
description: Import and categorize RBC credit card and bank account CSV statements into Google Sheets Expenses tab. Handles RBC-specific column format, negative debit amounts, and dual description fields.
version: 1.0.0
metadata:
  hermes:
    tags: [finance, rbc, bank-import, csv, canada]
    category: personal-finance
---

# RBC Statement Import

## When to Use
- Sophie uploads a CSV file from RBC online banking
- Says "import my RBC statement" / "here is my RBC CSV"
- File contains columns: Transaction Date, Description 1, Description 2, CAD$

## RBC CSV Format

### Credit Card CSV columns:
```
Account Type | Account Number | Transaction Date | Cheque Number | 
Description 1 | Description 2 | CAD$ | USD$
```

### Chequing/Savings CSV columns:
```
Account Type | Account Number | Transaction Date | Cheque Number |
Description 1 | Description 2 | CAD$ | USD$
```

### Key differences from Amex:
| Property | RBC | Amex |
|---|---|---|
| Date format | M/D/YYYY (e.g. 1/5/2026) | MM/DD/YYYY or YYYY-MM-DD |
| Debit amount | NEGATIVE number in CAD$ | POSITIVE number |
| Payment amount | POSITIVE number in CAD$ | POSITIVE with "PAYMENT" in description |
| Description | Two columns: Description 1 + Description 2 | One Description column |
| Currency | Separate CAD$ and USD$ columns | Single Amount column |

---

## Column Translation — RBC to Expenses Tab

```python
def translate_rbc_row(rbc_row):
    # Parse date — RBC uses M/D/YYYY
    raw_date = rbc_row.get('Transaction Date', '')
    date = reformat_date(raw_date, from_fmt='%m/%d/%Y', to_fmt='%Y-%m-%d')

    # Combine both description fields
    desc1 = rbc_row.get('Description 1', '').strip()
    desc2 = rbc_row.get('Description 2', '').strip()
    description = f"{desc1} {desc2}".strip()

    # RBC debits are NEGATIVE — flip sign for expenses
    cad_raw = rbc_row.get('CAD$', '0').replace(',', '')
    amount = float(cad_raw)
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
        'Notes':             '',
        'Dedup_Hash':        make_hash(date, expense_amount, description),
        'Source':            'rbc_csv',
        'Status':            'pending',
        '_is_debit':         is_debit,
        '_raw_amount':       amount
    }
```

---

## Rows to Skip

| Row type | How to detect | Action |
|---|---|---|
| Payment | CAD$ is POSITIVE AND description contains PAYMENT, THANK YOU | Skip Expenses. Update Debt tab — reduce RBC balance |
| Credit/refund | CAD$ is POSITIVE AND not a payment | Skip Expenses. Note as credit |
| Transfers | Description 1 contains TRANSFER, TRF, WWW TRF | Skip entirely |
| Cheque payments | Cheque Number column is populated | Log as expense unless >$500 (flag for review) |
| Interest charges | Description contains INTEREST, INT CHG | Log to Expenses → Debt category |
| NSF fees | Description contains NSF, NON-SUFFICIENT | Log to Expenses → Misc |

### RBC-specific skip patterns:
```
"INTERNET BANKING TRANSFER" → Skip
"PAYROLL DEPOSIT" → Skip Expenses, log to Income tab
"DIRECT DEPOSIT" → Skip Expenses, log to Income tab
"MOBILE DEPOSIT" → Skip Expenses, log to Income tab
"ATM WITHDRAWAL" → Log to Expenses → Misc (cash withdrawal)
"WEB TRF FR" / "WEB TRF TO" → Skip (internal transfer)
```

---

## RBC Merchant Mapping

Supplement the standard category mapping with RBC-specific merchant codes:

```
RBC Description Pattern      → Category
────────────────────────────────────────────────────
RCSS (Real Canadian SS)      → Groceries
LOBLAWS                      → Groceries
SAVE-ON-FOODS                → Groceries
REAL CDN SUPERSTORE          → Groceries
SAFEWAY                      → Groceries
IGA                          → Groceries

TIM HORTONS                  → Dining
MCDONALD'S                   → Dining
STARBUCKS                    → Dining
A&W                          → Dining
EARLS KITCHEN                → Dining
BROWN'S SOCIALHOUSE          → Dining

ESSO PETRO                   → Transport
PETRO CAN                    → Transport
CHEVRON                      → Transport
HUSKY                        → Transport
TRANSLINK                    → Transport
COMPASS CARD                 → Transport
AUTOPLAN                     → Transport (insurance)

BC HYDRO                     → Utilities & Subscriptions
FORTISBC / FORTIS            → Utilities & Subscriptions
SHAW CABLE                   → Utilities & Subscriptions
ROGERS                       → Utilities & Subscriptions
TELUS MOBILITY               → Utilities & Subscriptions

LONDON DRUGS                 → Health
SHOPPERS DRUG MART           → Health
REXALL                       → Health
PHARMASAVE                   → Health

RBC INSURANCE                → Housing
ICBC                         → Transport
MSP PREMIUM (if still active)→ Health

AMAZON.CA / AMZN             → Shopping
COSTCO                       → Groceries
SPORT CHEK                   → Shopping
WINNERS / HOMESENSE          → Shopping

ATM WITHDRAWAL               → Misc (cash — categorize manually)
```

---

## Income Detection (Chequing Account Imports)

When importing RBC chequing account (not credit card):

```python
INCOME_PATTERNS = [
    'PAYROLL', 'DIRECT DEPOSIT', 'SALARY',
    'EMPLOYER', 'DD ', 'PAYROLL DEP',
    'GOVT OF CANADA', 'CRA', 'CCB',  # government payments
    'GST/HST CREDIT', 'CERB', 'EI '
]

def is_income(description, amount):
    desc_upper = description.upper()
    is_positive = amount > 0  # RBC: positive = money in
    is_income_keyword = any(p in desc_upper for p in INCOME_PATTERNS)
    return is_positive and is_income_keyword
```

If income detected → log to Income tab, not Expenses tab.

---

## Reply Format After RBC Import

```
🏦 RBC Statement — [Month] [Year]
Account: [Credit Card / Chequing]

Processed:    X transactions
✅ Logged:    X expenses categorized
💰 Income:    X deposits logged to Income tab
⏭ Skipped:   X payments/transfers
🔁 Duplicate: X already in Sheets
❓ Review:    X flagged (ATM withdrawals or unrecognized)

Top spending:
[same format as Amex reply]
```

---

## Pitfalls

- **Negative = expense for RBC**: opposite of Amex — always flip sign
- **Two description columns**: always combine Description 1 + Description 2 for merchant matching
- **USD$ column**: if USD$ is populated, convert to CAD using memory rate (or prompt Sophie for rate)
- **Date format**: RBC uses M/D/YYYY not YYYY-MM-DD — parse carefully
- **Chequing vs Credit Card**: chequing has ATM withdrawals and deposits; credit card does not
- **ICBC**: auto insurance — log as Transport, not Insurance
