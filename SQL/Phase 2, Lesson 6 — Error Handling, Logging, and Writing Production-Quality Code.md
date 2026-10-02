
## Phase 2, Lesson 6 — Error Handling, Logging, and Writing Production-Quality Code

---

### Why this lesson is where most data engineers fail

Everything in the previous five lessons has been about making things work. This lesson is about making things work **reliably, observably, and recoverably** — the three properties that distinguish code running on your laptop from code running unattended in a production pipeline at 3am.

Consider what a production data pipeline actually does. It runs automatically on a schedule. Nobody is watching it. It processes data from external sources that can change their format without warning. It talks to networks that can fail. It writes to databases that can be unavailable. It runs for hours on datasets with billions of rows. And when something goes wrong — and something always goes wrong — the on-call engineer needs to figure out exactly what failed, where, and why, from a log file that is the only evidence of what happened.

Code that handles this well has three properties.

**Reliability** — the pipeline recovers from expected failures without crashing. Network timeouts, missing files, malformed records — these are predictable problems. Good code anticipates them, handles them specifically, and continues or stops gracefully.

**Observability** — when something does go wrong, there is a complete record of what happened. What data was processed, what decisions were made, what failed and why, at what time. This record is produced by logging, and it is the primary debugging tool for pipelines that run unattended.

**Recoverability** — failures are isolated. A bad record in one batch does not abort the entire pipeline. A failed API call does not corrupt the partial output already written. The pipeline can be safely rerun from the point of failure.

Without all three, a pipeline is not production-ready regardless of how well it processes data when everything is working.

---

### Python's exception hierarchy — understanding the structure before handling it

You have used try-except already. But to write good error handling, you need to understand the full exception hierarchy rather than just catching `Exception` everywhere.

Every Python exception is a class that inherits from `BaseException`. The hierarchy matters because catching a parent class catches all child classes:

```
BaseException
├── SystemExit              -- raised by sys.exit() -- not an error
├── KeyboardInterrupt       -- Ctrl+C -- not an error
├── GeneratorExit           -- generator cleanup -- not an error
└── Exception               -- all actual errors inherit from here
    ├── AttributeError      -- accessing nonexistent attribute
    ├── ImportError         -- failed import
    │   └── ModuleNotFoundError
    ├── LookupError
    │   ├── IndexError      -- list index out of range
    │   └── KeyError        -- dict key not found
    ├── NameError           -- undefined variable
    │   └── UnboundLocalError
    ├── OSError             -- operating system errors
    │   ├── FileNotFoundError
    │   ├── PermissionError
    │   └── TimeoutError
    ├── TypeError           -- wrong type
    ├── ValueError          -- right type, wrong value
    ├── RuntimeError        -- general runtime error
    ├── StopIteration       -- iterator exhausted
    └── ArithmeticError
        ├── ZeroDivisionError
        └── OverflowError
```

The three exceptions at the top — `SystemExit`, `KeyboardInterrupt`, `GeneratorExit` — do not inherit from `Exception`. This is intentional. When you write `except Exception`, you do not accidentally catch a `Ctrl+C` or a `sys.exit()`. This is the correct behavior — those are not errors, they are signals that the program should stop.

The practical implication: **always catch `Exception`, not `BaseException`**. Catching `BaseException` intercepts keyboard interrupts and exit signals, which can make your program unresponsive to Ctrl+C and impossible to stop cleanly.

```python
# Wrong -- catches Ctrl+C and sys.exit()
try:
    run_pipeline()
except BaseException:
    pass

# Correct -- leaves Ctrl+C and sys.exit() to propagate
try:
    run_pipeline()
except Exception as e:
    handle_error(e)
```

---

### The most important exception handling rule — specificity

The width of what you catch should match the width of what you can handle.

```python
def read_student_file(filepath: str) -> list[dict]:
    """Bad version -- catches everything, masks real bugs."""
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            import csv
            return list(csv.DictReader(f))
    except Exception:
        return []
```

This looks safe. It is dangerous. If there is a bug in your code — a typo, an import error, a wrong attribute access — this `except Exception` catches it silently. The function returns an empty list. The pipeline continues with zero records. The bug is invisible.

```python
def read_student_file(filepath: str) -> list[dict]:
    """Good version -- handles what it can, lets bugs propagate."""
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            import csv
            return list(csv.DictReader(f))
    except FileNotFoundError:
        # Expected -- file might not exist yet on first run
        print(f"Source file not found: {filepath}. Returning empty list.")
        return []
    except PermissionError:
        # Expected -- permissions issue is a handleable deployment problem
        raise PermissionError(
            f"No read permission for {filepath}. "
            f"Check file permissions and the service account."
        ) from None
    # csv.Error, UnicodeDecodeError, and anything else propagates naturally
    # because they indicate either corrupt data or bugs -- both need to be seen
```

This version handles `FileNotFoundError` with a specific, correct response — returning an empty list is appropriate if the file simply does not exist yet. It handles `PermissionError` by raising it with a more informative message. Everything else — encoding errors, CSV parsing failures, bugs — propagates naturally.

The rule crystallized: **catch the most specific exception you can, only when you know exactly what to do with it. Everything else should propagate.**

---

### Exception chaining — preserving the cause

