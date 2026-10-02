
## Phase 2, Lesson 4 — Pandas for Data Engineering

---

### Why Pandas exists — the problem before it

Before Pandas, doing tabular data manipulation in Python looked like this:

```python
students = [
    {"student_id": 1, "name": "Arjun", "city": "Mumbai", "gpa": 8.5},
    {"student_id": 2, "name": "Priya", "city": "Delhi",  "gpa": 9.1},
    {"student_id": 3, "name": "Rohan", "city": "Mumbai", "gpa": 7.8},
]

# Find average GPA of Mumbai students
mumbai_gpas = [s["gpa"] for s in students if s["city"] == "Mumbai"]
average = sum(mumbai_gpas) / len(mumbai_gpas)

# Add a new column -- has to be done manually for every row
for student in students:
    student["is_high_achiever"] = student["gpa"] >= 8.5

# Sort by GPA -- verbose
sorted_students = sorted(students, key=lambda s: s["gpa"], reverse=True)
```

Each operation requires a loop. Filtering requires a loop. Computing aggregates requires a loop. Adding derived columns requires a loop. Sorting is manageable but merging two datasets by a common key requires nested loops. On millions of records, Python loops are catastrophically slow because Python executes each iteration in the interpreter — there is no opportunity for the kind of bulk hardware optimization that modern CPUs are designed for.

Wes McKinney created Pandas in 2008 while working at a quantitative finance firm, specifically because he was frustrated with this exact problem. He needed to manipulate large tabular datasets — financial time series with millions of rows — and Python's existing tools were inadequate.

Pandas solves two distinct problems simultaneously.

The first is the **expressiveness problem**. SQL lets you express "give me the average GPA per city" in one line. Python lists-of-dicts require multiple lines and explicit loops. Pandas gives Python the expressive power of SQL-like operations — filtering, grouping, aggregating, joining — without leaving Python.

The second is the **performance problem**. Pandas is built on NumPy, which stores data in contiguous blocks of typed memory and executes operations using compiled C code on entire arrays at once rather than one element at a time in Python. A Pandas operation on a million rows executes one C function call on a million-element array. A Python loop executes one million Python interpreter iterations. The difference is commonly 10x to 100x in speed.

Understanding both of these — the expressiveness and the performance model — is what allows you to use Pandas correctly in data engineering pipelines and to know when it is the wrong tool.

---

### The fundamental data structure — what a DataFrame actually is

A DataFrame is not a new kind of thing. It is a specific arrangement of things you already know.

At the lowest level, a DataFrame is a **collection of NumPy arrays**, all of the same length, each corresponding to one column. These arrays are stored in contiguous memory — all values of one column are physically adjacent in RAM, exactly like Parquet's columnar storage on disk.

But arrays alone are just indexed sequences. Pandas adds a critical layer: an **index**. The index is a separate array that labels each row. By default it is integers: 0, 1, 2, 3... But it can be any hashable values — dates, strings, customer IDs, anything.

So a DataFrame is three things working together:

- A collection of column arrays (the data)
- An index that labels rows (the row identifier)
- Column labels that label columns (the column identifier)

The analogy to a database table is close but imperfect. A database table has no concept of row order — rows have no position, only identity through keys. A Pandas DataFrame has both position (row 0 comes before row 1) and identity (the index label). This distinction matters for how you select data.

A **Series** is one column — a single array with an index. A DataFrame is a collection of Series that share the same index. Every operation you do on a DataFrame is ultimately an operation on one or more Series.

```python
import pandas as pd

# A Series is a labeled one-dimensional array
gpa_series = pd.Series([8.5, 9.1, 7.8, 8.9, 6.5],
                        index=[1, 2, 3, 4, 5],
                        name="gpa")

print(gpa_series)
print(type(gpa_series))
print(gpa_series.dtype)    # float64 -- the NumPy type of the underlying array
print(gpa_series.values)   # the raw NumPy array
print(gpa_series.index)    # the index
```

Run this in a new file called `lesson4.py`. The output shows you the Series as a labeled sequence. The `dtype` is the NumPy data type of the underlying array — `float64` means 64-bit floating point numbers. The `values` attribute gives you the raw NumPy array directly. The `index` is a separate array of labels.

```python
# A DataFrame is a collection of Series sharing an index
students_df = pd.DataFrame({
    "student_id": [1,        2,        3,        4,        5       ],
    "name":       ["Arjun",  "Priya",  "Rohan",  "Sneha",  "Karan" ],
    "city":       ["Mumbai", "Delhi",  "Mumbai", "Chennai","Ahmedabad"],
    "gpa":        [8.5,      9.1,      7.8,      8.9,      None    ],
    "age":        [20,       21,       19,       22,       20      ],
})

print(students_df)
print()
print(students_df.dtypes)   # type of each column
print()
print(students_df.index)    # the row index -- default is 0,1,2,3,4
print()
print(students_df.columns)  # the column labels
```

