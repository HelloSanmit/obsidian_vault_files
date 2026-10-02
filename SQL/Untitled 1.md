
## Phase 3, Lesson 2 — Pipeline Design Principles

---

### Starting with the question I left you with

What happens to your Phase 2 pipeline if it fails halfway through and you run it again?

The pipeline does this in sequence: read the CSV, transform records, write to JSON. If it crashes after writing half the JSON file and you restart it, the `open(filepath, "w")` call overwrites the partial file and starts fresh. That actually works fine in this specific case — but only because the output is a single file opened in write mode.

Now imagine the pipeline instead appends records to a database table. It processes 10,000 records, writes 5,000, crashes, and restarts. The restart processes all 10,000 again and writes another 10,000 to the table. Now the table has 15,000 rows — 5,000 from the first run and 10,000 from the second. The 5,000 that were successfully written in the first run are now duplicated.

Or imagine the pipeline sends a Slack notification when it completes. The crash happens after sending the notification but before writing the output. The restart completes successfully and sends another notification. The person receiving the notification sees two success messages and does not know whether the data was written twice.

These are not hypothetical edge cases. They are the most common production failure modes. Networks time out. Servers restart. Disks fill up. Out-of-memory errors happen. Pipelines crash and get restarted constantly in production environments. A pipeline that produces wrong results when restarted is not a production-ready pipeline.

---

### Idempotency — the most important pipeline design principle

A pipeline is **idempotent** if running it multiple times produces the same result as running it once. The word comes from mathematics — an idempotent operation is one where applying it more than once has the same effect as applying it once. Multiplying by 1 is idempotent. Adding 1 is not.

The practical meaning for data engineering: if your pipeline is scheduled to run at 2am daily and for some reason it runs three times at 2am today — due to a retry mechanism, a deployment issue, or a scheduler bug — the output should be identical to what a single run would have produced. No duplicate records. No partial writes from the first run mixed with complete writes from the second. No missing data.

Idempotency is not a nice-to-have. In production systems with automatic retry logic, non-idempotent pipelines corrupt data silently. The corruption is often discovered days later when someone notices the numbers are wrong and has to trace it back through weeks of pipeline runs.

---

### How to design for idempotency — the mechanisms

There are several concrete patterns for achieving idempotency. Each is suited to different situations.

**Pattern 1 — Overwrite, never append.**

The simplest form of idempotency is to always write the complete output, overwriting anything that was there before. If the pipeline writes `students_2024_01_15.parquet`, every run on January 15th writes to exactly that file, replacing whatever was there. Running it three times produces the same file as running it once.

```python
from pathlib import Path
import json

def write_output(records: list[dict], output_path: str) -> None:
    """
    Always overwrites. Running this twice with the same records
    produces the same file as running it once.
    """
    path = Path(output_path)
    path.parent.mkdir(parents=True, exist_ok=True)
    
    with open(path, "w", encoding="utf-8") as f:
        json.dump(records, f, indent=2)
```

The `"w"` mode overwrites. If the file already exists from a previous run, it is replaced entirely. The second run produces exactly the same file as the first.

The limitation: this only works when the entire output can be recomputed from scratch on every run. For a daily pipeline that processes yesterday's data, that is fine. For a pipeline that accumulates data over months, recomputing everything every run becomes expensive.

**Pattern 2 — Delete then insert.**

When writing to a database, the append problem is that running twice inserts rows twice. The solution is to delete existing rows for the current processing window before inserting new ones.

```python
def load_daily_records(
    records: list[dict],
    date: str,
    db_connection,
) -> None:
    """
    Idempotent database write for a specific date.
    
    First deletes any existing rows for this date,
    then inserts the new rows. Running this twice for
    the same date and same records produces the same
    database state as running it once.
    """
    # Step 1: Remove any existing data for this date
    db_connection.execute(
        "DELETE FROM student_daily_stats WHERE processing_date = %s",
        (date,)
    )
    
    # Step 2: Insert the new data
    for record in records:
        db_connection.execute(
            """
            INSERT INTO student_daily_stats
            (student_id, date, gpa, city, processing_date)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (record["student_id"], record["date"],
             record["gpa"], record["city"], date)
        )
```

