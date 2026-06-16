# Part 1: Business Data Cleaning, Validation & Excel Reporting

## Problem Summary

A retail company exports order-level sales data from multiple internal systems. The raw dataset (`raw_orders.xlsx`) contains 932 rows and 21 columns but suffers from inconsistent text formatting, mixed date formats, duplicate records, missing values, invalid discounts, and order status inconsistencies.

The objective is to produce a clean, validated, analysis-ready dataset alongside summary reports for business review.

---

## Dataset Description

| Attribute | Value |
|---|---|
| File | `raw_orders.xlsx` |
| Raw rows | 932 |
| Columns | 21 |
| Cleaned rows | 912 (after removing 20 exact duplicates) |
| Key fields | order_id, order/ship dates, customer info, product, sales, cost, profit, discount, payment & order status |

---

## Tools Used

- **Python 3.12** — data cleaning, transformation, and report generation
  - `pandas` — data manipulation
  - `openpyxl` — Excel file creation with formatting
  - `matplotlib` — screenshot generation
- **Excel** (conceptually applied; all Excel-equivalent operations performed via Python)

---

## Cleaning Steps Performed

### 1. Text Field Standardisation
Applied to: `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`

- Removed leading/trailing whitespace (equivalent to Excel `TRIM`)
- Collapsed multiple internal spaces
- Applied consistent Title Case (equivalent to `PROPER`)
- Resolved 16 region variants → 5 clean values; 13 category variants → 3 clean values

### 2. Date Cleaning & Validation
- Detected 7+ different raw date formats across `order_date` and `ship_date`
- Implemented multi-format parser (DD Mon YYYY, MM/DD/YYYY, DD-MM-YYYY, YYYY-MM-DD, etc.)
- Standardised all valid dates to `YYYY-MM-DD`
- Calculated `shipping_delay_days` = ship_date − order_date
- Flagged 22 records where ship_date < order_date

### 3. Duplicate Handling
- Identified and removed 20 **exact duplicate rows**
- Identified 12 **order_id values with conflicting data** — retained both copies, flagged as `conflicting_duplicate`

### 4. Missing Value Treatment
- `region` (26 missing): filled with `"Unknown"`
- `ship_mode` (22 missing): filled with `"Unknown"`
- `discount` (18 missing): treated as `0.0` where other sales fields are valid

### 5. Discount Validation
- Parsed text percentage values (e.g. "70%", "85%") to decimal
- Flagged 16 records with **negative discounts**
- Flagged 10 records with **discount > 60%**

### 6. Calculated Columns
Added to `cleaned_orders.xlsx`:

| Column | Formula |
|---|---|
| `cleaned_discount` | Standardised discount (0.0 if missing) |
| `calculated_sales` | quantity × unit_price × (1 − cleaned_discount) |
| `calculated_profit` | calculated_sales − cost |
| `profit_margin` | calculated_profit / calculated_sales |
| `shipping_delay_days` | ship_date − order_date (days) |
| `order_month` | Month from order_date |
| `order_year` | Year from order_date |
| `data_quality_flag` | Pipe-separated quality flags per record |

---

## Business Rules Applied

| Rule | Action |
|---|---|
| Missing region | Fill `Unknown`; flag in quality report |
| Missing ship_mode | Fill `Unknown`; flag in quality report |
| Missing discount | Treat as 0 where other fields valid |
| Negative discount | Flag as `negative_discount` |
| Discount > 60% | Flag as `discount_above_60pct` |
| Cancelled orders | Excluded from completed sales summaries |
| Failed payments | Excluded from completed sales summaries |
| Refunded orders | Separately summarised |
| Ship date before order | Flag as `ship_before_order` |

---

## Summary of Data Quality Issues Found

| Issue | Count |
|---|---|
| Exact duplicate rows removed | 20 |
| Conflicting duplicate order IDs (flagged) | 12 pairs |
| Missing region (filled Unknown) | 26 |
| Missing ship_mode (filled Unknown) | 22 |
| Missing discount (filled 0) | 18 |
| Negative discounts | 16 |
| Discount > 60% (incl. text "70%","85%") | 10 |
| Ship date before order date | 22 |
| Cancelled orders | 146 |
| Failed payments | 69 |
| Refunded | 72 |

**Clean records (no flags): ~480 of 912 post-cleaning records**

---

## Summary of Pivot Reports

### `pivot_summary.xlsx` contains 6 sheets:

| Sheet | Description |
|---|---|
| Sales by Region | Total orders, sales, profit, and margin by region — sorted by sales descending |
| Sales by Category | Sales and profit by category and sub-category — sorted within each category |
| Orders by Ship Mode | Order count and average shipping delay per ship mode |
| Margin by Segment | Profit margin comparison across customer segments — sorted by margin descending |
| Bad Orders by Region | Count of cancelled / returned / failed / refunded orders per region |
| Monthly Sales Trend | Month-by-month sales and profit for completed, paid orders |

---

## Key Business Insights

1. **South region leads in sales** with ~₹1.55M in completed, paid sales — the highest of all regions.
2. **Technology is the most profitable category** — delivering the highest absolute profit among the three categories.
3. **Overall profit margin of 28.3%** across completed, paid orders — a healthy retail margin.
4. **22.3% of raw records have quality issues** — including cancellations, failed payments, and data errors that must be excluded from revenue reporting.
5. **602 of 912 cleaned records (66%)** qualify as completed, paid orders suitable for revenue analysis.
6. **146 cancelled + 69 failed payment orders** represent significant lost revenue that should be tracked and escalated.
7. **Discount discipline is inconsistent**: 16 records show negative discounts (possible surcharges) and 10 show discounts above 60%, suggesting data entry controls are needed.

---

## Assumptions and Limitations

### Assumptions
- Maximum valid discount is 60%; values above this are flagged but retained
- Missing discount implies no discount applied (treated as 0)
- Both copies of conflicting duplicate order_ids are retained for human review
- For India-based data, ambiguous dates (e.g. 01/07) are interpreted as DD/MM
- "Completed sales" = order_status Completed AND payment_status Paid

### Limitations
- Date format ambiguity cannot always be resolved programmatically
- Conflicting duplicates are unresolved without business input
- No master reference file available to validate product names, prices, or customer IDs
- Negative discounts may be intentional surcharges — cannot confirm without business context

---

## Screenshots Included

| File | Shows |
|---|---|
| `screenshots/raw_data_preview.png` | First 8 rows of raw dataset with original formatting issues visible |
| `screenshots/cleaned_data_preview.png` | First 9 rows of cleaned dataset with calculated columns and colour-coded flags |
| `screenshots/pivot_summary_1.png` | Sales & Profit by Region bar charts |
| `screenshots/pivot_summary_2.png` | Monthly Sales Trend line chart + Category Sales pie chart |

---

## Repository Structure

```
part1_data_cleaning/
├── data/
│   ├── raw_orders.xlsx          ← original dataset (unchanged)
│   └── cleaned_orders.xlsx      ← cleaned dataset with 26 columns
├── outputs/
│   ├── data_quality_report.xlsx ← 8-sheet quality report
│   ├── pivot_summary.xlsx       ← 6-sheet pivot analysis
│   └── cleaning_log.md          ← detailed cleaning documentation
├── screenshots/
│   ├── raw_data_preview.png
│   ├── cleaned_data_preview.png
│   ├── pivot_summary_1.png
│   └── pivot_summary_2.png
└── README.md
```
