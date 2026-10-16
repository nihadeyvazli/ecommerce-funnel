# E-Commerce Funnel Analysis – View to Purchase

## Dataset

[eCommerce Events History in Cosmetics Shop](https://www.kaggle.com/datasets/mkechinov/ecommerce-events-history-in-cosmetics-shop) (Kaggle).
| File | Rows |
|---|---|
| 2020-Jan.csv | 4,264,752 |
| 2020-Feb.csv | 4,156,682 |
| **Combined** | **8,421,434** |

Each row is one event: a user viewed a product, added it to the cart, removed it from the cart, or bought it.

## Methodology

**Loading.** The files are large, so I loaded only the 7 columns I needed (`usecols`), concatenated both months with `pd.concat`, converted `event_time` to datetime and added a `month` column (Jan / Feb).

**Event types.** The dataset has 4 event types, not 3:

| event_type | Count | Share |
|---|---|---|
| view | 3,991,194 | 47.39% |
| cart | 2,297,017 | 27.28% |
| remove_from_cart | 1,627,433 | 19.32% |
| purchase | 505,790 | 6.01% |

`remove_from_cart` is not a funnel stage, so I dropped it (6,794,001 rows left). Within the 3 funnel stages the split is view 58.75%, cart 33.81%, purchase 7.44%.

**Funnel = unique users, not events.** One user who viewed 30 products is still one viewer. So every stage is counted with `nunique()` on `user_id`:

```python
funnel = (df.groupby("event_type")
            .agg(events=("user_id", "size"),
                 users=("user_id", "nunique"))
            .reindex(["view", "cart", "purchase"]))
```

**Conversion rates.**

```
view → cart      = cart users / view users × 100
cart → purchase  = purchase users / cart users × 100
view → purchase  = purchase users / view users × 100
drop-off         = 100 − conversion rate
```

The same logic is reused for categories, months, brands and products, just grouped by a different column.

**Missing values.** `category_code` is empty in 98.2% of rows and `brand` in 42.8%. I did not fill them; I only dropped them inside the category and brand steps. The main funnel only needs `event_type` and `user_id`, which have no missing values. This means the category analysis is based on a small part of the data (~1.8% of rows).

**Price outliers (IQR).** Applied to the prices of purchase events:

```
IQR   = Q3 − Q1
lower = Q1 − 1.5 × IQR
upper = Q3 + 1.5 × IQR
```

## Results

### Funnel

| Stage | Events | Unique users | Events per user |
|---|---|---|---|
| View | 3,991,194 | 716,987 | 5.57 |
| Cart | 2,297,017 | 162,035 | 14.18 |
| Purchase | 505,790 | 49,473 | 10.22 |

| Step | Conversion | Drop-off |
|---|---|---|
| view → cart | 22.60% | 77.40% |
| cart → purchase | 30.53% | 69.47% |
| view → purchase (overall) | **6.90%** | 93.10% |

The overall rate is the product of the two steps: 22.60% × 30.53% ≈ 6.90%.

### By category (top 8 by views)

| Category | Viewers | view → cart | cart → purchase | view → purchase |
|---|---|---|---|---|
| apparel.glove | 4,965 | 83.14% | 46.80% | **38.91%** |
| stationery.cartrige | 4,313 | 52.52% | 45.25% | 23.77% |
| furniture.bathroom.bath | 3,602 | 32.32% | 32.99% | 10.66% |
| appliances.environment.vacuum | 15,431 | 25.16% | 35.18% | 8.85% |
| appliances.personal.massager | 1,335 | 11.31% | 23.18% | 2.62% |
| appliances.personal.hair_cutter | 1,141 | 9.90% | 24.78% | 2.45% |
| furniture.living_room.cabinet | 5,377 | 14.13% | 15.66% | 2.21% |
| accessories.bag | 3,147 | 4.64% | 30.82% | **1.43%** |

### January vs February

| Month | Viewers | Cart users | Buyers | view → cart | cart → purchase | view → purchase |
|---|---|---|---|---|---|---|
| Jan | 397,775 | 92,653 | 28,220 | 23.29% | 30.46% | 7.09% |
| Feb | 379,246 | 89,269 | 25,759 | 23.54% | 28.86% | 6.79% |
| **Change** | | | | +0.25 pp | **−1.60 pp** | **−0.30 pp** |

Overall conversion did **not** improve in February, it dropped by 0.30 pp. The drop comes from the cart → purchase step. Rates are compared rather than counts because February 2020 had 29 days vs 31. Note: 63,487 users were active in both months (60,034 of them viewed products in both), so monthly user counts add up to more than the total funnel.

### Top 10 brands by purchase count

| Brand | Purchases (items) | Buyers | view → purchase |
|---|---|---|---|
| runail | 45,274 | 15,562 | 17.92% |
| irisk | 31,210 | 13,332 | 18.91% |
| grattol | 21,320 | 7,905 | 16.18% |
| masura | 19,929 | 5,401 | 15.15% |
| ingarden | 10,276 | 4,972 | 20.96% |
| estel | 8,616 | 3,994 | 9.93% |
| bpw.style | 8,437 | 3,070 | **22.95%** |
| uno | 6,871 | 4,128 | 18.50% |
| italwax | 6,740 | 2,927 | 14.69% |
| kapous | 6,370 | 3,018 | **7.14%** |

Brands are ranked by purchase events (items sold); conversion uses unique users. The top 5 brands account for 25.31% of all purchase events (128,009 / 505,790).

### Price outliers (IQR)

| Q1 | Q3 | IQR | Lower bound | Upper bound |
|---|---|---|---|---|
| $1.59 | $5.56 | $3.97 | −$4.37 | $11.52 |

**34,947 purchases (6.91%)** are outliers:

| Side | Count | What they are |
|---|---|---|
| High (> $11.52) | 34,876 | Real, rarely bought expensive products, up to **$327.78** (the shop also sells vacuums, furniture and professional products). Kept. |
| Low (< −$4.37) | 71 | **Negative prices**, down to **−$79.37**. |

In total, 72 purchases have a price of zero or below. A real price cannot be zero or negative, so these are data-quality issues (for example refunds or corrections logged as purchases) and should be checked with the data owner. They are only 0.01% of purchases, so they do not change any funnel result. The task asks to identify outliers, not remove them.

### Bonus 1: time from first view to first purchase

For 45,571 buyers whose first view came before their first purchase:

| 25% | Median | 75% | Mean | Max |
|---|---|---|---|---|
| 0.40 h (~24 min) | **4.53 h** | 144.49 h (~6 days) | 131.81 h (5.49 days) | 1,415.91 h (~59 days) |

The mean is pulled up by a group of slow buyers, so the median describes the typical buyer better. 3,902 buyers were excluded because they had no view in Jan–Feb or bought before their first recorded view (probably viewed in December).

### Bonus 2: high view, low conversion products

"High view" = top 5% of products by unique viewers (95th percentile = 213 viewers), which gives 2,530 products. **22 of them had zero purchases in two months.** The most viewed ones:

| product_id | Brand | Viewers | Cart users | Buyers |
|---|---|---|---|---|
| 27759 | masura | 712 | 0 | 0 |
| 5678162 | – | 553 | 0 | 0 |
| 5906219 | strong | 462 | 24 | 0 |
| 5893947 | masura | 444 | 0 | 0 |
| 5894358 | – | 419 | 8 | 0 |

## Differences from the task notes

My numbers differ from the notes given with the task, so I used my own and list the differences here:

| Metric | Task notes | My result |
|---|---|---|
| Rows | ~2.6M + ~1.7M | 4.26M + 4.16M (full Kaggle files) |
| Event split | view 85.1% / cart 8.7% / purchase 6.2% | 58.75% / 33.81% / 7.44% (+ remove_from_cart in raw data) |
| view → cart | 10.2% | 22.60% |
| cart → purchase | 71.4% | 30.53% |
| view → purchase | 7.3% | 6.90% |
| Jan → Feb | +0.8 pp | −0.30 pp |
| Best category | accessories (12.1%) | apparel.glove (38.91%); accessories.bag is the lowest (1.43%) |
| Top 5 brands share | 34% | 25.31% |
| Max price | $2,140 | $327.78 |

The max price in the notes looks like a basket (order) value, while I worked with item prices, so those two are not directly comparable.

## Insights for the product / growth team

1. **The biggest leak is the first step.** 77.4% of viewers never add anything to the cart. Improving product pages (photos, descriptions, price visibility) has the largest potential, because this is where most users are lost.

2. **The cart is used like a wishlist.** Cart users add to the cart 14 times on average, `remove_from_cart` events equal 71% of cart events, and only 30.5% of cart users end up buying. Cart reminders and "items in your cart" emails could turn more of this intent into purchases.

3. **February got worse at checkout, not at interest.** People added to the cart slightly more (+0.25 pp) but finished the purchase less (−1.60 pp), so overall conversion fell by 0.30 pp. It is worth checking what changed in February: promotions ending, delivery costs, payment issues.

4. **Some products and brands get attention but not sales.** Vacuums are the most viewed category (15,431 viewers) but convert at only 8.85%. kapous and estel are among the most viewed top brands but convert at 7–10%, while bpw.style converts at 22.95% with the fewest views and deserves more exposure. 22 high-view products sold nothing at all; three masura products with 350–700 viewers were never even added to the cart, which looks like a stock or page problem rather than lack of interest.

5. **Half of buyers decide within hours, a quarter needs a week or more.** The median time from first view to first purchase is 4.5 hours, but 25% of buyers take more than 6 days. Retargeting during the first week after a user's first visit can help win back the slow deciders.