The `dtypes` output is important. `student_id` and `age` are `int64`. `name` and `city` are `object` — Pandas uses `object` dtype for columns containing Python strings because Python strings are variable-length and cannot be stored in fixed-width NumPy arrays. `gpa` is `float64`.

Notice that the `None` in gpa was converted to `NaN` — Pandas' representation of missing values. We will examine this carefully.

The default index is `RangeIndex(start=0, stop=5, step=1)` — essentially the range 0 through 4. This is an efficient representation that does not actually store all the values explicitly.

---

### The index — understanding what it actually does

The index is the feature of Pandas that confuses most beginners and causes the most bugs. Let us examine it carefully.

```python
# Create a DataFrame with a meaningful index
students_df = pd.DataFrame({
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5],
}, index=[101, 102, 103, 104, 105])  # student IDs as index

print(students_df)
print()
print(students_df.loc[103])     # access row by index label
print()
print(students_df.iloc[2])      # access row by position (2 = third row)
```

`loc` uses **label-based** selection. `students_df.loc[103]` means "give me the row whose index label is 103." It returns the row for Rohan.

`iloc` uses **position-based** selection. `students_df.iloc[2]` means "give me the row at position 2" (the third row, since positions start at 0). It also returns the row for Rohan — they happen to be the same row in this case, but that is only because the index label 103 corresponds to position 2.

The critical difference emerges when rows are reordered or filtered:

```python
# Filter to only Mumbai students
mumbai_df = students_df[students_df["city"] == "Mumbai"]
print(mumbai_df)
print()

# iloc uses current position in the filtered result
print(mumbai_df.iloc[0])   # Arjun -- first row of the filtered DataFrame

# loc uses the original index label
print(mumbai_df.loc[101])  # Arjun -- by label
print(mumbai_df.loc[103])  # Rohan -- by label
```

After filtering, `mumbai_df` contains rows with index labels 101 and 103. Position 0 is Arjun (label 101). Position 1 is Rohan (label 103). The labels from the original DataFrame are preserved. The positions have changed.

This is the source of many bugs. After filtering, slicing, or sorting a DataFrame, the positions change but the labels stay the same. If you assume `iloc[0]` always gives you a specific row regardless of prior operations, you will get wrong results.

The general rule: **use `loc` when you know the label you want. Use `iloc` when you know the position you want.** In data engineering, labels are usually more stable and meaningful than positions, so `loc` is more common in practice.

```python
# Resetting the index -- often needed after filtering
mumbai_df_reset = mumbai_df.reset_index(drop=False)
print(mumbai_df_reset)
```

`reset_index(drop=False)` creates a new sequential integer index starting from 0 and moves the old index into a column. `drop=True` discards the old index instead of keeping it. After a filter operation, if you do not need the original labels, resetting the index gives you a clean 0, 1, 2... sequence.

---

### NaN — Pandas' missing value, examined deeply

In Lesson 1 of Phase 1, we examined NULL in SQL extensively. Pandas has an analogous concept — NaN — but it behaves differently in important ways.

NaN stands for "Not a Number." It originated in the IEEE 754 floating point standard as a special value representing the result of undefined arithmetic operations like `0/0` or `sqrt(-1)`. Pandas adopted it as the missing value representation because it is a native floating point value that the underlying NumPy arrays can store without any special handling.

This has a consequence: **NaN is always a float.** When a column with integer values gets a missing value, Pandas must convert the entire column to float64 to accommodate NaN. The integer column `[1, 2, None, 4]` becomes `[1.0, 2.0, NaN, 4.0]`.

```python
import pandas as pd
import numpy as np

# NaN is a floating point value
print(type(np.nan))    # float
print(np.nan)          # nan

# The unusual arithmetic of NaN
print(np.nan == np.nan)    # False -- NaN is not equal to anything, including itself
print(np.nan > 5)          # False
print(np.nan + 1)          # nan -- any arithmetic with NaN produces NaN
print(np.nan is np.nan)    # True -- there is only one NaN object
```

The `np.nan == np.nan` returning `False` is identical to SQL's `NULL = NULL` returning UNKNOWN. A missing value is not equal to anything, not even another missing value. This is mathematically and logically correct — if you do not know a value, you cannot claim it equals another unknown value.

The practical consequence: **you can never test for NaN with `==`**.

```python
students_df = pd.DataFrame({
    "name": ["Arjun", "Priya", "Rohan"],
    "gpa": [8.5, None, 7.8],
})

# Wrong -- always returns False even for NaN values
mask = students_df["gpa"] == np.nan
print(mask)   # False, False, False

# Correct -- use pd.isna() or pd.notna()
mask = pd.isna(students_df["gpa"])
print(mask)   # False, True, False

mask = pd.notna(students_df["gpa"])
print(mask)   # True, False, True
```

