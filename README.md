[README.md](https://github.com/user-attachments/files/26185188/README.md)
# 🎄 Product Analysis for Christmas Campaign
> *Identifying top-performing products to inform advertising strategy during peak sale periods*

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Problem Statement](#problem-statement)
3. [Methodology](#methodology)
4. [Data Exploration & Preparation](#data-exploration--preparation)
5. [Analysis & Insights](#analysis--insights)
6. [Conclusion](#conclusion)
7. [Recommendations](#recommendations)

---

## Project Overview

This project analyzes e-commerce event data captured during the **November–December 2019** period — the Christmas campaign window — to identify which products performed best across multiple business-relevant dimensions.

| Property | Details |
|---|---|
| **Period Covered** | November 1, 2019 → December 31, 2019 |
| **Raw Dataset** | 7,033,125 rows × 9 columns |
| **Final Dataset** | 7,033,125 rows × 8 columns (after cleaning) |
| **Event Types** | `cart` (75%) · `purchase` (25%) |
| **Unique Products** | ~84,000+ product IDs |
| **Source Files** | `november.csv` (2,885,158 rows) · `december.csv` (4,147,967 rows) |

### Dataset Columns

| Column | Type | Description |
|---|---|---|
| `event_time` | datetime (UTC) | Timestamp of the user interaction |
| `event_type` | string | Type of event: `cart` or `purchase` |
| `product_id` | int | Unique product identifier |
| `brand` | string | Product brand (5% missing — expected) |
| `subcategory` | string | Granular product category (e.g., `electronics.smartphone`) |
| `category` | string | Top-level category (e.g., `electronics`) |
| `product_identifier` | string | Composite key: `product_id-brand subcategory` |
| `price` | float | Unit price in USD |

---

## Problem Statement

Senior stakeholders want to determine **which products performed best during the Christmas period** in order to streamline the products they advertise in future sale campaigns.

To answer this question rigorously, the business identified five dimensions of product performance:

| # | Metric | Business Question |
|---|---|---|
| 1 | **Volume** | Which products were purchased the most? |
| 2 | **Revenue** | Which products generated the most gross income? |
| 3 | **Popularity** | Which products attracted the most unique customers? |
| 4 | **Conversion Rate** | Which products most efficiently turned cart additions into purchases? |
| 5 | **November → December Growth** | Which products showed improved performance as Christmas approached? |

---

## Methodology

Each metric was computed at the **product level** using the cleaned event log. Below is how each was defined and calculated.

### 1. Volume — Sales Count
The number of purchase events associated with a product. Since no quantity field exists, each row represents a single purchase transaction.

```
Volume = COUNT(purchase events) per product_id
```

### 2. Revenue — Gross Income
The sum of the `price` column across all purchase events. This represents **gross revenue** — no shipping costs, wholesale costs, or tax data were available.

```
Revenue = SUM(price) for purchase events per product_id
```

### 3. Popularity — Unique Buyers
The number of **distinct users** who completed a purchase for a given product. Repeated purchases by the same user are counted once, making this a measure of reach rather than intensity.

```
Popularity = COUNT(DISTINCT user_id) for purchase events per product_id
```

### 4. Conversion Rate
The percentage of cart additions that resulted in a completed purchase. This reflects how effectively a product converts intent (adding to cart) into action (buying). Product views were excluded due to missing data in the dataset.

```
Conversion Rate = (Number of purchase events / Number of cart events) × 100
```

### 5. Change in Performance — November vs. December
Volume, revenue, and unique buyers were each computed separately for November and December. The difference between the two periods highlights products that gained momentum heading into Christmas.

```
Volume Δ = december_volume − november_volume
Revenue Δ = december_revenue − november_revenue
Unique Buyers Δ = december_users − november_users
```

---

## Data Exploration & Preparation

Before analysis, both monthly datasets were merged and subjected to a thorough quality review. Three key issues were identified and resolved.

---

### 1. Dataset Coverage Validation

After combining the November and December files into a single dataset of **7,033,125 rows**, a date-range check confirmed that every calendar day in both months had at least one recorded transaction — meaning there are no coverage gaps in the data.

| Month | Days with Transactions |
|---|---|
| November | 30 / 30 |
| December | 31 / 31 |

The event-type split was also validated:

| Event Type | Share |
|---|---|
| `cart` | 75.0% |
| `purchase` | 25.0% |

This distribution is consistent with typical e-commerce behavior — most customers add items to their cart without completing a purchase.

---

### 2. Category Inconsistency Fix

A product catalog audit revealed that **~13,000 products were assigned to two different categories simultaneously** — a sign of upstream data quality issues.

On closer inspection, the `construction.tools.light` category was acting as a misclassification bucket. Smartphone brands such as Samsung, Apple, and Xiaomi were among its top contributors — an obvious mismatch.

**Fix applied:** For any product assigned to both `construction.tools.light` and another category, the alternative category was used. For all other multi-category products, the most frequently occurring category was retained.

| Before Fix — Top Categories by Events | After Fix — Top Categories by Events |
|---|---|
| `construction.tools.light`: 1,823,398 | `electronics.smartphone`: 3,350,680 |
| `electronics.smartphone`: 1,577,185 | `sport.bicycle`: 384,120 |

This correction produced a dataset that more accurately reflects the actual product catalog.

---

### 3. Brand Label Inconsistency Fix

A further check found that **1,245 products were associated with more than one brand label** — a secondary data quality issue, likely caused by inconsistent data entry upstream.

**Fix applied:** For each affected product, the most frequently occurring brand label was retained and applied consistently across all rows.

| Top Brands by Event Count (after fix) |
|---|
| Samsung — 1,756,420 events |
| Apple — 1,368,510 events |
| Xiaomi — 719,910 events |
| Huawei — 263,429 events |
| Oppo — 128,661 events |

---

### 4. Product Identifier Creation

To enable unambiguous product-level aggregation, a composite `product_identifier` field was created:

```
product_identifier = product_id + "-" + brand + " " + subcategory
```

**Example:** `1005014-samsung electronics.smartphone`

This ensures that products sharing the same ID but belonging to different brand-category combinations are tracked separately. The cleaned dataset was exported as a compressed Parquet file for use in the analysis phase.

---

## Analysis & Insights

The analysis produced a product-level summary table (`purchases_table`) containing all five metrics for each product, split by month where applicable. The key findings across each dimension are reported below.

---

### Insight 1 — Volume: Most Purchased Products

The vast majority of high-volume products fall within the **electronics** category, with Samsung, Apple, and Xiaomi smartphones dominating the top of the purchase count rankings. This aligns with the overall dataset composition — electronics represent over 55% of all events after the category correction.

Products with high volume but relatively lower revenue (e.g., budget smartphones and accessories) indicate high customer throughput at lower margins.

---

### Insight 2 — Revenue: Highest Gross Income Products

Premium-tier smartphones generate the highest gross revenue per product. Apple products in particular show strong revenue contributions driven by high unit price (some products priced above $1,200–$1,275), even when volume is moderate.

**Notable example:** Product `1001618-apple electronics.smartphone` generated $18,059.76 in November revenue alone from 36 purchases — an average order value of approximately $501.

This highlights a segment of high-value, lower-frequency purchases that are disproportionately important from a revenue standpoint.

---

### Insight 3 — Popularity: Widest Customer Reach

Popularity (unique buyers) closely mirrors volume for most products, since repeat purchases of the same product by the same user are relatively rare in this dataset. However, a handful of products with moderate volume show notably high unique buyer counts — these are products that attract new customers rather than repeat purchasers, making them strong candidates for acquisition-focused advertising.

---

### Insight 4 — Conversion Rate: Cart-to-Purchase Efficiency

Conversion rate varies significantly across products. Products with a high conversion rate indicate strong purchase intent at the point of cart addition — customers who add them rarely abandon the cart. These products are likely well-priced relative to perceived value, or benefit from high brand trust.

Products with low conversion rates despite high cart addition volume represent a potential optimization opportunity — targeted reminders or promotional nudges during the Christmas period could recover a meaningful share of abandoned carts.

---

### Insight 5 — Growth from November to December

Products that recorded a **positive volume, revenue, and unique buyer delta** between November and December represent those that genuinely benefited from the Christmas season effect. Several Apple smartphone models showed strong December-only performance (November volume = 0, December volume > 10), suggesting new releases or targeted promotions drove a spike.

**Example of strong December growth:**

| Product | Nov Volume | Dec Volume | Δ Volume | Nov Revenue ($) | Dec Revenue ($) | Δ Revenue ($) |
|---|---|---|---|---|---|---|
| `1001605-apple electronics.smartphone` | 0 | 18 | +18 | 0.00 | 9,806.76 | +9,806.76 |
| `1001606-apple electronics.smartphone` | 0 | 11 | +11 | 0.00 | 5,662.69 | +5,662.69 |
| `1001588-meizu electronics.smartphone` | 6 | 13 | +7 | 766.29 | 1,652.55 | +886.26 |

Conversely, products such as `1001618-apple electronics.smartphone` showed strong November volume that tapered off in December (−29 units), indicating potential stock issues, a price shift, or early-season demand that was not sustained.

---

## Conclusion

The analysis confirms that **electronics — primarily smartphones — are the cornerstone of this retailer's Christmas campaign performance**, both in terms of volume and revenue. The top three brands (Samsung, Apple, Xiaomi) account for the overwhelming majority of both purchase events and gross revenue during the November–December 2019 period.

Key takeaways:

- **Volume leaders** are predominantly mid-range Samsung and Xiaomi smartphones, driven by accessible price points and brand recognition.
- **Revenue leaders** skew toward premium Apple devices, where high unit prices compensate for lower sales volumes.
- **Conversion efficiency** varies meaningfully across the catalog, identifying products where advertising spend would yield the highest return.
- **December growers** — particularly select Apple models — signal products that are especially responsive to the Christmas season, and therefore strong candidates for campaign prioritization.
- **Data quality issues** (misclassified categories, inconsistent brand labels) were identified and resolved prior to analysis. These issues should be flagged to the data engineering team to prevent recurrence in future campaigns.

---

## Recommendations

Based on the findings, the following actions are recommended for future sale period advertising strategy:

**1. Prioritize High-Revenue, High-Conversion Products**
Allocate the largest share of advertising budget to premium Apple smartphones and other high-revenue products that also show strong conversion rates. These products generate the greatest return per advertising dollar and attract buyers who have already demonstrated strong purchase intent.

**2. Use High-Volume Products for Reach and Awareness**
Samsung and Xiaomi products with high sales volume and wide unique buyer counts are well-suited for broad awareness campaigns. Their accessible price points and brand familiarity make them effective for customer acquisition.

**3. Capitalize on December Growers Early**
Products that surged in December with no prior November activity (e.g., specific Apple models) should be scheduled into campaigns at the start of November in future cycles. These products likely benefit from Christmas gifting intent and should be surfaced before competitor campaigns dominate ad placements.

**4. Address Low-Conversion, High-Cart Products**
A subset of products accumulates significant cart additions without converting. For the next campaign, consider deploying cart abandonment triggers (retargeting ads, email nudges, limited-time discounts) specifically for these products to recover lost revenue.

**5. Resolve Upstream Data Quality Issues**
The `construction.tools.light` category misclassification and the 1,245 products with inconsistent brand labels indicate gaps in the product catalog management process. Correcting these at the source will improve the accuracy of future analyses and reduce preparation overhead.

---

*Analysis conducted on November–December 2019 e-commerce event data. All revenue figures represent gross revenue (pre-cost). Conversion rates are calculated relative to cart events only; view-based funnel data was unavailable.*
