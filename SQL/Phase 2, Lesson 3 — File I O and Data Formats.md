
## Phase 2, Lesson 3 — File I/O and Data Formats

---

### Why this lesson is core data engineering, not a side topic

Every data engineering pipeline begins and ends with data moving between systems. Before data lives in a warehouse, before it flows through Kafka, before Spark processes it — it almost always starts as a file. CSV exports from databases. JSON responses from APIs. Parquet files in a data lake. Avro files from streaming systems.

Understanding file formats is not just about syntax for reading and writing. It is about making architectural decisions that affect pipeline performance, storage costs, and downstream query speed. A wrong format choice on a large dataset is not a minor inefficiency — it can mean the difference between a query that takes 3 seconds and one that takes 3 hours.

This lesson teaches you three things: how to handle files correctly in Python, what the major data formats are and how they work internally, and why you choose one over another in specific situations.

---

### Context managers — the foundation of all file and resource handling

Before opening a single file, you need to understand context managers completely. They appear in every file operation, every database connection, and every network call you write as a data engineer.

Consider this sequence without a context manager:

```python
f = open("data.csv", "r", encoding="utf-8")
data = f.read()
# imagine an exception occurs here
f.close()  # this line never runs if an exception happened above
```

If an exception occurs between opening and closing, the file remains open. In a long-running pipeline, this accumulates. Eventually the operating system hits its limit on open file handles and the pipeline crashes. The crash message will be confusing because it happens far from where the files were left open.

The `with` statement solves this:

```python
with open("data.csv", "r", encoding="utf-8") as f:
    data = f.read()
# file is closed here -- guaranteed -- even if an exception occurred inside
```

The `with` statement calls two special methods on the object. When entering the block it calls `__enter__`, which returns the resource to use (the file object, assigned to `f`). When leaving the block — whether normally or via exception — it calls `__exit__`, which closes the file.

You can implement this in your own classes. This is how database connection management works in data engineering:

```python
class ManagedConnection:
    
    def __init__(self, host: str, database: str):
        self.host = host
        self.database = database
        self._connection = None

    def __enter__(self):
        print(f"Opening connection to {self.database} on {self.host}")
        self._connection = f"connection_to_{self.database}"
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"Closing connection to {self.database}")
        self._connection = None
        if exc_type is not None:
            print(f"Connection closed after exception: {exc_val}")
        return False

    def execute(self, query: str) -> list:
        if self._connection is None:
            raise RuntimeError("Not connected")
        print(f"Executing: {query}")
        return []


with ManagedConnection("localhost", "production") as conn:
    results = conn.execute("SELECT * FROM students")
    print(f"Got {len(results)} results")

print("Outside the with block -- connection is closed")
```

`__exit__` receives three arguments about any exception that occurred: the exception type, the exception value, and the traceback. If no exception occurred, all three are `None`. Returning `False` from `__exit__` means do not suppress the exception — let it propagate normally. Returning `True` would suppress it. Almost always return `False`.

Create a new file called `lesson3.py` in your `de_learning` folder. All code for this lesson goes there.

---

### Reading and writing text files

```python
# Writing a file
with open("sample.txt", "w", encoding="utf-8") as f:
    f.write("Line one\n")
    f.write("Line two\n")
    f.write("Line three\n")

# Reading entire file at once
with open("sample.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# Reading line by line -- the correct way for large files
with open("sample.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())  # strip() removes the trailing newline character

# Appending to existing file
with open("sample.txt", "a", encoding="utf-8") as f:
    f.write("Line four\n")
```

The file mode is the second argument. The important modes:

`"r"` — read only. Raises `FileNotFoundError` if the file does not exist.

`"w"` — write. Creates the file if it does not exist. If it does exist, it is completely overwritten and the previous contents are gone. This is a common source of data loss in poorly written pipelines.

`"a"` — append. Creates the file if it does not exist. If it exists, new content is added after the existing content. Never overwrites.

`"x"` — exclusive creation. Creates the file but raises `FileExistsError` if it already exists. Useful when you want to guarantee you are not accidentally overwriting something.

`"rb"`, `"wb"` — binary read and write. Used for non-text files like images, Parquet files, and Avro files. Do not use with `encoding`.

**Always specify `encoding="utf-8"` explicitly.** Without it, Python uses the system's default encoding. On Windows this is typically `cp1252`. On Linux and Mac it is typically `utf-8`. A file written on Windows without specifying encoding and then read on a Linux cloud server may produce garbled text or an error. Specifying encoding makes your code portable.

**Never read a large file entirely into memory.** When you write `f.read()`, Python reads the entire file content into a single string in RAM. For a 10GB log file, that is 10GB of RAM consumed instantly. Iterate line by line instead — Python reads one line at a time, keeping memory usage constant regardless of file size:

