---
name: finance-analytics
description: Analyze spending habits, trends, anomalies, and patterns from Google Sheets financial data. Generates insights, comparisons, and weekly summaries for Sophie and Hai. Read-only — never writes to Sheets.
version: 1.0.0
metadata:
  hermes:
    tags: [finance, analytics, spending, habits, insights, trends]
    category: personal-finance
---

# Finance Analytics — Spending Habits & Insights

## When to Use
Load this skill when Sophie asks:
- "What are my spending habits?"
- "Where am I overspending?" / "What's my money leak?"
- "How does May compare to April?"
- "Which merchant do I spend most at?"
- "What day of the week do I spend most?"
- "Show me my dining trend"
- "Am I on track this month?"
- "Give me a spending summary"
- "What's one thing I could cut to save $X?"
- Weekly Sunday cron summary

## Prerequisites
- google-workspace skill must be authorized
- finance-sheets skill data must exist (at least 2 months for trends)
- Spreadsheet ID in config: `finance.sheets.spreadsheet_id`

## Read-Only Rule
This skill NEVER writes to Google Sheets.
All operations are GET only — no append, update, or delete.

---

## Data Sources

```bash
GAPI="python ${HERMES_HOME:-/opt/data}/skills/productivity/google-workspace/scripts/google_api.py"
SHEET_ID="YOUR_SPREADSHEET_ID"

# Load all tabs needed for analysis
expenses = $GAPI sheets get $SHEET_ID "Expenses!A:M"
budget   = $GAPI sheets get $SHEET_ID "Budget!A:B"
income   = $GAPI sheets get $SHEET_ID "Income!A:G"
upcoming = $GAPI sheets get $SHEET_ID "Upcoming!A:J"
```

---

## Analysis 1 — Category Breakdown

### When triggered
"Where does my money go" / "spending breakdown" / "budget status"

### Logic
```python
def category_breakdown(expenses, budget, month=None):
    month = month or current_month()  # YYYY-MM format

    # Filter to target month
    month_rows = [r for r in expenses if r['Month'] == month]

    # Sum by category
    by_cat = defaultdict(float)
    for r in month_rows:
        by_cat[r['Category']] += float(r['CAD Amount'])

    total = sum(by_cat.values())

    # Compare to budget
    budget_map = {r['Category']: float(r['Monthly Budget CAD'])
                  for r in budget}

    results = []
    for cat, spent in sorted(by_cat.items(), key=lambda x: -x[1]):
        target = budget_map.get(cat, 0)
        pct_of_budget = (spent / target * 100) if target else None
        pct_of_total = spent / total * 100
        results.append({
            'category': cat,
            'spent': spent,
            'budget': target,
            'pct_budget': pct_of_budget,
            'pct_total': pct_of_total
        })
    return results
```

### Reply format
```
📊 Where your money went — [Month] [Year]

🍽 Dining          $620  18% of spend  ↑ 29% vs last month  ⚠️ 155% of budget
🛒 Groceries       $540  16%           → stable              90% of budget
🏠 Housing         $3,000 88%          → fixed               100% of budget
🚗 Transport       $290   9%           ↓ improving           97% of budget
👶 Kids            $480  14%           → stable              96% of budget
🛍 Shopping        $420  12%           ↑↑ high               140% of budget ⚠️

Total spent:  $X / $X budget
Savings rate: X%
```

---

## Analysis 2 — Merchant Loyalty

### When triggered
"Which merchant do I spend most at" / "where do I shop most" / "top merchants"

### Logic
```python
def merchant_analysis(expenses, month=None):
    rows = filter_month(expenses, month) if month else expenses

    merchant_spend = defaultdict(float)
    merchant_visits = defaultdict(int)

    for r in rows:
        merchant = r['Description']
        amount = float(r['CAD Amount'])
        merchant_spend[merchant] += amount
        merchant_visits[merchant] += 1

    # Top 10 by spend
    top_by_spend = sorted(merchant_spend.items(),
                          key=lambda x: -x[1])[:10]

    # Top 10 by frequency
    top_by_freq = sorted(merchant_visits.items(),
                         key=lambda x: -x[1])[:10]

    # Calculate avg per visit
    avg_per_visit = {m: merchant_spend[m] / merchant_visits[m]
                     for m in merchant_spend}

    return top_by_spend, top_by_freq, avg_per_visit
```

