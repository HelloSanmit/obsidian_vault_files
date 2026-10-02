
# Phase 2 — Pandas

## Lesson 4: Thinking in Tables, Not Methods

---

## Before we write any Pandas code

You already know how Python works with ordinary objects:

```python
students = [
    {"id": 1, "name": "Arjun", "gpa": 8.5},
    {"id": 2, "name": "Priya", "gpa": 9.1},
    {"id": 3, "name": "Rohan", "gpa": 7.8},
]
```

You can loop over these records.

You can write functions that transform them.

You can put them into files.

You can validate them.

You can even build a small ETL pipeline around them.

So why do we need Pandas?

Because once the data becomes genuinely tabular, repeatedly treating it as a collection of Python dictionaries becomes cumbersome.

Imagine 5 million rows.

You want to:

- select 3 columns
    
- filter rows
    
- calculate a derived column
    
- group by city
    
- calculate averages
    
- join against another dataset
    
- handle missing values
    
- convert timestamps
    
- write Parquet
    

You _can_ implement all of that yourself with lists, dictionaries and loops.

But then you would be rebuilding a miniature table-processing system.

Pandas already provides one.

The important thing is that Pandas is not magic.

It gives you a set of data structures and operations designed around a particular mental model.

**Learn that model first. Learn the methods second.**

---

# 1. Where Pandas fits in data engineering

Think about a typical pipeline:

```text
API / CSV / Database
        ↓
     Python
        ↓
     Pandas
        ↓
  clean / validate / transform
        ↓
     Parquet
        ↓
Warehouse / Lake / Spark / dbt
```

Pandas is particularly useful when the dataset is small enough to comfortably process on one machine.

That last part matters enormously.

Pandas is **not Spark**.

Pandas generally operates on data in the memory of one Python process.

Spark is designed to distribute computation across machines.

So don't build the mental model:

```text
Pandas = small Spark
```

A better model is:

```text
Python data structures
        ↓
Pandas = rich in-memory tabular processing
        ↓
SQL / warehouse / Spark = systems for larger-scale data processing
```

Pandas is incredibly useful for:

- ingestion
    
- cleaning
    
- exploratory inspection
    
- small-to-medium transformations
    
- API responses
    
- CSV/Excel processing
    
- preparing datasets for analytical systems
    
- building local prototypes
    
- validating pipeline logic before scaling it
    

But when the dataset is larger than practical memory, or the computation needs distributed execution, Pandas is no longer the right engine.

That distinction will become important when you eventually learn Spark.

---

# 2. The three objects you must understand

There are many objects in Pandas, but three ideas form the foundation:

```text
Series
DataFrame
Index
```

Everything else becomes easier after these.

---

# 3. A Series is one labeled column

Start with ordinary Python:

```python
gpas = [8.5, 9.1, 7.8, 8.9]
```

This is just a list.

Pandas gives that sequence labels:

```python
import pandas as pd

gpa = pd.Series(
    [8.5, 9.1, 7.8, 8.9],
    index=[101, 102, 103, 104]
)
```

Conceptually:

```text
index     value
------    -----
101        8.5
102        9.1
103        7.8
104        8.9
```

That is a `Series`.

A Series is therefore not simply “a Python list with extra features.”

The important conceptual difference is:

> **A Series is a one-dimensional labeled data structure.**

The labels matter.

You can inspect:

```python
print(gpa)
print(gpa.index)
print(gpa.values)
print(gpa.dtype)
```

The Series has:

```text
values
+
labels
+
type information
```

---

# 4. A DataFrame is a collection of aligned Series

Now suppose we have:

```python
students = pd.DataFrame({
    "student_id": [101, 102, 103, 104],
    "name": ["Arjun", "Priya", "Rohan", "Sneha"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai"],
    "gpa": [8.5, 9.1, 7.8, 8.9]
})
```

Conceptually:

```text
   student_id   name    city       gpa
0       101   Arjun   Mumbai      8.5
1       102   Priya   Delhi       9.1
2       103   Rohan   Mumbai      7.8
3       104   Sneha   Chennai     8.9
```

The DataFrame has two axes:

```text
rows    → index
columns → column labels
```

And each column is essentially a Series sharing the DataFrame's row index.

You can see this:

```python
print(type(students))
print(type(students["name"]))
```

The result is conceptually:

```text
DataFrame
Series
```

This gives us one of the most important Pandas ideas:

> **A DataFrame is a labeled 2-dimensional table made of columns, and those columns are aligned along an index.**

The official Pandas documentation describes the DataFrame and Series in terms of these labeled axes and dtype information.

---

# 5. The Index is not “just the row number”

This is where beginners often develop a wrong mental model.

You see:

```text
   name
0  Arjun
1  Priya
2  Rohan
```

and naturally think:

```text
0 = first row
1 = second row
2 = third row
```

That's not quite what the index means.

Those are **labels**.

They happen to be integers by default.

You could instead have:

```python
students.index = ["A", "B", "C", "D"]
```

Now:

```text
   name
A  Arjun
B  Priya
C  Rohan
D  Sneha
```

The index is part of the data model.

This becomes extremely important because Pandas often performs operations using **labels**, not merely positions.

---

# 6. The concept that separates beginners from comfortable Pandas users: alignment

Consider:

```python
a = pd.Series(
    [10, 20, 30],
    index=["A", "B", "C"]
)

b = pd.Series(
    [100, 200, 300],
    index=["C", "A", "B"]
)
```

Look carefully.

The values are stored in different positions.

But the labels tell us:

```text
a:
A → 10
B → 20
C → 30

b:
C → 100
A → 200
B → 300
```

Now:

```python
a + b
```

Pandas does not simply add:

```text
10 + 100
20 + 200
30 + 300
```

It aligns by label:

```text
A → 10 + 200 = 210
B → 20 + 300 = 320
C → 30 + 100 = 130
```

So the result is:

```text
A    210
B    320
C    130
```

This is one of the deepest ideas in Pandas.

> **Pandas is label-aware.**

You should constantly ask yourself:

> “Is this operation positional or label-based?”

That question will save you from a huge number of bugs.

Pandas explicitly documents this automatic alignment behavior, including when assigning a Series to a DataFrame column.

---

# 7. Why alignment is powerful

Imagine you receive:

```text
customer spending
```

and separately:

```text
customer credit limit
```

You don't necessarily need both datasets to have their rows in exactly the same order.

If they share a meaningful index:

```text
customer_id
```

Pandas can align the data using that label.

This is one reason the index exists.

It is not merely decoration.

---

# 8. But don't confuse Pandas Index with a database index