```python
# This consumes memory proportional to file size -- dangerous for large files
with open("large_file.txt", "r", encoding="utf-8") as f:
    all_lines = f.read()

# This consumes memory proportional to one line -- safe for any file size
with open("large_file.txt", "r", encoding="utf-8") as f:
    for line in f:
        process(line.strip())
```

---

### Path handling — doing it the right way

File paths written as raw strings break across operating systems. Windows uses backslashes. Linux and Mac use forward slashes. Hardcoded paths make code non-portable.

```python
# Wrong -- breaks on Linux/Mac
path = "C:\Users\hp\data\students.csv"

# Also wrong -- backslash is an escape character in Python strings
# \n means newline, \t means tab, \U starts a unicode escape
path = "C:\new_folder\data.csv"  # \n and \d are interpreted

# Better -- raw string disables escape interpretation
path = r"C:\Users\hp\data\students.csv"

# Best -- use pathlib
from pathlib import Path

data_dir = Path("data")
students_file = data_dir / "students.csv"
output_file = Path("output") / "cleaned_students.csv"

print(students_file)        # data/students.csv on Linux, data\students.csv on Windows
print(students_file.name)   # students.csv
print(students_file.stem)   # students
print(students_file.suffix) # .csv
print(students_file.parent) # data
```

`pathlib.Path` handles path separators automatically for the current operating system. The `/` operator on Path objects joins path components — it does not divide. This looks odd at first but reads naturally: `data_dir / "students.csv"` means "the students.csv file inside data_dir."

```python
from pathlib import Path

# Create directories if they do not exist
output_dir = Path("output") / "processed"
output_dir.mkdir(parents=True, exist_ok=True)
# parents=True creates intermediate directories
# exist_ok=True does not raise error if directory already exists

# Check existence
data_file = Path("data.csv")
print(data_file.exists())   # True or False
print(data_file.is_file())  # True if it is a file
print(data_file.is_dir())   # True if it is a directory

# List all CSV files in a directory
data_dir = Path(".")
csv_files = list(data_dir.glob("*.csv"))
print(csv_files)

# Open using pathlib -- works exactly like a string path
with open(data_file, "r", encoding="utf-8") as f:
    content = f.read()
```

Use `pathlib.Path` for all path handling from this point forward. It is cleaner, more readable, and works correctly across operating systems without any special handling.

---

### CSV — the universal interchange format

CSV is everywhere. Every database, spreadsheet, and analytics tool can export and import it. It is also one of the worst formats for large-scale data engineering — no schema, no types, slow to parse, verbose, no compression. But you will deal with it constantly because the rest of the world uses it.

First, create some sample data to work with:

```python
import csv
from pathlib import Path

# Create the data directory
Path("data").mkdir(exist_ok=True)

students = [
    {"student_id": 1, "name": "Arjun Mehta",  "city": "Mumbai",    "gpa": 8.5},
    {"student_id": 2, "name": "Priya Sharma",  "city": "Delhi",     "gpa": 9.1},
    {"student_id": 3, "name": "Rohan Verma",   "city": "Mumbai",    "gpa": 7.8},
    {"student_id": 4, "name": "Sneha Iyer",    "city": "Chennai",   "gpa": 8.9},
    {"student_id": 5, "name": "Karan Patel",   "city": "Ahmedabad", "gpa": None},
]

# Writing CSV
with open("data/students.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["student_id", "name", "city", "gpa"])
    writer.writeheader()
    writer.writerows(students)

print("Written successfully")
```

`newline=""` is required on Windows when writing CSV. Without it, Python adds a `\r\n` line ending and then the `csv` module adds another `\r`, producing files with blank lines between every row. This is a Windows-specific quirk. Specifying `newline=""` disables Python's automatic line ending translation and lets the csv module handle it correctly.

`DictWriter` writes dictionaries as rows. It handles quoting automatically — if a field contains a comma or a newline, it wraps it in quotes. This is why you use the csv module instead of writing comma-separated strings manually.

Now read it back:

```python
# Reading CSV
with open("data/students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(dict(row))
        print(type(row["student_id"]))  # <class 'str'>
        print(type(row["gpa"]))         # <class 'str'>
```

Run this and look at the types. Every single value is a string. `row["student_id"]` is the string `"1"`, not the integer `1`. `row["gpa"]` is the string `"8.5"` or the string `"None"` or possibly an empty string, depending on how the None was serialized.

This is the fundamental limitation of CSV — it has no type information. You must convert every field to its correct type manually. This conversion step is called parsing and it is something every CSV-reading pipeline must do explicitly.

```python
def parse_student(row: dict) -> dict:
    """
    Convert a raw CSV row (all strings) into properly typed data.
    Handles missing/empty values gracefully.
    """
    gpa_str = row["gpa"].strip()

    return {
        "student_id": int(row["student_id"]),
        "name": row["name"].strip(),
        "city": row["city"].strip(),
        "gpa": float(gpa_str) if gpa_str and gpa_str.lower() != "none" else None,
    }


with open("data/students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    students_parsed = [parse_student(row) for row in reader]

for student in students_parsed:
    print(student)
    print(f"  GPA type: {type(student['gpa'])}")
```