### Reply format
```
🏪 Your top merchants — [Month]

By spend:
1. Whole Foods      $340  (8 visits · $42 avg)
2. Earls Kitchen    $280  (4 visits · $70 avg)
3. Amazon.ca        $220  (6 orders · $37 avg)
4. Starbucks        $96   (12 visits · $8 avg)
5. Costco           $180  (2 visits · $90 avg)

By frequency:
1. Starbucks        12 visits
2. Tim Hortons      10 visits ($4 avg)
3. Whole Foods       8 visits

💡 Coffee (Starbucks + Tim Hortons): $136/month = $1,632/year
```

---

## Analysis 3 — Time Pattern Analysis

### When triggered
"When do I spend most" / "what day" / "weekend spending"

### Logic
```python
def time_patterns(expenses, month=None):
    rows = filter_month(expenses, month) if month else expenses

    by_weekday = defaultdict(float)   # 0=Mon, 6=Sun
    by_week_of_month = defaultdict(float)  # 1-4

    for r in rows:
        date = parse_date(r['Date'])
        by_weekday[date.weekday()] += float(r['CAD Amount'])
        week_num = (date.day - 1) // 7 + 1
        by_week_of_month[week_num] += float(r['CAD Amount'])

    weekend_total = by_weekday[5] + by_weekday[6]  # Sat + Sun
    weekday_total = sum(v for k, v in by_weekday.items() if k < 5)
    weekday_avg = weekday_total / 5

    return by_weekday, by_week_of_month, weekend_total, weekday_avg
```

### Reply format
```
📅 When you spend — [Month]

By day:
Mon $180  ███
Tue $120  ██
Wed $95   ██
Thu $210  ████
Fri $380  ███████
Sat $420  ████████  ← peak
Sun $150  ███

Weekend = 2.4x weekday average
Fri + Sat = 52% of total weekly spend

By week of month:
Week 1 (1-7):    $840  — typically higher (fresh budget feel)
Week 2 (8-14):   $620
Week 3 (15-21):  $590
Week 4 (22-31):  $510  — typically lower
```

---

## Analysis 4 — 3-Month Trend Analysis

### When triggered
"Spending trend" / "am I improving" / "3 month comparison" / any multi-month query

### Logic
```python
def trend_analysis(expenses, months=3):
    # Get last N months
    recent_months = get_last_n_months(months)

    trends = {}
    for cat in ALL_CATEGORIES:
        monthly_totals = []
        for month in recent_months:
            total = sum(float(r['CAD Amount'])
                       for r in expenses
                       if r['Month'] == month
                       and r['Category'] == cat)
            monthly_totals.append(total)

        # Calculate trend direction
        if len(monthly_totals) >= 2:
            change = monthly_totals[-1] - monthly_totals[-2]
            pct_change = (change / monthly_totals[-2] * 100
                         if monthly_totals[-2] > 0 else 0)
            if pct_change > 15:
                direction = '↑↑'
            elif pct_change > 5:
                direction = '↑'
            elif pct_change < -15:
                direction = '↓↓'
            elif pct_change < -5:
                direction = '↓'
            else:
                direction = '→'

        trends[cat] = {
            'monthly': monthly_totals,
            'direction': direction,
            'pct_change': pct_change
        }

    return trends, recent_months
```

### Reply format
```
📈 3-month spending trend — Mar → Apr → May

Category       Mar    Apr    May    Trend
Dining         $510   $480   $620   ↑↑ accelerating ⚠️
Groceries      $580   $520   $540   → stable ✅
Shopping       $280   $310   $420   ↑↑ watch this ⚠️
Transport      $410   $380   $290   ↓↓ improving ✅
Kids           $460   $480   $480   → stable ✅
Health         $80    $120   $90    → stable ✅

Overall savings rate:
Mar 38% → Apr 41% → May 35%  ↓ dipped this month
```

---

## Analysis 5 — Anomaly Detection

### When triggered
"Anything unusual" / "did I overspend anywhere" / always runs as part of weekly summary

### Logic
```python
def detect_anomalies(expenses, lookback_months=3):
    anomalies = []

    # Get baseline averages from last N months (excluding current)
    baseline_months = get_last_n_months(lookback_months, exclude_current=True)

    for merchant in get_all_merchants(expenses):
        # Average spend per month at this merchant (baseline)
        baseline_amounts = [
            sum(float(r['CAD Amount']) for r in expenses
                if r['Month'] == m and r['Description'] == merchant)
            for m in baseline_months
        ]
        baseline_avg = mean(baseline_amounts) if baseline_amounts else 0

        # This month's spend at merchant
        this_month = sum(float(r['CAD Amount']) for r in expenses
                        if r['Month'] == current_month()
                        and r['Description'] == merchant)

        # Flag if >2x baseline and >$50 absolute
        if baseline_avg > 0 and this_month > baseline_avg * 2 and this_month > 50:
            anomalies.append({
                'merchant': merchant,
                'this_month': this_month,
                'baseline': baseline_avg,
                'multiple': this_month / baseline_avg
            })

    # Also flag any new merchant >$100 with no prior history
    new_merchants = [
        m for m in get_all_merchants(filter_month(expenses, current_month()))
        if m not in get_all_merchants(filter_older_months(expenses))
        and get_merchant_total(expenses, m, current_month()) > 100
    ]

    return anomalies, new_merchants
```

