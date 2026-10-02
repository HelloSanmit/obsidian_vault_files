
## Phase 2, Lesson 3 — File I/O and Data Formats

---

### Starting with the right question

Why does a data engineer need to deeply understand file I/O and data formats? You could argue that Spark, BigQuery, and other high-level tools abstract all of this away — you point them at data and they handle reading and writing.

That argument fails in practice for three reasons.

First, data almost always enters your pipeline as a raw file — a CSV export from a legacy system, a JSON dump from an API, a Parquet file from a previous pipeline stage. Before any high-level tool can process it, something has to read it. That something is almost always Python code you write.

Second, the format you choose for intermediate and output data has direct, measurable consequences on pipeline speed and storage cost. A wrong format choice is not a minor inefficiency. On a 500GB dataset, choosing CSV over Parquet can mean the difference between a 30-minute query and a 6-hour query. Understanding why requires understanding how these formats work internally.

Third, debugging pipeline failures at 2am almost always requires reading raw files directly in Python to see what the data actually looks like before and after each transformation. If you cannot read and inspect files fluently, debugging becomes guesswork.

This lesson has two layers. The first layer is the Python mechanics — how to open, read, and write files correctly, including the traps that cause silent failures and resource leaks in production. The second layer is the conceptual layer — what different file formats actually are, how they store data, and why those internal differences produce such dramatically different performance characteristics.

Create a new file called `lesson3.py` in your `de_learning` folder.

---

### The resource management problem — why it matters before anything else

Before you open a single file, you need to understand a problem that exists every time your program interacts with an external resource — a file, a database connection, a network socket.

When your program opens a file, the operating system gives it a **file handle** — a reference to an open file, with internal buffers and state maintained by the OS. The OS has a limit on how many file handles one process can have open simultaneously. On Linux (where most data engineering runs in production) this is typically 1024 by default.

Now consider this:

```python
def read_batch(filepath):
    f = open(filepath, "r", encoding="utf-8")
    data = f.read()
    # some processing happens
    # an exception occurs here
    f.close()   # this never runs
    return data
```

If an exception occurs between opening and closing the file, `f.close()` is never called. The file handle remains open. If this function is called in a loop that processes thousands of files — which is normal in data engineering — you accumulate open file handles until the OS limit is hit and your pipeline crashes with `OSError: [Errno 24] Too many open files`. The crash happens far from where the files were left open, making it extremely hard to diagnose.

This is not a hypothetical. It is a real failure mode that happens in production pipelines.

The `with` statement exists specifically to solve this. It is Python's mechanism for **guaranteed resource cleanup** regardless of whether the enclosed code succeeds or fails.

```python
with open("data.csv", "r", encoding="utf-8") as f:
    data = f.read()
# f is guaranteed to be closed here
# even if an exception occurred inside the with block
```

The `with` statement works through two special methods: `__enter__` and `__exit__`. When Python enters the `with` block, it calls `__enter__` on the object. When Python exits the block — whether normally or via exception — it calls `__exit__`. For file objects, `__exit__` closes the file.

This is not just a syntactic convenience. It is a guarantee made by the Python runtime. The `__exit__` call happens even if the block raises an exception, even if the exception is `SystemExit`, even if someone calls `sys.exit()`. The only exception is `os._exit()` which bypasses Python entirely.

The `__exit__` method receives three arguments that describe any exception that occurred:

```python
def __exit__(self, exc_type, exc_value, traceback):
    # exc_type:  the exception class, or None if no exception
    # exc_value: the exception instance, or None if no exception  
    # traceback: the traceback object, or None if no exception
    # return True to suppress the exception, False to let it propagate
```

For file objects, `__exit__` closes the file and returns `False` — it does not suppress exceptions.

You can implement this in your own classes. This is how every database connection library in Python works:

```python
class ManagedDatabaseConnection:
    
    def __init__(self, host: str, database: str, port: int = 5432):
        self.host = host
        self.database = database
        self.port = port
        self._connection = None
    
    def __enter__(self):
        print(f"Opening connection to {self.database} at {self.host}:{self.port}")
        # in real code this would be: psycopg2.connect(...)
        self._connection = f"<connection to {self.database}>"
        return self   # the return value is bound to the 'as' variable
    
    def __exit__(self, exc_type, exc_value, traceback):
        if self._connection:
            print(f"Closing connection to {self.database}")
            self._connection = None
        
        if exc_type is not None:
            print(f"Connection closed after exception: {exc_value}")
        
        return False  # do not suppress the exception
    
    def execute(self, query: str) -> list:
        if self._connection is None:
            raise RuntimeError("Not connected — use as context manager")
        print(f"Executing: {query}")
        return []


with ManagedDatabaseConnection("localhost", "production") as db:
    results = db.execute("SELECT * FROM students WHERE gpa > 8.0")
    print(f"Got {len(results)} results")

print("Outside the with block — connection is closed")
print(f"Connection is: {db._connection}")  # None
```

Run this. Then modify it to raise an exception inside the `with` block and observe that `__exit__` still runs and closes the connection before the exception propagates.

```python
try:
    with ManagedDatabaseConnection("localhost", "production") as db:
        results = db.execute("SELECT * FROM students")
        raise ValueError("Simulated processing error")
except ValueError as e:
    print(f"Caught outside: {e}")
```

The critical observation: the connection is closed before the exception reaches the `except` clause outside. This is the guarantee that prevents resource leaks.

---

### Why `encoding="utf-8"` is not optional

Every time you open a text file, Python must know how to convert raw bytes into characters. A file on disk is just bytes — the number 65 in a byte could be the letter 'A' in ASCII, or part of a multi-byte character in UTF-8, or something entirely different in a different encoding.

The encoding parameter tells Python which conversion to use.

Without specifying encoding, Python uses `locale.getpreferredencoding()` — the system's default encoding. On Windows this is typically `cp1252`. On Linux and Mac it is typically `utf-8`. On some Windows systems it might be `utf-16` or something else entirely.

The consequence: a file written on a Linux server in UTF-8 and read on a Windows machine without specifying encoding will either produce garbled text or raise a `UnicodeDecodeError` when it encounters a character that does not exist in `cp1252`. Your pipeline works perfectly in development and fails in production because they run on different operating systems.

The fix is always to specify `encoding="utf-8"` explicitly:

