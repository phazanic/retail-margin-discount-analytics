# 📊 Retail Margin & Discount Leakage Analytics (Power BI)

A **3-page Power BI dashboard** built to answer one core business question: *why is our profit margin falling short despite record-breaking sales?*

The short answer: unchecked discounting. This report traces exactly where the money is going, which product lines are bleeding, and which regional managers need a conversation.

---

## 🎯 The Business Problem

Top-line revenue grew nicely — **$733.22K**, up **+20.36% year-over-year**. But net profit margin landed at **12.74%**, well below the company's 15% target.

The gap wasn't random. Digging into the data revealed a consistent pattern: orders with discounts above 20% were systematically destroying margin, in some cases flipping a healthy 29.86% margin to a catastrophic **-44.31%**. The losses weren't spread evenly either — they were concentrated in specific product sub-categories and traceable back to specific regional decision-makers.

---

## 📈 Key Numbers at a Glance

| Metric | Value | Context |
|---|---|---|
| **Total Revenue** | **$733.22K** | YoY +20.36% — strong demand across all customer segments |
| **Net Profit** | **$93.44K** | YoY +14.24% — profit growing slower than revenue (warning sign) |
| **Profit Margin %** | **12.74%** | 2.26 percentage points below the 15% company target |
| **Loss-Making Orders** | **450 orders** | Generated **-$53.84K** in combined losses |
| **Margin at >20% Discount** | **-44.31%** | The cliff edge — every heavily discounted order destroys value |
| **Highest Return Rate** | **13.20%** | West region — highest sales volume but also the most product returns |

---

## 🖥️ What Each Dashboard Page Shows

### Page 1 — Executive Overview
The C-suite view. Tracks monthly sales vs. the prior year, breaks down profit margin by product category, and flags anything that needs immediate attention via an automated alert banner.

At a glance, Technology (18.65% margin) and Office Supplies (16.15%) are beating targets. Furniture (1.40%) is not — it triggers a critical alert and redirects attention to Page 2.

Customer mix: Consumer 45.27%, Corporate 32.98%, Home Office 21.75%.

---

### Page 2 — Profitability & Discount Leakage Deep-Dive
This is where the root cause becomes clear.

**The discount cliff, broken down by band:**
- **0% discount:** +29.86% margin
- **1–10% (safe zone):** +17.41% margin
- **11–20% (caution):** +13.00% margin
- **>20% (danger zone):** **-44.31% margin** — this is where the business is hemorrhaging cash

**The worst offenders at the sub-category level:**
- **Tables** — $60.9K in revenue, but lost **-$8,140.69** (avg. discount: 26.5%)
- **Bookcases** — a smaller but consistent drag of -$583.63 (avg. discount: 21.5%)

A scatter plot maps every sub-category's average discount against its profit margin, making the 20% threshold visually undeniable.

---

### Page 3 — Regional Governance & Accountability
Who's responsible?

This page benchmarks regional profit margins against the 15% target and links discount behavior directly to individual regional managers:

- **Kelly Williams (Central):** Granted discounts above 20% on **26.99%** of all transactions — the highest in the company. Her region's margin? **5.13%**. The connection is direct.
- **Anna Andreadi (West):** Strong sales ($250.1K) and the best margin in the company (17.51%) — but a **13.20% return rate** that warrants a separate investigation.

The goal of this page isn't to assign blame, it's to surface facts that a manager can act on.

---

## 🛠️ How It's Built

### Data Model (Star Schema)

```
         [ Dim_Date ]  ← DAX-generated calendar table
               │
               │ (1-to-Many on Order Date → Date)
               ↓
         [ Fact_Orders ] ──── (1-to-Many) ──── [ Dim_Product ]
         51K+ line-item rows                   Category, Sub-Category,
         Sales, Profit,                        Product ID & Name
         Discount, Quantity
               ↑
               │ (Virtual relationship via TREATAS)
               ↓
         [ Dim_Returns ] ──── (1-to-Many) ──── [ Dim_People ]
         Returned Order IDs                    Region & Regional Manager
```

A few deliberate choices worth noting:

- **`Dim_Date` was built in DAX**, not imported from a flat file. It uses `CALENDAR(MIN(...), MAX(...))` and generates its own sorting columns (`Year Month`, `Month No`) so the timeline always stays clean regardless of the data refresh.
- **`TREATAS` instead of bi-directional filtering.** The relationship between `Dim_Returns` and `Fact_Orders` is handled with a virtual filter in DAX rather than enabling bi-directional cross-filtering in the model. This keeps performance predictable and avoids ambiguous filter context — a common trap with many-to-many return relationships.

---

## 💻 Core DAX Logic

### Time Intelligence — YoY Growth

```dax
-- Prior year baseline for comparison
Sales LY =
CALCULATE(
    [Total Sales],
    SAMEPERIODLASTYEAR(Dim_Date[Date])
)

-- Growth percentage
Sales YoY % =
VAR CurrentSales = [Total Sales]
VAR PrevSales    = [Sales LY]
RETURN
DIVIDE(CurrentSales - PrevSales, PrevSales, 0)
```

### Loss Detection & Discount Banding

```dax
-- Isolate only the orders that lost money
Total Loss Amount =
CALCULATE(
    [Total Profit],
    Fact_Orders[Profit] < 0
)

-- Classify each order into a discount risk tier
Discount Band =
SWITCH(
    TRUE(),
    Fact_Orders[Discount] = 0,     "1. No Discount (0%)",
    Fact_Orders[Discount] <= 0.10, "2. Safe (1–10%)",
    Fact_Orders[Discount] <= 0.20, "3. Warning (11–20%)",
                                   "4. Danger (>20%)"
)

-- What share of total orders cross the 20% danger line?
Discount > 20% Share =
VAR HeavyDiscountRows = CALCULATE(COUNTROWS(Fact_Orders), Fact_Orders[Discount] > 0.20)
VAR TotalRows         = COUNTROWS(Fact_Orders)
RETURN
DIVIDE(HeavyDiscountRows, TotalRows, 0)
```

### Return Rate (using TREATAS)

```dax
-- Count distinct returned orders without bi-directional filtering
Returned Orders =
CALCULATE(
    DISTINCTCOUNT(Fact_Orders[Order ID]),
    TREATAS(VALUES(Dim_Returns[Order ID]), Fact_Orders[Order ID])
)

Return Rate % =
DIVIDE([Returned Orders], [Total Orders], 0)
```

### Conditional Formatting (Field Value Binding)

```dax
-- Green if growing, red if declining
Color YoY Sales =
IF([Sales YoY %] >= 0, "#107C41", "#D9534F")

-- Blue if above target margin, red if below
Color Margin Target =
IF([Profit Margin %] >= 0.15, "#2B579A", "#D9534F")
```

---

## 📂 Files

```
├── Retail_Margin_and_Discount_Analytics.pbix    ← Main Power BI report (open in Power BI Desktop)
├── assets/
│   ├── retail_bi_dashboard_Executive_Overview.png
│   ├── retail_bi_dashboard_Profitability_Deep-Dive.png
│   └── retail_bi_dashboard_Regional_Governance.png
└── README.md
```

To explore the report: download the `.pbix` file and open it in **Power BI Desktop** (free). No account or license needed to view.

---

## 👤 Author

**Worapol Boontawee**  
Data Analyst & BI Developer  
[LinkedIn](https://www.linkedin.com/in/boontawee-w/) · [Portfolio](https://phazanic.github.io)