This distinction is important for a future data engineer.

A Pandas `Index` is primarily about **labels and alignment**.

A database index is a physical/logical structure used by a database engine to accelerate lookups.

They are related only in the very broad sense that both involve indexing concepts.

Don't conclude:

```text
Pandas Index = PostgreSQL index
```

It isn't.

---

# 9. Creating a DataFrame

The most common ways you'll encounter DataFrames are:

### From dictionaries

```python
df = pd.DataFrame({
    "id": [1, 2, 3],
    "name": ["Arjun", "Priya", "Rohan"]
})
```

### From records

```python
records = [
    {"id": 1, "name": "Arjun"},
    {"id": 2, "name": "Priya"},
    {"id": 3, "name": "Rohan"},
]

df = pd.DataFrame(records)
```

### From files

```python
df = pd.read_csv("students.csv")
```

### From Parquet

```python
df = pd.read_parquet("students.parquet")
```

In real data engineering, the last two are particularly important.

---

# 10. Inspection comes before transformation

One of the worst habits you can develop is:

```python
df = pd.read_csv(...)
df["something"] = ...
df = df.merge(...)
df = df.groupby(...)
```

without first understanding what you loaded.

Your first task after ingestion should usually be **inspection**.

```python
print(df.shape)
print(df.columns)
print(df.dtypes)
print(df.head())
```

Useful tools include:

```python
df.head()
df.tail()
df.sample(5)
df.shape
df.columns
df.index
df.dtypes
df.info()
df.describe()
```

Think of these as your first debugging instruments.

---

# 11. What `shape` actually tells you

Suppose:

```python
df.shape
```

returns:

```text
(5000000, 14)
```

That means:

```text
5,000,000 rows
14 columns
```

This is immediately important from a data-engineering perspective.

You now know roughly what scale you're dealing with.

Before doing anything expensive, ask:

```text
How much memory will this consume?
Do I need all 14 columns?
Do I really need all 5 million rows?
Can I filter earlier?
```

---

# 12. `dtypes` are not a side detail

Suppose:

```python
df.dtypes
```

returns:

```text
student_id      int64
name           object
gpa           float64
```

The dtype tells Pandas how it is representing the values in each column.

That affects:

- memory
    
- operations
    
- missing values
    
- comparisons
    
- serialization
    
- performance
    
- downstream schema
    

This is why:

```python
"8.5"
```

and:

```python
8.5
```

are not equivalent from a data-processing perspective.

One is text.

The other is numeric.

---

# 13. `object` is a warning sign, not automatically a problem

You will frequently encounter:

```text
object
```

in older/common Pandas workflows.

It essentially means Pandas is using a general Python-object representation for those values rather than a more specific dtype.

For example:

```text
name       object
city       object
```

may be perfectly reasonable for ordinary text.

But an `object` column can also hide mixed or unexpected values:

```text
["Mumbai", "Delhi", 42, None]
```

So don't blindly think:

```text
object = string
```

Instead inspect the actual contents and choose an appropriate dtype.

Modern Pandas also provides dedicated extension dtypes such as string, nullable integer, nullable boolean, and others.

---

# 14. Missing values are a data-model problem

This deserves serious attention.

Suppose:

```python
df["gpa"]
```

contains:

```text
8.5
9.1
NaN
7.8
```

What does `NaN` mean?

Not zero.

Not `"unknown"`.

Not `"not applicable"` necessarily.

It means a value is missing/NA according to the representation being used.

Pandas has multiple missing-value representations depending on dtype, including `NaN`, `NaT`, and `pd.NA`.

The important engineering lesson is:

> **Missing is information.**

Before filling a missing value, determine why it is missing.

For example:

```text
customer_age = missing
```

could mean:

```text
not provided
unknown
not collected
not applicable
data ingestion failure
```

Those are different business meanings.

Don't automatically do:

```python
df = df.fillna(0)
```

just because your code stops complaining.

That can turn:

```text
unknown
```

into:

```text
zero
```

which is a data corruption bug.

---

# 15. Detecting missing values

Use:

```python
df.isna()
```

or:

```python
df.isna().sum()
```

For example:

```python
print(df.isna().sum())
```

might produce:

```text
student_id    0
name          0
city          0
gpa           3
```

Now you know exactly where missingness exists.

You can also use:

```python
df.notna()
```

`isnull()` and `notnull()` exist as aliases, but I recommend learning `isna()` / `notna()` first.

---

# 16. Selecting columns

Start simple:

```python
df["name"]
```

returns a Series.

While:

```python
df[["name", "gpa"]]
```

returns a DataFrame.

Notice the difference:

```python
df["name"]
```

one column → Series

```python
df[["name"]]
```

one column → DataFrame

That difference matters.

If you are writing reusable pipeline code, know what type you're producing.

---

# 17. Selecting rows: `.loc` and `.iloc`

These two deserve real understanding.

## `.loc` = labels

```python
df.loc[5]
```

asks for the row whose **index label** is `5`.

You can also select:

```python
df.loc[5, "name"]
```

meaning:

```text
row label = 5
column label = "name"
```

Or:

```python
df.loc[:, ["name", "gpa"]]
```

meaning:

```text
all rows
these columns
```

---

## `.iloc` = positions

```python
df.iloc[5]
```

means:

```text
give me the sixth row by position
```

And:

```python
df.iloc[5, 2]
```

means:

```text
sixth row
third column
```

The difference is:

```text
loc  → labels
iloc → positions
```

That distinction is fundamental.

---

# 18. `.at` and `.iat`

When you need a single scalar:

```python
df.at[5, "name"]
```

is label-based.

```python
df.iat[5, 2]
```

is position-based.

You don't need to obsess over their performance characteristics at the beginning.

Just remember the conceptual structure:

```text
loc → label-based general selection
iloc → position-based general selection

at → label-based scalar access
iat → position-based scalar access
```

---

# 19. Boolean filtering

Now Pandas starts becoming extremely powerful.

Suppose:

```python
df["gpa"] >= 8.5
```

This produces a Boolean Series:

```text
True
True
False
True
```

Then:

```python
df[df["gpa"] >= 8.5]
```

keeps the rows where the condition is true.

Conceptually:

```text
DataFrame
   +
Boolean mask
   ↓
filtered DataFrame
```

This is called boolean indexing.

---

# 20. Multiple conditions

Suppose we want:

```text
GPA >= 8
AND
city == Mumbai
```

Write:

```python
df[
    (df["gpa"] >= 8) &
    (df["city"] == "Mumbai")
]
```

For OR:

```python
df[
    (df["city"] == "Mumbai") |
    (df["city"] == "Delhi")
]
```