```python
# Never do this -- encoding is system-dependent
with open("data.csv", "r") as f:
    content = f.read()

# Always do this -- encoding is explicit and consistent
with open("data.csv", "r", encoding="utf-8") as f:
    content = f.read()
```

The only exception is when you know for certain the file uses a different encoding — legacy systems often produce `latin-1` or `cp1252` files, and you must specify the correct encoding to read them properly.

UTF-8 is the right default for all new files you create and for all files produced by modern systems.

---

### Path handling — the right way

A path is just a string in the simplest sense. But treating paths as plain strings causes problems:

```python
# This fails on Linux/Mac -- backslash is not a path separator there
path = "C:\Users\hp\data\students.csv"

# This looks fine but has a hidden bug on Windows
path = "data\new_folder\students.csv"
# \n is the newline escape sequence in Python strings
# \f is the form feed escape sequence
# The path is corrupted silently
```

Two ways to avoid this:

Raw strings disable escape sequence interpretation:

```python
path = r"C:\Users\hp\data\students.csv"
```

But raw strings still hardcode Windows-specific paths that break on Linux.

The correct solution for all cross-platform path handling:

```python
from pathlib import Path

# Create paths using Path objects
data_dir = Path("data")
students_file = data_dir / "students.csv"
output_dir = Path("output") / "processed" / "2024"

# The / operator joins path components using the correct separator
# for the current operating system
print(students_file)          # data/students.csv on Linux, data\students.csv on Windows

# Path objects have useful attributes
print(students_file.name)     # students.csv
print(students_file.stem)     # students
print(students_file.suffix)   # .csv
print(students_file.parent)   # data

# Check existence
print(students_file.exists())
print(students_file.is_file())
print(students_file.is_dir())

# Create directories including intermediate ones
output_dir.mkdir(parents=True, exist_ok=True)
# parents=True: create data/, output/, output/processed/ if they don't exist
# exist_ok=True: don't raise an error if the directory already exists

# List files matching a pattern
csv_files = list(data_dir.glob("*.csv"))
parquet_files = list(data_dir.rglob("**/*.parquet"))  # recursive search
```

`pathlib.Path` is not just cosmetic. It handles path separators, provides a clean API for file system operations, and works correctly on every operating system without any special handling.

From this point forward, every path in your code should be a `Path` object, not a string.

---

### Reading files — the memory question

When you open a file for reading, you have a fundamental choice: how much of it do you load into memory at once?

```python
# Reading the entire file at once
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()   # entire file is now one string in RAM
```

For a 10GB log file, this consumes 10GB of RAM. Most machines do not have that. The process will be killed by the OS before it finishes.

```python
# Reading one line at a time
with open("data.txt", "r", encoding="utf-8") as f:
    for line in f:
        process(line.strip())
```

The `for line in f` loop reads one line at a time from disk. RAM usage is constant — it holds only one line at any moment regardless of file size. A 10GB file processes with a few kilobytes of RAM.

This is the correct pattern for any file whose size might exceed available RAM. In data engineering, always assume files are larger than available RAM. The line-by-line pattern handles any size.

```python
# Reading in chunks -- for binary files or when lines don't make sense
with open("data.bin", "rb") as f:
    while True:
        chunk = f.read(8192)   # read 8KB at a time
        if not chunk:
            break
        process(chunk)
```

For binary files, iterate by fixed-size chunks instead of lines.

---

### File modes — what each one does and the one that destroys data

```python
"r"   # read text -- raises FileNotFoundError if file doesn't exist
"w"   # write text -- CREATES file if it doesn't exist, OVERWRITES if it does
"a"   # append text -- creates file if it doesn't exist, adds to end if it does  
"x"   # exclusive create -- raises FileExistsError if file already exists
"rb"  # read binary
"wb"  # write binary
"ab"  # append binary
```

The dangerous one is `"w"`. It silently destroys existing file contents. There is no warning, no confirmation, no way to recover the overwritten data. A pipeline bug that opens an output file in `"w"` mode on every retry will destroy the partial output from previous attempts.

In data pipelines, think carefully about which mode is correct:

- First run, no existing file: `"w"` is fine
- Appending daily data to an accumulated file: `"a"` is correct
- Writing to a path that must not already exist: `"x"` is the safest choice
- Rerunning a failed pipeline: consider whether you want to overwrite or append

---

### CSV — what it actually is and why it is problematic

CSV stands for Comma-Separated Values. It is not a formal standard — different tools implement it differently. The variations cause endless interoperability problems.

The structure is simple in concept: each line is a record, and fields within a record are separated by a delimiter (usually comma, sometimes semicolon or tab). The first line is often but not always a header row with field names.

The problems:

**No type information.** Every value is a string. The number `8.5` and the string `"8.5"` are indistinguishable in CSV. The parser that reads the file must infer or be told the correct type for each field. This is a source of constant bugs when the inference is wrong.

**No schema.** A CSV file does not declare what columns exist, what types they are, or whether fields are required. Different files from the same source can have different columns if the source system changes. Your pipeline will fail silently or produce wrong results.

**No standard for missing values.** Depending on the tool that produced the CSV, a missing value might be an empty string, the text "NULL", the text "None", the text "N/A", the text "nan", or something else. You must handle all variants.

**Quoting rules.** What happens when a field value contains a comma? It must be wrapped in quotes. What happens when a field value contains quotes? They must be escaped — usually by doubling them. Different tools implement these rules differently, and manually splitting on commas always breaks on these cases.

**Line ending inconsistency.** Windows uses `\r\n`, Unix uses `\n`, old Mac used `\r`. Files transferred between systems may have inconsistent line endings.

Despite all of this, CSV is everywhere. Legacy systems produce it. Stakeholders demand it. External partners send it. You will deal with it constantly.

The Python `csv` module handles the quoting and line ending problems correctly. Never parse CSV manually by splitting on commas. Always use the module.

```python
import csv
from pathlib import Path

Path("data").mkdir(exist_ok=True)

students = [
    {"student_id": 1, "name": "Arjun Mehta",  "city": "Mumbai",    "gpa": 8.5},
    {"student_id": 2, "name": "Priya Sharma",  "city": "Delhi",     "gpa": 9.1},
    {"student_id": 3, "name": "Rohan Verma",   "city": "Mumbai",    "gpa": 7.8},
    {"student_id": 4, "name": "Sneha Iyer",    "city": "Chennai",   "gpa": 8.9},
    {"student_id": 5, "name": "Karan Patel",   "city": "Ahmedabad", "gpa": None},
]

with open("data/students.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["student_id", "name", "city", "gpa"])
    writer.writeheader()
    writer.writerows(students)
```