Run this. Observe that GPA is now `float` or `None`, not a string. This is the kind of explicit parsing every CSV pipeline needs.

Now a more realistic scenario — what if a CSV has values that contain commas? For example, a name field with "Smith, John":

```python
# CSV with tricky values
tricky_data = [
    {"id": 1, "name": 'Smith, John', "note": 'Said "hello"'},
    {"id": 2, "name": "O'Brien, Mary", "note": "Normal note"},
]

with open("data/tricky.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["id", "name", "note"])
    writer.writeheader()
    writer.writerows(tricky_data)

# Read it back -- csv module handles the quoting automatically
with open("data/tricky.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)
```

Open `data/tricky.csv` in a text editor after running this. You will see that the csv module automatically wrapped the tricky values in quotes and escaped the internal quotes. Reading it back, `DictReader` correctly parses them. This is why you never manually split on commas to parse CSV — it breaks on these cases. Always use the csv module.

---

### JSON — for APIs and nested data

JSON is the standard format for API responses and configuration files. Unlike CSV, JSON preserves basic type information — numbers are numbers, strings are strings, booleans are booleans, null is null.

```python
import json
from pathlib import Path

pipeline_metadata = {
    "pipeline_name": "student_etl",
    "run_date": "2024-01-15",
    "config": {
        "batch_size": 1000,
        "max_retries": 3,
        "source": {
            "type": "csv",
            "path": "/data/students.csv"
        }
    },
    "stats": {
        "rows_read": 5,
        "rows_written": 4,
        "rows_failed": 1
    },
    "tags": ["daily", "students", "production"],
    "active": True
}

# Write JSON to file
with open("data/pipeline_run.json", "w", encoding="utf-8") as f:
    json.dump(pipeline_metadata, f, indent=2)

# Read JSON from file
with open("data/pipeline_run.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)

print(loaded["pipeline_name"])
print(loaded["config"]["batch_size"])
print(type(loaded["stats"]["rows_read"]))  # <class 'int'> -- not a string
print(type(loaded["active"]))              # <class 'bool'>
```

The four JSON functions you use constantly:

`json.dump(obj, file)` — write a Python object to a file as JSON.  
`json.load(file)` — read JSON from a file into a Python object.  
`json.dumps(obj)` — convert a Python object to a JSON string (s = string).  
`json.loads(string)` — parse a JSON string into a Python object.

The `indent=2` parameter in `json.dump` makes the output human-readable with 2-space indentation. Without it, the entire JSON is written on one line — valid JSON but unreadable. Use `indent=2` when writing files humans will read. Omit it for maximum compactness in machine-to-machine communication.

Now the critical trap. JSON cannot serialize Python types that have no JSON equivalent:

```python
from datetime import datetime
from decimal import Decimal

broken_data = {
    "timestamp": datetime.now(),  # Python datetime -- not JSON serializable
    "amount": Decimal("1234.56"), # Python Decimal -- not JSON serializable
}

try:
    json.dumps(broken_data)
except TypeError as e:
    print(f"Error: {e}")
```

Run this. You get `TypeError: Object of type datetime is not JSON serializable`.

This happens constantly in data engineering because PostgreSQL returns `datetime` objects for timestamp columns and `Decimal` objects for numeric columns. You need a custom encoder:

```python
import json
from datetime import datetime, date
from decimal import Decimal

class PipelineEncoder(json.JSONEncoder):
    """
    Custom JSON encoder that handles types common in data engineering.
    """
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()  # "2024-01-15T14:30:00.123456"
        if isinstance(obj, date):
            return obj.isoformat()  # "2024-01-15"
        if isinstance(obj, Decimal):
            return float(obj)
        if isinstance(obj, set):
            return list(obj)        # sets are not JSON serializable
        return super().default(obj)


data_with_special_types = {
    "run_timestamp": datetime(2024, 1, 15, 14, 30, 0),
    "run_date": date(2024, 1, 15),
    "amount": Decimal("1234.56"),
    "tags": {"daily", "production"},
}

serialized = json.dumps(data_with_special_types, cls=PipelineEncoder, indent=2)
print(serialized)
```

Pass `cls=PipelineEncoder` to any `json.dump` or `json.dumps` call to use the custom encoder. The `default` method is called for any object that the standard encoder does not know how to handle.

**JSON Lines format** — this is important for data engineering. Standard JSON cannot be streamed or processed line by line because the entire document must be parsed before any record is available. JSON Lines (also called NDJSON — Newline Delimited JSON) stores one JSON object per line. Each line is independently valid JSON.

