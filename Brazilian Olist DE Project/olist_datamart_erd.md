# Olist gold layer — datamart diagrams

Full-column star schema diagrams for the gold layer dimensional model: three fact tables, plus four reporting views built on top of them.

## 1. Payments datamart

```mermaid
erDiagram
    DIM_OLIST_ORDERS ||--o{ FACT_OLIST_PAYMENTS : order_sk
    DIM_OLIST_CUSTOMERS ||--o{ FACT_OLIST_PAYMENTS : customer_sk

    DIM_OLIST_ORDERS {
        bigint order_sk PK
        varchar order_id
        varchar customer_unique_id
        varchar order_status
        datetime2 order_purchase_timestamp
        datetime2 order_approved_at
        datetime2 order_delivered_carrier_date
        datetime2 order_delivered_customer_date
        datetime2 order_estimated_delivery_date
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_OLIST_CUSTOMERS {
        bigint customer_sk PK
        varchar customer_unique_id
        varchar customer_zip_code_prefix
        varchar customer_city
        varchar customer_state
        varchar customer_state_name
        decimal geo_lat
        decimal geo_lng
        char active_flag
        datetime2 start_date
        datetime2 end_date
        datetime2 created_at
        datetime2 updated_at
    }

    FACT_OLIST_PAYMENTS {
        bigint payment_sk PK
        bigint order_sk FK
        bigint customer_sk FK
        varchar order_id
        int payment_sequential
        varchar payment_type
        int payment_installments
        decimal payment_value
        datetime2 created_at
        datetime2 updated_at
    }
```

## 2. Order items datamart

```mermaid
erDiagram
    DIM_OLIST_ORDERS ||--o{ FACT_OLIST_ORDER_ITEMS : order_sk
    DIM_OLIST_PRODUCTS ||--o{ FACT_OLIST_ORDER_ITEMS : product_sk
    DIM_OLIST_SELLERS ||--o{ FACT_OLIST_ORDER_ITEMS : seller_sk
    DIM_OLIST_CUSTOMERS ||--o{ FACT_OLIST_ORDER_ITEMS : customer_sk
    DIM_DATE ||--o{ FACT_OLIST_ORDER_ITEMS : shipping_limit_date_sk

    DIM_OLIST_ORDERS {
        bigint order_sk PK
        varchar order_id
        varchar customer_unique_id
        varchar order_status
        datetime2 order_purchase_timestamp
        datetime2 order_approved_at
        datetime2 order_delivered_carrier_date
        datetime2 order_delivered_customer_date
        datetime2 order_estimated_delivery_date
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_OLIST_PRODUCTS {
        bigint product_sk PK
        varchar product_id
        varchar product_category_name
        varchar product_category_name_english
        int product_name_length
        int product_description_length
        int product_photos_qty
        int product_weight_g
        decimal product_length_cm
        decimal product_height_cm
        decimal product_width_cm
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_OLIST_SELLERS {
        bigint seller_sk PK
        varchar seller_id
        varchar seller_zip_code_prefix
        varchar seller_city
        char seller_state
        decimal geo_lat
        decimal geo_lng
        varchar seller_state_name
        char active_flag
        datetime2 start_date
        datetime2 end_date
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_OLIST_CUSTOMERS {
        bigint customer_sk PK
        varchar customer_unique_id
        varchar customer_zip_code_prefix
        varchar customer_city
        varchar customer_state
        varchar customer_state_name
        decimal geo_lat
        decimal geo_lng
        char active_flag
        datetime2 start_date
        datetime2 end_date
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_DATE {
        int date_sk PK
        date calendar_date
        int year_number
        int quarter_number
        int month_number
        varchar month_name
        varchar month_name_short
        int day_of_month
        int day_of_year
        int week_of_year
        int day_of_week_number
        varchar day_name
        varchar day_name_short
        bit is_weekend
        varchar year_month
        varchar year_quarter
    }

    FACT_OLIST_ORDER_ITEMS {
        bigint order_item_sk PK
        bigint order_sk FK
        bigint product_sk FK
        bigint seller_sk FK
        bigint customer_sk FK
        varchar order_id
        int order_item_id
        int shipping_limit_date_sk FK
        datetime2 shipping_limit_date
        int quantity
        decimal price
        decimal freight_value
        decimal total_item_value
        float freight_pct_of_total
        datetime2 created_at
        datetime2 updated_at
    }
```

## 3. Order reviews datamart