`newline=""` when writing on Windows is critical. Without it, Python's text mode adds `\r` to every `\n`, and then the `csv` module adds its own `\r`, producing files with blank lines between every data row. Specifying `newline=""` disables Python's automatic translation and lets the csv module control line endings correctly.

Open the written file in a text editor and look at what was produced. Notice how `None` was written — as the string `"None"` or as an empty field. This is the missing value problem. When you read it back:

```python
with open("data/students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(dict(row))
        print(f"  student_id type: {type(row['student_id'])}")
        print(f"  gpa type: {type(row['gpa'])}")
        print(f"  gpa value: {repr(row['gpa'])}")
```

Every value is a string. `student_id` is `"1"`, not `1`. `gpa` is `"8.5"` or `"None"` or `""`. This is the type problem. You must explicitly convert every field.

The parsing function that every CSV-reading pipeline needs:

```python
def parse_student_record(raw_row: dict) -> dict:
    """
    Convert a raw CSV row (all string values) to properly typed data.
    
    The 'raw_row' comes from csv.DictReader which gives us strings for everything.
    We must explicitly convert each field to its correct type.
    Missing/null indicators vary: empty string, "None", "null", "N/A" — handle all.
    """
    def parse_optional_float(value: str) -> float | None:
        cleaned = value.strip().lower()
        if cleaned in ("", "none", "null", "n/a", "nan"):
            return None
        try:
            return float(cleaned)
        except ValueError:
            return None

    return {
        "student_id": int(raw_row["student_id"]),
        "name": " ".join(raw_row["name"].split()).title(),
        "city": raw_row["city"].strip(),
        "gpa": parse_optional_float(raw_row["gpa"]),
    }


with open("data/students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    parsed_students = [parse_student_record(row) for row in reader]

for student in parsed_students:
    print(student)
    print(f"  gpa type: {type(student['gpa'])}")
```

Notice the nested function `parse_optional_float`. It is defined inside `parse_student_record` because it is only needed there. This is a legitimate use of function nesting — the inner function is a helper that has no reason to exist outside its parent.

Now the quoting problem — why you must use the csv module:

```python
tricky_students = [
    {"id": 1, "name": "Smith, John",       "note": 'Scored "excellent"'},
    {"id": 2, "name": "O'Brien, Mary",     "note": "Normal student"},
    {"id": 3, "name": "Multi\nLine Name",  "note": "Has newline in name"},
]

with open("data/tricky.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "name", "note"])
    writer.writeheader()
    writer.writerows(tricky_students)
```

Open `data/tricky.csv`. See how the csv module correctly quoted the fields containing commas, escaped the internal quotes, and handled the newline in the name. Read it back with `csv.DictReader` and verify the values are recovered correctly. Now imagine doing this manually by splitting on commas — it breaks on the very first record.

---

### JSON — what it is, how it works, where it fails

JSON (JavaScript Object Notation) is a text-based format derived from JavaScript object syntax. It solves CSV's type problem — a JSON number is recognized as a number, a string as a string, a boolean as a boolean, null as null, an array as an array, a nested object as an object.

This type preservation is why JSON became the universal format for API communication. When a REST API returns student data, the client receives a JSON object where `student_id` is already an integer and `gpa` is already a float — no parsing step needed for basic types.

JSON's structure maps directly to Python:

|JSON|Python|
|---|---|
|object `{}`|dictionary|
|array `[]`|list|
|string `"..."`|str|
|number|int or float|
|boolean `true/false`|bool `True/False`|
|null|None|

The Python `json` module serializes Python objects to JSON strings and deserializes JSON strings back to Python objects.

```python
import json
from pathlib import Path

pipeline_run = {
    "pipeline_name": "student_etl",
    "run_date": "2024-01-15",
    "configuration": {
        "batch_size": 1000,
        "max_retries": 3,
        "source": {
            "type": "csv",
            "path": "/data/students.csv",
            "delimiter": ","
        },
        "destination": {
            "type": "parquet",
            "path": "/output/students/",
            "partitioned_by": ["city"]
        }
    },
    "statistics": {
        "rows_read": 1500,
        "rows_written": 1498,
        "rows_failed": 2,
        "duration_seconds": 4.73
    },
    "tags": ["daily", "students", "production"],
    "success": True
}

# Write JSON to file
with open("data/pipeline_run.json", "w", encoding="utf-8") as f:
    json.dump(pipeline_run, f, indent=2)

# Read JSON from file
with open("data/pipeline_run.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)

print(loaded["pipeline_name"])
print(type(loaded["statistics"]["rows_read"]))      # int
print(type(loaded["statistics"]["duration_seconds"]))  # float
print(type(loaded["success"]))                      # bool
print(type(loaded["tags"]))                         # list
print(type(loaded["configuration"]["source"]))      # dict
```

Run this. The key observation is the types. Unlike CSV where everything was a string, JSON preserves types through the round-trip.

The `indent=2` parameter formats the output with 2-space indentation for readability. Without it, the entire JSON is on one line — valid but unreadable. Use `indent=2` for files humans will read, omit it for files passed between machines where size matters.

Now the critical failure modes.

**JSON cannot serialize many Python types.** This is the most common JSON-related bug in data engineering:

```python
from datetime import datetime, date
from decimal import Decimal

problematic_data = {
    "timestamp": datetime.now(),     # not JSON serializable
    "run_date": date.today(),        # not JSON serializable
    "amount": Decimal("1234.56"),    # not JSON serializable
    "tags": {"daily", "production"}, # set -- not JSON serializable
}

try:
    json.dumps(problematic_data)
except TypeError as e:
    print(f"Serialization failed: {e}")
```

Run this. The error tells you exactly what failed. These types come up constantly in data engineering — database queries return `datetime` objects, financial calculations use `Decimal`, intermediate computations often produce sets.

The solution is a custom encoder:

