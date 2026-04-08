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

This project analyses e-commerce event data captured during the **November–December 2019** period — the Christmas campaign window — to identify which products performed best across multiple business-relevant dimensions.

| Property | Details |
|:---|:---|
| **Period Covered** | November 1, 2019 → December 31, 2019 |
| **Raw Dataset** | 7,033,125 rows × 9 columns |
| **Final Dataset** | 7,033,125 rows × 11 columns (after cleaning & feature engineering) |
| **Event Types** | `cart` (75%) · `purchase` (25%) |
| **Unique Products** | 83,838 product IDs |
| **Products with Purchase Activity** | 45,403 |
| **Source Files** | `november.csv` (2,885,158 rows) · `december.csv` (4,147,967 rows) |

### Dataset Columns

| Column | Type | Description |
|:---|:---|:---|
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

Senior stakeholders want to determine **which products performed best during the Christmas period** in order to streamline the products advertised in future sale campaigns.

To answer this question rigorously, the business identified five dimensions of product performance:

| # | Metric | Business Question |
|:---|:---|:---|
| 1 | **Volume** | Which products were purchased the most? |
| 2 | **Revenue** | Which products generated the most gross income? |
| 3 | **Popularity** | Which products attracted the most unique customers? |
| 4 | **November → December Growth** | Which products showed improved performance as Christmas approached? |
| 5 | **Conversion Rate** | Which products most efficiently turned cart additions into purchases? |

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

### 4. Change in Performance — November vs. December
Volume, revenue, and unique buyers were each computed separately for November and December. The percentage change in volume was used as the primary ranking signal, normalising for product size.

```
Volume Δ% = ((december_volume − november_volume) / november_volume) × 100
```

Products that had zero sales in November but meaningful December volumes (≥ 100 units) were captured separately as **December-only breakout performers**.

### 5. Conversion Rate
The percentage of cart additions that resulted in a completed purchase. This reflects how effectively a product converts intent into action.

```
Conversion Rate = (Number of purchase events / Number of cart events) × 100
```

> **Note:** A small number of products showed conversion rates above 100%, indicating cases where purchases occurred without a logged cart event — likely due to an order history / reorder flow that bypasses the cart screen. These were excluded from the conversion ranking.

---

## Data Exploration & Preparation

Before analysis, both monthly datasets were merged and subjected to a thorough quality review. Three key issues were identified and resolved.

---

### 1. Dataset Coverage Validation

After combining November and December into a single dataset of **7,033,125 rows**, a date-range check confirmed that every calendar day in both months had at least one recorded transaction — no coverage gaps exist.

| Month | Days with Transactions |
|:---|:---|
| November | 30 / 30 |
| December | 31 / 31 |

The event-type split was also validated:

| Event Type | Share |
|:---|:---|
| `cart` | 75.0% |
| `purchase` | 25.0% |

This distribution is consistent with typical e-commerce behaviour — most customers add items to their cart without completing a purchase.

---

### 2. Category Inconsistency Fix

A product catalogue audit revealed that **~13,000 products were assigned to two different categories simultaneously**, indicating upstream data quality issues.

On closer inspection, `construction.tools.light` was acting as a misclassification bucket. Smartphone brands such as Samsung, Apple, and Xiaomi were among its top contributors — an obvious mismatch.

| | Category | Event Count |
|:---|:---|:---|
| **Before fix** | `construction.tools.light` | 1,823,398 |
| **Before fix** | `electronics.smartphone` | 1,577,185 |
| **After fix** | `electronics.smartphone` | 3,350,680 |
| **After fix** | `sport.bicycle` | 384,120 |

**Fix applied:** For any product assigned to both `construction.tools.light` and another category, the alternative category was retained. For all other multi-category products, the most frequently occurring category was used.

> The majority rule was deliberately not applied universally — doing so would risk re-assigning correctly labelled products *to* `construction.tools.light`, since it had accumulated the most events precisely through misclassification.

---

### 3. Brand Label Inconsistency Fix

**1,245 products** were associated with more than one brand label — a secondary data quality issue, likely caused by inconsistent data entry upstream.

**Fix applied:** For each affected product, the most frequently occurring brand label was retained across all rows.

| Top 5 Brands by Event Count (after fix) | Events |
|:---|:---|
| Samsung | 1,756,420 |
| Apple | 1,368,510 |
| Xiaomi | 719,910 |
| Huawei | 263,429 |
| Oppo | 128,661 |

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

The analysis produced a unified **metrics table** (45,403 products) combining purchase volumes, revenue, unique buyers, month-on-month deltas, and conversion rates. Three lenses were applied to surface the top performers.

---

### Insight 1 — Month-on-Month Volume Growth: December High Performers

Products were ranked by their **percentage increase in sales volume** from November to December. Two segments were identified:

- **Sustained growers:** Products with a November baseline > 10 units that grew by more than 200% in December.
- **December breakouts:** Products with zero November sales that exceeded 100 units sold in December.

This yielded **391 high-performing products**. Their distribution by category:

| Category | Products |
|:---|:---|
| Apparel | 100 |
| Appliances | 97 |
| Electronics | 82 |
| Furniture | 30 |
| Computers | 25 |
| Kids | 20 |
| Sport | 13 |
| Construction | 11 |
| Auto | 5 |
| Other | 8 |

The top brand-category combinations among December high performers:

| Category | Subcategory | Brand | Count |
|:---|:---|:---|:---|
| Appliances | `appliances.kitchen.coffee_grinder` | Lucente | 33 |
| Electronics | `electronics.smartphone` | Xiaomi | 12 |
| Apparel | `apparel.shoes` | Sony | 11 |
| Kids | `kids.toys` | Lucente | 10 |
| Electronics | `electronics.smartphone` | Huawei | 9 |
| Electronics | `electronics.smartphone` | Samsung | 8 |
| Computers | `computers.peripherals.printer` | Lucente | 7 |
| Apparel | `apparel.shoes` | LG | 7 |
| Electronics | `electronics.smartphone` | Apple | 6 |
| Furniture | `furniture.kitchen.chair` | Xiaomi | 6 |

