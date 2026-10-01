# Bronze layer

The raw ingestion layer. This is the landing zone for the Olist source files — no cleaning, no business logic, no dimensional modeling happens here.

## Pipeline flow

1. **`Ingest_Data` (Notebook)** — authenticates to the Kaggle API and downloads the Olist dataset's CSV files into a new, uniquely timestamped folder under `Files/raw/{timestamp}/` for each run. This is what makes incremental, metadata-driven processing possible downstream — every run's files are isolated in their own folder rather than overwriting a shared location.

2. **`Lookup_Processed_Folders` (Lookup)** — checks the metadata/audit control tables to identify which folders (and the files within them) haven't been processed yet, so the pipeline only acts on new arrivals rather than reprocessing the full raw history on every run.

3. **`Loop_through_all_unprocessed_files` (ForEach)** — iterates over the unprocessed folder/file list returned by the lookup. For each item:
   - **`Copy data1`** — copies the file into its corresponding Bronze Delta table.
   - **`Wait1`** — a short pause between iterations.

4. **On completion** — once the ForEach completes, **`Update_Metadata_Tables`** (Notebook) writes the newly processed folder/file/row-count information back into the metadata and audit control tables, so the next pipeline run knows this batch is done.

5. **On failure** — a failure anywhere in the notebook or the ForEach loop routes to **`Email_Notification`** (Office 365 Email), so a failed ingestion run doesn't fail silently.

## Pipeline Architecture:

![Pipeline Design Diagram](./Bronze%20Olist%20Data%20Pipeline%20-%20Visual.png)


## Design notes

- **Schema drift is intentionally tolerated at this layer.** New source columns aren't expected to fail the pipeline — Bronze isn't yet business-modeled, so there's nothing to protect against a wider schema here.
- **A known limitation, found and documented rather than silently left broken:** the `Copy data1` activity does not reliably widen an *existing* Bronze Delta table's schema when a source file gains a new column — it fails with `SourceColumnIsNotDefinedInDeltaMetadata` instead of adding the column automatically. This was tested directly (see the main project README for the reproduction and root cause). The fix — replacing this Copy Data step with a Notebook activity using the same Spark `mergeSchema`/`autoMerge` pattern already proven at the Silver layer — is scoped out of this iteration but is the clear next step if schema drift needs to be fully supported end-to-end.
- **Every ingestion run is isolated.** Because each run lands in its own timestamped folder, reprocessing or backfilling a specific batch never risks colliding with another run's data.