```python
import json
from datetime import datetime, date
from decimal import Decimal


class DataEngineeringEncoder(json.JSONEncoder):
    """
    Extends the default JSON encoder to handle types
    common in data engineering that the standard encoder cannot serialize.
    """
    
    def default(self, obj):
        # datetime must be checked before date because datetime is a subclass of date
        if isinstance(obj, datetime):
            return obj.isoformat()      # "2024-01-15T14:30:00.123456"
        if isinstance(obj, date):
            return obj.isoformat()      # "2024-01-15"
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, set):
            return sorted(list(obj))    # sorted for deterministic output
        if isinstance(obj, bytes):
            return obj.decode("utf-8")
        # for anything else, call the parent's default which raises TypeError
        return super().default(obj)


data = {
    "timestamp": datetime(2024, 1, 15, 14, 30, 0),
    "run_date": date(2024, 1, 15),
    "amount": Decimal("1234.56789"),
    "tags": {"daily", "production", "students"},
}

serialized = json.dumps(data, cls=DataEngineeringEncoder, indent=2)
print(serialized)
```

The `default` method is called only for objects the standard encoder cannot handle. For everything it can handle (strings, numbers, lists, dicts, booleans, None), the standard encoder takes over and `default` is never called. The order of `isinstance` checks matters — `datetime` must be checked before `date` because `datetime` is a subclass of `date`, and an `isinstance(obj, date)` check would match `datetime` objects too.

**JSON Lines — the streaming-friendly variant**

Standard JSON wraps everything in one top-level object or array. This means you must read and parse the entire file before processing any record — for a file with one million records, you load them all into memory first.

JSON Lines (also called NDJSON) stores one independent JSON object per line. Each line is complete and independently parseable. This enables line-by-line processing regardless of file size.

```python
records = [
    {"student_id": 1, "name": "Arjun Mehta",  "gpa": 8.5, "city": "Mumbai"},
    {"student_id": 2, "name": "Priya Sharma",  "gpa": 9.1, "city": "Delhi"},
    {"student_id": 3, "name": "Rohan Verma",   "gpa": 7.8, "city": "Mumbai"},
]

# Writing JSON Lines
with open("data/students.jsonl", "w", encoding="utf-8") as f:
    for record in records:
        f.write(json.dumps(record) + "\n")

# Reading JSON Lines -- one line at a time, constant memory regardless of file size
with open("data/students.jsonl", "r", encoding="utf-8") as f:
    for line in f:
        line = line.strip()
        if not line:    # skip empty lines
            continue
        record = json.loads(line)
        print(f"Processing student {record['student_id']}: {record['name']}")
```

Open `data/students.jsonl` in a text editor. Each line is a complete, valid JSON object. You can process this file line by line without ever holding more than one record in memory. For a file with 100 million records, memory usage is constant.

JSON Lines is the format Kafka uses for its data files, what many ETL tools use for intermediate storage, and what cloud services often provide as their export format. You will encounter it constantly.

---

### Parquet — why it exists and how it actually works

To understand Parquet, you need to understand the problem it was designed to solve. That problem did not exist until datasets got large enough that I/O speed became the bottleneck.

**The I/O bottleneck.** When you run a query on a large dataset, the slowest part is usually not the computation — it is reading the data from disk. Modern SSDs can read at roughly 500MB/second. A 100GB file takes 200 seconds to read completely, before any processing. If your query only needs 3 out of 50 columns, you still read all 50 columns — 47 columns of wasted I/O.

This is the problem with row-oriented storage like CSV and JSON. Every row is a complete record with all its column values adjacent. To access one column, you must read every row's complete record.

**Columnar storage.** Parquet stores data column by column instead of row by row.

Imagine a table with 4 rows and 5 columns:

Row-oriented (CSV):

```
Row 1:  1,  "Arjun",  "Mumbai",   8.5,  20
Row 2:  2,  "Priya",  "Delhi",    9.1,  21
Row 3:  3,  "Rohan",  "Mumbai",   7.8,  19
Row 4:  4,  "Sneha",  "Chennai",  8.9,  22
```

Columnar (Parquet):

```
Column student_id: 1, 2, 3, 4
Column name:       "Arjun", "Priya", "Rohan", "Sneha"
Column city:       "Mumbai", "Delhi", "Mumbai", "Chennai"
Column gpa:        8.5, 9.1, 7.8, 8.9
Column age:        20, 21, 19, 22
```

A query that reads only `gpa` reads only the gpa column's data — 4 values. With row storage, reading only gpa still requires reading all 5 columns for all 4 rows to locate the gpa values. The I/O savings scale with the number of columns — for a 50-column table, reading 3 columns with Parquet reads 6% of what CSV would require.

**Why compression is dramatically better on columnar data.**

Compression algorithms exploit repetition and patterns. When you compress column data, all the values are the same type and often have similar magnitudes. The `city` column might be `"Mumbai", "Delhi", "Mumbai", "Chennai", "Mumbai"` — three "Mumbai" values in close proximity. The `age` column might be `20, 21, 19, 22, 20, 21` — all integers in a narrow range.

Compression algorithms can represent "Mumbai" appearing 3 times in 5 values extremely efficiently. In row storage, these similar values are spread across the file interleaved with completely different data types and magnitudes, making pattern-based compression far less effective.

Real-world consequence: a 1GB CSV file typically becomes 50-100MB as Parquet with Snappy compression. Reads are 10-20x faster even ignoring column pruning, purely from the smaller file size.

**Schema embedding.** Parquet files embed a complete schema — column names, data types, nullability. Every field has an explicit type. When the file is read, the reader already knows exactly what to expect. No type inference, no parsing ambiguity. A `gpa` field written as a `float32` is always read as a `float32`, regardless of who writes or reads the file or what language they use.

**Row groups and predicate pushdown.**

Parquet does not store all column values in one contiguous block. It divides rows into **row groups** — typically 64MB to 128MB of uncompressed data each. Within each row group, data is stored column by column.

For each column in each row group, Parquet stores **statistics**: the minimum value, maximum value, and null count. These are stored in the file's footer and can be read without reading the actual data.

This enables **predicate pushdown**: when you filter with `WHERE gpa > 9.0`, the reader checks the max gpa statistic for each row group. If the maximum gpa in a row group is 8.5, no rows in that group can satisfy `gpa > 9.0`, and the entire row group is skipped without reading it.

Combined, column pruning (only read needed columns) and predicate pushdown (skip row groups that cannot match the filter) make Parquet queries on large datasets dramatically faster than equivalent queries on CSV.

---

### Working with Parquet in Python

Install PyArrow:

```
pip install pyarrow
```

PyArrow is the Python library for working with columnar data and Parquet files. It is the foundation that Pandas, Spark's Python API, and most other data tools use internally for Parquet I/O.