```mermaid
erDiagram
    DIM_OLIST_ORDERS ||--o{ FACT_OLIST_ORDER_REVIEWS : order_sk
    DIM_OLIST_CUSTOMERS ||--o{ FACT_OLIST_ORDER_REVIEWS : customer_sk
    DIM_DATE ||--o{ FACT_OLIST_ORDER_REVIEWS : review_creation_date_sk
    DIM_DATE ||--o{ FACT_OLIST_ORDER_REVIEWS : review_answer_date_sk

    DIM_OLIST_ORDERS {
        bigint order_sk PK
        varchar order_id
        varchar customer_unique_id
        varchar order_status
        datetime2 order_purchase_timestamp
        datetime2 order_approved_at
        datetime2 order_delivered_carrier_date
        datetime2 order_delivered_customer_date
        datetime2 order_estimated_delivery_date
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_OLIST_CUSTOMERS {
        bigint customer_sk PK
        varchar customer_unique_id
        varchar customer_zip_code_prefix
        varchar customer_city
        varchar customer_state
        varchar customer_state_name
        decimal geo_lat
        decimal geo_lng
        char active_flag
        datetime2 start_date
        datetime2 end_date
        datetime2 created_at
        datetime2 updated_at
    }

    DIM_DATE {
        int date_sk PK
        date calendar_date
        int year_number
        int quarter_number
        int month_number
        varchar month_name
        varchar month_name_short
        int day_of_month
        int day_of_year
        int week_of_year
        int day_of_week_number
        varchar day_name
        varchar day_name_short
        bit is_weekend
        varchar year_month
        varchar year_quarter
    }

    FACT_OLIST_ORDER_REVIEWS {
        bigint review_sk PK
        bigint order_sk FK
        bigint customer_sk FK
        varchar review_id
        varchar order_id
        int review_creation_date_sk FK
        int review_answer_date_sk FK
        smallint review_score
        bit has_comment_title
        bit has_comment_message
        int comment_title_length
        int comment_message_length
        float response_time_hours
        bit is_positive_review
        bit is_negative_review
        varchar review_comment_title
        varchar review_comment_message
        datetime2 created_at
        datetime2 updated_at
    }
```

## 4. `vw_order_payments` — order value vs. amount paid

Reconciles what an order was actually worth (summed from `fact_olist_order_items.total_item_value`) against what was actually paid for it (summed from `fact_olist_payments.payment_value`), surfacing any `outstanding_amount` gap between the two.

```mermaid
erDiagram
    FACT_OLIST_PAYMENTS ||--o| VW_ORDER_PAYMENTS : order_id
    FACT_OLIST_ORDER_ITEMS ||--o| VW_ORDER_PAYMENTS : order_id
    DIM_OLIST_ORDERS ||--o| VW_ORDER_PAYMENTS : order_id

    FACT_OLIST_PAYMENTS {
        varchar order_id
        decimal payment_value
    }

    FACT_OLIST_ORDER_ITEMS {
        varchar order_id
        decimal total_item_value
    }

    DIM_OLIST_ORDERS {
        varchar order_id
        varchar customer_unique_id
    }

    VW_ORDER_PAYMENTS {
        varchar order_id
        varchar customer_unique_id
        decimal total_payment_made
        decimal total_order_value
        decimal outstanding_amount
    }
```

## 5. `vw_olist_order_delivery_metrics` — delivery performance

Computed directly on `dim_olist_orders` — no separate fact table needed, since these measures sit at the same 1:1 grain as the order itself.

```mermaid
erDiagram
    DIM_OLIST_ORDERS ||--|| VW_OLIST_ORDER_DELIVERY_METRICS : order_sk

    DIM_OLIST_ORDERS {
        bigint order_sk PK
        varchar order_id
        varchar customer_unique_id
        varchar order_status
        datetime2 order_purchase_timestamp
        datetime2 order_approved_at
        datetime2 order_delivered_carrier_date
        datetime2 order_delivered_customer_date
        datetime2 order_estimated_delivery_date
    }

    VW_OLIST_ORDER_DELIVERY_METRICS {
        bigint order_sk PK, FK
        varchar order_id
        varchar customer_unique_id
        float hours_to_approve
        float hours_to_carrier
        float hours_carrier_to_customer
        float hours_purchase_to_delivery
        int delivery_delay_days
        bit is_delivered
        bit is_delivered_late
        bit is_canceled
    }
```

## 6. `vw_olist_sales_by_customer` — sales rolled up per customer

`fact_olist_order_items` joined to `dim_olist_customers` via `customer_sk`, aggregated to one row per `customer_unique_id`.

```mermaid
erDiagram
    FACT_OLIST_ORDER_ITEMS ||--o| VW_OLIST_SALES_BY_CUSTOMER : customer_sk
    DIM_OLIST_CUSTOMERS ||--o| VW_OLIST_SALES_BY_CUSTOMER : customer_sk

    FACT_OLIST_ORDER_ITEMS {
        bigint customer_sk FK
        int quantity
        decimal price
        decimal freight_value
        decimal total_item_value
    }

    DIM_OLIST_CUSTOMERS {
        bigint customer_sk PK
        varchar customer_unique_id
    }

    VW_OLIST_SALES_BY_CUSTOMER {
        varchar customer_unique_id
        int total_qty
        decimal total_item_cost
        decimal total_freight_cost
        decimal order_total
    }
```

## 7. `vw_olist_sales_by_order` — sales rolled up per order

`dim_olist_orders` joined to `fact_olist_order_items` on `order_id`, aggregated to one row per order, including the average freight cost as a percentage of order value.

```mermaid
erDiagram
    DIM_OLIST_ORDERS ||--o| VW_OLIST_SALES_BY_ORDER : order_id
    FACT_OLIST_ORDER_ITEMS ||--o| VW_OLIST_SALES_BY_ORDER : order_id

    DIM_OLIST_ORDERS {
        varchar order_id
        varchar customer_unique_id
    }

    FACT_OLIST_ORDER_ITEMS {
        varchar order_id
        int quantity
        decimal price
        decimal freight_value
        decimal total_item_value
        float freight_pct_of_total
    }

    VW_OLIST_SALES_BY_ORDER {
        varchar order_id
        varchar customer_unique_id
        int total_qty
        decimal total_item_cost
        decimal total_freight_cost
        decimal order_total
        float avg_freight_pct
    }
```