When you catch one exception and raise another, Python allows you to explicitly chain them, preserving the original as context.

```python
def parse_gpa(value: str) -> float:
    """Convert a string GPA value to float."""
    try:
        return float(value)
    except ValueError as e:
        raise ValueError(f"Cannot parse GPA from '{value}': expected a number") from e
```

The `from e` chains the new `ValueError` to the original `ValueError`. When this exception is printed, Python shows both:

```
ValueError: invalid literal for float(): [value='abc']

The above exception was the direct cause of the following exception:

ValueError: Cannot parse GPA from 'abc': expected a number
```

The full chain is preserved in the traceback, giving debugging context about both what went wrong at the low level and what it meant at the higher level.

`from None` is the other option — it suppresses the original exception entirely, showing only the new one. Use this when the original exception contains implementation details that the caller should not see:

```python
def connect_to_database(host: str, database: str):
    try:
        return internal_library.connect(host, database)
    except internal_library.InternalConnectionError as e:
        raise RuntimeError(
            f"Failed to connect to {database} at {host}. "
            f"Check that the database is running and credentials are correct."
        ) from None
        # 'from None' hides the internal library's exception details
        # which would be confusing and expose implementation internals
```

---

### Custom exceptions — defining your own error vocabulary

A data engineering codebase should have its own exception hierarchy. This gives you precise, meaningful error types that callers can handle specifically.

```python
class DataEngineeringError(Exception):
    """Base class for all data engineering errors in this project."""
    pass


class ExtractionError(DataEngineeringError):
    """
    Raised when data cannot be extracted from a source.
    Includes information about what was being extracted.
    """
    def __init__(self, source: str, reason: str, records_extracted: int = 0):
        self.source = source
        self.reason = reason
        self.records_extracted = records_extracted
        super().__init__(
            f"Extraction failed from '{source}': {reason}. "
            f"Records extracted before failure: {records_extracted}"
        )


class TransformationError(DataEngineeringError):
    """Raised when a record fails transformation."""
    def __init__(self, record: dict, step: str, reason: str):
        self.record = record
        self.step = step
        self.reason = reason
        super().__init__(
            f"Transformation failed at step '{step}': {reason}. "
            f"Record: {record}"
        )


class LoadError(DataEngineeringError):
    """Raised when data cannot be written to the destination."""
    def __init__(self, destination: str, reason: str, records_attempted: int = 0):
        self.destination = destination
        self.reason = reason
        self.records_attempted = records_attempted
        super().__init__(
            f"Load failed to '{destination}': {reason}. "
            f"Records attempted: {records_attempted}"
        )


class SchemaValidationError(DataEngineeringError):
    """Raised when data does not conform to the expected schema."""
    def __init__(self, field: str, expected_type: type, actual_value):
        self.field = field
        self.expected_type = expected_type
        self.actual_value = actual_value
        super().__init__(
            f"Schema validation failed for field '{field}': "
            f"expected {expected_type.__name__}, "
            f"got {type(actual_value).__name__} ({actual_value!r})"
        )
```

Now in your pipeline code:

```python
def extract(source_path: str) -> list[dict]:
    if not Path(source_path).exists():
        raise ExtractionError(
            source=source_path,
            reason="Source file does not exist",
        )
    # ...

# Caller can handle specifically
try:
    records = extract("data/students.csv")
except ExtractionError as e:
    # handle extraction failure -- maybe retry, maybe alert
    print(f"Extraction from {e.source} failed: {e.reason}")
    print(f"Got {e.records_extracted} records before failure")
except DataEngineeringError as e:
    # handle any other DE error
    print(f"Pipeline error: {e}")
except Exception as e:
    # handle unexpected errors -- these are bugs
    print(f"Unexpected error: {e}")
    raise   # re-raise bugs so they are visible
```

This is far more useful than catching generic `Exception` everywhere. The caller knows exactly what kind of failure occurred and has access to structured information about it — not just a string error message.

---

### Python's logging module — the right way to record what your pipeline does

`print()` is for development. `logging` is for production. The difference is fundamental.

`print()` always outputs to stdout. It has no severity levels. You cannot silence it without removing the calls. It has no timestamps. It provides no way to route different messages to different destinations. In a pipeline running as a background process, print output is often lost entirely.

The `logging` module fixes all of this. It provides:

**Severity levels** — five levels in ascending order of severity. Only messages at or above the configured level are emitted:

```
DEBUG    -- detailed diagnostic information (variable values, step entry/exit)
INFO     -- normal operational events (pipeline started, 1000 records processed)
WARNING  -- something unexpected but not fatal (missing optional field, slow response)
ERROR    -- a failure that affects correctness (record failed, file not found)
CRITICAL -- a failure that stops the pipeline entirely (database unreachable)
```

**Structured output** — each log record includes the timestamp, severity level, logger name, and message. This makes log files parseable by log management systems.

**Handlers** — log records can be sent to multiple destinations simultaneously: a file, the console, a remote logging service.

**Logger hierarchy** — loggers are organized in a tree by name. A logger named `"pipeline.extract"` is a child of `"pipeline"`. Configuration applied to the parent applies to all children.