```python
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

Path("data").mkdir(exist_ok=True)
```

**Creating a PyArrow table with an explicit schema:**

```python
data = {
    "student_id": [1,        2,        3,        4,        5       ],
    "name":       ["Arjun",  "Priya",  "Rohan",  "Sneha",  "Karan" ],
    "city":       ["Mumbai", "Delhi",  "Mumbai", "Chennai","Ahmedabad"],
    "gpa":        [8.5,      9.1,      7.8,      8.9,      None    ],
    "age":        [20,       21,       19,       22,       20      ],
}

schema = pa.schema([
    pa.field("student_id", pa.int32()),
    pa.field("name",       pa.string()),
    pa.field("city",       pa.string()),
    pa.field("gpa",        pa.float32()),
    pa.field("age",        pa.int16()),
])

table = pa.table(data, schema=schema)

print(table.schema)
print(f"Rows: {table.num_rows}")
print(f"Columns: {table.num_columns}")
```

Why explicitly define the schema? Because PyArrow will infer types if you omit it, and the inferred types may not be what you want. It might infer `int64` for student_id when `int32` is sufficient. It might infer `float64` for gpa when `float32` is sufficient. Smaller types use less memory and disk space, and reduce I/O. At 100 million rows, the difference between `int32` and `int64` is 400MB. Always define your schema explicitly in production pipelines.

The `None` in the gpa list becomes a null value in the Parquet file. Parquet has native null support — each column has a separate bitmap tracking which values are null. This is more efficient than CSV's approach of storing the string "None".

**Writing and reading:**

```python
# Write with Snappy compression
pq.write_table(table, "data/students.parquet", compression="snappy")

# Read the entire file
loaded = pq.read_table("data/students.parquet")
print(loaded.to_pydict())

# Read only specific columns -- column pruning in action
gpa_only = pq.read_table("data/students.parquet", columns=["name", "gpa"])
print(gpa_only.to_pydict())
print(f"Schema of partial read: {gpa_only.schema}")
```

The `columns=["name", "gpa"]` argument triggers column pruning. PyArrow reads only those two columns' data from the file. The student_id, city, and age columns are not read at all. On a 50-column file with 100 million rows, reading 2 columns instead of 50 reduces I/O by 96%.

```python
# Read with a filter -- predicate pushdown
high_achievers = pq.read_table(
    "data/students.parquet",
    filters=[("gpa", ">", 8.5)]
)
print(high_achievers.to_pydict())
```

The filter is evaluated at the storage level. Row groups where the maximum gpa does not exceed 8.5 are skipped entirely. On your 5-row file this makes no measurable difference. On a file with 100 million rows in 800 row groups, most row groups might be skipped.

**Compression choices:**

```python
import os

# Write with different compressions to compare
for compression in ["none", "snappy", "gzip", "zstd"]:
    path = f"data/students_{compression}.parquet"
    pq.write_table(table, path, compression=compression)
    size = os.path.getsize(path)
    print(f"{compression:8}: {size:,} bytes")
```

On a 5-row table the differences will be minimal or even reversed (headers and metadata add overhead). On a large table:

**Snappy** — fast compression and decompression. The most common choice for data lakes and pipelines where read speed matters more than maximum compression. Decompression is so fast it adds negligible latency.

**Gzip** — better compression ratio, significantly slower decompression. Use when storage cost is the primary concern and data is accessed infrequently.

**Zstd** — modern algorithm. Balances compression ratio and speed better than both Snappy and Gzip. Increasingly preferred in new systems.

**None** — no compression. Use for intermediate data that will be immediately processed and deleted.

---

### Partitioned Parquet — organizing large datasets

A single Parquet file works for datasets up to a few GB. For larger datasets, partitioning divides the data into multiple files organized by one or more column values.

The directory structure partitioned by city:

```
students_partitioned/
  city=Mumbai/
    part-0.parquet      (contains only Mumbai rows)
  city=Delhi/
    part-0.parquet      (contains only Delhi rows)
  city=Chennai/
    part-0.parquet      (contains only Chennai rows)
```

When a query filters by city, the query engine looks at the directory names and skips entire directories that do not match. It never opens the Delhi or Chennai files when you ask for Mumbai data.

```python
data_large = {
    "student_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan", "Divya", "Amit", "Meera"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad", "Delhi", "Pune", "Hyderabad"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5, 9.4, 7.2, 8.1],
}

large_table = pa.table(data_large)

# Write partitioned by city
pq.write_to_dataset(
    large_table,
    root_path="data/students_partitioned",
    partition_cols=["city"],
    compression="snappy",
    existing_data_behavior="overwrite_or_ignore",
)
```

Look at the directory structure created. Then read with a city filter:

```python
# Only the Mumbai directory is opened -- all others are ignored
mumbai = pq.read_table(
    "data/students_partitioned",
    filters=[("city", "=", "Mumbai")]
)
print(mumbai.to_pydict())
print(f"Mumbai rows: {mumbai.num_rows}")
```

The `city` column is not in the output data — it is encoded in the directory name and PyArrow removes it from the read result by default. This is partition column behavior.

**Choosing partition columns** is a design decision with real consequences.

Partition by columns you frequently filter on. If most queries filter by date, partition by date. If most filter by region, partition by region.

Avoid too many partitions. Each partition creates at least one file. If you partition by user_id and have 10 million users, you create 10 million directories with 10 million files. Opening files has overhead — 10 million files is slower than 100 files even if the total data is the same. This is called the **small files problem** and it is a real operational issue in data lakes.

Avoid too few partitions. If you have 1TB of data in one file, you cannot parallelize reading it — only one worker can read it at a time. Partitioning enables parallel reads.

The sweet spot for most data engineering workloads: partitions between 100MB and 1GB of data each, with a moderate number of distinct partition values (days, months, regions, categories — not user IDs or transaction IDs).

---

### Putting all three formats together — a realistic pipeline

This pipeline reads CSV (as it arrives from a source system), validates and transforms the data, and writes Parquet (for efficient downstream querying). It also writes a JSON metadata file recording what happened.