`pd.isna()` and `pd.notna()` are the Pandas equivalents of SQL's `IS NULL` and `IS NOT NULL`. Use them exclusively for missing value checks.

Pandas also provides `.isna()` and `.notna()` as methods on Series and DataFrames:

```python
print(students_df["gpa"].isna())
print(students_df.isna())          # shows missing values across entire DataFrame
print(students_df.isna().sum())    # count of missing values per column
```

How aggregate functions handle NaN in Pandas:

```python
students_df = pd.DataFrame({
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "gpa": [8.5, 9.1, 7.8, None, 6.5],
})

print(students_df["gpa"].sum())    # 31.9 -- NaN excluded automatically
print(students_df["gpa"].mean())   # 7.975 -- NaN excluded, divides by 4 not 5
print(students_df["gpa"].count())  # 4 -- counts non-NaN values only
print(len(students_df))            # 5 -- total rows including NaN
```

This is the same behavior as SQL's aggregate functions — NaN is silently excluded from calculations. The same data quality judgment applies: if NaN means "data not collected," the mean of the remaining values may be misleading. If it means "student has no GPA yet," excluding them from the average may be correct.

Filling or dropping NaN:

```python
# Replace NaN with a specific value
filled = students_df["gpa"].fillna(0.0)
print(filled)

# Replace NaN with the column mean
mean_gpa = students_df["gpa"].mean()
filled_with_mean = students_df["gpa"].fillna(mean_gpa)
print(filled_with_mean)

# Drop rows that have any NaN value
dropped = students_df.dropna()
print(dropped)

# Drop rows where specific columns are NaN
dropped_gpa = students_df.dropna(subset=["gpa"])
print(dropped_gpa)
```

In data engineering, the decision between filling and dropping NaN is a business logic decision, not a technical one. Dropping removes data. Filling can introduce bias if the fill value is not representative. Always make this decision explicitly and document it — never let NaN propagate silently through a pipeline.

---

### Selecting data — the four methods and when to use each

Selecting specific rows, columns, or combinations is something you do in almost every transformation. There are four methods and they are not interchangeable.

**Column selection by label:**

```python
students_df = pd.DataFrame({
    "student_id": [1, 2, 3, 4, 5],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad"],
    "gpa": [8.5, 9.1, 7.8, 8.9, None],
    "age": [20, 21, 19, 22, 20],
})

# Select one column -- returns a Series
gpa_series = students_df["gpa"]
print(type(gpa_series))   # Series

# Select multiple columns -- returns a DataFrame
subset_df = students_df[["name", "gpa"]]
print(type(subset_df))    # DataFrame
```

Single column with `["column"]` returns a Series. Double brackets `[["col1", "col2"]]` returns a DataFrame. The double brackets are passing a list to the selection operator — the outer brackets are the selection, the inner brackets define the list.

**Row selection with boolean masks:**

This is the most important selection pattern. A boolean mask is a Series of True/False values, one per row. When used to select rows, it returns only the rows where the mask is True.

```python
# Boolean mask -- one True/False per row
high_gpa_mask = students_df["gpa"] > 8.0
print(high_gpa_mask)

# Apply the mask to select rows
high_achievers = students_df[high_gpa_mask]
print(high_achievers)

# Usually written inline
high_achievers = students_df[students_df["gpa"] > 8.0]
```

The `students_df["gpa"] > 8.0` expression compares every value in the gpa column to 8.0 simultaneously using NumPy's vectorized operations. The result is a boolean array. Passing this array to `students_df[...]` returns only the rows where the value is True. No Python loop is involved anywhere in this operation.

Combining conditions:

```python
# AND -- both conditions must be True
mumbai_high = students_df[
    (students_df["city"] == "Mumbai") & (students_df["gpa"] > 8.0)
]

# OR -- either condition can be True
mumbai_or_delhi = students_df[
    (students_df["city"] == "Mumbai") | (students_df["city"] == "Delhi")
]

# NOT
not_mumbai = students_df[~(students_df["city"] == "Mumbai")]
```

The parentheses around each condition are mandatory. Pandas uses `&` for AND, `|` for OR, and `~` for NOT — the bitwise operators, not Python's `and`, `or`, `not`. The reason is that `and`, `or`, `not` cannot be overloaded for NumPy arrays in the way Pandas needs — they always return a single boolean, not a boolean array. The bitwise operators work element-wise on arrays and can be overloaded.

**`loc` — label-based selection for both rows and columns:**

```python
# Select specific rows by label AND specific columns by name
subset = students_df.loc[1:3, ["name", "gpa"]]
print(subset)

# Single row by label
row = students_df.loc[2]
print(row)

# All rows, specific columns
cols = students_df.loc[:, ["name", "city"]]
print(cols)

# Boolean mask with loc
high_achievers = students_df.loc[students_df["gpa"] > 8.0, ["name", "gpa"]]
print(high_achievers)
```