```python
import logging
from pathlib import Path

def setup_logging(
    log_file: str = None,
    level: int = logging.INFO,
    log_format: str = None,
) -> logging.Logger:
    """
    Configure logging for a data pipeline.
    
    Sets up a root logger that writes to the console,
    and optionally also to a file.
    
    Args:
        log_file: Optional path to write logs to file
        level: Minimum severity level to log
        log_format: Optional custom format string
    
    Returns:
        Configured logger for this pipeline
    """
    if log_format is None:
        log_format = (
            "%(asctime)s | %(levelname)-8s | %(name)s | %(message)s"
        )
    
    formatter = logging.Formatter(
        fmt=log_format,
        datefmt="%Y-%m-%d %H:%M:%S",
    )
    
    # Get the root logger and set its level
    root_logger = logging.getLogger()
    root_logger.setLevel(level)
    
    # Remove any existing handlers (avoid duplicate output on re-runs)
    root_logger.handlers.clear()
    
    # Console handler -- always present
    console_handler = logging.StreamHandler()
    console_handler.setFormatter(formatter)
    root_logger.addHandler(console_handler)
    
    # File handler -- optional
    if log_file:
        Path(log_file).parent.mkdir(parents=True, exist_ok=True)
        file_handler = logging.FileHandler(log_file, encoding="utf-8")
        file_handler.setFormatter(formatter)
        root_logger.addHandler(file_handler)
    
    return logging.getLogger("pipeline")


logger = setup_logging(log_file="data/logs/pipeline.log")

logger.debug("This only appears when level is set to DEBUG")
logger.info("Pipeline started")
logger.warning("Source file is older than 24 hours")
logger.error("Failed to parse record: %s", {"student_id": "abc"})
logger.critical("Database connection lost. Cannot continue.")
```

Run this and look at the output on the console and in `data/logs/pipeline.log`. Every line has the same structure: timestamp, level, logger name, message.

**The format string `%(asctime)s | %(levelname)-8s | %(name)s | %(message)s`:**

`%(asctime)s` — the formatted timestamp from `datefmt`.

`%(levelname)-8s` — the level name, left-aligned in a field of width 8. The `-8s` means left-align in 8 characters. This makes log levels column-aligned: `INFO` , `WARNING` , `ERROR` . Aligned columns make log files much easier to scan.

`%(name)s` — the logger's name. When you create a logger with `logging.getLogger("pipeline.extract")`, the name appears here, telling you which part of the code emitted this log.

`%(message)s` — the log message.

---

### Logger naming — how to structure logging in a pipeline

Each module or component should have its own named logger. This lets you configure verbosity per component and identify where each log message came from.

```python
import logging

# At the top of each module or class file:
logger = logging.getLogger(__name__)

# __name__ is a Python built-in that evaluates to the current module's name
# In a file called 'pipeline/extract.py', __name__ is 'pipeline.extract'
# This creates a logger named 'pipeline.extract'
```

`__name__` is a special Python variable that evaluates to the current module's fully qualified name. In a file called `pipeline/extract.py`, `__name__` is `"pipeline.extract"`. In the main script, `__name__` is `"__main__"`.

Using `__name__` as the logger name is the standard convention. It means every log message automatically carries the exact file and module it came from, without you having to hardcode anything.

```python
import logging

logger = logging.getLogger(__name__)

class DataExtractor:
    
    def __init__(self, source_path: str):
        self.source_path = source_path
        self.logger = logging.getLogger(f"{__name__}.DataExtractor")
    
    def extract(self) -> list[dict]:
        self.logger.info("Starting extraction from %s", self.source_path)
        
        records = []
        errors = 0
        
        try:
            with open(self.source_path, "r", encoding="utf-8") as f:
                import csv
                reader = csv.DictReader(f)
                for line_num, row in enumerate(reader, start=2):
                    try:
                        records.append(self._parse_row(row))
                    except ValueError as e:
                        errors += 1
                        self.logger.warning(
                            "Line %d: skipping malformed row: %s", line_num, e
                        )
            
            self.logger.info(
                "Extraction complete: %d records, %d errors", len(records), errors
            )
            return records
        
        except FileNotFoundError:
            self.logger.error("Source file not found: %s", self.source_path)
            raise
    
    def _parse_row(self, row: dict) -> dict:
        gpa_str = row.get("gpa", "").strip()
        return {
            "student_id": int(row["student_id"]),
            "name": row["name"].strip(),
            "gpa": float(gpa_str) if gpa_str and gpa_str.lower() != "none" else None,
        }
```

**String formatting in log messages — an important performance note:**

```python
# Wrong -- formats the string even if the message is never emitted
logger.debug(f"Processing record: {record}")

# Correct -- formatting only happens if DEBUG level is active
logger.debug("Processing record: %s", record)
```

The `%s` style formatting is not just stylistic — it is a performance optimization. The logger only formats the message string if that level is actually going to be emitted. With f-strings, the string is formatted immediately regardless of whether the log level is active. In a loop processing millions of records at INFO level, DEBUG messages with f-strings would format millions of strings that are immediately discarded.

Always use `%s`, `%d`, `%f` placeholders with additional arguments for log messages.

---

### The `finally` block — guaranteed cleanup

`try-except` catches errors. Sometimes you need code that runs regardless of whether an error occurred — to close a connection, release a lock, or write summary statistics. That is `finally`.

