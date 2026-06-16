# Data Cleaning Log — Retail Orders Dataset

**Date:** 2025  
**Analyst:** Business Analyst  
**Dataset:** raw_orders.xlsx (932 rows × 21 columns)  
**Tool:** Python (pandas, openpyxl)

---

## 1. Issues Found

### 1.1 Text Field Issues
| Field | Issues Detected |
|---|---|
| `customer_name` | Mixed casing (e.g. "PRIYA MENON", "Vikram  Iyer" with double space) |
| `segment` | 16 unique raw values due to case/space variations (e.g. "SMALL BUSINESS", "Small  Business", "  Consumer ") |
| `region` | 16 raw variants (e.g. "  North ", "NORTH", "west", "EAST"); 26 missing values |
| `category` | 13 raw variants (e.g. "OFFICE SUPPLIES", "  Furniture ", "Technology ") |
| `ship_mode` | 12 raw variants (e.g. "STANDARD CLASS", "Standard  Class"); 22 missing values |
| `payment_status` | 8 raw variants ("Paid ", "PENDING", "  Pending ", "failed") |
| `order_status` | 10 raw variants ("  Completed ", "COMPLETED", "completed", "  Cancelled ") |

### 1.2 Date Issues
- Mixed date formats across `order_date` and `ship_date`: at least 7 different formats detected:
  - `DD Mon YYYY` (e.g. "21 Jul 2024")
  - `MM/DD/YYYY` (e.g. "07/27/2024")
  - `DD-MM-YYYY` (e.g. "28-11-2024")
  - `YYYY-MM-DD` (e.g. "2024-05-24")
  - `DD-Mon-YYYY` and others
- **22 records** where ship_date is before order_date (invalid shipping records)
- 0 completely unparseable dates after multi-format parsing

### 1.3 Duplicate Records
- **20 exact duplicate rows** (identical on all 21 columns)
- **12 order_id values** with conflicting data across rows (same order_id, different field values)

### 1.4 Discount Issues
- **18 records** with missing discount
- **16 records** with negative discount values (e.g. -0.19, -0.23, -0.15)
- **10 records** with discount above 60% — including 2 records with text format values ("70%", "85%")

### 1.5 Missing Values
| Column | Missing Count |
|---|---|
| `region` | 26 |
| `ship_mode` | 22 |
| `discount` | 18 |

### 1.6 Order/Payment Status Issues
- Cancelled orders: 146
- Failed payments: 69
- Refunded: 72
- Returned: various (overlapping with above)

---

## 2. Cleaning Actions Performed

### 2.1 Text Standardisation
- Applied `.strip()` to remove all leading/trailing whitespace
- Collapsed multiple internal spaces using `' '.join(str.split())`
- Applied `.title()` case to: `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`
- Result: all text fields now have consistent Title Case with no extra spaces

### 2.2 Date Standardisation
- Implemented a multi-format parser that tries 7 known date formats in sequence
- Fell back to pandas `pd.to_datetime()` for edge cases
- All valid dates standardised to `YYYY-MM-DD` format in `cleaned_orders.xlsx`
- Added `shipping_delay_days` = `ship_date` − `order_date` in calendar days

### 2.3 Duplicate Handling
- Removed all 20 **exact duplicate rows** (drop_duplicates())
- Identified 12 order_ids with conflicting data — **both copies retained**, flagged with `conflicting_duplicate` in `data_quality_flag`
- Net records after deduplication: **912**

### 2.4 Discount Cleaning
- Parsed text percentage values ("70%", "85%") to decimal (0.70, 0.85)
- Filled missing discounts with `0.0` (where other sales fields were populated and valid)
- Retained original raw discount in audit trail via `discount_raw` column (internal)
- Used `cleaned_discount` for all calculated columns