This is called **delete-then-insert** or sometimes **upsert** pattern. The first run deletes nothing (there is no existing data for this date) and inserts the records. The second run deletes the records inserted by the first run and inserts the same records again. The end state is identical.

**Pattern 3 — Upsert (INSERT ON CONFLICT UPDATE).**

A more sophisticated version that handles the case where you want to update existing rows rather than delete and reinsert them.

```python
def upsert_students(records: list[dict], db_connection) -> None:
    """
    Insert student records. If a student_id already exists,
    update their record instead of inserting a duplicate.
    
    This is idempotent because inserting the same student twice
    updates them to the same values rather than creating two rows.
    """
    for record in records:
        db_connection.execute(
            """
            INSERT INTO students (student_id, name, city, gpa)
            VALUES (%s, %s, %s, %s)
            ON CONFLICT (student_id)
            DO UPDATE SET
                name = EXCLUDED.name,
                city = EXCLUDED.city,
                gpa = EXCLUDED.gpa
            """,
            (record["student_id"], record["name"],
             record["city"], record["gpa"])
        )
```

`ON CONFLICT (student_id) DO UPDATE SET` is PostgreSQL's upsert syntax. If the `INSERT` would violate the unique constraint on `student_id`, it instead runs the `UPDATE`. `EXCLUDED` refers to the row that was attempted to be inserted. Running this with the same records twice produces the same final state as running it once.

**Pattern 4 — Partition-based overwrite.**

For large datasets in Parquet, the natural idempotency mechanism is partitioned writes where you overwrite a specific partition.

```python
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

def write_daily_partition(
    records: list[dict],
    date: str,
    output_dir: str,
) -> None:
    """
    Write records for a specific date to their own partition.
    
    The partition path encodes the date: output_dir/date=2024-01-15/
    Running this twice for the same date overwrites the same partition.
    Running it for different dates creates different partitions.
    This makes the entire pipeline idempotent -- reprocessing any date
    only affects that date's partition.
    """
    partition_path = Path(output_dir) / f"date={date}"
    partition_path.mkdir(parents=True, exist_ok=True)
    
    # Always write to the same filename within the partition
    # Overwrites any previous run for this date
    output_file = partition_path / "part-0.parquet"
    
    table = pa.table({
        "student_id": [r["student_id"] for r in records],
        "name": [r["name"] for r in records],
        "gpa": [r["gpa"] for r in records],
    })
    
    pq.write_table(table, str(output_file), compression="snappy")
```

The directory structure becomes:

```
output/
  date=2024-01-13/
    part-0.parquet
  date=2024-01-14/
    part-0.parquet
  date=2024-01-15/
    part-0.parquet
```

Reprocessing January 15th writes to `date=2024-01-15/part-0.parquet`, overwriting the previous run. January 13th and 14th are untouched. This is the standard pattern used in production data lakes.

---

### Backfilling — what it is and why idempotency makes it possible

**Backfilling** is the process of reprocessing historical data. You need it when:

Your pipeline was broken for a week and produced wrong data — you need to reprocess those seven days.

You changed the transformation logic — the new logic should apply to historical data too, not just future runs.

A new column was added to the source system retroactively — you need to populate it for historical records.

Backfilling is only safe if your pipeline is idempotent. If reprocessing January 13th duplicates or corrupts January 13th's data, backfilling is dangerous and cannot be done easily. If reprocessing January 13th simply overwrites January 13th's partition with freshly computed data, backfilling is safe and trivial — you just re-run the pipeline for each historical date.

```python
from datetime import date, timedelta

def backfill(
    start_date: date,
    end_date: date,
    pipeline_fn,
    source_fn,
    output_dir: str,
) -> dict:
    """
    Re-run the pipeline for every date in the specified range.
    
    Only safe because the pipeline is idempotent -- rerunning a date
    produces the same output as the original run, not a duplicate.
    """
    results = {}
    current = start_date
    
    while current <= end_date:
        date_str = current.strftime("%Y-%m-%d")
        print(f"Backfilling {date_str}...")
        
        try:
            records = source_fn(date_str)
            pipeline_fn(records, date_str, output_dir)
            results[date_str] = "success"
        except Exception as e:
            results[date_str] = f"failed: {e}"
            print(f"  Failed: {e}")
        
        current += timedelta(days=1)
    
    return results
```