```python
def run_pipeline_step(step_name: str, records: list[dict]) -> dict:
    """
    Run a pipeline step and always record metrics, even if it fails.
    """
    start_time = time.time()
    result = {"step": step_name, "status": "unknown", "records_in": len(records)}
    
    try:
        processed = transform(records)
        result["records_out"] = len(processed)
        result["status"] = "success"
        return processed
    
    except TransformationError as e:
        result["status"] = "failed"
        result["error"] = str(e)
        logger.error("Step %s failed: %s", step_name, e)
        raise
    
    except Exception as e:
        result["status"] = "error"
        result["error"] = str(e)
        logger.exception("Unexpected error in step %s", step_name)
        raise
    
    finally:
        # This runs no matter what -- success, handled error, or unhandled error
        result["duration_seconds"] = round(time.time() - start_time, 3)
        logger.info(
            "Step %s: %s in %.3fs",
            step_name, result["status"], result["duration_seconds"]
        )
```

`finally` runs after the `try` block completes (normally or via exception) and after any matching `except` block completes. If the `except` block re-raises, `finally` still runs before the exception propagates to the caller.

This makes `finally` the right place for:

- Closing database connections and file handles (though `with` is usually better for this)
- Writing metrics and statistics that should always be recorded
- Releasing locks
- Logging completion regardless of outcome

---

### `logger.exception` — capturing tracebacks

```python
try:
    process_record(record)
except Exception as e:
    logger.exception("Failed to process record %s", record["student_id"])
    # DO NOT do: logger.error("Failed: %s", e) -- this loses the traceback
```

`logger.exception()` is identical to `logger.error()` except it automatically appends the current exception's traceback to the log message. The traceback shows exactly which line raised the exception and the full call chain.

`logger.error("Failed: %s", e)` only logs the exception message — `"invalid literal for int() with base 10: 'abc'"`. It tells you what went wrong but not where. Without the traceback, debugging a production failure can take hours.

Always use `logger.exception()` inside `except` blocks, not `logger.error()`.

---

### Structuring pipeline code for reliability — the three-layer pattern

Production data pipelines are structured in a specific way that makes them reliable and debuggable. Understanding this pattern is one of the most practically valuable things in this lesson.

A well-structured pipeline has three layers:

**The operation layer** — individual functions that do exactly one thing. Extract data from one source. Transform one batch. Write to one destination. These functions are small, focused, and testable. They raise exceptions on failure — they do not handle pipeline-level concerns.

**The step layer** — functions that wrap operations, handle step-level errors, collect metrics, and log progress. They decide whether a step failure is fatal or recoverable.

**The orchestration layer** — the function that runs steps in sequence, manages overall pipeline state, and produces the final run report. It handles step failures in the context of the full pipeline.

