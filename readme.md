# 📊 Customer Cohort & Category LTV Analysis (< 999)

![SQL](https://img.shields.io/badge/Language-SQL-blue.svg)
![Database](https://img.shields.io/badge/Engine-PostgreSQL%20%2F%20Redshift-orange.svg)
![Domain](https://img.shields.io/badge/Domain-E--commerce%20Analytics-green.svg)

## 📌 Overview

This repository contains an analytical SQL script designed to extract target customer profiles for marketing reactivation and remarketing campaigns. 

The query filters for **low-value (LTV < 999), recently acquired customers** who have purchased within key product categories on **Store 1**. To ensure maximum delivery success for outreach, contact information (phone number and full name) is dynamically pulled from each customer's **most recent shipping address**.

---

## 🎯 Business Use Case & Cohort Criteria

> **Goal:** Identify low-spend customers acquired within the last 24 months to run targeted win-back, cross-sell, or high-value conversion campaigns.

| Rule / Filter | Parameter Criteria | Business Rationale |
| :--- | :--- | :--- |
| **Product Categories** | `Health & Immunity`, `Natural Food`, `Lifestyle` | Focuses on specific core consumer segments. |
| **Recency Window** | `min(created_at) >= current_date - interval '24 months'` | Isolates recently acquired users within 2 years. |
| **Spend Threshold** | `sum(item_value) < 999` | Targets low-tier LTV customers for value growth. |
| **Order Status** | `complete`, `processing`, `delivered`, `shipped`, `in-transit`, etc. | Excludes canceled, refunded, or abandoned orders. |
| **Store Scope** | `store_id = 1` | Restricts analysis to the primary store view. |

---

## 🏗️ Technical Architecture

The query utilizes three **Common Table Expressions (CTEs)** to process data sequentially before serving the final dataset:

```
┌─────────────────────────────────────────────────────────┐
│                     SOURCE TABLES                       │
│ sales_order | sales_order_item | category_mapping_india │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│                  1. category_orders                     │
│  - Normalizes SKUs (regex removal of '-IN' / 'old')     │
│  - Filters target categories, store ID & valid statuses │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│                  2. customer_cohort                     │
│  - Aggregates spend per email (category_ltv)            │
│  - Applies HAVING filters (Recency >= -24m, LTV < 999)  │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│                   3. ranked_orders                      │
│  - Joins sales_order_address (shipping)                 │
│  - Ranks orders via ROW_NUMBER() per customer email     │
└────────────────────────────┬────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────┐
│                      FINAL OUTPUT                       │
│  - Filters for order_rank = 1 (Most recent address)     │
│  - Outputs: customer_email, telephone, firstname, last  │
└─────────────────────────────────────────────────────────┘
```

---

## 🔍 Logic Breakdown

### 1. `category_orders` (CTE)
* **Calculates Line Item Value:** `soi.base_price * soi.qty_ordered`
* **SKU Cleaning:** Applies `regexp_replace(soi.sku, '(-IN|old)$', '')` to strip suffix variations (`-IN`, `old`) for clean mapping against `category_table_mapping_india`.

### 2. `customer_cohort` (CTE)
* **Aggregates Metrics:** Computes `first_order_date`, `last_order_date`, and `category_ltv` grouped by `customer_email`.
* **Filters Cohort:** Uses `HAVING` logic to strictly keep customers whose first purchase was within **24 months** and whose category spend is **under 999**.

### 3. `ranked_orders` (CTE)
* **Contact Detail Isolation:** Joins `sales_order_address` on `address_type = 'shipping'` to get accurate delivery details.
* **Recency Ranking:** Applies `ROW_NUMBER() OVER (PARTITION BY so.customer_email ORDER BY so.created_at DESC, so.entity_id DESC)` to rank customer addresses by their latest order.

### 4. Final Query Execution
* Retrieves records where `order_rank = 1`, guaranteeing **one unique record per customer email** with their most updated phone number and name.

---

## 💻 SQL Script

```sql
WITH category_orders AS (
    SELECT
        so.customer_email,
        so.created_at,
        soi.base_price * soi.qty_ordered AS item_value
    FROM sales_order so
    JOIN sales_order_item soi
        ON so.entity_id = soi.order_id
    JOIN category_table_mapping_india ctm
        ON regexp_replace(soi.sku, '(-IN|old)$', '') = ctm.sku
    WHERE so.status IN (
            'complete', 'processing', 'delivered', 'Packed', 'shipped',
            'in-transit', 'dispatching_process', 'partial_delivered', 'out-for-delivery'
          )
      AND so.store_id = 1
      AND ctm.l1 IN ('Health & Immunity', 'Natural Food', 'Lifestyle')
),

customer_cohort AS (
    SELECT
        customer_email,
        MIN(created_at) AS first_order_date,
        SUM(item_value) AS category_ltv,
        MAX(created_at) AS last_order_date
    FROM category_orders
    GROUP BY customer_email
    HAVING MIN(created_at) >= CURRENT_DATE - INTERVAL '24 months'
       AND SUM(item_value) < 999
),

ranked_orders AS (
    SELECT
        so.customer_email,
        soa.telephone,
        soa.firstname,
        soa.lastname,
        ROW_NUMBER() OVER (
            PARTITION BY so.customer_email
            ORDER BY so.created_at DESC, so.entity_id DESC
        ) AS order_rank
    FROM sales_order so
    JOIN sales_order_address soa
        ON so.entity_id = soa.parent_id
       AND soa.address_type = 'shipping'
    JOIN customer_cohort c
        ON c.customer_email = so.customer_email
    WHERE so.status IN (
            'complete', 'processing', 'delivered', 'Packed', 'shipped',
            'in-transit', 'dispatching_process', 'partial_delivered', 'out-for-delivery'
          )
      AND so.store_id = 1
)

SELECT 
    customer_email, 
    telephone, 
    firstname, 
    lastname
FROM ranked_orders
WHERE order_rank = 1
ORDER BY customer_email;
```

---

## 📋 Sample Output Schema

| customer_email | telephone | firstname | lastname |
| :--- | :--- | :--- | :--- |
| `alex.d@example.com` | `+919876543210` | Alex | Doe |
| `jane.smith@example.com` | `+919812345678` | Jane | Smith |