```python
import csv
import json
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path
from datetime import datetime


def extract_from_csv(filepath: str) -> tuple[list[dict], list[str]]:
    """
    Read and parse a CSV file. Returns valid records and error messages.
    Handles missing files, empty files, and malformed rows gracefully.
    """
    path = Path(filepath)
    errors = []
    records = []

    if not path.exists():
        errors.append(f"Source file not found: {filepath}")
        return records, errors

    def parse_gpa(value: str) -> float | None:
        cleaned = value.strip().lower()
        if cleaned in ("", "none", "null", "n/a"):
            return None
        try:
            parsed = float(cleaned)
            if not (0.0 <= parsed <= 10.0):
                raise ValueError(f"GPA {parsed} outside valid range [0.0, 10.0]")
            return parsed
        except ValueError as e:
            raise ValueError(f"Cannot parse GPA '{value}': {e}") from e

    with open(path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        
        if reader.fieldnames is None:
            errors.append("CSV file is empty or has no header")
            return records, errors

        for line_num, row in enumerate(reader, start=2):
            try:
                records.append({
                    "student_id": int(row["student_id"]),
                    "name": " ".join(row["name"].split()).title(),
                    "city": row["city"].strip().title(),
                    "gpa": parse_gpa(row.get("gpa", "")),
                })
            except (ValueError, KeyError) as e:
                errors.append(f"Line {line_num}: {e} — raw: {dict(row)}")

    return records, errors


def transform(records: list[dict]) -> list[dict]:
    """
    Apply business logic transformations.
    Adds derived fields, normalizes values.
    """
    transformed = []
    for record in records:
        enriched = {
            **record,
            "is_high_achiever": record["gpa"] is not None and record["gpa"] >= 8.5,
            "gpa_category": (
                "outstanding" if record["gpa"] is not None and record["gpa"] >= 9.0
                else "good" if record["gpa"] is not None and record["gpa"] >= 7.5
                else "needs_improvement" if record["gpa"] is not None
                else "ungraded"
            ),
        }
        transformed.append(enriched)
    return transformed


def load_to_parquet(records: list[dict], output_path: str) -> int:
    """
    Write records to a Parquet file with an explicit schema.
    Returns number of records written.
    """
    if not records:
        return 0

    Path(output_path).parent.mkdir(parents=True, exist_ok=True)

    schema = pa.schema([
        pa.field("student_id",      pa.int32()),
        pa.field("name",            pa.string()),
        pa.field("city",            pa.string()),
        pa.field("gpa",             pa.float32()),
        pa.field("is_high_achiever", pa.bool_()),
        pa.field("gpa_category",    pa.string()),
    ])

    table = pa.table(
        {field.name: [r[field.name] for r in records] for field in schema},
        schema=schema,
    )

    pq.write_table(table, output_path, compression="snappy")
    return len(records)


def write_run_metadata(metadata: dict, output_path: str) -> None:
    """Write pipeline run metadata as JSON."""
    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    with open(output_path, "w", encoding="utf-8") as f:
        json.dump(metadata, f, indent=2, cls=DataEngineeringEncoder)


class DataEngineeringEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)


def run_pipeline(source_csv: str, output_dir: str) -> dict:
    """
    Execute the full extract-transform-load pipeline.
    Returns a metadata dictionary describing the run.
    """
    run_start = datetime.now()
    output = Path(output_dir)

    print(f"Pipeline starting at {run_start.isoformat()}")

    raw_records, extract_errors = extract_from_csv(source_csv)
    print(f"Extracted {len(raw_records)} records, {len(extract_errors)} errors")

    transformed_records = transform(raw_records)
    print(f"Transformed {len(transformed_records)} records")

    parquet_path = str(output / "students_clean.parquet")
    rows_written = load_to_parquet(transformed_records, parquet_path)
    print(f"Wrote {rows_written} records to {parquet_path}")

    run_end = datetime.now()
    duration = (run_end - run_start).total_seconds()

    metadata = {
        "pipeline": "student_etl",
        "run_start": run_start,
        "run_end": run_end,
        "duration_seconds": round(duration, 3),
        "statistics": {
            "rows_extracted": len(raw_records),
            "rows_written": rows_written,
            "rows_with_errors": len(extract_errors),
            "extraction_errors": extract_errors,
        },
        "outputs": {
            "parquet": parquet_path,
        }
    }

    write_run_metadata(metadata, str(output / "run_metadata.json"))
    print(f"Metadata written. Pipeline complete in {duration:.3f}s")

    return metadata


run_stats = run_pipeline(
    source_csv="data/students.csv",
    output_dir="data/output",
)

# Verify the output
print("\nVerifying output:")
result = pq.read_table("data/output/students_clean.parquet")
print(result.to_pydict())
print(f"\nSchema: {result.schema}")
```

Run this end to end. Read every output line and trace it to the code that produced it.

Notice the structure: three focused functions — `extract_from_csv`, `transform`, `load_to_parquet` — each doing exactly one thing, each returning data rather than producing side effects. The `run_pipeline` function orchestrates them. This is the Extract-Transform-Load pattern that you will implement at much larger scale with Spark and Airflow in later phases. The same structure, just bigger tools.

---

### The format decision framework

Every time you choose a format in a data engineering project, these questions guide the decision:

**Who produces this data?** External systems often only produce CSV or JSON. You cannot always choose.

**Who consumes this data?** Humans and spreadsheet tools want CSV. APIs want JSON. Analytical tools want Parquet.

**How large will it get?** Under a few MB, format performance barely matters. Above 1GB, format choice has enormous impact.

**Will it be queried analytically?** If queries will scan specific columns across many rows, Parquet's column pruning is essential.

**Does the schema change?** Parquet's embedded schema detects schema changes. CSV changes silently break parsers.

**How often is it written vs read?** Write-once-read-many (data lake files) favors Parquet. Write-frequently (log files being appended) may favor JSON Lines.

**The default** for any output your pipelines produce: Parquet with Snappy compression, partitioned by the most common filter column. This is the right answer unless one of the above questions points elsewhere.

---

### Exercises — Lesson 3

1. Write a function `safe_read_csv` that reads a CSV file defensively. It must handle: file not found, empty file, missing expected columns, rows with wrong number of fields, and type conversion failures in any field. Accept the expected column names as a parameter. Return a dataclass `ReadResult` with fields `records`, `errors`, and `rows_skipped`. For each error, include the line number and what went wrong.
    
2. Write a function that reads a JSON file that might contain either a JSON array of records or a JSON Lines file (one record per line). It should auto-detect which format it is — do not require the caller to specify. Return the records as a list of dictionaries regardless of input format. Handle malformed JSON gracefully.
    