```python
import logging
import time
from dataclasses import dataclass, field
from pathlib import Path
from datetime import datetime
import csv
import json

logger = logging.getLogger(__name__)


# ── Custom exceptions ─────────────────────────────────────────────

class PipelineError(Exception):
    pass

class ExtractionError(PipelineError):
    def __init__(self, source: str, reason: str):
        self.source = source
        super().__init__(f"Extraction failed from '{source}': {reason}")

class LoadError(PipelineError):
    def __init__(self, destination: str, reason: str):
        self.destination = destination
        super().__init__(f"Load failed to '{destination}': {reason}")


# ── Metrics tracking ──────────────────────────────────────────────

@dataclass
class StepMetrics:
    step_name: str
    start_time: datetime = field(default_factory=datetime.now)
    end_time: datetime = None
    records_in: int = 0
    records_out: int = 0
    records_failed: int = 0
    status: str = "pending"
    error: str = None

    def complete(self, status: str = "success", error: str = None) -> None:
        self.end_time = datetime.now()
        self.status = status
        self.error = error

    @property
    def duration_seconds(self) -> float:
        if self.end_time is None:
            return (datetime.now() - self.start_time).total_seconds()
        return (self.end_time - self.start_time).total_seconds()

    def to_dict(self) -> dict:
        return {
            "step": self.step_name,
            "status": self.status,
            "records_in": self.records_in,
            "records_out": self.records_out,
            "records_failed": self.records_failed,
            "duration_seconds": round(self.duration_seconds, 3),
            "error": self.error,
        }


# ── Operation layer ────────────────────────────────────────────────

def read_csv(filepath: str) -> list[dict]:
    """Read a CSV file and return raw rows. Raises ExtractionError on failure."""
    path = Path(filepath)
    if not path.exists():
        raise ExtractionError(source=filepath, reason="File not found")
    
    with open(path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        if reader.fieldnames is None:
            raise ExtractionError(source=filepath, reason="Empty file or no header")
        return [dict(row) for row in reader]


def parse_record(raw: dict) -> dict:
    """Parse and validate one raw CSV row. Raises ValueError on bad data."""
    gpa_str = raw.get("gpa", "").strip().lower()
    gpa = None
    if gpa_str not in ("", "none", "null"):
        gpa = float(gpa_str)
        if not (0.0 <= gpa <= 10.0):
            raise ValueError(f"GPA {gpa} is outside valid range [0.0, 10.0]")
    
    return {
        "student_id": int(raw["student_id"]),
        "name": " ".join(raw["name"].split()).title(),
        "city": raw.get("city", "").strip().title(),
        "gpa": gpa,
    }


def write_json(records: list[dict], filepath: str) -> None:
    """Write records to a JSON file. Raises LoadError on failure."""
    path = Path(filepath)
    try:
        path.parent.mkdir(parents=True, exist_ok=True)
        with open(path, "w", encoding="utf-8") as f:
            json.dump(records, f, indent=2)
    except OSError as e:
        raise LoadError(destination=filepath, reason=str(e)) from e


# ── Step layer ─────────────────────────────────────────────────────

def step_extract(source_path: str) -> tuple[list[dict], StepMetrics]:
    """Extract step with metrics and logging."""
    metrics = StepMetrics(step_name="extract")
    
    logger.info("Step EXTRACT: reading from %s", source_path)
    
    try:
        raw_records = read_csv(source_path)
        metrics.records_in = len(raw_records)
        metrics.records_out = len(raw_records)
        metrics.complete("success")
        logger.info("Step EXTRACT: %d records read", len(raw_records))
        return raw_records, metrics
    
    except ExtractionError:
        metrics.complete("failed", error="Extraction error")
        logger.exception("Step EXTRACT failed")
        raise


def step_transform(raw_records: list[dict]) -> tuple[list[dict], StepMetrics]:
    """Transform step -- invalid records are skipped, not fatal."""
    metrics = StepMetrics(step_name="transform", records_in=len(raw_records))
    
    logger.info("Step TRANSFORM: processing %d records", len(raw_records))
    
    valid = []
    for i, raw in enumerate(raw_records):
        try:
            valid.append(parse_record(raw))
        except (ValueError, KeyError) as e:
            metrics.records_failed += 1
            logger.warning(
                "Step TRANSFORM: skipping row %d: %s — raw: %s", i + 2, e, raw
            )
    
    metrics.records_out = len(valid)
    metrics.complete("success")
    logger.info(
        "Step TRANSFORM: %d valid, %d failed",
        metrics.records_out, metrics.records_failed
    )
    return valid, metrics


def step_load(records: list[dict], output_path: str) -> StepMetrics:
    """Load step with metrics and logging."""
    metrics = StepMetrics(step_name="load", records_in=len(records))
    
    logger.info("Step LOAD: writing %d records to %s", len(records), output_path)
    
    try:
        write_json(records, output_path)
        metrics.records_out = len(records)
        metrics.complete("success")
        logger.info("Step LOAD: complete")
        return metrics
    
    except LoadError:
        metrics.complete("failed", error="Load error")
        logger.exception("Step LOAD failed")
        raise


# ── Orchestration layer ────────────────────────────────────────────

def run_pipeline(source_path: str, output_path: str) -> dict:
    """
    Orchestrate the full ETL pipeline.
    
    Returns a run report dictionary with step metrics and overall status.
    Always returns a report -- never raises. Failures are captured in the report.
    """
    run_start = datetime.now()
    step_metrics = []
    overall_status = "success"
    
    logger.info("=" * 60)
    logger.info("Pipeline starting at %s", run_start.isoformat())
    logger.info("Source: %s", source_path)
    logger.info("Output: %s", output_path)
    logger.info("=" * 60)
    
    try:
        raw_records, extract_metrics = step_extract(source_path)
        step_metrics.append(extract_metrics.to_dict())
        
        valid_records, transform_metrics = step_transform(raw_records)
        step_metrics.append(transform_metrics.to_dict())
        
        if not valid_records:
            logger.warning("No valid records after transformation. Skipping load.")
            overall_status = "warning"
        else:
            load_metrics = step_load(valid_records, output_path)
            step_metrics.append(load_metrics.to_dict())
    
    except ExtractionError as e:
        overall_status = "failed"
        logger.error("Pipeline failed at EXTRACT: %s", e)
    
    except LoadError as e:
        overall_status = "failed"
        logger.error("Pipeline failed at LOAD: %s", e)
    
    except Exception as e:
        overall_status = "error"
        logger.exception("Pipeline failed with unexpected error")
    
    finally:
        run_end = datetime.now()
        duration = (run_end - run_start).total_seconds()
        
        report = {
            "pipeline": "student_etl",
            "run_start": run_start.isoformat(),
            "run_end": run_end.isoformat(),
            "duration_seconds": round(duration, 3),
            "overall_status": overall_status,
            "steps": step_metrics,
        }
        
        logger.info("=" * 60)
        logger.info("Pipeline %s in %.3fs", overall_status.upper(), duration)
        logger.info("=" * 60)
        
        return report


# ── Main ───────────────────────────────────────────────────────────

if __name__ == "__main__":
    import logging as _logging
    
    _logging.basicConfig(
        level=_logging.INFO,
        format="%(asctime)s | %(levelname)-8s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S",
        handlers=[
            _logging.StreamHandler(),
            _logging.FileHandler("data/logs/pipeline.log", encoding="utf-8"),
        ],
    )
    
    Path("data/logs").mkdir(parents=True, exist_ok=True)
    
    report = run_pipeline(
        source_path="data/students.csv",
        output_path="data/output/students_clean.json",
    )
    
    print("\nRun report:")
    print(json.dumps(report, indent=2))
```

