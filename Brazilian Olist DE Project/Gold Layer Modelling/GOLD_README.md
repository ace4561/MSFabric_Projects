# Gold layer

The business-ready dimensional model. This pipeline updates the Gold layer entirely from metadata — the stored procedure to run for each table is looked up dynamically rather than hardcoded activity-by-activity, so adding a new dimension or fact later means adding a metadata row, not editing the pipeline

## Gold Olist Data Pipeline: Pipeline flow

1. **`lookup metadata dim table` (Lookup)** — retrieves the list of dimension tables to load and the stored procedure associated with each one.

2. **`loop through and exec all dim...` (ForEach)** — iterates over that list, dynamically building and executing the correct stored procedure name (`schema.procedure_name`) for each dimension via a Stored Procedure activity.
   - **On success** — proceeds to the fact table stage.
   - **On failure** — routes to **`Email Notification on dim table load...`**, so a failed dimension load is flagged rather than silently allowed to feed a fact load with stale or missing dimension keys.

3. **`lookup metadata fact table` (Lookup)** — same pattern as step 1, but for the fact tables, run only after dimensions have loaded successfully.

4. **`loop though and exec all fact...` (ForEach)** — iterates over the fact table list, executing each one's stored procedure (the MERGE-based, insert-only or upsert logic built per table).
   - **On success/failure** — routes to **`Email Notification on fact table load...`**, confirming completion or flagging a problem with the fact load stage specifically.

## Pipeline Architecture

![Pipeline_Design_Diagram](./Gold%20Olist%20Data%20Pipeline%20Visual.png)

## Design notes

- **Dimensions load before facts, deliberately.** Since fact tables resolve their surrogate keys via lookups against dimensions during their MERGE (`product_sk`, `seller_sk`, `customer_sk`), running dimensions first — and gating the fact stage behind the dimension stage's success — avoids fact rows landing with unresolved or null foreign keys.
- **Metadata-driven execution keeps the pipeline stable as the model grows.** Both ForEach loops build their stored procedure names dynamically from a metadata lookup rather than one activity per table, so the pipeline doesn't need editing every time a table is added or renamed.
- **Separate failure notifications per stage** make it immediately clear whether a failed run needs investigation in the dimension logic or the fact logic, rather than one generic "Gold pipeline failed" alert.