**Notable examples of strong December growth:**

| Product | Nov Volume | Dec Volume | Δ Volume | Dec Revenue ($) |
|:---|:---|:---|:---|:---|
| `1002527-apple electronics.smartphone` | 14 | 455 | +441 (+3,150%) | 303,348 |
| `1003770-huawei electronics.smartphone` | 11 | 55 | +44 (+400%) | 12,116 |
| `1003533-samsung electronics.smartphone` | 123 | 450 | +327 (+266%) | 138,199 |
| `1003712-samsung electronics.smartphone` | 519 | 1,806 | +1,287 (+248%) | 920,217 |
| `1003801-apple electronics.smartphone` | 107 | 333 | +226 (+211%) | 208,191 |

---

### Insight 2 — Conversion Rate: Most Efficient Products in December

Filtering to products with meaningful activity (≥ 5 unique buyers, ≥ 10 purchases, conversion between 70% and 100%), **41 products** were identified as top converters in December.

Their distribution by category:

| Category | Products |
|:---|:---|
| Electronics | 6 |
| Appliances | 6 |
| Kids | 6 |
| Computers | 5 |
| Furniture | 5 |
| Apparel | 4 |
| Construction | 3 |
| Auto | 3 |
| Sport | 2 |
| Accessories | 1 |

Top subcategories among the highest-converting categories (electronics, appliances, kids):

| Subcategory | Products |
|:---|:---|
| `kids.carriage` | 3 |
| `appliances.environment.air_conditioner` | 2 |
| `electronics.video.tv` | 2 |
| `electronics.audio.acoustic` | 1 |
| `appliances.kitchen.hood` | 1 |
| `kids.dolls` | 1 |
| `kids.swing` | 1 |
| `electronics.tablet` | 1 |
| `appliances.kitchen.refrigerators` | 1 |
| `appliances.personal.massager` | 1 |

When conversion is the primary lens, **electronics, appliances, and kids products** emerge as the most purchase-intent-driven categories — customers who add these to their cart rarely abandon them.

---

### Insight 3 — Volume Distribution: Most Products Did Not Grow

Examining the full distribution of month-on-month volume change reveals an important baseline finding: **the majority of products showed little to no change** in sales volume between November and December. A notable portion actually recorded zero December sales, corresponding to a -100% change.

This underscores that the 391 high-growth products and 41 top converters identified above represent a genuinely exceptional subset of the catalogue — not a broad seasonal lift across all products.

---

## Conclusion

The analysis confirms that **December performance gains are concentrated in a specific subset of products**, rather than reflecting a uniform Christmas uplift across the catalogue. Key takeaways:

- **391 products** showed exceptional month-on-month volume growth — these are the strongest candidates for campaign prioritisation in future sale periods.
- **Smartphones dominate** the high-growth segment. Apple, Samsung, Xiaomi, and Huawei all appear prominently across both growth and revenue dimensions, confirming electronics as the backbone of Christmas performance.
- **Lucente** emerged as a notable high performer across coffee grinders, printers, and kids toys — a brand whose December momentum warrants further investigation and potential campaign investment.
- **41 products** demonstrated elite conversion efficiency in December (70–100%), led by electronics, appliances, and kids items — signalling strong purchase intent in these categories once customers engage.
- **Most products did not benefit** from the Christmas season. Concentrating advertising resources on high-growth and high-conversion segments will yield significantly better returns than broad catalogue promotion.
- **Data quality issues** — misclassified categories and inconsistent brand labels — were identified and resolved prior to analysis. These should be flagged to the data engineering team to prevent recurrence.

---

## Recommendations

**1. Prioritise the 391 December High-Growth Products**
Allocate the largest share of advertising budget to products that demonstrated the strongest month-on-month volume increases. These have already proven they respond to seasonal demand — amplifying their visibility will compound that effect in future campaigns.

**2. Double Down on Smartphones**
Apple, Samsung, Xiaomi, and Huawei smartphones consistently appear across both the growth and revenue rankings. These products combine high unit prices with strong December momentum and should anchor any premium advertising placements.

**3. Investigate and Elevate Lucente**
Lucente's strong showing across coffee grinders, printers, and kids toys is noteworthy. If this reflects genuine product-market fit, it represents an underexposed brand that could deliver strong returns with targeted campaign support.

**4. Use High-Conversion Products for Retargeting**
The 41 products with 70–100% December conversion rates are ideal candidates for retargeting campaigns. Customers who have viewed or wishlisted these items are highly likely to convert with a timely nudge — particularly in the kids and appliances categories where gifting intent is strong.

**5. Activate December Breakout Products Early**
Products with zero November sales that surpassed 100 December units were likely driven by Christmas gifting intent arriving late. In future cycles, surface these products at the start of November to capture demand earlier and ahead of competitor campaigns.

**6. Resolve Upstream Data Quality Issues**
The `construction.tools.light` misclassification and the 1,245 products with inconsistent brand labels point to gaps in the product catalogue management process. Correcting these at the source will improve the accuracy of future analyses and reduce preparation overhead significantly.

---

*Analysis conducted on November–December 2019 e-commerce event data. All revenue figures represent gross revenue (pre-cost). Conversion rates are calculated relative to cart events only; view-based funnel data was unavailable. Products with conversion rates exceeding 100% were excluded from the conversion ranking due to suspected data collection anomalies.*