Run this. Read the log output carefully. Then deliberately break it — change the source_path to a file that does not exist. Run it again. Observe how the error is handled, logged, and reflected in the report without crashing.

Then introduce a bad row into `data/students.csv` — change one student_id to "abc". Run again. Observe that the bad row is skipped with a warning while the rest of the pipeline completes successfully.

This is the behavior a production pipeline must have.

---

### The `if __name__ == "__main__"` guard — what it is and why it exists

You saw this in the pipeline above. Every Python file can be either run directly or imported as a module by another file.

When Python runs a file directly, it sets `__name__` to `"__main__"`. When a file is imported, `__name__` is set to the module's name (like `"pipeline"` or `"pipeline.extract"`).

```python
# In file: my_module.py

def useful_function():
    return "useful result"

# This runs when the file is run directly (python my_module.py)
# but NOT when it is imported (import my_module)
if __name__ == "__main__":
    result = useful_function()
    print(result)
```

Without the guard, any code at the module's top level runs both when the file is executed directly AND when it is imported. If that code starts a pipeline, makes network calls, or has side effects, importing the module for its functions triggers all of that unexpectedly.

In data engineering, modules define functions and classes that are imported by other modules. The `if __name__ == "__main__"` guard contains the code that runs the module as a script. This keeps imports side-effect-free.

---

### New built-ins and concepts introduced in this lesson

**`logging.getLogger(name)`** — returns a Logger object. Called with the same `name` string, it always returns the same Logger object — loggers are singletons keyed by name. This means you can call `logging.getLogger("pipeline")` from ten different files and always get the same logger.

**`logging.Formatter`** — formats log records into strings. The `fmt` string uses `%` style placeholders from the `LogRecord` attributes. The `datefmt` formats the timestamp using `strftime` codes: `%Y` is 4-digit year, `%m` is 2-digit month, `%d` is 2-digit day, `%H` is 24-hour hour, `%M` is minute, `%S` is second.

**`logging.StreamHandler()`** — a handler that writes log records to a stream. Without arguments, it defaults to `sys.stderr`. You can pass any writable stream object.

**`logging.FileHandler(filepath)`** — a handler that writes log records to a file. The file is opened in append mode by default — log records are added to the end. Pass `mode="w"` to overwrite on each run.

**`logger.handlers.clear()`** — removes all handlers from a logger. Necessary when reconfiguring logging to avoid duplicate output when running multiple times in the same Python process (common in Jupyter notebooks and testing).

**`%-8s` in format strings** — the `-` means left-align. `8` means minimum field width of 8 characters. If the string is shorter than 8 characters, spaces are added on the right. If it is longer, no truncation occurs. This is standard Python `%` string formatting, the older style that predates f-strings.

**`__name__`** — a built-in module-level variable. When the file is the main script, it is `"__main__"`. When imported, it is the module's dotted path. Used to name loggers (`logging.getLogger(__name__)`) and to guard script code (`if __name__ == "__main__"`).

**`time.time()`** — returns the current time as a float representing seconds since the Unix epoch (January 1, 1970). The difference between two `time.time()` calls gives elapsed time in seconds. More precise than `datetime` arithmetic for measuring small durations.

**`(datetime.now() - earlier_datetime).total_seconds()`** — subtracting two `datetime` objects gives a `timedelta`. `.total_seconds()` converts it to a single float. `timedelta` objects also have `.days`, `.seconds`, and `.microseconds` attributes, but `.total_seconds()` is what you want for elapsed time.

**`logging.basicConfig(...)`** — configures the root logger with sensible defaults in one call. It is simpler than manually creating formatters and handlers. It accepts `level`, `format`, `datefmt`, `filename` (for file output), and `handlers` (a list of pre-built handler objects). It only has an effect the first time it is called in a process — subsequent calls do nothing unless you pass `force=True`.

---

### Exercises — Lesson 6

1. Write a complete exception hierarchy for a data pipeline that has three stages: extract, transform, and load. Each stage should have its own exception class that inherits from a common base. Each exception class should carry structured information relevant to its stage — not just a message string. Write a function for each stage that raises its exception when given specific bad inputs. Then write an orchestrator function that runs all three and handles each exception type differently.
    
2. Write a function `setup_pipeline_logging` that creates a logger with two handlers: a console handler that shows WARNING and above, and a file handler that shows DEBUG and above. Use a custom format that includes the log level, timestamp, logger name, line number, and message. The line number should be the actual line in your Python file where the log call was made (hint: look up `%(lineno)d` in the logging format attributes).
    
3. Write a class `PipelineMetricsCollector` that tracks metrics across all steps of a pipeline run. It should support: `start_step(name)` which begins timing a step, `end_step(name, records_in, records_out, status)` which records completion, `add_error(step, message)` which records an error without ending the step, and `generate_report()` which returns a structured dictionary summarizing the entire run including total duration, total records processed, success rate per step, and all errors. Use `datetime` for timing.
    
4. Write a decorator `pipeline_step` that wraps a function with the following behavior: logs when the function starts and ends, catches any exception and logs it using `logger.exception`, records how long the function took, and returns a `StepResult` dataclass containing `success` (bool), `result` (the return value or None), `error` (the exception or None), and `duration_seconds` (float). The decorator should work on any function regardless of its arguments.
    