`date.strftime("%Y-%m-%d")` formats a `date` object as a string like `"2024-01-15"`. `%Y` is the 4-digit year, `%m` is the 2-digit month, `%d` is the 2-digit day. The `strftime` method (string format time) is available on both `date` and `datetime` objects and uses the same format codes.

`timedelta(days=1)` represents a duration of one day. Adding it to a `date` gives the next day's `date`. This is how you iterate through date ranges in Python — `timedelta` is the standard tool.

---

### Late-arriving data — a problem idempotency alone does not solve

Idempotency handles the case where you re-run a pipeline that already ran. Late-arriving data is a different problem: data that belongs to a past time window arrives after that window's pipeline has already run.

Consider a pipeline that processes student attendance records daily. The attendance system records when students swipe their card to enter the building. On January 15th, the network was slow and 200 swipe records from January 14th were not transmitted until the morning of January 15th. The January 14th pipeline already ran at 2am and missed these records.

Without handling late-arriving data, those 200 records are simply lost from January 14th's attendance report. The report is permanently wrong.

There are two approaches.

**Approach 1 — Watermark with grace period.** Instead of processing data for exactly the previous day, the pipeline processes data for the previous day plus a grace period. The January 15th pipeline processes all records with a `created_at` timestamp between January 13th 00:00 and January 14th 23:59. Because the pipeline is idempotent (delete-then-insert or partition overwrite), reprocessing the previous day is safe. Late records from January 14th that arrived on January 15th get included.

```python
from datetime import datetime, timedelta

def get_processing_window(
    processing_date: datetime,
    grace_period_days: int = 1,
) -> tuple[datetime, datetime]:
    """
    Return the time window to process for a given date.
    
    The window starts grace_period_days before processing_date
    to capture late-arriving records.
    
    For processing_date=2024-01-15, grace_period_days=1:
    Returns (2024-01-14 00:00:00, 2024-01-14 23:59:59)
    The January 15th pipeline reprocesses January 14th's full day.
    """
    end = processing_date - timedelta(days=1)
    end = end.replace(hour=23, minute=59, second=59)
    start = end.replace(hour=0, minute=0, second=0)
    return start, end
```

`datetime.replace(hour=0, minute=0, second=0)` returns a new `datetime` with the specified fields replaced. It does not modify the original. `replace` is how you set specific time components of a `datetime` while keeping the date the same.

**Approach 2 — Append-only with reprocessing triggers.** Some systems track which time windows have been updated since the last pipeline run. If records for January 14th are modified after the January 14th pipeline ran, this triggers a reprocessing of January 14th's data.

Which approach to use depends on how late data typically arrives, how critical accuracy is, and how expensive reprocessing is. For most data engineering pipelines, the grace period approach is simpler and sufficient.

---

### Incremental loading — only processing new data

Running a full pipeline on the entire historical dataset every day is wasteful when only a small fraction of data has changed. **Incremental loading** means processing only the data that is new or changed since the last successful run.

The implementation requires tracking where the last run ended. The most common mechanism is a **watermark** — a value (usually a timestamp or an ID) that marks the boundary between data that has been processed and data that has not.