`loc[row_selector, column_selector]` takes two arguments separated by a comma. The first selects rows (by label, slice of labels, or boolean mask), the second selects columns (by name, list of names, or slice of names).

Important: `loc` slicing is **inclusive on both ends**. `loc[1:3]` returns rows with labels 1, 2, and 3. This is different from Python's normal slice behavior where `list[1:3]` excludes the end.

**`iloc` — position-based selection:**

```python
# Select by position
first_row = students_df.iloc[0]       # first row
last_row = students_df.iloc[-1]       # last row

# Slice by position -- exclusive end, like Python slices
first_three = students_df.iloc[0:3]   # rows at positions 0, 1, 2 (not 3)

# Rows and columns by position
subset = students_df.iloc[0:3, 1:3]   # rows 0-2, columns 1-2
```

`iloc` follows Python's normal slice conventions — the end is exclusive. `iloc[0:3]` returns positions 0, 1, 2. This is the opposite of `loc`'s inclusive slicing.

---

### The copy vs view problem — one of Pandas' most dangerous traps

This is the most important Pandas concept for data engineering that most tutorials skip. It has caused real data corruption bugs in production pipelines.

When you perform a selection operation on a DataFrame, Pandas may return either a **view** of the original data or a **copy** of it. A view shares memory with the original — modifying the view modifies the original. A copy is independent — modifying it has no effect on the original.

The problem: Pandas does not always tell you which one you got. And the rules for when you get a view versus a copy are complex and have changed across Pandas versions.

```python
students_df = pd.DataFrame({
    "name": ["Arjun", "Priya", "Rohan"],
    "city": ["Mumbai", "Delhi", "Mumbai"],
    "gpa": [8.5, 9.1, 7.8],
})

# This MIGHT modify the original -- or it might not -- undefined behavior
mumbai = students_df[students_df["city"] == "Mumbai"]
mumbai["gpa"] = 0.0   # SettingWithCopyWarning -- are you modifying the original?

print(students_df)   # did the original change? maybe, maybe not
```

Pandas issues a `SettingWithCopyWarning` warning because it cannot guarantee whether `mumbai` is a view or a copy. Depending on the specific operation and the version of Pandas, `students_df` might or might not be modified.

This non-deterministic behavior is unacceptable in data engineering pipelines that run unattended. The solution is explicit:

```python
# Always use .copy() when you intend to modify independently
mumbai = students_df[students_df["city"] == "Mumbai"].copy()
mumbai["gpa"] = 0.0   # no warning, definitely does not affect original

print(students_df)   # unchanged -- guaranteed
print(mumbai)        # modified
```

The `.copy()` call creates an independent copy. Any modification to `mumbai` cannot affect `students_df` because they do not share memory.

**The rule:** Whenever you create a subset DataFrame that you intend to modify, always call `.copy()` immediately. Never rely on Pandas deciding whether to give you a view or a copy.

**Pandas 2.0 and Copy-on-Write:** Pandas 2.0 introduced Copy-on-Write as an opt-in feature, which became the default in Pandas 3.0. With Copy-on-Write, all indexing operations return views that become independent copies only when modified. This eliminates the SettingWithCopyWarning entirely. The behavior is cleaner and more predictable. You are using Python 3.12 which likely has a recent Pandas version — but the habit of using `.copy()` explicitly is still good practice for clarity.

---

### Adding and modifying columns — the right way

```python
students_df = pd.DataFrame({
    "student_id": [1, 2, 3, 4, 5],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "gpa": [8.5, 9.1, 7.8, 8.9, None],
    "age": [20, 21, 19, 22, 20],
})

# Add a new column -- vectorized, operates on entire column at once
students_df["is_adult"] = students_df["age"] >= 18

# Add column based on another column
students_df["gpa_rounded"] = students_df["gpa"].round(1)

# Add column based on multiple columns
students_df["age_gpa_product"] = students_df["age"] * students_df["gpa"]

print(students_df)
```

Every one of these operations is vectorized — it operates on the entire column array simultaneously using compiled C code. No Python loop is involved.

**Conditional columns using `np.where`:**

```python
import numpy as np

students_df["category"] = np.where(
    students_df["gpa"] >= 8.5,
    "high_achiever",
    "standard"
)
print(students_df[["name", "gpa", "category"]])
```

`np.where(condition, value_if_true, value_if_false)` applies a conditional element-wise across the entire column. This is the Pandas equivalent of SQL's CASE WHEN expression.

For multiple conditions, use `np.select`:

```python
conditions = [
    students_df["gpa"] >= 9.0,
    students_df["gpa"] >= 8.0,
    students_df["gpa"] >= 7.0,
    students_df["gpa"].notna(),
]
choices = [
    "outstanding",
    "good",
    "satisfactory",
    "needs_improvement",
]

students_df["gpa_category"] = np.select(conditions, choices, default="ungraded")
print(students_df[["name", "gpa", "gpa_category"]])
```