---

### Answers — Lesson 6

**Exercise 1**

```python
import logging

logger = logging.getLogger(__name__)


class PipelineError(Exception):
    """Base class for all pipeline errors."""
    pass


class ExtractionError(PipelineError):
    def __init__(self, source: str, reason: str, rows_read: int = 0):
        self.source = source
        self.reason = reason
        self.rows_read = rows_read
        super().__init__(
            f"Extraction from '{source}' failed: {reason} "
            f"(rows read before failure: {rows_read})"
        )


class TransformationError(PipelineError):
    def __init__(self, step_name: str, record: dict, reason: str):
        self.step_name = step_name
        self.record = record
        self.reason = reason
        super().__init__(
            f"Transformation failed at step '{step_name}': {reason} "
            f"— record: {record}"
        )


class LoadError(PipelineError):
    def __init__(self, destination: str, reason: str, rows_attempted: int = 0):
        self.destination = destination
        self.reason = reason
        self.rows_attempted = rows_attempted
        super().__init__(
            f"Load to '{destination}' failed: {reason} "
            f"(rows attempted: {rows_attempted})"
        )


def extract(source_path: str) -> list[dict]:
    if source_path == "bad_path":
        raise ExtractionError(
            source=source_path,
            reason="File not found",
            rows_read=0,
        )
    return [{"student_id": 1, "name": "Arjun", "gpa": "8.5"}]


def transform(records: list[dict]) -> list[dict]:
    result = []
    for record in records:
        try:
            gpa = float(record["gpa"])
        except ValueError:
            raise TransformationError(
                step_name="parse_gpa",
                record=record,
                reason=f"Cannot convert gpa '{record['gpa']}' to float",
            )
        result.append({**record, "gpa": gpa})
    return result


def load(records: list[dict], destination: str) -> None:
    if destination == "bad_destination":
        raise LoadError(
            destination=destination,
            reason="Permission denied",
            rows_attempted=len(records),
        )
    print(f"Loaded {len(records)} records to {destination}")


def orchestrate(source: str, destination: str) -> str:
    try:
        raw = extract(source)
        logger.info("Extracted %d records", len(raw))
        
        transformed = transform(raw)
        logger.info("Transformed %d records", len(transformed))
        
        load(transformed, destination)
        logger.info("Load complete")
        
        return "success"
    
    except ExtractionError as e:
        logger.error("Cannot read from source '%s': %s", e.source, e.reason)
        return f"failed_extract: {e.reason}"
    
    except TransformationError as e:
        logger.warning(
            "Bad record at step '%s': %s — skipping", e.step_name, e.reason
        )
        return f"failed_transform: {e.reason}"
    
    except LoadError as e:
        logger.error(
            "Cannot write to '%s': %s (%d rows lost)",
            e.destination, e.reason, e.rows_attempted
        )
        return f"failed_load: {e.reason}"


logging.basicConfig(level=logging.DEBUG,
                    format="%(levelname)-8s | %(message)s")

print(orchestrate("data/students.csv", "data/output.json"))
print(orchestrate("bad_path", "data/output.json"))
print(orchestrate("data/students.csv", "bad_destination"))
```

**Exercise 2**

```python
import logging
from pathlib import Path


def setup_pipeline_logging(
    log_file: str,
    console_level: int = logging.WARNING,
    file_level: int = logging.DEBUG,
) -> logging.Logger:
    """
    Configure a logger with a console handler (WARNING+) and file handler (DEBUG+).
    The line number in the format refers to where the log call was made.
    """
    log_format = (
        "%(asctime)s | %(levelname)-8s | %(name)s:%(lineno)d | %(message)s"
    )
    formatter = logging.Formatter(
        fmt=log_format,
        datefmt="%Y-%m-%d %H:%M:%S",
    )
    
    Path(log_file).parent.mkdir(parents=True, exist_ok=True)
    
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.handlers.clear()
    
    console_handler = logging.StreamHandler()
    console_handler.setLevel(console_level)
    console_handler.setFormatter(formatter)
    root.addHandler(console_handler)
    
    file_handler = logging.FileHandler(log_file, encoding="utf-8")
    file_handler.setLevel(file_level)
    file_handler.setFormatter(formatter)
    root.addHandler(file_handler)
    
    return logging.getLogger("pipeline")


logger = setup_pipeline_logging("data/logs/detailed.log")

logger.debug("Debug: only in file")
logger.info("Info: only in file")
logger.warning("Warning: console and file")
logger.error("Error: console and file")

print("\nCheck data/logs/detailed.log for DEBUG and INFO messages.")
```

**Exercise 3**