### 2.5 Calculated Columns Added
| Column | Formula |
|---|---|
| `cleaned_discount` | Parsed, standardised discount (0.0 if missing) |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit / calculated_sales` |
| `shipping_delay_days` | `ship_date − order_date` (days) |
| `order_month` | Month extracted from `order_date` |
| `order_year` | Year extracted from `order_date` |
| `data_quality_flag` | See below |

---

## 3. Business Rules Applied

| Rule | Action Taken |
|---|---|
| Missing `region` | Filled with `"Unknown"`; flagged in quality report |
| Missing `ship_mode` | Filled with `"Unknown"`; flagged in quality report |
| Missing `discount` | Treated as `0.0` where quantity, unit_price and cost are all valid |
| Negative discount | Flagged as `negative_discount`; value retained in `cleaned_discount` for transparency |
| Discount > 60% | Flagged as `discount_above_60pct` |
| Cancelled orders | Excluded from completed sales pivot summaries |
| Failed payments | Excluded from completed sales pivot summaries |
| Refunded orders | Separately summarised in `pivot_summary.xlsx → Bad Orders by Region` |
| Ship date before order date | Flagged as `ship_before_order` |

---

## 4. `data_quality_flag` Values

Records may carry multiple pipe-separated flags:

| Flag Value | Meaning |
|---|---|
| `clean` | No issues detected |
| `negative_discount` | Discount < 0 |
| `discount_above_60pct` | Discount > 0.60 |
| `ship_before_order` | ship_date < order_date |
| `invalid_order_date` | Date could not be parsed |
| `invalid_ship_date` | Date could not be parsed |
| `cancelled_order` | order_status = Cancelled |
| `failed_payment` | payment_status = Failed |
| `refunded` | payment_status = Refunded |
| `conflicting_duplicate` | Same order_id, different field values — retained for review |

Multiple flags are joined with `|` (e.g. `cancelled_order|failed_payment`).

---

## 5. Records Removed

| Action | Count |
|---|---|
| Exact duplicate rows removed | 20 |
| Conflicting duplicates removed | 0 (flagged & retained) |
| **Total removed** | **20** |

---

## 6. Records Flagged

| Flag Type | Count |
|---|---|
| Conflicting duplicate | 24 (12 pairs) |
| Cancelled order | 146 |
| Failed payment | 69 |
| Refunded | 72 |
| Negative discount | 16 |
| Discount > 60% | 10 |
| Ship before order | 22 |
| Missing region (now Unknown) | 26 |
| Missing ship_mode (now Unknown) | 22 |

> Note: Records can carry multiple flags simultaneously, so totals above overlap.

---

## 7. Assumptions Made

1. **Discount range**: Maximum valid discount is 60% (0.60). Values above this are flagged but not removed — they may represent legitimate bulk/clearance discounts requiring business sign-off.
2. **Missing discount = 0**: Where quantity, unit_price, and cost are all present and non-zero, a missing discount is assumed to mean no discount was applied.
3. **Conflicting duplicates**: Both records are retained for human review. The first occurrence is not assumed to be authoritative.
4. **Negative discounts**: These likely represent surcharges or data entry errors. Flagged but not removed, as the underlying intent may be recoverable.
5. **Date parsing**: Where two formats could produce different results, `DD-MM-YYYY` is preferred over `MM-DD-YYYY` for an India-based retail dataset.
6. **"Unknown" region/ship_mode**: Where region or ship_mode is blank, this is filled as "Unknown" rather than dropped, to preserve order-level data for all other analyses.
7. **Completed sales definition**: Only `order_status = "Completed"` AND `payment_status = "Paid"` records are included in the main pivot sales summaries.

---

## 8. Limitations

- **Date ambiguity**: For dates like "01/07/2024", it is impossible to determine definitively whether this means 1 July or 7 January without business confirmation. DD-MM-YYYY was assumed.
- **No external reference data**: Cannot verify whether product names, customer IDs, or unit prices are correct without a master reference file.
- **Conflicting duplicates unresolved**: The 12 conflicting order_id pairs remain unresolved — a business stakeholder must determine which record is authoritative.
- **Negative discounts**: Cannot determine whether these are intentional surcharges or data entry errors without business context.
- **Screenshot tool**: Screenshots are generated programmatically (matplotlib) rather than from a live Excel interface, but faithfully represent the data content.