`np.select` evaluates conditions in order and assigns the corresponding choice for the first True condition. If no condition is True, the `default` value is used. The `default="ungraded"` handles the case where gpa is NaN — none of the conditions involving NaN comparisons will be True.

---

### Groupby — the most powerful Pandas operation for data engineers

GroupBy is the Pandas equivalent of SQL's GROUP BY. It splits the DataFrame into groups based on one or more column values, applies an operation within each group, and combines the results.

But there is something important to understand about how Pandas GroupBy actually works, because it affects how you use it.

When you call `groupby()`, Pandas does not immediately compute anything. It creates a **GroupBy object** — a lazy structure that records which column to group by and remembers the original DataFrame. The computation happens only when you apply an aggregation function.

```python
students_df = pd.DataFrame({
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan", "Divya"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Mumbai", "Delhi"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5, 9.4],
    "age": [20, 21, 19, 22, 20, 23],
})

# GroupBy object -- no computation yet
grouped = students_df.groupby("city")
print(type(grouped))   # DataFrameGroupBy

# Computation happens here
city_stats = grouped["gpa"].agg(["mean", "max", "min", "count"])
print(city_stats)
```

The `agg()` method applies named aggregation functions to the column. You can pass a list of function names as strings, or a dictionary mapping column names to functions.

```python
# Multiple columns, different aggregations per column
city_summary = students_df.groupby("city").agg(
    avg_gpa=("gpa", "mean"),
    max_gpa=("gpa", "max"),
    student_count=("name", "count"),
    avg_age=("age", "mean"),
)
print(city_summary)
```

The named aggregation syntax — `result_column_name=(source_column, function)` — is the modern, readable way to do GroupBy aggregations. The result column names are exactly what you specify. This is equivalent to SQL:

```sql
SELECT city,
    AVG(gpa) AS avg_gpa,
    MAX(gpa) AS max_gpa,
    COUNT(name) AS student_count,
    AVG(age) AS avg_age
FROM students
GROUP BY city;
```

After GroupBy aggregation, the grouped column becomes the index of the result:

```python
print(city_summary.index)   # Index(['Ahmedabad', 'Chennai', 'Delhi', ...])
```

If you want the grouped column as a regular column instead of the index, either call `reset_index()` after aggregation or pass `as_index=False` to `groupby()`:

```python
city_summary = students_df.groupby("city", as_index=False).agg(
    avg_gpa=("gpa", "mean"),
    student_count=("name", "count"),
)
print(city_summary)   # city is now a regular column
```

**GroupBy with transform — adding group-level values back to each row:**

This is the Pandas equivalent of SQL window functions with PARTITION BY. `transform` applies a function within each group but returns a result the same size as the original DataFrame, aligned with the original rows.

```python
# Add city average GPA back to each row -- like AVG(gpa) OVER (PARTITION BY city)
students_df["city_avg_gpa"] = students_df.groupby("city")["gpa"].transform("mean")
print(students_df[["name", "city", "gpa", "city_avg_gpa"]])
```

Each student's row now shows both their own GPA and their city's average GPA. The computation happened per city group but the result is aligned back to the original row positions. Compare this to the SQL window function `AVG(gpa) OVER (PARTITION BY city)` from Phase 1 — identical semantics, different syntax.

```python
# Add city rank -- like RANK() OVER (PARTITION BY city ORDER BY gpa DESC)
students_df["city_rank"] = (
    students_df.groupby("city")["gpa"]
    .rank(method="dense", ascending=False)
)
print(students_df[["name", "city", "gpa", "city_rank"]])
```

---

### Merging DataFrames — the Pandas equivalent of SQL JOINs

Merging combines two DataFrames based on common column values — exactly like SQL JOINs. Pandas uses the word "merge" for what SQL calls "JOIN."

```python
students_df = pd.DataFrame({
    "student_id": [1, 2, 3, 4, 5],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad"],
})

enrollments_df = pd.DataFrame({
    "student_id": [1, 1, 2, 3, 4],
    "course_id": [101, 102, 101, 102, 103],
    "grade": ["A", "B+", "A+", "B", "A"],
})

# INNER JOIN
inner = pd.merge(students_df, enrollments_df, on="student_id", how="inner")
print("Inner join:")
print(inner)
print()

# LEFT JOIN -- all students, even unenrolled
left = pd.merge(students_df, enrollments_df, on="student_id", how="left")
print("Left join:")
print(left)
```

The `how` parameter controls the join type exactly like SQL:

- `"inner"` — only rows with matches in both DataFrames
- `"left"` — all rows from the left DataFrame, matched rows from right (NaN where no match)
- `"right"` — all rows from the right DataFrame, matched rows from left
- `"outer"` — all rows from both DataFrames