Use:

```text
&   → AND
|   → OR
~   → NOT
```

Don't use:

```python
and
or
not
```

for element-wise Series logic.

---

# 21. Why the parentheses?

This is worth understanding rather than memorizing.

Python's operator precedence interacts with the Pandas expressions.

So:

```python
df["gpa"] >= 8 & df["city"] == "Mumbai"
```

does not mean what a beginner might expect.

Write the conditions clearly:

```python
(df["gpa"] >= 8) & (df["city"] == "Mumbai")
```

Readable code is safer code here.

---

# 22. Selecting and filtering are two different questions

This distinction is useful:

```text
What columns do I want?
```

versus:

```text
What rows do I want?
```

For example:

```python
df.loc[
    df["gpa"] >= 8,
    ["student_id", "name", "gpa"]
]
```

means:

```text
rows     → students with GPA >= 8
columns  → student_id, name, gpa
```

This is conceptually very close to SQL:

```sql
SELECT student_id, name, gpa
FROM students
WHERE gpa >= 8;
```

Learning to translate between SQL and Pandas will become extremely useful.

---

# 23. Assignment: the dangerous-looking part

Suppose:

```python
df["gpa"] = df["gpa"].round(1)
```

You are replacing the `gpa` column.

Or:

```python
df.loc[df["gpa"] < 0, "gpa"] = pd.NA
```

You are conditionally assigning values.

The general pattern I recommend is:

```python
df.loc[condition, column] = value
```

It makes the target explicit.

---

# 24. A modern note about copies

You may encounter older Pandas tutorials talking extensively about:

> views vs copies

and warnings such as:

> `SettingWithCopyWarning`

This area of Pandas has changed.

In current Pandas 3.0, Copy-on-Write is the standard behavior: objects derived from another DataFrame/Series behave as independent objects for mutation, while copying is deferred when possible. The Pandas documentation specifically describes Copy-on-Write as the mechanism intended to make the indexing API more predictable.

So I don't want you spending hours memorizing historical Pandas folklore.

You should instead develop these habits:

```python
filtered = df.loc[df["gpa"] >= 8].copy()
```

when you intentionally want an explicitly independent working object, and:

```python
df.loc[condition, "column"] = value
```

when you are intentionally modifying a DataFrame.

The deeper lesson is:

> **Know which object you intend to modify.**

---

# 25. Creating derived columns

This is one of the most common Pandas tasks.

Suppose:

```python
df["gpa_percent"] = df["gpa"] * 10
```

This creates a new column.

Or:

```python
df["is_high_performer"] = df["gpa"] >= 8.5
```

You are doing an element-wise operation.

This is one of the things Pandas is designed to make easy.

---

# 26. Vectorization: the idea you need before learning `.apply()`

Consider:

```python
results = []

for gpa in df["gpa"]:
    results.append(gpa * 10)
```

This is ordinary Python iteration.

You could write it.

But Pandas provides:

```python
df["gpa_percent"] = df["gpa"] * 10
```

This is the style you should generally prefer.

Why?

Because Pandas and the numerical libraries underneath it can operate on whole arrays/columns using optimized operations rather than repeatedly executing Python-level code for every row.

So your default mental model should be:

```text
whole-column operation
        ↓
prefer this
```

rather than:

```text
loop through every row
        ↓
do something manually
```

---

# 27. Then why does `.apply()` exist?

Because not every transformation can be expressed naturally using built-in vectorized operations.

For example:

```python
def classify_gpa(gpa):
    if pd.isna(gpa):
        return "missing"
    if gpa >= 9:
        return "excellent"
    if gpa >= 8:
        return "good"
    return "needs_attention"
```

You could use:

```python
df["category"] = df["gpa"].apply(classify_gpa)
```

That's perfectly reasonable for certain operations.

But don't learn the dangerous rule:

> “Whenever I need to transform data, use `.apply()`.”

Instead:

> **First look for a native Pandas/vectorized operation. Use `.apply()` when the logic genuinely needs custom Python-level processing.**

For example, string normalization can often use:

```python
df["name"] = df["name"].str.strip().str.title()
```

rather than a custom function applied row by row.

---

# 28. String operations

Pandas provides vectorized string operations through `.str`.

For example:

```python
df["name"].str.strip()
df["name"].str.lower()
df["name"].str.upper()
df["name"].str.contains("an")
df["name"].str.startswith("A")
df["name"].str.replace("-", "_")
```

These operations are particularly common in ingestion pipelines.

Real-world data frequently contains:

```text
" Mumbai "
"mumbai"
"MUMBAI"
"Mumbai\t"
```

These are semantically similar but technically different strings.

A cleaning pipeline should normalize them intentionally.

---

# 29. Numeric conversion

Suppose a CSV gives you:

```text
"8.5"
"9.1"
"7.8"
""
```

You may need:

```python
df["gpa"] = pd.to_numeric(
    df["gpa"],
    errors="coerce"
)
```

`errors="coerce"` means invalid values become missing rather than causing the entire conversion to fail.

But this raises an engineering question:

> Should invalid data be silently converted to missing?

Sometimes yes.

Sometimes no.

For a production pipeline, you may instead want to count and quarantine invalid records.

Again, Pandas gives you a mechanism; **you still have to define the data-quality policy.**

---

# 30. Datetime is another major Pandas capability

Data engineers spend an enormous amount of time dealing with timestamps.

Suppose:

```python
df["event_time"]
```

contains strings:

```text
2026-09-20 10:30:00
2026-09-20 11:15:00
```

Convert them explicitly:

```python
df["event_time"] = pd.to_datetime(
    df["event_time"],
    errors="coerce"
)
```

Then you get datetime-aware operations.

For example:

```python
df["event_time"].dt.year
df["event_time"].dt.month
df["event_time"].dt.day
df["event_time"].dt.hour
df["event_time"].dt.day_name()
```

You can also derive:

```python
df["event_date"] = df["event_time"].dt.date
```

or a normalized timestamp:

```python
df["event_day"] = df["event_time"].dt.floor("D")
```

This becomes extremely important later for:

```text
partitioning
incremental loads
time-window processing
event-time analysis
late-arriving data
```

---

# 31. Timezones: don't ignore them

Suppose two systems produce:

```text
2026-09-20 10:00
```

and:

```text
2026-09-20 10:00
```

Are they the same instant?

Not necessarily.

One could mean:

```text
10:00 UTC
```

and another:

```text
10:00 Asia/Kolkata
```

Those are different instants.

A serious data engineer should develop the habit:

> **Store and reason about timestamps with timezone semantics explicitly understood.**

