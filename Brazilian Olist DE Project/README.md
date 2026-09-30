# Olist E-Commerce — Microsoft Fabric Data Platform

This is an end-to-end data engineering project built on Microsoft Fabric, using the [Olist Brazilian E-Commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). The goal wasn't just to model the data, but to simulate a production-style pipeline. 

Although the olist dataset is a one-off dataset, I have developed the solution so that it will ingest the data based on a trigger or a schedule in order to replicate a real world production scenario where files will be required to be ingested on a daily basis and each day the source system will produce data files with new or updated data in them from the previous day's transactions. Each dataset download from the Kaggle website extracts the files to a new timestamped folder in the Fabric Lakehouse files section with a fresh set of data files in them. The data in these files will then be appended to the existing tables within the Fabric Lakehouse tables section. This is done via a metadata driven Fabric Data Factory Pipeline

I chose the Olist dataset as it mirrors the messy relational complexity of a real-world production environment to some extent and because this project replicates a similar scenario I faced at work, where I needed to produce ETL processes that would extract data files on a daily schedule from a source system, to be loaded into another system via ETL processes so that we could provide an integrated and updated view of sales across different divisions of the business. 

This Microsoft Fabric project demonstrates the following approaches and best practices:

- A metadata-driven approach
- A medallion architecture
- Incremental batch loading
- Idempotency

## Aim

The aim of this project was to create an end-to-end data engineering solution in Microsoft Fabric using data engineering best practices and principles. My aim was to use several years of commercial data engineering experience that I have obtained to build a commercial/enterprise-level solution in the cloud facilitating the migration of skills and knowledge gained on on-premise and legacy systems to cloud-based environments and orchestration tools. I have intentionally used a hybrid architecture to try and make use of the various data ingestion and transformation capabilities of Microsoft Fabric including the use of Fabric Pipelines, Notebooks, Copy Data Activity and Dataflow Gen 2.


## Architecture

The platform follows a medallion architecture:

- **Bronze** — raw, unmodified landing zone. CSV files are pulled from Kaggle via the Kaggle API into timestamped folders (`Files/raw/{timestamp}/`), then loaded into Delta tables with no business logic applied. Each ingestion run gets its own folder, which is what makes incremental, metadata-driven processing possible downstream.
- **Silver** — cleaned and conformed. Deduplication, type casting, column renaming (fixing source-side inconsistencies), and null-handling happen here. This is also where schema evolution is actually implemented, using Spark's `mergeSchema`/`autoMerge` options.
- **Gold** — business-ready dimensional model. Star schema, surrogate keys, MERGE-based incremental loads.

A metadata-driven control layer has been implemented which includes tables that track which files have been ingested and their row counts, supporting incremental loads without reprocessing the full file history each run.

See [`olist_datamart_erd.md`](./olist_datamart_erd.md) for the star schema diagrams implemented in the Gold layer.



- **Architecture Design** - The medallion model has been built on Fabric Lakehouse and Warehouse platforms. Bronze and silver layers have been built in a Lakehouse while the gold layer has been built using a Warehouse. I have chosen this approach as the Warehouse is highly optimised for dimension modelling, better suited for structured data and heavy BI related joins while the Lakehouse is better suited for raw file ingestion, cleaning and transformation using notebooks, pyspark API and spark sql. Most business and data analysts are also familiar with SQL/T-SQL which makes the data more accessible to them via the gold layer. However in reality, a major factor in this decision within a commercial environment would depend on the use case and the skills/experience within the relevant teams.

- **Pipeline Development** - Four Fabric Data Factory Pipelines were built to enable the data ingestion, cleaning, transformation and modelling.    

        Bronze Olist Data Pipeline - used for bronze layer ingestion
        Silver Olist Data Pipeline - used for silver ETL/ELT
        Gold Olist Data Pipeline - used to update dimensional model
        Master Olist Pipeline - This is the pipeline that runs all the other pipelines in sequence and can be triggered on an event or set up on a schedule

- **Master Olist Pipeline Architecture Design**

![Master_Pipeline](./Master%20Olist%20Data%20Pipeline%20Visual.png)

![Master_Pipeline_success](./Master%20Olist%20Data%20Pipeline%20-%20Successful%20Run%20Visual.png)

## The dimensional model


**Dimensions**

- **`dim_olist_orders`** — one row per order. Holds descriptive attributes only (status, timestamps, `customer_unique_id`) — no measures. All three fact tables join to it on `order_sk`.
- **`dim_olist_customers`** — one row per real person (`customer_unique_id`), not per raw `customer_id`. Tracks address/location with SCD2 attributes (`active_flag`, `start_date`, `end_date`) so historical changes aren't lost on update.
- **`dim_olist_sellers`** — one row per seller, also SCD2-tracked for address/location changes, with `geo_lat`/`geo_lng` refreshed as a derived lookup rather than its own tracked attribute.
- **`dim_olist_products`** — one row per product, with physical attributes (weight, dimensions, category) carried through from the source.
- **`dim_date`** — standard calendar dimension, joined to wherever a date needs day-of-week/month/quarter attributes rather than just a raw timestamp.