```python
# Writing JSON Lines
records = [
    {"student_id": 1, "name": "Arjun", "gpa": 8.5},
    {"student_id": 2, "name": "Priya", "gpa": 9.1},
    {"student_id": 3, "name": "Rohan", "gpa": 7.8},
]

with open("data/students.jsonl", "w", encoding="utf-8") as f:
    for record in records:
        f.write(json.dumps(record) + "\n")

# Reading JSON Lines -- one line at a time, memory efficient
with open("data/students.jsonl", "r", encoding="utf-8") as f:
    for line in f:
        record = json.loads(line.strip())
        print(record)
```

JSON Lines is the format used by many data engineering tools for intermediate storage because it can be processed line by line without loading the entire file. Kafka, many ETL tools, and cloud data services use JSON Lines for their output files.

---

### Parquet — the format you will actually use in production

CSV and JSON are human-readable but inefficient for large-scale analytical work. Parquet is a binary columnar format designed specifically for analytical workloads. It is the dominant file format in modern data engineering — data lakes, Spark pipelines, BigQuery exports, and dbt materializations all use Parquet.

To understand why Parquet exists, you need to understand why column-oriented storage matters.

---

### Row storage vs column storage — the fundamental difference

Consider a table with 5 columns and 4 rows. Here is how it looks stored row by row (like CSV):

```
Row 1:  1, "Arjun",  "Mumbai",  8.5, 20
Row 2:  2, "Priya",  "Delhi",   9.1, 21
Row 3:  3, "Rohan",  "Mumbai",  7.8, 19
Row 4:  4, "Sneha",  "Chennai", 8.9, 22
```

On disk, all values for row 1 are physically adjacent. Then all values for row 2. And so on.

Now suppose a query needs only the `gpa` column across all rows. With row storage, the database must read all 5 columns for all 4 rows even though it only needs 1 column. For a table with 50 columns and 100 million rows, that means reading 50x more data than necessary.

In columnar storage (like Parquet), the same data is stored like this:

```
Column student_id: 1, 2, 3, 4
Column name:       "Arjun", "Priya", "Rohan", "Sneha"
Column city:       "Mumbai", "Delhi", "Mumbai", "Chennai"
Column gpa:        8.5, 9.1, 7.8, 8.9
Column age:        20, 21, 19, 22
```

All values for a column are physically adjacent on disk. A query reading only `gpa` reads only the gpa column's data — exactly 4 values — and skips the other 4 columns entirely. This is called **column pruning** and it is why analytical queries on Parquet are dramatically faster than on CSV.

**Why compression works better on columns.** When you store all values of a column together, they tend to be similar — all ages between 18 and 25, all cities from a small set of possible values. Compression algorithms exploit repetition and similarity. A column of 1 million cities where only 10 distinct values exist compresses to almost nothing. Interleaved row data has much less locality and compresses far less effectively.

A concrete example from real systems: a 1GB CSV file commonly becomes 50-100MB as Parquet with Snappy compression. The query that read 1GB from CSV reads 50MB from Parquet. Even ignoring the columnar access benefit, compression alone makes Parquet 10-20x faster for large analytical workloads.

---

### Working with Parquet using PyArrow

Install pyarrow first:

```
pip install pyarrow
```

```python
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

Path("data").mkdir(exist_ok=True)

# Define data as a dictionary of column arrays
data = {
    "student_id": [1,        2,        3,        4,        5       ],
    "name":       ["Arjun",  "Priya",  "Rohan",  "Sneha",  "Karan" ],
    "city":       ["Mumbai", "Delhi",  "Mumbai", "Chennai","Ahmedabad"],
    "gpa":        [8.5,      9.1,      7.8,      8.9,      None    ],
    "age":        [20,       21,       19,       22,       20      ],
}

# Define schema explicitly with types
schema = pa.schema([
    pa.field("student_id", pa.int32()),
    pa.field("name",       pa.string()),
    pa.field("city",       pa.string()),
    pa.field("gpa",        pa.float32()),
    pa.field("age",        pa.int16()),
])

# Create PyArrow table
table = pa.table(data, schema=schema)
print(table)
print(f"Rows: {table.num_rows}")
print(f"Columns: {table.num_columns}")
print(f"Schema: {table.schema}")
```

A PyArrow table is not a Pandas DataFrame. It is PyArrow's own in-memory columnar data structure. The data is stored column by column in memory, matching how it will be stored on disk.

The explicit schema definition is important. It forces specific types. `pa.int32()` means each student_id uses 4 bytes. `pa.int16()` means each age uses 2 bytes (max value 32767 — fine for age). Choosing smaller types reduces file size and improves cache efficiency. Compare to CSV where every value is a string regardless of its actual type.