UTC will become especially important when you later work with distributed systems and cloud pipelines.

---

# 32. Sorting

Simple:

```python
df.sort_values("gpa")
```

Descending:

```python
df.sort_values("gpa", ascending=False)
```

Multiple columns:

```python
df.sort_values(
    ["city", "gpa"],
    ascending=[True, False]
)
```

Remember that most Pandas operations return a new DataFrame rather than magically rewriting your existing variable.

So normally write:

```python
df = df.sort_values("gpa")
```

when you want the sorted result to become the current working DataFrame.

---

# 33. Removing duplicates

Suppose your pipeline accidentally loaded the same student twice.

Inspect duplicates:

```python
df.duplicated()
```

Count them:

```python
df.duplicated().sum()
```

Remove them:

```python
df = df.drop_duplicates()
```

Or define business uniqueness:

```python
df = df.drop_duplicates(
    subset=["student_id"]
)
```

This is more important in DE than it might initially appear.

A duplicate record isn't merely a cosmetic problem.

It can cause:

```text
double-counted revenue
duplicate events
incorrect aggregates
incorrect customer counts
```

---

# 34. `dropna`, `fillna`, and `replace`

You have:

```python
df.dropna()
```

which removes rows containing missing values according to the operation's parameters.

You have:

```python
df.fillna(...)
```

which fills missing values.

And:

```python
df.replace(...)
```

which replaces specific values or patterns.

But don't treat them as equivalent.

These operations should reflect a **data-quality decision**.

For example:

```python
df["country"] = df["country"].fillna("UNKNOWN")
```

may make sense.

But:

```python
df["revenue"] = df["revenue"].fillna(0)
```

could be completely wrong.

The correct value depends on what “missing revenue” means.

---

# 35. Now we reach one of the central Pandas operations: `groupby`

Suppose our table contains:

```text
student_id   name    city       gpa
101          Arjun   Mumbai     8.5
102          Priya   Delhi      9.1
103          Rohan   Mumbai     7.8
104          Sneha   Chennai    8.9
105          Karan   Mumbai     8.2
```

We want average GPA by city.

In SQL:

```sql
SELECT
    city,
    AVG(gpa)
FROM students
GROUP BY city;
```

In Pandas:

```python
df.groupby("city")["gpa"].mean()
```

That is not just a convenient syntax trick.

It represents a major computational pattern:

```text
split
  ↓
apply
  ↓
combine
```

---

# 36. The split-apply-combine mental model

Suppose:

```python
df.groupby("city")
```

Conceptually:

```text
Mumbai group
Delhi group
Chennai group
```

Then:

```python
["gpa"].mean()
```

says:

```text
within each group,
take gpa,
calculate mean
```

Then Pandas combines the results.

So:

```python
df.groupby("city")["gpa"].mean()
```

means:

```text
group rows by city
→ select gpa
→ calculate mean within each group
→ return result
```

Once you understand this, `groupby` stops being magic.

---

# 37. Multiple aggregations

You can do:

```python
df.groupby("city")["gpa"].agg(
    ["count", "mean", "min", "max"]
)
```

Or define named outputs:

```python
summary = (
    df.groupby("city")
      .agg(
          student_count=("student_id", "count"),
          avg_gpa=("gpa", "mean"),
          max_gpa=("gpa", "max")
      )
      .reset_index()
)
```

This is much closer to real data-engineering work.

---

# 38. `agg` and `transform` are not the same

This is a concept I want you to understand early.

Suppose:

```python
df.groupby("city")["gpa"].mean()
```

produces one result per city.

That's an aggregation.

But suppose you want every student to know the average GPA of their city.

Then you want:

```python
df["city_avg_gpa"] = (
    df.groupby("city")["gpa"]
      .transform("mean")
)
```

Now each original row remains.

Example:

```text
city       gpa    city_avg_gpa
Mumbai     8.5       8.17
Mumbai     7.8       8.17
Mumbai     8.2       8.17
Delhi      9.1       9.10
Chennai    8.9       8.90
```

The distinction:

```text
agg()
    ↓
reduces groups

transform()
    ↓
returns something aligned back to the original rows
```

That distinction becomes extremely useful.

---

# 39. Think of `transform` as “group calculation + broadcast”

This is a powerful mental model:

```text
group
 ↓
calculate one value
 ↓
broadcast that value back to every row in the group
```

You'll see this pattern repeatedly in analytical transformations.

---

# 40. Filtering groups

Sometimes you don't want to filter individual rows.

You want to filter entire groups.

For example:

> Keep cities that contain at least 3 students.

That is conceptually different from:

> Keep students whose GPA is above 8.

One operates on rows.

The other operates on groups.

This distinction is important because group-level logic is one of the places where people initially write overly complicated code.

---

# 41. Joining DataFrames: think SQL

Suppose we have:

### Students

```text
student_id   name
101          Arjun
102          Priya
103          Rohan
```

### Departments

```text
student_id   department
101          CS
102          EE
103          ME
```

We want:

```text
student_id   name    department
101          Arjun   CS
102          Priya   EE
103          Rohan   ME
```

Use:

```python
result = students.merge(
    departments,
    on="student_id",
    how="inner"
)
```

This is SQL-style joining.

Pandas' `merge()` is explicitly designed for database-style joins.

---

# 42. Join types

You should know the conceptual meaning before memorizing syntax.

```text
inner
left
right
outer
cross
```

### Inner join

Keep keys appearing on both sides.

### Left join

Keep every row from the left DataFrame.

### Right join

Keep every row from the right DataFrame.

### Outer join

Keep keys from both sides.

### Cross join

Cartesian product.

Think in SQL terms.

You already learned joins in SQL.

Use that knowledge.

---

# 43. The most important join question isn't `how=`

Before you merge two datasets, ask:

> **What is the cardinality of the relationship?**

For example:

```text
student_id
```

might be:

```text
students.student_id → one row
departments.student_id → one row
```

That's one-to-one.

But perhaps:

```text
customers
orders
```

has:

```text
customer_id
```

with:

```text
one customer → many orders
```

That's one-to-many.

And perhaps both sides have duplicates:

```text
many-to-many
```

which can produce a much larger result than you intended.

This is one of the most dangerous Pandas mistakes.

---

# 44. Use `validate` when joining important data

For example:

```python
result = customers.merge(
    customer_dim,
    on="customer_id",
    how="left",
    validate="many_to_one"
)
```

This tells Pandas:

> I expect many customer rows to match at most one dimension row.

If that assumption is violated, fail rather than silently produce an unexpected multiplication of rows.