3. Write a function `csv_to_parquet` that converts a CSV file to a Parquet file. It must infer the schema from the data — use these rules: if every non-null value in a column can be parsed as an integer, use `int32`; if every non-null value can be parsed as a float, use `float32`; otherwise use string. Write a separate function `compare_sizes` that reports the original CSV size and the Parquet size, and the compression ratio. Test with `data/students.csv`.
    
4. Implement a class `PartitionedParquetWriter` that writes records to partitioned Parquet files. The constructor accepts an output directory and a partition column name. The `write_batch(records: list[dict])` method accepts records and writes each partition's records to its own Parquet file (e.g., `output/city=Mumbai/part-0.parquet`). The `finalize()` method prints a summary of how many files were written and how many records went to each partition.
    

---

### Answers — Lesson 3

**Exercise 1**

```python
import csv
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class ReadResult:
    records: list[dict] = field(default_factory=list)
    errors: list[str] = field(default_factory=list)
    rows_skipped: int = 0

    def add_error(self, line_num: int, message: str) -> None:
        self.errors.append(f"Line {line_num}: {message}")
        self.rows_skipped += 1


def safe_read_csv(filepath: str, expected_columns: list[str]) -> ReadResult:
    """
    Read a CSV file with comprehensive error handling.

    Args:
        filepath: Path to the CSV file
        expected_columns: Column names that must be present

    Returns:
        ReadResult with records, errors, and skip count
    """
    result = ReadResult()
    path = Path(filepath)

    if not path.exists():
        result.add_error(0, f"File not found: {filepath}")
        return result

    if path.stat().st_size == 0:
        result.add_error(0, "File is empty")
        return result

    try:
        with open(path, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f)

            if reader.fieldnames is None:
                result.add_error(1, "No header row found")
                return result

            missing_cols = set(expected_columns) - set(reader.fieldnames)
            if missing_cols:
                result.add_error(
                    1,
                    f"Missing expected columns: {sorted(missing_cols)}. "
                    f"Found: {list(reader.fieldnames)}"
                )
                return result

            for line_num, row in enumerate(reader, start=2):
                try:
                    if len(row) != len(reader.fieldnames):
                        result.add_error(
                            line_num,
                            f"Expected {len(reader.fieldnames)} fields, "
                            f"got {len(row)}"
                        )
                        continue

                    record = {}
                    for col in expected_columns:
                        value = row.get(col, "").strip()
                        record[col] = value if value else None

                    result.records.append(record)

                except Exception as e:
                    result.add_error(line_num, f"Unexpected error: {e}")

    except PermissionError:
        result.add_error(0, f"Permission denied: {filepath}")
    except UnicodeDecodeError as e:
        result.add_error(0, f"Encoding error: {e}. Try specifying encoding.")

    return result


output = safe_read_csv("data/students.csv", ["student_id", "name", "city", "gpa"])
print(f"Records: {len(output.records)}")
print(f"Errors: {len(output.errors)}")
print(f"Skipped: {output.rows_skipped}")
for record in output.records:
    print(f"  {record}")
for error in output.errors:
    print(f"  ERROR: {error}")

bad_result = safe_read_csv("nonexistent.csv", ["student_id"])
print(f"\nNonexistent file errors: {bad_result.errors}")
```

**Exercise 2**

```python
import json
from pathlib import Path


def read_json_auto(filepath: str) -> tuple[list[dict], list[str]]:
    """
    Read a JSON file that is either a JSON array or JSON Lines format.
    Auto-detects the format by attempting JSON array parse first.

    Returns:
        Tuple of (records, errors)
    """
    errors = []
    path = Path(filepath)

    if not path.exists():
        return [], [f"File not found: {filepath}"]

    with open(path, "r", encoding="utf-8") as f:
        content = f.read().strip()

    if not content:
        return [], ["File is empty"]

    # Try JSON array format first (starts with '[')
    if content.startswith("["):
        try:
            data = json.loads(content)
            if isinstance(data, list):
                valid = []
                for i, item in enumerate(data):
                    if isinstance(item, dict):
                        valid.append(item)
                    else:
                        errors.append(f"Item {i} is not a dict: {type(item).__name__}")
                return valid, errors
        except json.JSONDecodeError as e:
            errors.append(f"Failed to parse as JSON array: {e}")

    # Try JSON Lines format (one object per line)
    records = []
    for line_num, line in enumerate(content.splitlines(), start=1):
        line = line.strip()
        if not line:
            continue
        try:
            record = json.loads(line)
            if isinstance(record, dict):
                records.append(record)
            else:
                errors.append(
                    f"Line {line_num}: expected dict, got {type(record).__name__}"
                )
        except json.JSONDecodeError as e:
            errors.append(f"Line {line_num}: invalid JSON — {e}")

    return records, errors


# Test with JSON array
with open("data/test_array.json", "w", encoding="utf-8") as f:
    json.dump([
        {"student_id": 1, "name": "Arjun", "gpa": 8.5},
        {"student_id": 2, "name": "Priya", "gpa": 9.1},
    ], f, indent=2)

# Test with JSON Lines
with open("data/test_lines.jsonl", "w", encoding="utf-8") as f:
    for record in [
        {"student_id": 3, "name": "Rohan", "gpa": 7.8},
        {"student_id": 4, "name": "Sneha", "gpa": 8.9},
    ]:
        f.write(json.dumps(record) + "\n")

records_array, errors = read_json_auto("data/test_array.json")
print(f"JSON array: {len(records_array)} records, {len(errors)} errors")
print(records_array)

records_lines, errors = read_json_auto("data/test_lines.jsonl")
print(f"JSON Lines: {len(records_lines)} records, {len(errors)} errors")
print(records_lines)
```

**Exercise 3**