```python
# Write Parquet with Snappy compression
pq.write_table(table, "data/students.parquet", compression="snappy")

# Check the file size
import os
csv_size = os.path.getsize("data/students.csv")
parquet_size = os.path.getsize("data/students.parquet")
print(f"CSV size:     {csv_size:,} bytes")
print(f"Parquet size: {parquet_size:,} bytes")
```

On your small 5-row dataset the sizes will be similar — the Parquet file may even be larger due to its header and metadata overhead. Parquet's advantages appear at scale. At millions of rows the size ratio reverses dramatically.

Compression options:

- `"snappy"` — fast compression and decompression. Prioritizes speed. The most common choice for data engineering pipelines where read speed matters more than maximum compression.
- `"gzip"` — better compression ratio, slower. Use when storage cost matters more than query speed.
- `"zstd"` — modern algorithm that balances speed and compression well. Increasingly common.
- `"none"` — no compression. Useful when the data will be immediately processed and never stored long-term.

```python
# Read entire Parquet file
loaded_table = pq.read_table("data/students.parquet")
print(loaded_table.to_pydict())

# Read only specific columns -- this is where Parquet shines
# Only the gpa and name columns are read from disk
partial = pq.read_table("data/students.parquet", columns=["name", "gpa"])
print(partial.to_pydict())

# Read with row filter -- only reads matching row groups
# (on small files this does not give a speed benefit, but at scale it does)
filtered = pq.read_table(
    "data/students.parquet",
    filters=[("gpa", ">", 8.0)]
)
print(filtered.to_pydict())
```

The `columns=` parameter is column pruning in action. The Parquet reader skips the student_id, city, and age columns entirely — they are not even read from disk. On a file with 50 columns and 100 million rows, this is transformative for performance.

The `filters=` parameter is predicate pushdown. The reader evaluates the filter as close to the storage as possible and skips entire row groups (blocks of rows) that cannot contain matching data. This is possible because Parquet stores min/max statistics for each column in each row group. If a row group's max gpa is 7.5 and you are filtering for gpa > 8.0, the entire row group is skipped without reading it.

---

### Parquet with Pandas — the most common pattern in practice

In practice, you will often use Pandas as an intermediary between your data and Parquet files. PyArrow handles the actual Parquet reading and writing under the hood, but Pandas provides the familiar DataFrame interface.

```python
import pandas as pd

# Write DataFrame to Parquet
students_df = pd.DataFrame({
    "student_id": [1, 2, 3, 4, 5],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad"],
    "gpa": [8.5, 9.1, 7.8, 8.9, None],
})

students_df.to_parquet("data/students_pandas.parquet", compression="snappy", index=False)
# index=False prevents writing the DataFrame index as a column

# Read Parquet into DataFrame
loaded_df = pd.read_parquet("data/students_pandas.parquet")
print(loaded_df)
print(loaded_df.dtypes)

# Read only specific columns
names_and_gpa = pd.read_parquet(
    "data/students_pandas.parquet",
    columns=["name", "gpa"]
)
print(names_and_gpa)
```

`index=False` is important. By default Pandas writes the DataFrame's row index (0, 1, 2, 3...) as a column in the Parquet file. This adds an unnecessary column that was never part of your data. Always use `index=False` unless you specifically need the index.

---

### Partitioned Parquet — how large datasets are organized

A single Parquet file works for small to medium datasets. For large datasets — millions or billions of rows — you need partitioning. Partitioning splits the data into multiple files organized by column values in a directory structure.

```python
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

data = {
    "student_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan", "Divya", "Amit", "Meera"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad", "Delhi", "Pune", "Hyderabad"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5, 9.4, 7.2, 8.1],
}

table = pa.table(data)

# Write partitioned by city
pq.write_to_dataset(
    table,
    root_path="data/students_partitioned",
    partition_cols=["city"]
)
```

After running this, look at the directory structure created inside `data/students_partitioned/`. You will see:

```
data/students_partitioned/
  city=Ahmedabad/
    part-0.parquet
  city=Chennai/
    part-0.parquet
  city=Delhi/
    part-0.parquet
  city=Hyderabad/
    part-0.parquet
  city=Mumbai/
    part-0.parquet
  city=Pune/
    part-0.parquet
```

Each city's data lives in its own directory and its own file. When you read with a filter for a specific city:

```python
# Only reads the Mumbai directory -- all other directories are skipped
mumbai_students = pq.read_table(
    "data/students_partitioned",
    filters=[("city", "=", "Mumbai")]
)
print(mumbai_students.to_pydict())
```

The reader looks at the directory names and immediately knows which directories to skip. It never opens the Delhi, Chennai, Pune, or other directories. For a dataset with 1 billion rows spread across 100 city directories, a query for one city reads 1/100th of the data.

This is **partition pruning** — one of the most important performance mechanisms in large-scale data engineering. Data lakes in S3, Azure Data Lake, and GCS organize data in exactly this structure. Spark, BigQuery, Athena, and other query engines all exploit partitioning automatically when you provide filters.