**Fact tables**

- **`fact_olist_payments`** — one row per payment method used on an order (grain: `order_id` + `payment_sequential`). Insert-only.
- **`fact_olist_order_items`** — one row per unit purchased (grain: `order_id` + `order_item_id`). Insert-only, carries the actual sales measures (`price`, `freight_value`), plus `shipping_limit_date_sk` linking to `dim_date`.
- **`fact_olist_order_reviews`** — one row per review (grain: `review_id`), with review score, sentiment flags (`is_positive_review`, `is_negative_review`), and both a creation and an answer date linked to `dim_date`.

A separate fact table for delivery performance was deliberately not built. See *Trade-offs*, below.

## Key design decisions

**Insert-only vs. upsert, decided per table, not applied uniformly.** Payments and order items represent immutable historical events — once a payment or a sold item exists, it doesn't change — so both use insert-only MERGE logic (`WHEN NOT MATCHED THEN INSERT`, no update branch), which protects transaction history and keeps loads idempotent against reprocessed batches. `dim_orders`, by contrast, tracks a genuinely mutable attribute (`order_status` progresses over the life of a real order), so it uses a full upsert with change-detection logic to avoid rewriting rows that haven't actually changed.

**Grain decided deliberately, not assumed.** Olist's `order_items` table has no quantity column — multiple units of the same product appear as separate rows with incrementing `order_item_id`s, all sharing the same `product_id`. Building the MERGE match condition on `order_id` + `product_id` instead of `order_id` + `order_item_id` would have silently dropped legitimate repeat-purchase rows. A `quantity = 1` column was added deliberately — not as a derived calculation, but as a constant reflecting the table's true grain, making `SUM(quantity)` self-documenting for anyone querying the table later.

**`customer_id` vs. `customer_unique_id`.** Olist's raw data gives every order its own throwaway `customer_id` — even for the same real person ordering twice. `dim_customers` is rolled up to `customer_unique_id` instead, which is what actually makes repeat-customer analysis (lifetime value, retention) possible. The trade-off: this means `dim_customers` only holds one "current" address per person, not the address used on any specific historical order — a real limitation if order-level geography analysis (delivery time by region, freight cost by state) becomes a priority.

**Schema evolution: tested, and scoped by risk.** Spark's `mergeSchema`/`autoMerge.enabled` handles new columns cleanly at the Silver layer. Bronze ingestion (via Fabric's Copy Data activity) was tested against the same requirement and found to **fail** — Copy Data activity does not reliably widen an existing Lakehouse table's schema on append; it throws `SourceColumnIsNotDefinedInDeltaMetadata` instead of adding the column. This was diagnosed, not just noticed — the fix (replacing Copy Data ingestion with a Notebook activity using the same Spark pattern already proven in Silver) is scoped out of this iteration, but documented rather than left as a silent gap. Schema evolution is also deliberately **not** applied uniformly: PII-adjacent tables (customer data) use a detect-and-alert pattern rather than automatic evolution, since an unreviewed new column landing in a customer table is a data governance risk, not just a technical inconvenience.


## Trade-offs and scope decisions

- **No row-level audit logging in silver and gold.** Deliberately scoped out of silver anf gold layers to keep the project moving — the pattern (`@@ROWCOUNT` captured inside each stored procedure, written to an audit table in the same transaction) is understood and I have implemented it before, but it just wasn't worth the additional build time here.
- **No `fact_order_delivery` table.** Delivery performance measures (`hours_to_approve`, `delivery_delay_days`, `is_delivered_late`) would have sat at the exact same grain as `dim_orders` with no independent measures repeating — a legitimate but non-default Kimball pattern. Rather than force a 1:1 fact table into the model, these are exposed via `vw_order_delivery_metrics`, a view computed directly on `dim_orders`. Always current, zero load logic to maintain.
- **No gross margin analysis.** Genuinely not possible from this dataset — there's no cost/COGS data anywhere in Olist's public files. `freight_value` as a percentage of order value is used as the closest available cost-related proxy.
- **Dataflow Gen2 cross-workspace deployment.** Fabric's deployment pipelines don't rebind a Dataflow Gen2's source/destination connections when moving between workspaces (Dev → Test) — this needs to be retargeted manually after each deployment. A known platform limitation, not something fixable from the dataflow side.

## What I'd do differently

- Move all Bronze ingestion to the notebook-based Spark pattern from the start, rather than discovering Copy Data activity's schema-evolution limitation mid-project.
- Add the row-level audit logging (silver and gold layers) — it's a small addition once the rest of the pipeline is stable, and it's the kind of thing that matters more in a real production setting than it does for a portfolio build.
- Consider whether order-time geography (the address a specific order actually shipped to, vs. a customer's current rolled-up address) is worth its own dimension, if regional/delivery analysis becomes a bigger focus.