The `on` parameter specifies the column to join on. When the join column has different names in the two DataFrames:

```python
students_df = pd.DataFrame({
    "id": [1, 2, 3],
    "name": ["Arjun", "Priya", "Rohan"],
})

enrollments_df = pd.DataFrame({
    "student_id": [1, 2, 4],
    "course": ["Math", "Science", "History"],
})

merged = pd.merge(
    students_df, enrollments_df,
    left_on="id", right_on="student_id",
    how="left"
)
print(merged)
```

After the merge, both `id` and `student_id` columns appear because they had different names. You can drop the duplicate:

```python
merged = merged.drop(columns=["student_id"])
```

**The row multiplication problem — identical to SQL JOINs:**

Exactly as in SQL, a JOIN in Pandas multiplies rows when there are multiple matches. A student enrolled in 3 courses will have 3 rows after merging students with enrollments. If you then compute the average GPA per city, that student's GPA is counted 3 times.

```python
# Wrong -- student GPA counted once per enrollment
wrong_avg = left.groupby("city")["gpa"].mean()
print("Wrong city averages:")
print(wrong_avg)

# Correct -- compute average from students before joining
correct_avg = students_df.groupby("city")["gpa"].mean()
print("Correct city averages:")
# (need city in students_df for this to work, adding it back)
```

The rule is the same as SQL: aggregate before joining when possible, or be conscious of row multiplication when joining before aggregating.

---

### Apply — when vectorization is not enough

Vectorized operations (arithmetic, comparisons, aggregations) are the fastest way to transform data in Pandas. But sometimes the transformation logic is complex enough that it cannot be expressed as a single vectorized operation. For those cases, `apply` lets you apply a custom function to each row or column.

```python
students_df = pd.DataFrame({
    "name": ["Arjun Mehta", "Priya Sharma", "rohan verma", "  Sneha Iyer  "],
    "gpa": [8.5, 9.1, 7.8, 8.9],
})

# Apply a function to each value in a column
def normalize_name(name: str) -> str:
    return " ".join(name.strip().split()).title()

students_df["name_clean"] = students_df["name"].apply(normalize_name)
print(students_df)
```

`apply` on a Series calls the function once per element. `apply` on a DataFrame with `axis=1` calls the function once per row, passing the row as a Series.

```python
def classify_student(row) -> str:
    """
    Classify a student based on multiple column values.
    Called once per row with axis=1.
    """
    if pd.isna(row["gpa"]):
        return "unclassified"
    if row["gpa"] >= 9.0:
        return f"outstanding (GPA: {row['gpa']:.1f})"
    if row["gpa"] >= 7.5:
        return "good"
    return "needs_improvement"

students_df["classification"] = students_df.apply(classify_student, axis=1)
print(students_df)
```

**The performance cost of `apply`:** `apply` with `axis=1` iterates through rows in Python — it is essentially a loop in disguise. On 1 million rows, it executes 1 million Python function calls. It is dramatically slower than vectorized operations.

The guidance:

- Use vectorized operations (`np.where`, `np.select`, arithmetic) whenever possible
- Use `apply` on a column (Series.apply) for element-wise transformations that cannot be vectorized — the function is called per element but Pandas can sometimes optimize these
- Use `apply(axis=1)` only for transformations that genuinely require multiple column values simultaneously and cannot be expressed with vectorized operations

```python
# Slow -- apply with axis=1
students_df["label"] = students_df.apply(
    lambda row: "high" if row["gpa"] >= 8.5 else "low", axis=1
)

# Fast -- np.where is vectorized
students_df["label"] = np.where(students_df["gpa"] >= 8.5, "high", "low")
```

Always prefer the vectorized version. The difference on large datasets is 10x to 100x.

---

### Reading and writing DataFrames

Pandas has built-in functions to read and write the file formats we covered in Lesson 3.

```python
import pandas as pd
from pathlib import Path

# Reading CSV
students_df = pd.read_csv(
    "data/students.csv",
    dtype={
        "student_id": "int32",
        "name": "string",
        "city": "string",
        "gpa": "float32",
    },
    na_values=["", "None", "NULL", "N/A"],   # what to treat as NaN
    encoding="utf-8",
)
print(students_df.dtypes)
```

The `dtype` parameter forces specific column types instead of letting Pandas infer them. Inference can be wrong and is slower. Specifying types explicitly makes your pipeline deterministic — the same file always produces the same types regardless of the data values.

The `na_values` parameter specifies which string values should be read as NaN. By default Pandas only treats empty strings as NaN. Specifying `"None"` and `"NULL"` catches the common representations we discussed in Lesson 3.

```python
# Writing CSV
students_df.to_csv(
    "data/students_output.csv",
    index=False,        # do not write the row index as a column
    encoding="utf-8",
    na_rep="",          # how to represent NaN -- empty string matches common convention
)
```