That is exactly the kind of defensive practice I want you to develop as a data engineer.

---

# 45. An important Pandas-vs-SQL trap: NULL join keys

There is a subtle difference between Pandas and SQL.

Pandas' current `merge()` documentation warns that rows with null values in the join keys can match each other, which differs from normal SQL null-join semantics.

So:

```text
SQL intuition
NULL = NULL
→ UNKNOWN

Pandas merge
null key on left + null key on right
→ may match
```

Never assume:

```text
Pandas merge == SQL join in every semantic detail
```

It isn't.

This is precisely why understanding the underlying systems matters.

---

# 46. `indicator=True` is useful for debugging joins

You can write:

```python
result = left.merge(
    right,
    on="id",
    how="outer",
    indicator=True
)
```

Now Pandas can indicate whether a row came from:

```text
left_only
right_only
both
```

This is extremely useful when debugging ingestion and reconciliation pipelines.

For example:

> Which customers exist in source A but not source B?

That's not merely an analysis question.

It can be a data-quality check.

---

# 47. `concat()` is different from `merge()`

This distinction must be clear.

Suppose you have:

```text
January data
February data
March data
```

and all three have the same schema.

You generally want:

```python
df = pd.concat(
    [jan, feb, mar],
    ignore_index=True
)
```

You're stacking datasets.

Conceptually:

```text
A
↓
B
↓
C
↓
combined
```

That's `concat`.

`merge` is about relating datasets using keys.

```text
merge
→ relational join
```

```text
concat
→ concatenate objects along an axis
```

Pandas documentation makes this distinction explicit.

---

# 48. Why you should not repeatedly concatenate inside a loop

Avoid:

```python
result = pd.DataFrame()

for file in files:
    chunk = pd.read_csv(file)
    result = pd.concat([result, chunk])
```

This can repeatedly create larger objects and cause unnecessary copying.

Prefer:

```python
frames = []

for file in files:
    frames.append(pd.read_csv(file))

result = pd.concat(
    frames,
    ignore_index=True
)
```

The current Pandas documentation explicitly recommends collecting objects and concatenating once rather than repeatedly concatenating.

This is a small example of a larger engineering principle:

> **Don't repeatedly rebuild a growing data structure when you can accumulate and combine once.**

---

# 49. Wide versus long data

Now we enter reshaping.

Suppose you have:

```text
student   math   physics   chemistry
Arjun     90     85       88
Priya     95     91       93
```

This is wide.

A long representation might be:

```text
student   subject      score
Arjun     math         90
Arjun     physics      85
Arjun     chemistry    88
Priya     math         95
Priya     physics      91
Priya     chemistry    93
```

These represent the same information differently.

---

# 50. `melt()` turns wide into long

```python
long_df = df.melt(
    id_vars=["student"],
    var_name="subject",
    value_name="score"
)
```

The conceptual transformation is:

```text
many subject columns
        ↓
one subject column
+
one value column
```

This is particularly useful when preparing data for systems that expect normalized/long-form structures.

---

# 51. `pivot_table()` moves in the opposite direction

Suppose you have:

```text
student   subject      score
Arjun     math         90
Arjun     physics      85
Priya     math         95
Priya     physics      91
```

You may want:

```python
df.pivot_table(
    index="student",
    columns="subject",
    values="score",
    aggfunc="mean"
)
```

Conceptually:

```text
long
 ↓
group by dimensions
 ↓
put one dimension across columns
 ↓
aggregate if necessary
 ↓
wide table
```

Don't memorize these as random methods.

Think:

```text
melt       → wide → long
pivot      → long → wide, requiring unique combinations
pivot_table → long → wide with aggregation
```

Pandas' reshaping guide covers `pivot`, `pivot_table`, `melt`, `stack`, `unstack` and related operations.

---

# 52. Reading CSV correctly

The simplest code is:

```python
df = pd.read_csv("students.csv")
```

But production-quality ingestion requires more thought.

You should ask:

```text
What encoding?
Which columns?
What types?
What represents missing values?
Are dates parsed?
How large is the file?
Are there malformed rows?
```

For example:

```python
df = pd.read_csv(
    "students.csv",
    usecols=["student_id", "name", "city", "gpa"]
)
```

Selecting only required columns can reduce memory.

You can also explicitly control type interpretation where appropriate.

Do not blindly trust inferred dtypes when schema correctness matters.

---

# 53. Pandas is not a schema-management system

This is an important DE principle.

Suppose your source sends:

```text
student_id,name,gpa
101,Arjun,8.5
102,Priya,9.1
```

Tomorrow it sends:

```text
student_id,name,gpa
101,Arjun,8.5
102,Priya,unknown
```

Pandas may infer a different dtype or convert values in a way you didn't expect.

This means:

> **Schema should be an explicit pipeline concern.**

Your pipeline should know:

```text
student_id → integer
name       → string
gpa        → numeric, nullable
```

and should have a policy for violations.

---

# 54. Reading Parquet

For Parquet:

```python
df = pd.read_parquet("students.parquet")
```

This is one of the places where Pandas becomes directly relevant to your future data-engineering work.

A common pattern is:

```text
CSV / JSON
    ↓
Pandas
    ↓
cleaned DataFrame
    ↓
Parquet
```

You are using a text-oriented input format and converting it into a columnar analytical format.

That connects directly to the file-format lesson you already studied.

---

# 55. Writing Parquet

```python
df.to_parquet(
    "output/students.parquet",
    index=False
)
```

The `index=False` deserves attention.

Usually your Pandas index is an internal labeling mechanism, not actual business data.

You often don't want it serialized as a separate column.

But don't mechanically use `index=False` everywhere.

Ask:

> Is this index meaningful data that needs to survive serialization?

Usually, in a data pipeline, the answer is no unless you deliberately made it part of the data model.

---

# 56. CSV versus Parquet in your pipeline

A common architecture is:

```text
External system
      ↓
    CSV/JSON
      ↓
    Pandas
      ↓
  validation
      ↓
 transformation
      ↓
   Parquet
      ↓
 downstream systems
```

Why?

Because CSV/JSON are convenient interchange formats.

Parquet is much more appropriate for analytical workloads.

This isn't because Parquet is universally “better.”

It's because the formats have different jobs.

You already learned the internal reasons for this in the file-format lesson.

---

# 57. Memory is part of Pandas programming

This is one of the most important differences between writing toy Pandas code and writing data-engineering code.

Suppose:

```python
df = pd.read_csv("huge.csv")
```

If the file is several gigabytes, that's not automatically safe.

Your computer needs enough memory not merely for the file on disk, but for the in-memory representation of the data and intermediate objects created during processing.

