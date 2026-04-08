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
[README (1).md](https://github.com/user-attachments/files/26581419/README.1.md)
ata-exploration--preparation)
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
