# Silver layer

The cleaning and conforming layer. This is where Bronze's raw, unmodified data gets deduplicated, type-cast, renamed, and validated before it's trusted enough to feed the Gold dimensional model.

## Silver Olist Data Pipeline: Pipeline flow

1. **`Go-No-Go` (Lookup)** — a data check. Before any transformation runs, this checks whether the Bronze load actually completed as expected (e.g., comparing row counts recorded in the audit control tables) rather than assuming Bronze finished cleanly just because its own pipeline reported success.

2. **`If Condition1`** — branches on the Go-No-Go result:
   - **True** — Bronze data passed the check, so **`Notebook1`**, **`Notebook2`** and **`DataFlow1`** activities run, performing the actual Silver transformations: deduplication (`drop_duplicates`), column renaming to fix source-side inconsistencies, type casting, null-handling defaults, and schema evolution via Spark's `mergeSchema`/`autoMerge` options.

   - **False** — Bronze data failed the check, so **`Office365Email1`** fires instead, alerting that the Silver load was skipped rather than proceeding against incomplete or inconsistent Bronze data.

3. **`Office365Email2`** — fires if the if condition activity or sub activities fail.

## Pipeline Architecture

![Pipeline_design_diagram](./Silver%20Olist%20Data%20Pipeline%20Visual.png)

## Dataflow Gen 2 Transformation for the Product Category table

![Dataflow_Gen_2](./SIlver%20Dataflow%20Gen%202%20-%20Product%20Category%20Name%20Translation.png)

## Design notes

- **The data check exists to prevent a bad Bronze load from silently propagating.** Rather than letting Silver run unconditionally and only discovering a problem downstream in Gold, the Go-No-Go check stops the pipeline early and routes via email to a person or team if Bronze's actual row counts don't match what was expected.
- **Schema evolution is handled deliberately, not uniformly.** Most Silver tables use Spark's `mergeSchema`/`autoMerge.enabled` so new Bronze columns flow through automatically. Customer data is the exception — schema changes there are gated behind manual review rather than auto-applied, since an unreviewed new column landing in a table containing customer data is a governance risk, not just a technical inconvenience.
- **Load pattern is create-once, merge-forever.** Each Silver table is created on first run only if it doesn't already exist, then every subsequent run uses an idempotent MERGE (insert-only for immutable data like payments and order items, upsert with change-detection for data that can legitimately be corrected, like product attributes).
- **No dimension modelling occurs here or adding of surrogate keys** Data is cleaned and transformed, but wholesale schema changes are avoided. So this layer will generally match the schema of bronze. The cleaned data is ready to be consumed by certain groups or departments such as data scientists for machine learning etc.