A compressed 2 GB Parquet file does not mean:

```text
2 GB disk
→
2 GB RAM
```

The in-memory representation can be substantially different.

Therefore:

> **Always think about the memory lifecycle of a DataFrame.**

Ask:

```text
How big is the source?
How many columns do I actually need?
What dtypes are being used?
How many intermediate DataFrames am I creating?
Can I process in chunks?
```

---

# 58. Don't force Pandas to solve a problem it cannot solve

Imagine:

```text
500 GB dataset
```

You have:

```python
pd.read_parquet(...)
```

and you hope it will somehow process the whole thing efficiently on your laptop.

That's not what Pandas is for.

At that point you'd investigate:

```text
Spark
DuckDB
Polars
warehouse SQL
distributed processing
query engines
```

depending on the workload.

Pandas is one tool in the ecosystem.

The goal is not:

> “Use Pandas for everything.”

The goal is:

> **Know when Pandas is the right abstraction and when it isn't.**

---

# 59. Chunked processing

For CSVs that are too large to load all at once, Pandas can read chunks:

```python
for chunk in pd.read_csv(
    "large.csv",
    chunksize=100_000
):
    process(chunk)
```

Now your pipeline can operate on batches rather than the complete dataset.

Conceptually:

```text
large file
   ↓
chunk 1 → process
chunk 2 → process
chunk 3 → process
...
```

This connects directly to the earlier Python lesson about streaming and memory management.

Notice how concepts from different parts of Python are now converging.

That is exactly what we want.

---

# 60. `query()` exists, but don't make it your foundation

You may encounter:

```python
df.query("gpa >= 8 and city == 'Mumbai'")
```

This can be convenient.

But I want you to understand ordinary boolean indexing first:

```python
df[
    (df["gpa"] >= 8) &
    (df["city"] == "Mumbai")
]
```

Why?

Because the second form makes the underlying Series operations explicit.

Once you understand the fundamental mechanics, convenience APIs become much easier to learn.

---

# 61. Method chaining

You may start seeing code like:

```python
result = (
    df
    .loc[df["gpa"] >= 8]
    .assign(
        gpa_percent=lambda x: x["gpa"] * 10
    )
    .sort_values("gpa", ascending=False)
)
```

This can be elegant.

But don't learn chaining as a style requirement.

The underlying operations are:

```text
filter
↓
create column
↓
sort
```

First learn to understand those operations separately.

Then learn to compose them.

A readable pipeline is good.

A 15-line chain that nobody can debug is not.

---

# 62. `assign()` is useful for transformations

Instead of:

```python
df["gpa_percent"] = df["gpa"] * 10
df["passed"] = df["gpa"] >= 5
```

you can write:

```python
df = df.assign(
    gpa_percent=lambda x: x["gpa"] * 10,
    passed=lambda x: x["gpa"] >= 5
)
```

This can be useful in transformation pipelines because it returns a new DataFrame and makes the sequence of transformations explicit.

Again:

> Learn the underlying idea before the syntactic convenience.

---

# 63. A very important operation: `reset_index()`

Suppose:

```python
result = df.groupby("city")["gpa"].mean()
```

Now:

```text
city
Chennai     8.9
Delhi       9.1
Mumbai      8.2
```

`city` is the index.

Sometimes that's useful.

Sometimes you want it to become an ordinary column again:

```python
result = result.reset_index()
```

Now:

```text
city       gpa
Chennai    8.9
Delhi      9.1
Mumbai     8.2
```

This distinction is important because indexes can be introduced as a side effect of many operations.

When passing data between systems, you should know whether something is:

```text
a column
```

or:

```text
an index
```

---

# 64. Never treat the index as invisible magic

When something looks strange in Pandas, inspect:

```python
df.index
```

and:

```python
df.columns
```

A surprising number of bugs come from not realizing:

```text
what you thought was a column
```

is actually:

```text
an index level
```

or vice versa.

This becomes even more important when you later encounter `MultiIndex`.

You don't need deep MultiIndex knowledge yet.

Just know that Pandas' label system can become hierarchical.

---

# 65. `rename()`

Column names are often messy in source systems:

```text
Student ID
student name
GPA (%)
```

Normalize them:

```python
df = df.rename(
    columns={
        "Student ID": "student_id",
        "student name": "student_name",
        "GPA (%)": "gpa"
    }
)
```

A common pipeline pattern is to establish a canonical schema early.

For example:

```text
source schema
     ↓
normalize names
     ↓
normalize types
     ↓
validate schema
     ↓
transform
```

That's much safer than letting source-system naming leak throughout your entire pipeline.

---

# 66. A realistic cleaning example

Suppose we ingest:

```text
student_id,name,city,gpa
101, " Arjun ",mumbai,8.5
102,Priya,Delhi,9.1
103,Rohan,MUMBAI,7.8
104,Sneha, Chennai ,N/A
```

A reasonable transformation might be:

```python
df["name"] = df["name"].str.strip()

df["city"] = (
    df["city"]
    .str.strip()
    .str.title()
)

df["gpa"] = pd.to_numeric(
    df["gpa"],
    errors="coerce"
)
```

Then inspect:

```python
print(df.dtypes)
print(df.isna().sum())
print(df)
```

The key is not the exact code.

The important pipeline logic is:

```text
raw
 ↓
normalize strings
 ↓
convert types
 ↓
inspect missing values
 ↓
validate
```

---

# 67. Data quality should come after cleaning—but not only after cleaning

Suppose:

```python
df["student_id"].is_unique
```

returns:

```text
False
```

That tells you the supposed identifier isn't unique.

Or:

```python
df["gpa"].between(0, 10).all()
```

checks whether all GPAs fall into the expected range.

Or:

```python
df["name"].notna().all()
```

checks whether names are present.

These are data-quality assertions.

You should increasingly think:

```text
transform
+
validate
```

rather than:

```text
transform
and hope
```

---

# 68. Pandas as a bridge to SQL

Because you've already learned SQL, use that knowledge aggressively.

Think:

|SQL|Pandas|
|---|---|
|`SELECT col`|`df["col"]`|
|`SELECT col1, col2`|`df[["col1", "col2"]]`|
|`WHERE`|boolean filtering|
|`GROUP BY`|`groupby()`|
|aggregate|`agg()`, `sum()`, `mean()`|
|window-like transformation|`transform()`, rolling operations|
|`JOIN`|`merge()`|
|`UNION ALL`|`concat()`|
|`ORDER BY`|`sort_values()`|
|`DISTINCT`|`drop_duplicates()`|
|rename|`rename()`|
|reshape|`pivot_table()`, `melt()`|