Choosing partition columns is a design decision with real consequences:

- Partition by columns you frequently filter on in queries
- Choose columns with reasonable cardinality — too many distinct values creates too many tiny files (hundreds of city files is fine, millions of user_id files is not)
- Partition by date in time-series data — almost universally the right choice for event data

---

### A realistic pipeline using all three formats

This brings the entire lesson together. A real pipeline reads from one format, transforms the data, and writes to another.

```python
import csv
import json
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path
from datetime import datetime

def read_csv_source(filepath: str) -> list[dict]:
    """Read CSV and return properly typed records."""
    records = []
    with open(filepath, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            gpa_str = row.get("gpa", "").strip()
            records.append({
                "student_id": int(row["student_id"]),
                "name": row["name"].strip(),
                "city": row["city"].strip(),
                "gpa": float(gpa_str) if gpa_str and gpa_str.lower() != "none" else None,
            })
    return records


def transform_records(records: list[dict]) -> list[dict]:
    """Clean and enrich records."""
    cleaned = []
    for record in records:
        if record["gpa"] is not None and not (0.0 <= record["gpa"] <= 10.0):
            print(f"Skipping invalid GPA: {record}")
            continue
        record["name"] = " ".join(record["name"].split()).title()
        record["city"] = record["city"].strip().title()
        record["is_high_achiever"] = record["gpa"] is not None and record["gpa"] >= 8.5
        cleaned.append(record)
    return cleaned


def write_parquet_output(records: list[dict], output_path: str) -> None:
    """Write cleaned records to Parquet."""
    if not records:
        print("No records to write")
        return

    table = pa.table({
        "student_id":       [r["student_id"] for r in records],
        "name":             [r["name"] for r in records],
        "city":             [r["city"] for r in records],
        "gpa":              [r["gpa"] for r in records],
        "is_high_achiever": [r["is_high_achiever"] for r in records],
    })

    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    pq.write_table(table, output_path, compression="snappy")
    print(f"Written {len(records)} records to {output_path}")


def write_run_metadata(stats: dict, metadata_path: str) -> None:
    """Write pipeline run metadata as JSON."""
    metadata = {
        "run_timestamp": datetime.now().isoformat(),
        "stats": stats,
    }
    with open(metadata_path, "w", encoding="utf-8") as f:
        json.dump(metadata, f, indent=2)
    print(f"Metadata written to {metadata_path}")


def run_pipeline(source_csv: str, output_parquet: str, metadata_json: str) -> None:
    print(f"Starting pipeline at {datetime.now().isoformat()}")

    raw = read_csv_source(source_csv)
    print(f"Read {len(raw)} records from CSV")

    cleaned = transform_records(raw)
    print(f"Transformed to {len(cleaned)} clean records")

    write_parquet_output(cleaned, output_parquet)

    stats = {
        "rows_read": len(raw),
        "rows_written": len(cleaned),
        "rows_dropped": len(raw) - len(cleaned),
    }
    write_run_metadata(stats, metadata_json)

    print("Pipeline complete")


run_pipeline(
    source_csv="data/students.csv",
    output_parquet="data/output/students_clean.parquet",
    metadata_json="data/output/run_metadata.json",
)

# Verify output
result = pq.read_table("data/output/students_clean.parquet")
print(result.to_pydict())
```

Run this end to end. Read the code carefully. Notice how each function does exactly one thing. Notice how the pipeline function orchestrates them in sequence without knowing their internal details. Notice how the context managers inside `read_csv_source` and `write_run_metadata` guarantee file closure. This structure — small focused functions composed into a pipeline — is the foundation of real data engineering code.

---

### Format comparison — when to use what

This is the decision you will make constantly as a data engineer.

**Use CSV when:**

- Receiving data from external systems that only support CSV
- Delivering data to non-technical stakeholders who open files in Excel
- Working with very small datasets where performance does not matter
- You need human readability for debugging

**Use JSON when:**

- Consuming or producing API responses
- Storing configuration and metadata
- The data is genuinely nested or semi-structured — different records have different fields
- You need human readability for debugging

**Use JSON Lines when:**

- Storing streaming data where records arrive one at a time
- You need to process a large collection of JSON records line by line without loading everything into memory

**Use Parquet when:**

- Storing any dataset larger than a few megabytes that will be queried analytically
- Writing output from a data pipeline to a data lake
- Any situation where read performance and storage efficiency matter
- You are working with Spark, BigQuery, Athena, Redshift Spectrum, or any modern analytical system — they all read Parquet natively and efficiently

As a data engineer, your default output format for any pipeline should be Parquet unless there is a specific reason to use something else.

---

### Exercises — Lesson 3