```python
import json
from pathlib import Path
from datetime import datetime


def read_watermark(watermark_file: str) -> datetime | None:
    """
    Read the last successful processing timestamp.
    Returns None if no previous run has completed.
    """
    path = Path(watermark_file)
    if not path.exists():
        return None
    
    with open(path, "r", encoding="utf-8") as f:
        data = json.load(f)
    
    return datetime.fromisoformat(data["last_processed_at"])


def write_watermark(watermark_file: str, processed_at: datetime) -> None:
    """
    Record the timestamp of the last successful run.
    Called only after the pipeline completes successfully.
    """
    path = Path(watermark_file)
    path.parent.mkdir(parents=True, exist_ok=True)
    
    with open(path, "w", encoding="utf-8") as f:
        json.dump({"last_processed_at": processed_at.isoformat()}, f)


def extract_incremental(
    all_records: list[dict],
    watermark: datetime | None,
) -> list[dict]:
    """
    Return only records created after the watermark.
    If watermark is None (first run), return all records.
    """
    if watermark is None:
        return all_records
    
    return [
        record for record in all_records
        if datetime.fromisoformat(record["created_at"]) > watermark
    ]


def run_incremental_pipeline(
    source_records: list[dict],
    output_dir: str,
    watermark_file: str,
) -> int:
    """
    Run the pipeline incrementally -- only process new records.
    Updates the watermark only on success.
    """
    watermark = read_watermark(watermark_file)
    print(f"Last processed at: {watermark or 'never (first run)'}")
    
    new_records = extract_incremental(source_records, watermark)
    print(f"New records to process: {len(new_records)}")
    
    if not new_records:
        print("No new records. Nothing to do.")
        return 0
    
    # Process and write new records
    # ... transformation and loading logic ...
    
    # Only update watermark after successful completion
    # If we crash here, the watermark is not updated and
    # the next run will reprocess these same records
    # This ensures we never skip records, only potentially
    # process them twice (which idempotency handles)
    run_completed_at = datetime.now()
    write_watermark(watermark_file, run_completed_at)
    
    return len(new_records)
```

`datetime.fromisoformat(string)` parses an ISO 8601 datetime string like `"2024-01-15T14:30:00"` into a `datetime` object. It is the inverse of `.isoformat()`. Available from Python 3.7.

The critical design decision: the watermark is written **after** successful completion, not before. If the pipeline crashes during processing, the watermark is not updated. The next run re-reads the same watermark and reprocesses the same data. Combined with idempotent writes, this guarantees that records are never permanently lost — they are simply processed more than once, which is safe.

---

### The pipeline contract — what every well-designed pipeline guarantees

These principles together define what a well-designed pipeline commits to:

**Completeness.** Every record from the source appears in the output. No records are silently lost due to pipeline failures.

**Idempotency.** Running the pipeline multiple times produces the same result as running it once. Retries and backfills are safe.

**Correctness.** The output accurately reflects the transformation logic applied to the input. Bad input records are explicitly logged, not silently dropped or corrupted.

**Observability.** Every run produces a record of what happened — how many records were processed, how many failed, how long it took, and what errors occurred.

**Recoverability.** When the pipeline fails, it fails cleanly. Partial state does not corrupt existing output. The next run can pick up where the previous one left off.

This is the contract your Phase 2 pipeline partially satisfies. The `PipelineRun` dataclass provides observability. The JSON output uses `"w"` mode which provides simple idempotency. But it has no watermark, no incremental loading, and no partition-based organization. The remaining lessons in Phase 3 build the missing pieces.

---

### Immutability — designing data that does not change

There is a deeper principle underlying idempotency that is worth naming explicitly: **immutability**. In well-designed data pipelines, data is never updated in place. Instead, new versions replace old versions.

Instead of updating a record's GPA from 8.5 to 8.7, you write a new version of the record with the updated GPA. The old version is either replaced entirely or kept as a historical record depending on your requirements.

Why does this matter? Because updates in the middle of a dataset are the primary cause of idempotency problems and data consistency bugs. If you update individual rows, a failed pipeline may have updated some rows but not others, leaving the dataset in an inconsistent half-updated state. Detecting and recovering from this is difficult.

If instead you write the entire output for a time window from scratch (overwriting the previous output), a failed pipeline simply leaves the previous complete output intact. The next run overwrites it again. The dataset is always either the previous complete output or the new complete output — never a mixture.

This immutable, append-only design is why columnar storage formats like Parquet work so well for data engineering. You write a complete Parquet file for January 15th. If it needs to be corrected, you write a new complete Parquet file for January 15th that replaces the old one. You never edit the file in place.

---

### Lesson 2 complete

The concepts in this lesson are not tied to specific tools. They are principles. When you use Airflow in Phase 4, you will write DAGs that implement these patterns. When you use Spark in Phase 6, the partitioned write pattern is how Spark outputs data. When you use Kafka in Phase 7, watermarks are how stream processing handles time. Every tool you learn from here on is implementing one or more of these principles.

Tell me when you are ready for Lesson 3.