But don't conclude:

> Pandas is SQL in Python.

It isn't.

SQL is a declarative language executed by a database/query engine.

Pandas is an in-memory Python object model where you compose operations programmatically.

The similar concepts are valuable precisely because you already understand relational thinking.

---

# 69. One of the most important distinctions: eager vs lazy

When you write:

```python
df2 = df[df["gpa"] >= 8]
```

Pandas performs the operation and produces a result object.

You're working in an eager model.

This is different from systems you'll later learn where transformations may build a logical plan and execution happens later.

That difference becomes important when you move from:

```text
Pandas
```

to:

```text
Spark
```

and eventually to:

```text
query engines / warehouses
```

Don't assume the same code style implies the same execution model.

---

# 70. Pandas operations are not automatically cheap

This is another important engineering mindset.

Something can look small:

```python
df.sort_values(...)
```

but sorting millions of rows is still substantial work.

Likewise:

```python
df.merge(...)
```

can become expensive if the join keys have high duplication.

And:

```python
df.copy()
```

can matter for very large DataFrames.

The syntactic size of the line tells you almost nothing about its computational cost.

Learn to think:

```text
What is the number of rows?
What are the dtypes?
What is the cardinality?
What intermediate objects are created?
```

---

# 71. Categorical data

Suppose:

```python
city
```

contains only:

```text
Mumbai
Delhi
Chennai
Pune
```

but you have 20 million rows.

There are only a few distinct values.

A categorical representation can be useful for such low-cardinality columns.

You may encounter:

```python
df["city"] = df["city"].astype("category")
```

This is an optimization/data-representation tool.

But don't blindly convert every string to category.

Think about:

```text
number of unique values
memory
operations
downstream format
workload
```

Optimization should follow measurement and workload understanding.

---

# 72. A note about Pandas 3.0

You will find many tutorials online that were written for older Pandas versions.

That matters.

Pandas 3.0 introduced and standardized Copy-on-Write behavior and contains other API changes relative to older tutorials. The official documentation currently corresponds to Pandas 3.0.6.

So when you encounter an old tutorial saying:

```text
Pandas always behaves this way...
```

don't automatically believe it.

Check the version.

Data-engineering code lives in environments with explicit dependencies for a reason.

---

# 73. A complete small ETL example

Now let's put the concepts together.

Imagine:

```text
students.csv
```

contains:

```text
student_id,name,city,gpa
1, Arjun ,Mumbai,8.5
2,Priya,Delhi,9.1
3,Rohan,Mumbai,7.8
4,Sneha,Chennai,N/A
5,Karan,Mumbai,8.2
```

Our pipeline:

```python
from pathlib import Path

import pandas as pd


INPUT = Path("data/students.csv")
OUTPUT = Path("data/output/students.parquet")


def extract(path: Path) -> pd.DataFrame:
    return pd.read_csv(path)


def transform(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()

    df["name"] = df["name"].str.strip()

    df["city"] = (
        df["city"]
        .str.strip()
        .str.title()
    )

    df["gpa"] = pd.to_numeric(
        df["gpa"],
        errors="coerce"
    )

    df["passed"] = df["gpa"] >= 5

    return df


def validate(df: pd.DataFrame) -> None:
    assert df["student_id"].notna().all()
    assert df["student_id"].is_unique
    assert df["gpa"].between(0, 10).all() | df["gpa"].isna().all()


def load(df: pd.DataFrame, path: Path) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    df.to_parquet(path, index=False)


def run_pipeline() -> None:
    df = extract(INPUT)

    print("Raw:")
    print(df.dtypes)

    df = transform(df)

    validate(df)

    print("Clean:")
    print(df)

    load(df, OUTPUT)


if __name__ == "__main__":
    run_pipeline()
```

Don't just copy this.

Look at the architecture.

```text
extract
   ↓
transform
   ↓
validate
   ↓
load
```

That's the same ETL pattern you have already encountered.

Pandas is the transformation engine inside the middle of the pipeline.

---

# 74. Notice what we deliberately did not do

We didn't write:

```python
for row in df.iterrows():
    ...
```

for ordinary transformations.

We didn't manually parse CSV.

We didn't convert everything into Python dictionaries.

We didn't blindly fill every missing value.

We didn't merge without considering key uniqueness.

We didn't create 25 unnecessary intermediate copies.

We didn't ask Pandas to process hundreds of gigabytes.

This is the beginning of **engineering judgment**, not merely Pandas syntax.

---

# 75. The most common beginner mistakes

### Mistake 1: treating DataFrame like a list of dictionaries

```python
for row in df:
    ...
```

does not iterate over rows.

It iterates over column labels.

Understand what object you're holding before iterating over it.

---

### Mistake 2: using `.apply()` for everything

Just because you can write:

```python
df["x"].apply(...)
```

doesn't mean it's the best expression.

Look for built-in/vectorized operations first.

---

### Mistake 3: ignoring dtypes

If:

```text
revenue = "100"
```

you don't have numeric revenue.

You have text that looks like a number.

---

### Mistake 4: blindly filling missing values

```python
fillna(0)
```

is not a universal data-cleaning solution.

---

### Mistake 5: merging without validating cardinality

A join can silently multiply rows.

Always understand:

```text
one-to-one
one-to-many
many-to-one
many-to-many
```

when joining important datasets.

---

### Mistake 6: confusing index with business keys

The Pandas index is not automatically:

```text
primary key
```

A business identifier such as:

```python
customer_id
```

should remain an explicit column unless there is a deliberate reason to make it an index.

---

### Mistake 7: loading everything

This:

```python
pd.read_csv("huge.csv")
```

isn't automatically good engineering.

Always consider scale.

---

### Mistake 8: assuming Pandas semantics equal SQL semantics

They're similar enough to help you.

They're different enough to bite you.

Null joins are one example.

---

# 76. The Pandas mental model I want you to remember

You don't need to memorize hundreds of methods.

Remember this:

```text
DataFrame
│
├── rows → Index
│
├── columns → Series
│
└── values → typed data
```

Then think in operations:

```text
Inspect
   ↓
Select
   ↓
Filter
   ↓
Transform
   ↓
Aggregate
   ↓
Join
   ↓
Reshape
   ↓
Validate
   ↓
Write
```

And continually ask:

```text
What are my rows?
What are my columns?
What are my labels?
What are the dtypes?
What does missing mean?
What is the cardinality?
How much memory does this require?
What exactly happens to the row count?
```

Those questions are more valuable than memorizing method names.

---

# 77. Stop-and-think questions