```python
import csv
import os
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path


def infer_column_type(values: list[str]) -> pa.DataType:
    """
    Infer the PyArrow type for a column based on its non-null string values.
    Rules: int32 if all parseable as int, float32 if all parseable as float,
    else string.
    """
    non_null = [v for v in values if v and v.strip().lower() not in ("", "none", "null")]

    if not non_null:
        return pa.string()

    try:
        for v in non_null:
            int(v)
        return pa.int32()
    except ValueError:
        pass

    try:
        for v in non_null:
            float(v)
        return pa.float32()
    except ValueError:
        pass

    return pa.string()


def parse_value(value: str, arrow_type: pa.DataType):
    """Convert a string value to the correct Python type for a PyArrow column."""
    cleaned = value.strip() if value else ""
    if cleaned.lower() in ("", "none", "null", "n/a"):
        return None
    if arrow_type == pa.int32():
        return int(cleaned)
    if arrow_type == pa.float32():
        return float(cleaned)
    return cleaned


def csv_to_parquet(
    csv_path: str,
    parquet_path: str,
    compression: str = "snappy"
) -> dict:
    """
    Convert a CSV file to Parquet, inferring column types automatically.

    Returns:
        Dictionary with conversion statistics
    """
    Path(parquet_path).parent.mkdir(parents=True, exist_ok=True)

    # First pass: read all data as strings to infer types
    with open(csv_path, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        column_names = reader.fieldnames
        all_rows = list(reader)

    if not column_names or not all_rows:
        raise ValueError("CSV file is empty or has no data")

    # Infer types from all values in each column
    column_types = {}
    for col in column_names:
        col_values = [row[col] for row in all_rows]
        column_types[col] = infer_column_type(col_values)

    print("Inferred schema:")
    for col, dtype in column_types.items():
        print(f"  {col}: {dtype}")

    # Build PyArrow table with inferred types
    schema = pa.schema([
        pa.field(col, dtype) for col, dtype in column_types.items()
    ])

    columns = {}
    for col in column_names:
        columns[col] = [
            parse_value(row[col], column_types[col])
            for row in all_rows
        ]

    table = pa.table(columns, schema=schema)
    pq.write_table(table, parquet_path, compression=compression)

    return {
        "rows": len(all_rows),
        "columns": len(column_names),
        "schema": {col: str(dtype) for col, dtype in column_types.items()},
    }


def compare_sizes(csv_path: str, parquet_path: str) -> None:
    """Print a size comparison between CSV and Parquet files."""
    csv_size = os.path.getsize(csv_path)
    parquet_size = os.path.getsize(parquet_path)
    ratio = csv_size / parquet_size if parquet_size > 0 else 0

    print(f"\nFile size comparison:")
    print(f"  CSV:     {csv_size:,} bytes ({csv_size/1024:.1f} KB)")
    print(f"  Parquet: {parquet_size:,} bytes ({parquet_size/1024:.1f} KB)")
    print(f"  Ratio:   {ratio:.2f}x (Parquet is {ratio:.2f}x smaller)")
    if parquet_size > csv_size:
        print("  Note: on small datasets Parquet overhead exceeds savings.")
        print("  At millions of rows, Parquet is 10-20x smaller than CSV.")


stats = csv_to_parquet("data/students.csv", "data/students_converted.parquet")
print(f"\nConversion complete: {stats['rows']} rows, {stats['columns']} columns")

compare_sizes("data/students.csv", "data/students_converted.parquet")

result = pq.read_table("data/students_converted.parquet")
print(f"\nVerification — schema: {result.schema}")
print(result.to_pydict())
```

**Exercise 4**

```python
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path
from collections import defaultdict


class PartitionedParquetWriter:
    """
    Writes records to partitioned Parquet files organized by a partition column.
    
    Creates directory structure:
        output_dir/
          {partition_col}={value}/
            part-0.parquet
    """

    def __init__(self, output_dir: str, partition_col: str):
        self.output_dir = Path(output_dir)
        self.partition_col = partition_col
        self._partitions: dict[str, list[dict]] = defaultdict(list)
        self._files_written: dict[str, str] = {}

        self.output_dir.mkdir(parents=True, exist_ok=True)
        print(
            f"PartitionedParquetWriter initialized. "
            f"Output: {self.output_dir}, "
            f"Partitioned by: {partition_col}"
        )

    def write_batch(self, records: list[dict]) -> None:
        """
        Accept a batch of records and group them by the partition column value.
        Records missing the partition column are placed in a '_null' partition.
        """
        for record in records:
            partition_value = str(record.get(self.partition_col, "_null"))
            self._partitions[partition_value].append(record)

        print(
            f"Batch of {len(records)} records accepted. "
            f"Active partitions: {list(self._partitions.keys())}"
        )

    def finalize(self) -> dict:
        """
        Write all buffered records to their respective Parquet files.
        Returns a summary of what was written.
        """
        if not self._partitions:
            print("No records to write.")
            return {}

        summary = {}
        for partition_value, records in self._partitions.items():
            partition_dir = (
                self.output_dir / f"{self.partition_col}={partition_value}"
            )
            partition_dir.mkdir(parents=True, exist_ok=True)
            output_file = partition_dir / "part-0.parquet"

            all_keys = set()
            for r in records:
                all_keys.update(r.keys())

            columns = {key: [r.get(key) for r in records] for key in all_keys}
            table = pa.table(columns)
            pq.write_table(table, str(output_file), compression="snappy")

            self._files_written[partition_value] = str(output_file)
            summary[partition_value] = len(records)

        print(f"\nFinalize complete. Wrote {len(summary)} partition files:")
        for partition_value, count in sorted(summary.items()):
            file_path = self._files_written[partition_value]
            print(f"  {self.partition_col}={partition_value}: {count} records -> {file_path}")

        total_records = sum(summary.values())
        print(f"Total records written: {total_records}")
        return summary


writer = PartitionedParquetWriter("data/partitioned_output", "city")

batch_1 = [
    {"student_id": 1, "name": "Arjun Mehta",  "city": "Mumbai",  "gpa": 8.5},
    {"student_id": 2, "name": "Priya Sharma",  "city": "Delhi",   "gpa": 9.1},
    {"student_id": 3, "name": "Rohan Verma",   "city": "Mumbai",  "gpa": 7.8},
]
batch_2 = [
    {"student_id": 4, "name": "Sneha Iyer",   "city": "Chennai", "gpa": 8.9},
    {"student_id": 5, "name": "Karan Patel",  "city": "Mumbai",  "gpa": 6.5},
    {"student_id": 6, "name": "Divya Nair",   "city": "Delhi",   "gpa": 9.4},
]

writer.write_batch(batch_1)
writer.write_batch(batch_2)
summary = writer.finalize()

print("\nVerifying Mumbai partition:")
mumbai_data = pq.read_table("data/partitioned_output/city=Mumbai/part-0.parquet")
print(mumbai_data.to_pydict())
```

Lesson 3 is complete. Tell me when you are ready for Lesson 4.