`index=False` is critical. Without it, Pandas writes the row index (0, 1, 2...) as the first column in the CSV. This is almost never what you want.

```python
# Reading Parquet
students_df = pd.read_parquet(
    "data/students.parquet",
    columns=["name", "gpa"],   # column pruning
)

# Writing Parquet
students_df.to_parquet(
    "data/students_out.parquet",
    compression="snappy",
    index=False,
)
```

Pandas delegates Parquet reading and writing to PyArrow (or optionally fastparquet) under the hood. The `columns` parameter in `read_parquet` triggers PyArrow's column pruning.

---

### When Pandas is the wrong tool

Pandas loads the entire dataset into RAM. It is designed for datasets that fit in memory. As a rough rule, Pandas works well for datasets up to about 1GB on a typical development machine — beyond that, you will hit memory pressure and the process will slow dramatically or crash.

In data engineering, you often deal with datasets far larger than this. When that happens, Pandas is the wrong tool and you need one of these alternatives:

**Polars** — a modern DataFrame library written in Rust. Same API style as Pandas but dramatically faster and memory-efficient through lazy evaluation. Growing rapidly in adoption for data engineering.

**Dask** — parallelizes Pandas operations across multiple cores or machines. Exposes a Pandas-like API but operates lazily on datasets larger than memory.

**Apache Spark (PySpark)** — the standard for truly large-scale data (tens of GB to petabytes). Distributes computation across a cluster of machines. We cover this in Phase 6.

The practical rule for data engineering: use Pandas for datasets under 1GB, for transformation logic that runs as part of a larger pipeline stage, and for exploratory analysis. Use Spark for anything larger.

---

### Exercises — Lesson 4

1. Load `data/students.csv` into a DataFrame. Do the following in sequence: cast all columns to appropriate types explicitly, handle NaN in the gpa column by filling with the median GPA of the same city (not the overall median — the city-specific median), add a column `gpa_vs_city_avg` showing how much each student's GPA differs from their city average (positive means above, negative means below), add a column `rank_in_city` using dense rank within each city ordered by GPA descending. Print the final DataFrame.
    
2. Create two DataFrames: `students_df` with the student data from Lesson 1 of Phase 1, and `enrollments_df` with the enrollment data. Perform a LEFT JOIN. After joining, find which students are not enrolled in any course. Then find the course (by course_id) in which the average GPA of enrolled students is highest. Show your work in steps, using a new variable for each intermediate result.
    
3. Write a function `clean_dataframe` that accepts a DataFrame and a configuration dictionary. The config dictionary maps column names to dictionaries specifying: the target dtype, what values to treat as null (a list of strings), and optionally a fill strategy for nulls ("drop", "mean", "median", or a specific value). The function should apply all cleaning steps and return a cleaned DataFrame. It must not modify the original. Test it on the students data.
    
4. Write a function `profile_dataframe` that accepts a DataFrame and returns a summary DataFrame with one row per column. For each column, show: column name, dtype, count of non-null values, count of null values, percentage null, number of unique values, and (for numeric columns) the min, mean, and max. For non-numeric columns, show None for those statistics. This is called data profiling and it is one of the first things you do on any new dataset in a real pipeline.
    

---

### Answers — Lesson 4

**Exercise 1**

```python
import pandas as pd
import numpy as np

df = pd.read_csv(
    "data/students.csv",
    dtype={"student_id": "int32", "name": "string", "city": "string"},
    na_values=["", "None", "NULL", "N/A"],
    encoding="utf-8",
)
df["gpa"] = df["gpa"].astype("float32")

# Fill NaN gpa with city-specific median
def fill_with_city_median(group):
    median = group.median()
    return group.fillna(median)

df["gpa"] = df.groupby("city")["gpa"].transform(fill_with_city_median)

# GPA vs city average
df["city_avg_gpa"] = df.groupby("city")["gpa"].transform("mean")
df["gpa_vs_city_avg"] = (df["gpa"] - df["city_avg_gpa"]).round(2)

# Rank within city
df["rank_in_city"] = (
    df.groupby("city")["gpa"]
    .rank(method="dense", ascending=False)
    .astype(int)
)

print(df.to_string())
```

**Exercise 2**

