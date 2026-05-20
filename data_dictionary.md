# Imbewu Retail — Data Dictionary

---

## Table 1: `stores`

| Column | Type | Description | Notes |
|---|---|---|---|
| store_id | string | Unique store identifier (e.g. S001) | Primary key |
| store_name | string | Full store name including format and suburb | |
| format | string | Store format: Express / Market / Mega | 3 formats |
| province | string | Province where store is located | **Data quality issue — see below** |
| city | string | City of store | |
| suburb | string | Suburb of store | |
| store_manager | string | Name of store manager | **1 NULL found** — one store has no manager recorded |
| opened_date | date | Date store opened | |

**Row count:** 45  
**Data quality issues:**
- `province` column has inconsistent casing: "Western Cape" and "western cape" both appear, as do "Gauteng" and "gauteng". Standardised to Title Case for all analysis.
- 1 NULL in `store_manager` — not material for revenue analysis.

**Store format breakdown:**
| Format | Count | Typical size |
|---|---|---|
| Express | 23 | Small convenience |
| Market | 16 | Mid-size neighbourhood |
| Mega | 6 | Large-format destination store |

**Stores by province (after cleaning):**
| Province | Store count |
|---|---|
| Gauteng | 16 |
| Western Cape | 12 |
| KwaZulu-Natal | 12 |
| Eastern Cape | 5 |

---

## Table 2: `products`

| Column | Type | Description | Notes |
|---|---|---|---|
| product_id | string | Unique product identifier (e.g. P001) | Primary key |
| product_name | string | Full product name including size/variant | |
| category | string | Top-level category | 5 categories |
| sub_category | string | Sub-category within category | |
| brand | string | Product brand | |
| unit_cost | float | Cost price to Imbewu (ZAR) | |
| unit_price | float | Standard selling price (ZAR) | |

**Row count:** 48  
**Data quality issues:** None. Clean table.

**Category breakdown:**
| Category | Products |
|---|---|
| Groceries | 24 |
| Household | 8 |
| Health & Beauty | 8 |
| Electronics | 5 |
| Apparel | 3 |

---

## Table 3: `customers`

| Column | Type | Description | Notes |
|---|---|---|---|
| customer_id | string | Unique customer identifier (e.g. C00001) | Primary key |
| first_name | string | Customer first name | |
| last_name | string | Customer last name | |
| gender | string | Gender (M/F) | **139 NULLs** |
| birth_year | integer | Year of birth | **161 NULLs** |
| loyalty_tier | string | Loyalty programme tier: Bronze / Silver / Gold | |
| home_suburb | string | Customer's home suburb | |
| signup_date | date | Date joined loyalty programme | |

**Row count:** 3,000  
**Data quality issues:**
- 139 NULLs in `gender` (4.6%) — not used in primary analysis
- 161 NULLs in `birth_year` (5.4%) — not used in primary analysis
- No NULLs in `loyalty_tier` — this field is reliable for analysis

**Loyalty tier breakdown:**
| Tier | Count | % |
|---|---|---|
| Bronze | 2,091 | 69.7% |
| Silver | 660 | 22.0% |
| Gold | 249 | 8.3% |

**Note:** Transactions also have ~3,911 NULL `customer_id` values — these represent non-loyalty (anonymous) shoppers. They are excluded from loyalty-tier analysis but included in all revenue totals.

---

## Table 4: `transactions`

| Column | Type | Description | Notes |
|---|---|---|---|
| transaction_id | string | Unique till receipt ID (e.g. T000001) | Primary key |
| store_id | string | Store where transaction occurred | FK → stores |
| customer_id | string | Loyalty card customer (nullable) | FK → customers; **3,911 NULLs** = anonymous shoppers |
| transaction_date | datetime | Date and time of transaction | |
| payment_method | string | Payment method: Cash / Card / SnapScan | |

**Row count:** 9,164  
**Data quality issues:**
- 3,911 NULLs in `customer_id` (42.7%) — expected; represents walk-in customers not on the loyalty programme. Treated as anonymous; included in all revenue totals.

**Decision:** All revenue analysis includes anonymous transactions. Loyalty tier analysis is limited to matched customers only.

---

## Table 5: `transaction_items`

| Column | Type | Description | Notes |
|---|---|---|---|
| item_id | string | Unique line item ID (e.g. I0000001) | Primary key |
| transaction_id | string | Parent transaction | FK → transactions |
| product_id | string | Product purchased | FK → products |
| quantity | integer | Number of units purchased | |
| unit_price_at_sale | float | Actual selling price at time of sale (ZAR) | May differ from product.unit_price during promotions |
| discount_applied | float | Total discount amount applied to this line (ZAR) | |

**Row count:** 48,641  
**Data quality issues:** None. Clean table.

**Derived field used in analysis:**
```
revenue = (quantity × unit_price_at_sale) - discount_applied
```

---

## Table 6: `promotions`

| Column | Type | Description | Notes |
|---|---|---|---|
| promo_id | string | Unique promotion ID (e.g. PR001) | Primary key |
| promo_name | string | Promotion name | |
| category_targeted | string | Product category the promo applies to | |
| start_date | date | Promotion start date | |
| end_date | date | Promotion end date | |
| discount_pct | float | Percentage discount applied | |

**Row count:** 4  
**Data quality issues:** None. Clean table.

**Promotions in the dataset:**
| Promo ID | Name | Category | Period | Discount |
|---|---|---|---|---|
| PR001 | Pap Power Promo | Maize Meal | Apr 2025 | 33.33% (Buy 2 Get 1) |
| PR002 | Winter Warmer Sale | Beverages | Jun–Jul 2024 | 15% |
| PR003 | Back to School | Apparel | Jan 2025 | 20% |
| PR004 | Festive Cheer Special | Snacks | Dec 2024 | 10% |

---

## Schema Relationships

```
stores (store_id)
    └── transactions (store_id → store_id)
            ├── customers (customer_id → customer_id)  [nullable]
            └── transaction_items (transaction_id → transaction_id)
                        └── products (product_id → product_id)

promotions — standalone table; linked to transaction_items via category/date overlap
```

---

## Key Analysis Decisions

1. **Province casing:** Standardised all province values to Title Case (e.g. "gauteng" → "Gauteng") before any grouping.
2. **Year comparison:** The dataset contains full-year 2024 but only H1 2025 (Jan–Jun). All year-over-year comparisons are done on H1 only (January–June) to ensure like-for-like.
3. **Revenue definition:** `revenue = (quantity × unit_price_at_sale) - discount_applied`. This is net revenue after discounts.
4. **Anonymous shoppers:** Transactions with NULL `customer_id` are included in all revenue and basket metrics. They are excluded only from loyalty tier segmentation.
5. **Promotion attribution:** The Pap Power Promo (PR001) is identified by matching `sub_category = 'Maize Meal'` during the promo period (April 2025) and confirmed via non-zero `discount_applied`.