1. Write a function `read_csv_safe` that reads a CSV file and returns a list of dictionaries. It must handle three failure cases gracefully without crashing: the file does not exist, the file is empty, and a row has missing or malformed values. For malformed rows, skip them and collect the error messages. Return both the valid records and the list of error messages.
    
2. Write a function `json_to_parquet` that reads a JSON Lines file (one JSON object per line), converts the records to a PyArrow table with an explicit schema, and writes them as a Parquet file with Snappy compression. Test it by first writing a JSON Lines file with at least 5 records, then converting it.
    
3. Write a function `compare_formats` that takes a list of dictionaries, writes the same data to CSV, JSON, and Parquet, and prints the file size of each. Then reads each format back and verifies the data is identical. Print a clear summary showing the size difference.
    
4. Write a class `FileProcessor` that uses a context manager pattern (`__enter__` and `__exit__`). It should accept a source CSV path and an output directory. On entering the context, it reads and parses the CSV. On exiting normally, it writes the processed data to Parquet. On exiting due to an exception, it writes an error log as JSON instead. Test it with both a successful case and a simulated failure.
    

---

### Answers — Lesson 3

**Exercise 1**

```python
import csv
from pathlib import Path

def read_csv_safe(filepath: str) -> tuple[list[dict], list[str]]:
    """
    Read a CSV file safely, handling common failure cases.

    Returns:
        Tuple of (valid_records, error_messages)
    """
    errors = []
    records = []
    path = Path(filepath)

    if not path.exists():
        errors.append(f"File not found: {filepath}")
        return records, errors

    try:
        with open(path, "r", encoding="utf-8") as f:
            content = f.read().strip()
            if not content:
                errors.append(f"File is empty: {filepath}")
                return records, errors

        with open(path, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for line_num, row in enumerate(reader, start=2):
                try:
                    gpa_str = row.get("gpa", "").strip()
                    record = {
                        "student_id": int(row["student_id"]),
                        "name": row["name"].strip(),
                        "city": row.get("city", "").strip(),
                        "gpa": float(gpa_str) if gpa_str and gpa_str.lower() != "none" else None,
                    }
                    records.append(record)
                except (ValueError, KeyError) as e:
                    errors.append(f"Line {line_num}: {e} — raw row: {dict(row)}")

    except PermissionError:
        errors.append(f"Permission denied reading: {filepath}")

    return records, errors


records, errors = read_csv_safe("data/students.csv")
print(f"Valid records: {len(records)}")
for r in records:
    print(f"  {r}")
if errors:
    print(f"Errors: {len(errors)}")
    for e in errors:
        print(f"  {e}")

_, errors = read_csv_safe("nonexistent.csv")
print(errors)
```

**Exercise 2**

```python
import json
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

def write_jsonl(filepath: str, records: list[dict]) -> None:
    with open(filepath, "w", encoding="utf-8") as f:
        for record in records:
            f.write(json.dumps(record) + "\n")


def json_to_parquet(jsonl_path: str, parquet_path: str) -> int:
    """
    Read a JSON Lines file and write it as a Parquet file.

    Returns:
        Number of records converted
    """
    records = []
    with open(jsonl_path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                records.append(json.loads(line))

    if not records:
        print("No records found in JSON Lines file")
        return 0

    schema = pa.schema([
        pa.field("student_id", pa.int32()),
        pa.field("name",       pa.string()),
        pa.field("city",       pa.string()),
        pa.field("gpa",        pa.float32()),
    ])

    table = pa.table({
        "student_id": [r["student_id"] for r in records],
        "name":       [r["name"] for r in records],
        "city":       [r["city"] for r in records],
        "gpa":        [r.get("gpa") for r in records],
    }, schema=schema)

    Path(parquet_path).parent.mkdir(parents=True, exist_ok=True)
    pq.write_table(table, parquet_path, compression="snappy")
    print(f"Converted {len(records)} records from {jsonl_path} to {parquet_path}")
    return len(records)


sample_records = [
    {"student_id": 1, "name": "Arjun Mehta",  "city": "Mumbai",    "gpa": 8.5},
    {"student_id": 2, "name": "Priya Sharma",  "city": "Delhi",     "gpa": 9.1},
    {"student_id": 3, "name": "Rohan Verma",   "city": "Mumbai",    "gpa": 7.8},
    {"student_id": 4, "name": "Sneha Iyer",    "city": "Chennai",   "gpa": 8.9},
    {"student_id": 5, "name": "Karan Patel",   "city": "Ahmedabad", "gpa": None},
]

write_jsonl("data/students.jsonl", sample_records)
json_to_parquet("data/students.jsonl", "data/students_from_jsonl.parquet")

result = pq.read_table("data/students_from_jsonl.parquet")
print(result.to_pydict())
```

**Exercise 3**