```python
import pandas as pd

students_df = pd.DataFrame({
    "student_id": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
    "name": ["Arjun Mehta", "Priya Sharma", "Rohan Verma", "Sneha Iyer",
             "Karan Patel", "Divya Nair", "Amit Joshi", "Meera Reddy",
             "Sahil Gupta", "Tanvi Desai"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad",
             "Delhi", "Pune", "Hyderabad", "Mumbai", "Pune"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5, 9.4, 7.2, 8.1, None, 7.6],
})

enrollments_df = pd.DataFrame({
    "student_id": [1, 1, 2, 2, 3, 4, 4, 5, 6, 6, 7, 8, 9, 10],
    "course_id": [101, 102, 101, 103, 102, 101, 103, 104, 101, 102, 105, 103, 101, 102],
    "grade": ["A", "B+", "A+", "A", "B", "A", "A+", "C", "A+", "A", "B", "A", "B+", "A"],
})

# Left join
merged = pd.merge(students_df, enrollments_df, on="student_id", how="left")
print("After left join:")
print(merged.head(10))
print()

# Students not enrolled in any course
not_enrolled = merged[merged["course_id"].isna()][["student_id", "name"]]
print("Not enrolled in any course:")
print(not_enrolled)
print()

# Course with highest average GPA of enrolled students
course_gpa = (
    pd.merge(enrollments_df, students_df[["student_id", "gpa"]], on="student_id")
    .groupby("course_id")["gpa"]
    .mean()
    .reset_index()
    .rename(columns={"gpa": "avg_gpa"})
    .sort_values("avg_gpa", ascending=False)
)
print("Courses by average GPA of enrolled students:")
print(course_gpa)
print()
print(f"Highest average GPA course: {course_gpa.iloc[0]['course_id']}")
```

**Exercise 3**

```python
import pandas as pd
import numpy as np


def clean_dataframe(df: pd.DataFrame, config: dict) -> pd.DataFrame:
    """
    Clean a DataFrame according to a column configuration.

    Config format:
    {
        "column_name": {
            "dtype": "float32",
            "null_values": ["", "None", "N/A"],
            "fill": "mean" | "median" | "drop" | <specific_value>
        }
    }

    Does not modify the original DataFrame.
    Returns a cleaned copy.
    """
    result = df.copy()

    for col, col_config in config.items():
        if col not in result.columns:
            print(f"Warning: column '{col}' not in DataFrame, skipping")
            continue

        null_values = col_config.get("null_values", [])
        if null_values:
            result[col] = result[col].replace(null_values, np.nan)

        target_dtype = col_config.get("dtype")
        if target_dtype:
            try:
                result[col] = result[col].astype(target_dtype)
            except (ValueError, TypeError) as e:
                print(f"Warning: could not cast '{col}' to {target_dtype}: {e}")

        fill = col_config.get("fill")
        if fill is not None:
            if fill == "drop":
                result = result.dropna(subset=[col])
            elif fill == "mean":
                result[col] = result[col].fillna(result[col].mean())
            elif fill == "median":
                result[col] = result[col].fillna(result[col].median())
            else:
                result[col] = result[col].fillna(fill)

    return result


raw_df = pd.DataFrame({
    "student_id": ["1", "2", "3", "4", "5"],
    "name": ["Arjun", "Priya", "rohan", "N/A", "Karan"],
    "gpa": ["8.5", "9.1", "None", "8.9", ""],
    "age": ["20", "21", "19", "None", "20"],
})

config = {
    "student_id": {"dtype": "int32"},
    "name": {"null_values": ["N/A", ""], "fill": "Unknown"},
    "gpa": {"dtype": "float32", "null_values": ["None", ""], "fill": "median"},
    "age": {"dtype": "float32", "null_values": ["None", ""], "fill": "mean"},
}

cleaned = clean_dataframe(raw_df, config)
print("Original:")
print(raw_df)
print("\nCleaned:")
print(cleaned)
print("\nOriginal unchanged:")
print(raw_df)
```

**Exercise 4**

```python
import pandas as pd
import numpy as np


def profile_dataframe(df: pd.DataFrame) -> pd.DataFrame:
    """
    Generate a data profile summary with one row per column.

    Returns a DataFrame with columns:
        column, dtype, non_null_count, null_count, pct_null,
        unique_count, min, mean, max
    """
    rows = []
    total_rows = len(df)

    for col in df.columns:
        series = df[col]
        non_null = series.notna().sum()
        null_count = series.isna().sum()
        pct_null = round((null_count / total_rows * 100), 2) if total_rows > 0 else 0.0
        unique_count = series.nunique(dropna=True)

        is_numeric = pd.api.types.is_numeric_dtype(series)

        row = {
            "column": col,
            "dtype": str(series.dtype),
            "non_null_count": int(non_null),
            "null_count": int(null_count),
            "pct_null": pct_null,
            "unique_count": int(unique_count),
            "min": round(float(series.min()), 4) if is_numeric and non_null > 0 else None,
            "mean": round(float(series.mean()), 4) if is_numeric and non_null > 0 else None,
            "max": round(float(series.max()), 4) if is_numeric and non_null > 0 else None,
        }
        rows.append(row)

    return pd.DataFrame(rows)


students_df = pd.read_csv(
    "data/students.csv",
    na_values=["", "None", "NULL"],
    encoding="utf-8",
)

profile = profile_dataframe(students_df)
print(profile.to_string(index=False))
```

Lesson 4 is complete. Tell me when you are ready for Lesson 5.