### Reply format
```
🔍 Spending anomalies — [Month]

⚠️  Earls Kitchen  $280 this month (normally $120 · 2.3x)
⚠️  Amazon         $220 this month (normally $80 · 2.8x)
🆕 New: Structube  $450 — first time seeing this merchant

✅ Everything else within normal range
```

---

## Analysis 6 — The One Insight

### Always append to any habit analysis

Find the single most actionable insight from the data.
Priority: biggest dollar opportunity > biggest trend change > closest to goal.

```python
def generate_insight(category_data, trend_data, budget_data):
    # Find category most over budget with upward trend
    worst = max(
        [(c, d) for c, d in category_data.items()
         if d['pct_budget'] and d['pct_budget'] > 110],
        key=lambda x: x[1]['pct_budget'],
        default=None
    )

    if worst:
        cat, data = worst
        overage = data['spent'] - data['budget']
        annual = overage * 12
        return f"{cat} is ${overage:.0f} over budget this month. " \
               f"At this pace that's ${annual:.0f}/year over target."
```

### Format
```
💡 One thing to know:

Dining is $220 over budget this month — 4 visits to Earls.
Cutting 2 restaurant meals/month saves ~$140 = $1,680/year.
That covers your Vietnam trip budget entirely.
```

---

## Weekly Sunday Summary (Cron Job)

### Schedule
```
0 8 * * 0   (8am every Sunday)
```

### Logic
Run last 7 days analysis + month-to-date status. Keep under 12 lines.

### Reply format
```
📊 Week of May 4–10 · Day 10 of 31

Spent this week:  $X
Month to date:    $X of $X budget (X% · pace: on track ✅)

Top 3 this week:
🍽 Earls Kitchen   $140  (2 dinners)
🛒 Whole Foods     $95
☕ Starbucks        $40  (5 visits)

💡 [One sentence — data observation, no judgment]

Next payday: X days | Upcoming: [nearest bill] in X days
```

---

## Sophie vs Hai Breakdown

### When triggered
"How does Hai spend vs me" / "who spends more" / "compare our spending"

```python
def by_person(expenses, month=None):
    rows = filter_month(expenses, month) if month else expenses
    sophie = [r for r in rows if r['Who'] == 'Sophie']
    hai = [r for r in rows if r['Who'] == 'Hai']
    joint = [r for r in rows if r['Who'] == 'Joint']
```

### Reply format
```
👤 Spending by person — [Month]

Sophie:  $X (X%)
  Top: Groceries $X · Kids $X · Health $X

Hai:     $X (X%)
  Top: Dining $X · Transport $X · Shopping $X

Joint:   $X (X%)
  Top: Housing $X · Utilities $X

Note: X transactions have no Who tag — reply "fix tags"
to review and assign them
```

---

## Tone Rules

Per SOUL.md — all analysis must be:
- **Data first**: show numbers before any commentary
- **Non-judgmental**: state facts, never shame
- **Actionable**: always end with one option, not a lecture
- **Concise**: Sophie is busy — no padding, no preamble

```
❌ NOT: "You've been spending way too much on dining again"
✅ YES: "Dining: $620 this month vs $400 budget (+$220)"

❌ NOT: "Your savings rate is dangerously low"
✅ YES: "Savings rate: 35% — above the 20% target ✅"

❌ NOT: "You really need to cut back on Amazon"
✅ YES: "Amazon: $220 this month (normally $80) — what changed?"
```

---

## Pitfalls

- **Not enough data**: if <2 months of data, run analysis but note results are preliminary
- **Missing Who tags**: many Amex imports default to Sophie — remind Sophie to tag Hai's transactions
- **Category Misc overflow**: if Misc >10% of total, flag and ask Sophie to recategorize
- **Holiday months**: December and travel months will spike — note context before flagging anomalies
- **VND transactions**: convert to CAD for all analysis using stored rate
- **Upcoming expenses**: always factor Upcoming tab into month projections — raw Expenses tab alone understates July/big-bill months