```python
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class StepRecord:
    name: str
    start_time: datetime
    end_time: datetime = None
    records_in: int = 0
    records_out: int = 0
    status: str = "pending"
    errors: list[str] = field(default_factory=list)

    @property
    def duration_seconds(self) -> float:
        end = self.end_time or datetime.now()
        return round((end - self.start_time).total_seconds(), 3)

    @property
    def success_rate(self) -> float:
        if self.records_in == 0:
            return 100.0
        return round(self.records_out / self.records_in * 100, 1)


class PipelineMetricsCollector:
    """Tracks metrics across all steps of a pipeline run."""
    
    def __init__(self, pipeline_name: str):
        self.pipeline_name = pipeline_name
        self.run_start = datetime.now()
        self._steps: dict[str, StepRecord] = {}
        self._step_order: list[str] = []

    def start_step(self, name: str) -> None:
        record = StepRecord(name=name, start_time=datetime.now())
        self._steps[name] = record
        self._step_order.append(name)

    def end_step(
        self,
        name: str,
        records_in: int,
        records_out: int,
        status: str = "success",
    ) -> None:
        if name not in self._steps:
            raise KeyError(f"Step '{name}' was not started")
        step = self._steps[name]
        step.end_time = datetime.now()
        step.records_in = records_in
        step.records_out = records_out
        step.status = status

    def add_error(self, step: str, message: str) -> None:
        if step not in self._steps:
            raise KeyError(f"Step '{step}' was not started")
        self._steps[step].errors.append(message)

    def generate_report(self) -> dict:
        run_end = datetime.now()
        total_duration = (run_end - self.run_start).total_seconds()
        total_records_in = sum(s.records_in for s in self._steps.values())
        total_records_out = sum(s.records_out for s in self._steps.values())
        
        failed_steps = [
            name for name, s in self._steps.items()
            if s.status != "success"
        ]
        
        return {
            "pipeline": self.pipeline_name,
            "run_start": self.run_start.isoformat(),
            "run_end": run_end.isoformat(),
            "total_duration_seconds": round(total_duration, 3),
            "overall_status": "success" if not failed_steps else "failed",
            "total_records_in": total_records_in,
            "total_records_out": total_records_out,
            "steps": [
                {
                    "name": self._steps[name].name,
                    "status": self._steps[name].status,
                    "duration_seconds": self._steps[name].duration_seconds,
                    "records_in": self._steps[name].records_in,
                    "records_out": self._steps[name].records_out,
                    "success_rate_pct": self._steps[name].success_rate,
                    "errors": self._steps[name].errors,
                }
                for name in self._step_order
            ],
            "failed_steps": failed_steps,
        }


import time
import json

collector = PipelineMetricsCollector("student_etl")

collector.start_step("extract")
time.sleep(0.05)
collector.end_step("extract", records_in=0, records_out=100, status="success")

collector.start_step("transform")
time.sleep(0.03)
collector.add_error("transform", "Row 5: GPA 'abc' could not be parsed")
collector.add_error("transform", "Row 12: student_id missing")
collector.end_step("transform", records_in=100, records_out=98, status="success")

collector.start_step("load")
time.sleep(0.02)
collector.end_step("load", records_in=98, records_out=98, status="success")

report = collector.generate_report()
print(json.dumps(report, indent=2))
```

**Exercise 4**

```python
import logging
import time
import functools
from dataclasses import dataclass
from typing import Any

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO, format="%(levelname)-8s | %(message)s")


@dataclass
class StepResult:
    success: bool
    result: Any
    error: Exception | None
    duration_seconds: float
    function_name: str

    def __str__(self) -> str:
        if self.success:
            return (
                f"StepResult[{self.function_name}]: "
                f"OK in {self.duration_seconds:.3f}s"
            )
        return (
            f"StepResult[{self.function_name}]: "
            f"FAILED in {self.duration_seconds:.3f}s — {self.error}"
        )


def pipeline_step(func):
    """
    Decorator that wraps a pipeline function with logging, timing,
    and structured result capture.
    
    The wrapped function never raises. Failures are captured in StepResult.
    """
    @functools.wraps(func)
    def wrapper(*args, **kwargs) -> StepResult:
        start = time.time()
        func_name = func.__name__
        
        logger.info("Starting step: %s", func_name)
        
        try:
            result = func(*args, **kwargs)
            duration = time.time() - start
            logger.info(
                "Completed step: %s in %.3fs", func_name, duration
            )
            return StepResult(
                success=True,
                result=result,
                error=None,
                duration_seconds=round(duration, 3),
                function_name=func_name,
            )
        
        except Exception as e:
            duration = time.time() - start
            logger.exception(
                "Step failed: %s after %.3fs — %s", func_name, duration, e
            )
            return StepResult(
                success=False,
                result=None,
                error=e,
                duration_seconds=round(duration, 3),
                function_name=func_name,
            )
    
    return wrapper


@pipeline_step
def successful_step(records: list[dict]) -> list[dict]:
    """A step that works fine."""
    time.sleep(0.01)
    return [dict(r, processed=True) for r in records]


@pipeline_step
def failing_step(records: list[dict]) -> list[dict]:
    """A step that fails."""
    time.sleep(0.005)
    raise ValueError("Simulated processing error")


sample_data = [{"id": 1, "name": "Arjun"}, {"id": 2, "name": "Priya"}]

result1 = successful_step(sample_data)
print(result1)
print(f"  Result data: {result1.result}")

result2 = failing_step(sample_data)
print(result2)
print(f"  Error type: {type(result2.error).__name__}")
print(f"  Error message: {result2.error}")
```

Lesson 6 is complete. Tell me when you are ready for Lesson 7.