```python
import csv
import json
import os
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path

def compare_formats(records: list[dict]) -> None:
    """
    Write records in CSV, JSON, and Parquet formats.
    Compare file sizes and verify data round-trips correctly.
    """
    Path("data/format_comparison").mkdir(parents=True, exist_ok=True)

    csv_path     = "data/format_comparison/data.csv"
    json_path    = "data/format_comparison/data.json"
    parquet_path = "data/format_comparison/data.parquet"

    # Write CSV
    with open(csv_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=records[0].keys())
        writer.writeheader()
        writer.writerows(records)

    # Write JSON
    with open(json_path, "w", encoding="utf-8") as f:
        json.dump(records, f, indent=2)

    # Write Parquet
    table = pa.table({
        key: [r[key] for r in records]
        for key in records[0].keys()
    })
    pq.write_table(table, parquet_path, compression="snappy")

    # Compare sizes
    csv_size     = os.path.getsize(csv_path)
    json_size    = os.path.getsize(json_path)
    parquet_size = os.path.getsize(parquet_path)

    print("Format size comparison:")
    print(f"  CSV:     {csv_size:,} bytes")
    print(f"  JSON:    {json_size:,} bytes")
    print(f"  Parquet: {parquet_size:,} bytes")

    # Verify CSV round-trip
    with open(csv_path, "r", encoding="utf-8") as f:
        csv_back = list(csv.DictReader(f))
    csv_names = [r["name"] for r in csv_back]
    original_names = [r["name"] for r in records]
    print(f"\nCSV round-trip names match: {csv_names == original_names}")

    # Verify JSON round-trip
    with open(json_path, "r", encoding="utf-8") as f:
        json_back = json.load(f)
    print(f"JSON round-trip count match: {len(json_back) == len(records)}")

    # Verify Parquet round-trip
    parquet_back = pq.read_table(parquet_path).to_pydict()
    print(f"Parquet round-trip names match: {parquet_back['name'] == original_names}")


sample = [
    {"student_id": 1, "name": "Arjun Mehta",  "city": "Mumbai",    "gpa": 8.5},
    {"student_id": 2, "name": "Priya Sharma",  "city": "Delhi",     "gpa": 9.1},
    {"student_id": 3, "name": "Rohan Verma",   "city": "Mumbai",    "gpa": 7.8},
    {"student_id": 4, "name": "Sneha Iyer",    "city": "Chennai",   "gpa": 8.9},
    {"student_id": 5, "name": "Karan Patel",   "city": "Ahmedabad", "gpa": 6.5},
]
compare_formats(sample)
```

**Exercise 4**

```python
import csv
import json
import pyarrow as pa
import pyarrow.parquet as pq
from pathlib import Path
from datetime import datetime

class FileProcessor:
    """
    Context manager that reads a CSV on entry and writes
    Parquet on clean exit or an error log on exception.
    """

    def __init__(self, source_csv: str, output_dir: str):
        self.source_csv = source_csv
        self.output_dir = Path(output_dir)
        self.records: list[dict] = []
        self._exception_occurred = False

    def __enter__(self):
        print(f"Reading source: {self.source_csv}")
        with open(self.source_csv, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f)
            for row in reader:
                gpa_str = row.get("gpa", "").strip()
                self.records.append({
                    "student_id": int(row["student_id"]),
                    "name": row["name"].strip(),
                    "city": row.get("city", "").strip(),
                    "gpa": float(gpa_str) if gpa_str and gpa_str.lower() != "none" else None,
                })
        print(f"Loaded {len(self.records)} records")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.output_dir.mkdir(parents=True, exist_ok=True)

        if exc_type is None:
            output_path = self.output_dir / "output.parquet"
            table = pa.table({
                "student_id": [r["student_id"] for r in self.records],
                "name":       [r["name"] for r in self.records],
                "city":       [r["city"] for r in self.records],
                "gpa":        [r["gpa"] for r in self.records],
            })
            pq.write_table(table, str(output_path), compression="snappy")
            print(f"Success: wrote {len(self.records)} records to {output_path}")
        else:
            error_log = {
                "timestamp": datetime.now().isoformat(),
                "error_type": exc_type.__name__,
                "error_message": str(exc_val),
                "records_loaded": len(self.records),
            }
            log_path = self.output_dir / "error_log.json"
            with open(log_path, "w", encoding="utf-8") as f:
                json.dump(error_log, f, indent=2)
            print(f"Failure: error log written to {log_path}")

        return False  # do not suppress the exception


# Successful case
with FileProcessor("data/students.csv", "data/processor_output") as processor:
    print(f"Processing {len(processor.records)} records")

# Simulated failure case
try:
    with FileProcessor("data/students.csv", "data/processor_output_error") as processor:
        print(f"Processing {len(processor.records)} records")
        raise ValueError("Simulated transformation error")
except ValueError as e:
    print(f"Caught expected error: {e}")
```

Lesson 3 complete. Tell me when you are ready for Lesson 4 — Pandas for Data Engineering.