Don't run the code immediately.

Predict first.

### Question 1

```python
a = pd.Series(
    [10, 20, 30],
    index=["A", "B", "C"]
)

b = pd.Series(
    [100, 200],
    index=["C", "A"]
)

print(a + b)
```

What happens to `B`?

---

### Question 2

```python
df = pd.DataFrame({
    "city": ["Mumbai", "Delhi", "Mumbai"],
    "gpa": [8.5, 9.0, 7.5]
})

result = df.groupby("city")["gpa"].mean()
```

Is `city` now a normal column or an index?

What would `reset_index()` do?

---

### Question 3

```python
df["high"] = df["gpa"] >= 8
```

Is `high` one Boolean value or a collection of Boolean values?

Why?

---

### Question 4

Suppose:

```text
left:
id = [1, 2, 3]

right:
id = [2, 3, 4]
```

Predict the row count of:

```python
left.merge(right, on="id", how="inner")
```

Then:

```python
left.merge(right, on="id", how="left")
```

Then:

```python
left.merge(right, on="id", how="outer")
```

---

### Question 5

Why can this be dangerous?

```python
result = customers.merge(
    orders,
    on="customer_id"
)
```

What happens if each customer has 20 orders?

---

# 78. Exercises

Do these without looking up complete solutions.

## Exercise 1 — Build a DataFrame

Create a DataFrame containing:

```text
student_id
name
city
age
gpa
```

Create at least 10 records.

Use realistic missing values.

Then inspect:

```text
shape
columns
dtypes
head
missing counts
```

Write down what you learned about the dataset before transforming it.

---

## Exercise 2 — Cleaning

Introduce deliberately messy data:

```text
" Mumbai "
"mumbai"
"MUMBAI"
" Delhi"
```

and GPA values such as:

```text
"8.5"
"9.1"
"bad"
""
```

Normalize:

```text
name
city
gpa
```

Then determine:

```text
How many rows had invalid GPA values?
How many missing GPA values exist?
```

---

## Exercise 3 — Filtering

Find:

```text
students with GPA >= 8
students from Mumbai
students from Mumbai with GPA >= 8
students whose names begin with A
```

Do these using boolean expressions.

---

## Exercise 4 — Grouping

Calculate by city:

```text
number of students
average GPA
minimum GPA
maximum GPA
```

Then add:

```text
city_avg_gpa
```

to every original student row using `transform()`.

---

## Exercise 5 — Joins

Create:

```text
students
departments
```

with a shared:

```text
student_id
```

Perform:

```text
inner
left
outer
```

joins.

Then deliberately introduce duplicate `student_id` values and see what happens.

Use:

```python
validate=
```

to express your expected relationship.

---

## Exercise 6 — Data-quality pipeline

Build:

```text
CSV
 ↓
Pandas
 ↓
clean
 ↓
validate
 ↓
Parquet
```

Your validation should check:

```text
required columns exist
student_id is not null
student_id is unique
gpa is between 0 and 10 when present
city is not empty
```

Don't merely print errors.

Make the pipeline fail when critical invariants are violated.

---

## Exercise 7 — Memory experiment

Create a DataFrame with a few million rows.

Compare:

```text
memory usage
different dtypes
different representations
```

Use:

```python
df.memory_usage(deep=True)
```

The goal isn't to memorize numbers.

The goal is to understand:

> **A DataFrame's memory footprint depends on representation, not simply on the number of rows.**

---

## Exercise 8 — Chunked ingestion

Create a large CSV.

Read it using:

```python
chunksize=
```

Process each chunk independently.

Calculate a final result without holding the entire input dataset in memory.

This exercise connects Pandas to the streaming/file-I/O concepts you already learned.

---

# 79. A miniature DE challenge

Now build something closer to an actual pipeline.

Use a public dataset containing transaction/event records.

Your pipeline should have:

```text
extract()
transform()
validate()
aggregate()
load()
```

The output should be:

```text
Parquet
```

Your transformation should include at least:

```text
column normalization
type conversion
missing-value handling
timestamp conversion
duplicate handling
derived columns
groupby aggregation
at least one join
```

And your validation should explicitly answer:

```text
How many rows entered?
How many rows left?
How many records were invalid?
What happened to missing values?
Did the join preserve expected cardinality?
What is the output schema?
```

This is where Pandas stops being “a library you're learning” and becomes part of a data pipeline.

---

# 80. What you actually need to know before leaving Pandas

You do **not** need to memorize every Pandas method.

You should be comfortable explaining these concepts without looking them up:

```text
Series
DataFrame
Index
alignment
dtype
missing values
.loc
.iloc
boolean filtering
assignment
vectorized operations
.str
.dt
groupby
agg
transform
merge
join cardinality
concat
melt
pivot_table
sorting
duplicates
reading CSV
reading Parquet
writing Parquet
memory usage
chunked processing
```

More importantly, you should be able to answer:

> What happens to the rows?

after every major operation.

For example:

```text
filter
→ fewer rows

groupby + agg
→ generally fewer rows

transform
→ same number of rows

left merge
→ at least as many left-side matches, but potentially MORE due to duplicate keys

inner merge
→ only matching keys, but potentially MORE rows due to many-to-many matches

concat
→ usually more rows when concatenating vertically

melt
→ usually more rows because multiple columns become rows
```

That way of thinking is much more valuable than memorizing syntax.

---

# 81. The final mental model

At the beginning, you had:

```python
[
    {"id": 1, "name": "Arjun"},
    {"id": 2, "name": "Priya"}
]
```

You can think about that as:

```text
a list of independent records
```

Pandas asks you to think differently:

```text
                      DataFrame
                          │
             ┌────────────┴────────────┐
             │                         │
         row labels                columns
             │                         │
           Index                  Series objects
                                       │
                                  typed values
```

And then transformations become operations on that table:

```text
                 DATAFRAME
                     │
        ┌────────────┼─────────────┐
        ↓            ↓             ↓
     select       filter        transform
        │            │             │
        └────────────┼─────────────┘
                     ↓
                 aggregate
                     ↓
                   merge
                     ↓
                 reshape
                     ↓
                validate
                     ↓
                  Parquet
```

That's the Pandas you need for Data Engineering.

Not:

> “I know `groupby()`, `merge()`, `fillna()` and 50 other functions.”

But:

> **“I understand the data model, I can predict what an operation will do to the data, I can reason about types/missingness/cardinality/memory, and I can compose those operations into a reliable transformation pipeline.”**

Once you have that foundation, learning the rest of Pandas becomes much easier because you're no longer memorizing an API. You're learning different ways to express operations you already understand.