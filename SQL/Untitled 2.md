
## Phase 3, Lesson 3 — Data Modeling for Analytics

---

### Why data modeling is a data engineering skill, not just a database administrator skill

Most people learn data modeling as something database administrators do when setting up an OLTP system — defining tables, foreign keys, and normalization rules. That is one kind of data modeling.

The data modeling we care about in data engineering is different. It is the art of organizing analytical data so that the questions business people ask can be answered quickly, correctly, and intuitively. This is called **dimensional modeling**, and it was developed specifically for analytical systems, not operational ones.

Understanding dimensional modeling is essential for a data engineer because it determines the structure of everything you load into a data warehouse. Every dbt model you write in Phase 5, every Spark output you design in Phase 6, every warehouse table you create — the structure comes from dimensional modeling decisions. Writing a pipeline that loads data into a poorly modeled warehouse is like building a road that leads nowhere — technically functional, practically useless.

---

### The problem with normalized data for analysis

In Phase 3 Lesson 1, you learned that OLTP databases are highly normalized — data is split across many tables to minimize redundancy. Let us look at what that actually means for a concrete query.

Suppose a business analyst wants to answer: "What is the average GPA per department for students who enrolled in 2023 and live in cities with more than 1 million population?"

In a normalized OLTP database, the data needed for this question lives across at least five tables:

```
students (student_id, name, city_id, enrollment_year)
cities (city_id, city_name, population)
enrollments (enrollment_id, student_id, course_id, enrollment_date)
courses (course_id, course_name, department_id)
departments (department_id, department_name)
grades (grade_id, enrollment_id, gpa)
```

The SQL to answer this question:

```sql
SELECT 
    d.department_name,
    AVG(g.gpa) AS avg_gpa
FROM students s
JOIN cities c ON s.city_id = c.city_id
JOIN enrollments e ON s.student_id = e.student_id
JOIN courses co ON e.course_id = co.course_id
JOIN departments d ON co.department_id = d.department_id
JOIN grades g ON e.enrollment_id = g.enrollment_id
WHERE s.enrollment_year = 2023
AND c.population > 1000000
GROUP BY d.department_name;
```

Six tables, five joins. This query is not hard to write, but it has problems at scale. Five joins on tables with millions of rows is expensive. The business analyst must know how all six tables relate to each other to write or understand this query. If the analyst uses a BI tool that generates SQL automatically, the tool must understand the entire table structure and all the relationships.

In a production data warehouse serving dozens of analysts and hundreds of dashboards, this complexity multiplies. Every query needs to know about all the joins. A mistake in one join silently returns wrong data. The cognitive overhead is enormous.

Dimensional modeling solves this by transforming the data into a structure designed specifically for how analysts think about business questions rather than how databases minimize redundancy.

---

### How analysts actually think about data

An analyst asking "what is the average GPA per department in 2023" is thinking about this differently from how a normalized schema represents it. They are thinking:

"I want to measure something (GPA) broken down by different dimensions (department, year, city, enrollment status)."

This is the fundamental insight of dimensional modeling. Business questions always have the same structure: **a measurement of something, filtered and grouped by characteristics of that something**.

The measurement is called a **fact**. The characteristics are called **dimensions**.

The GPA is a fact. Department is a dimension. Year is a dimension. City is a dimension. The entire question is: give me the fact, aggregated across groups defined by the dimensions.

Every analytical question about a business has this structure. Total revenue (fact) by product category (dimension) and region (dimension). Average delivery time (fact) by courier company (dimension) and month (dimension). Student dropout rate (fact) by department (dimension) and admission year (dimension).

Dimensional modeling takes this observation and uses it to design the analytical schema from the ground up.

---

### The star schema — the core structure

A star schema has two types of tables: one **fact table** at the center, surrounded by multiple **dimension tables**. The shape resembles a star — the fact table in the middle, dimension tables radiating outward. This is not just aesthetic; it directly reflects the analytical structure of business questions.

The **fact table** contains:

- Foreign keys pointing to each dimension table
- The measurable facts (numeric values you aggregate — count, sum, average, min, max)

The **dimension tables** contain:

- A primary key
- Descriptive attributes (the characteristics you filter and group by)

Let us rebuild the university example as a star schema.

The fact table captures what actually happened — a student received a grade in a course. The facts (measurements) are the GPA and the credit hours. The dimensions describe the who, what, when, and where.

```sql
-- Fact table
CREATE TABLE fact_grades (
    grade_key        BIGINT PRIMARY KEY,
    student_key      INT REFERENCES dim_students(student_key),
    course_key       INT REFERENCES dim_courses(course_key),
    date_key         INT REFERENCES dim_date(date_key),
    city_key         INT REFERENCES dim_cities(city_key),
    gpa              DECIMAL(3,2),
    credit_hours     INT,
    is_passing       BOOLEAN
);

-- Dimension tables
CREATE TABLE dim_students (
    student_key      INT PRIMARY KEY,
    student_id       INT,
    student_name     VARCHAR(100),
    enrollment_year  INT,
    enrollment_type  VARCHAR(20)
);

CREATE TABLE dim_courses (
    course_key       INT PRIMARY KEY,
    course_id        INT,
    course_name      VARCHAR(100),
    department_name  VARCHAR(100),
    credits          INT,
    level            VARCHAR(20)
);

CREATE TABLE dim_date (
    date_key         INT PRIMARY KEY,
    full_date        DATE,
    year             INT,
    quarter          INT,
    month            INT,
    month_name       VARCHAR(20),
    day_of_week      VARCHAR(20),
    is_weekend       BOOLEAN,
    academic_term    VARCHAR(20)
);

CREATE TABLE dim_cities (
    city_key         INT PRIMARY KEY,
    city_name        VARCHAR(100),
    state            VARCHAR(100),
    country          VARCHAR(100),
    population       BIGINT,
    tier             VARCHAR(20)
);
```

Now the same analytical query becomes:

```sql
SELECT 
    c.department_name,
    AVG(f.gpa) AS avg_gpa
FROM fact_grades f
JOIN dim_courses c ON f.course_key = c.course_key
JOIN dim_students s ON f.student_key = s.student_key
JOIN dim_cities ci ON f.city_key = ci.city_key
WHERE s.enrollment_year = 2023
AND ci.population > 1000000
GROUP BY c.department_name;
```

Still five joins — but something fundamentally different happened. Look at what changed.

The normalized schema required the analyst to understand six tables and their relationships to write this query. In the star schema, the fact table is the center and every dimension table connects directly to it. There are no joins between dimension tables. The structure is flat and uniform. An analyst can look at the star schema once and immediately understand how to query any combination of facts and dimensions.

The fact table contains exactly what was measured. The dimension tables contain exactly the descriptive context. The query structure — join fact to whatever dimensions you need, filter on dimension attributes, aggregate the fact — is always the same regardless of which dimensions you choose.

---

### Surrogate keys — why dimension tables have their own primary keys

You noticed that the dimension tables have `student_key`, `course_key`, `date_key` — their own primary keys — separate from the natural identifiers like `student_id` and `course_id`. These are called **surrogate keys**. Understanding why they exist is important because it affects how your loading pipelines work.

**Natural keys** are identifiers that come from the source system — student_id, course_id, city name. They have real-world meaning.

**Surrogate keys** are artificial identifiers generated by the data warehouse — sequential integers with no real-world meaning, assigned purely for warehouse use.

Why use surrogate keys instead of natural keys?

**Natural keys change.** A university might change its course numbering system. A city might be renamed. A student's ID might be reissued to a different student after a system migration. When natural keys change, all foreign key references in the fact table that use the old key become invalid or incorrect. Surrogate keys never change — once assigned, they stay assigned to that specific version of that entity forever.

**Natural keys from different source systems conflict.** If your data warehouse combines data from two different university systems that both use student_id starting from 1, you have a collision. Surrogate keys are generated by the warehouse and are unique across all sources.

**Slowly Changing Dimensions require version tracking.** This is the most important reason, and we cover it next.

The surrogate key is always an integer. It is always system-generated. It has no meaning outside the data warehouse. In your loading pipeline, before writing a record to the fact table, you look up the surrogate key for each dimension value from the dimension table. If the dimension value does not exist yet, you insert it and get back the new surrogate key. This lookup process is called a **dimension lookup** and it is a core step in every ETL pipeline that loads a star schema.

---

### The date dimension — the most important dimension in any warehouse

Every star schema has a date dimension, and it is always pre-populated with every date for the foreseeable future (typically 10 to 20 years). This might seem strange — why not just store the date directly in the fact table?

Because storing a date as a foreign key to a dimension table lets you add rich time-based attributes to every fact without modifying the fact table. The date dimension contains: year, quarter, month, month name, week of year, day of week, day name, whether it is a weekend, whether it is a public holiday, what academic term it is.

A query asking "what is the average GPA per academic term" does not need to calculate academic terms from raw dates. It joins to the date dimension and groups by `academic_term`. The calculation was done once when the date dimension was built.

```python
from datetime import date, timedelta
import csv
from pathlib import Path


def build_date_dimension(
    start_date: date,
    end_date: date,
    output_path: str,
) -> int:
    """
    Pre-populate the date dimension with every date in the range.
    
    The date_key is an integer in YYYYMMDD format -- for example,
    January 15 2024 is 20240115. This format is readable as a number
    (you can compare date_keys directly) and is a common convention
    in data warehousing.
    """
    rows = []
    current = start_date
    
    while current <= end_date:
        date_key = int(current.strftime("%Y%m%d"))
        
        rows.append({
            "date_key": date_key,
            "full_date": current.isoformat(),
            "year": current.year,
            "quarter": (current.month - 1) // 3 + 1,
            "month": current.month,
            "month_name": current.strftime("%B"),
            "week_of_year": current.isocalendar()[1],
            "day_of_week": current.isoweekday(),
            "day_name": current.strftime("%A"),
            "is_weekend": current.isoweekday() >= 6,
            "academic_term": _get_academic_term(current),
        })
        
        current += timedelta(days=1)
    
    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    
    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=rows[0].keys())
        writer.writeheader()
        writer.writerows(rows)
    
    return len(rows)


def _get_academic_term(d: date) -> str:
    """Determine the academic term for a given date."""
    if d.month in (7, 8, 9, 10):
        return f"Monsoon {d.year}"
    if d.month in (11, 12, 1, 2):
        year = d.year if d.month >= 11 else d.year - 1
        return f"Winter {year}-{year + 1}"
    return f"Summer {d.year}"


rows_written = build_date_dimension(
    start_date=date(2020, 1, 1),
    end_date=date(2030, 12, 31),
    output_path="data/dim_date.csv",
)
print(f"Date dimension: {rows_written} rows")
```

`current.strftime("%B")` returns the full month name — "January", "February". `%A` returns the full day name — "Monday", "Tuesday".

`current.isocalendar()` returns a named tuple with year, week number, and weekday according to the ISO 8601 calendar. `[1]` gets the week number. ISO week numbers run from 1 to 52 or 53 and always start on Monday.

`current.isoweekday()` returns the day of the week as an integer where Monday is 1 and Sunday is 7. So `>= 6` means Saturday or Sunday — a weekend.

`(current.month - 1) // 3 + 1` is the quarter calculation. Month 1-3 is quarter 1, 4-6 is quarter 2, 7-9 is quarter 3, 10-12 is quarter 4. Subtracting 1 shifts months to 0-11. Integer dividing by 3 groups them into 0, 1, 2, 3. Adding 1 shifts back to 1, 2, 3, 4.

---

### Slowly Changing Dimensions — tracking history

This is the concept that most data modeling tutorials explain poorly because they describe it mechanically without explaining why it matters.

Consider a student who transfers from Mumbai to Delhi midway through their studies. The `dim_students` table has a `city` attribute. What do you do when the student's city changes?

**Option 1 — Overwrite.** Update the student's row: set `city = "Delhi"`. The student now looks like they have always been in Delhi. All historical fact records for this student now incorrectly say Delhi. If you run "average GPA by city" for last year, this student's grades from when they were in Mumbai are now counted under Delhi. The historical query is wrong.

**Option 2 — Add new row.** Insert a new row for the student with city = Delhi and a new surrogate key. Keep the old row with the Mumbai city. Historical fact records that point to the old surrogate key (Mumbai version) still correctly show Mumbai. New fact records point to the new surrogate key (Delhi version) and correctly show Delhi. Historical queries remain accurate.

These options are called **Slowly Changing Dimension types**, abbreviated SCD.

**SCD Type 1 — Overwrite (no history).**

Update the existing row. No historical record of the old value is kept. Use this when the change is a correction (the city was recorded wrong originally, not actually changed) or when historical accuracy does not matter for this attribute.

```python
def scd_type_1_update(
    dimension_table: list[dict],
    student_id: int,
    new_attributes: dict,
) -> list[dict]:
    """
    SCD Type 1: overwrite the existing record.
    No history is preserved. Previous values are lost.
    """
    for record in dimension_table:
        if record["student_id"] == student_id:
            record.update(new_attributes)
            break
    return dimension_table
```

**SCD Type 2 — Add new row (full history).**

Insert a new row with the new values. Mark the old row as expired. Use date ranges to indicate which version was current at what time. This is the standard approach for any attribute where historical accuracy matters.

```python
from datetime import date


def scd_type_2_update(
    dimension_table: list[dict],
    student_id: int,
    new_attributes: dict,
    change_date: date,
    next_surrogate_key: int,
) -> list[dict]:
    """
    SCD Type 2: add a new row and expire the old one.
    
    The dimension table gains one row per change.
    Every row has:
      - effective_date: when this version became current
      - expiry_date: when this version was superseded (NULL = still current)
      - is_current: boolean flag for easy filtering
    
    The fact table's surrogate key references will still point to the
    correct historical version of the dimension.
    """
    for record in dimension_table:
        if record["student_id"] == student_id and record["is_current"]:
            record["expiry_date"] = change_date
            record["is_current"] = False
            break
    
    new_record = {
        "student_key": next_surrogate_key,
        "student_id": student_id,
        "effective_date": change_date,
        "expiry_date": None,
        "is_current": True,
        **new_attributes,
    }
    dimension_table.append(new_record)
    
    return dimension_table


students = [
    {
        "student_key": 1,
        "student_id": 101,
        "name": "Arjun Mehta",
        "city": "Mumbai",
        "effective_date": date(2020, 1, 1),
        "expiry_date": None,
        "is_current": True,
    }
]

print("Before transfer:")
for s in students:
    print(f"  {s}")

students = scd_type_2_update(
    dimension_table=students,
    student_id=101,
    new_attributes={"name": "Arjun Mehta", "city": "Delhi"},
    change_date=date(2023, 6, 1),
    next_surrogate_key=2,
)

print("\nAfter transfer to Delhi:")
for s in students:
    print(f"  {s}")
```

Run this. Look at what happens. Two rows exist for student 101. The first row has `expiry_date=2023-06-01` and `is_current=False`. The second row has `expiry_date=None` and `is_current=True`.

Now any fact record with `student_key=1` represents a grade earned before the transfer — and the dimension lookup gives city="Mumbai". Any fact record with `student_key=2` represents a grade after the transfer — dimension lookup gives city="Delhi". History is perfectly preserved.

To query current data only:

```sql
SELECT * FROM dim_students WHERE is_current = TRUE;
```

To query what the data looked like on a specific date:

```sql
SELECT * FROM dim_students 
WHERE student_id = 101
AND effective_date <= '2022-01-01'
AND (expiry_date > '2022-01-01' OR expiry_date IS NULL);
```

**SCD Type 3 — Add new column (limited history).**

Add a column to the existing row: `current_city` and `previous_city`. This preserves exactly one previous value — useful when you only care about the most recent change and need to compare current versus previous.

```python
def scd_type_3_update(
    record: dict,
    new_city: str,
) -> dict:
    """
    SCD Type 3: store current and previous value in separate columns.
    Only one level of history is preserved.
    """
    record["previous_city"] = record.get("current_city", record.get("city"))
    record["current_city"] = new_city
    return record


student = {
    "student_id": 101,
    "name": "Arjun Mehta",
    "current_city": "Mumbai",
    "previous_city": None,
}

student = scd_type_3_update(student, "Delhi")
print(student)
# {"student_id": 101, "name": "Arjun Mehta",
#  "current_city": "Delhi", "previous_city": "Mumbai"}
```

The right SCD type depends entirely on the business requirement. Ask: do analysts need to see how this attribute looked at historical points in time? If yes, SCD Type 2. Is the change always a correction with no historical value? SCD Type 1. Do we only care about current versus one previous value? SCD Type 3.

---

### The snowflake schema — and why you usually should not use it

A snowflake schema is a normalized version of a star schema. Instead of dimension tables being flat, they have their own dimension tables. The `dim_courses` table would not store `department_name` directly — it would have a foreign key to a `dim_departments` table. The shape looks like a snowflake because the dimension tables branch out into further tables.

```
fact_grades
├── dim_students
├── dim_courses ──── dim_departments
├── dim_date
└── dim_cities ──── dim_states ──── dim_countries
```

Snowflake schemas reduce storage by eliminating redundancy — "Computer Science" is stored once in `dim_departments` rather than on every course row. This was important when storage was expensive.

Modern data warehouses make storage extremely cheap. A terabyte on Snowflake or BigQuery costs a few dollars per month. The savings from normalization are negligible.

What is not negligible is the query complexity that snowflake schemas add. Every query that needs department information must now join `fact_grades → dim_courses → dim_departments` instead of just `fact_grades → dim_courses`. Analysts must understand more relationships. Queries are harder to write. Query optimization becomes more complex.

The default choice in modern data warehousing is the star schema with flat, denormalized dimension tables. Use snowflake schemas only when you have a specific reason — very high cardinality dimensions, extremely large dimension tables, or a specific tool that benefits from the normalization.

---

### Grain — the most important decision in dimensional modeling

Every fact table has a **grain** — the level of detail that each row represents. Defining the grain is the first and most important decision when designing a fact table.

"What does one row in this fact table represent?"

For `fact_grades`, the grain is one grade for one student in one course in one academic term. One row per student-course-term combination.

If you tried to mix grains — some rows representing individual grades and other rows representing semester averages — the fact table would be unusable. A SUM of the gpa column would mix raw grades with averages, producing a meaningless number.

The grain must be consistent and atomic — the lowest level of detail that the business process generates. You can always aggregate atomic grain data to higher levels (average GPA by semester from individual grades). You can never disaggregate higher-level data back to atomic level (you cannot recover individual grades from a semester average).

```python
from dataclasses import dataclass
from datetime import date


@dataclass
class GradeFactRow:
    """
    One row in fact_grades.
    
    Grain: one grade for one student in one course in one term.
    Every row represents exactly this level of detail -- not a summary,
    not a student's overall record, not a course's overall statistics.
    
    Measurable facts (numeric values we aggregate):
    - gpa: the grade received (we average, min, max this)
    - credit_hours: the course weight (we sum this for credit totals)
    
    Everything else is a foreign key to a dimension table.
    """
    grade_key: int
    student_key: int    # FK to dim_students
    course_key: int     # FK to dim_courses
    date_key: int       # FK to dim_date
    city_key: int       # FK to dim_cities
    gpa: float | None
    credit_hours: int
    is_passing: bool
```

A common mistake is including attributes in the fact table that belong in a dimension table. If you put `student_name` in the fact table, you are storing the same name on every row for every grade that student received. This is redundancy. The student's name belongs in `dim_students`. The fact table only holds foreign keys and measurable facts.

Another common mistake is choosing too coarse a grain. If you design `fact_grades` at the semester level (one row per student per semester with average GPA), you cannot later answer questions about individual course performance. Start at the finest grain available. You can always aggregate — you can never disaggregate.

---

### Building a complete dimensional model — bringing it together

Let us build a complete, working dimensional model for the student data from our pipeline, including the loading code.

```python
from dataclasses import dataclass, field
from datetime import date
from pathlib import Path
import csv
import json


@dataclass
class DimStudent:
    student_key: int
    student_id: int
    student_name: str
    enrollment_year: int
    city_name: str
    effective_date: date
    expiry_date: date | None
    is_current: bool


@dataclass
class DimCourse:
    course_key: int
    course_id: int
    course_name: str
    department_name: str
    credits: int


@dataclass
class FactGrade:
    grade_key: int
    student_key: int
    course_key: int
    date_key: int
    gpa: float | None
    credit_hours: int
    is_passing: bool


class DimensionalModelLoader:
    """
    Loads source data into a star schema.
    
    This is a simplified in-memory implementation.
    In production, these tables live in a data warehouse like BigQuery
    and the lookups use SQL queries rather than Python dictionaries.
    """

    def __init__(self):
        self._students: dict[int, DimStudent] = {}
        self._courses: dict[int, DimCourse] = {}
        self._facts: list[FactGrade] = []

        self._student_key_counter = 1
        self._course_key_counter = 1
        self._grade_key_counter = 1

        self._student_id_to_key: dict[int, int] = {}
        self._course_id_to_key: dict[int, int] = {}

    def _get_or_create_student(
        self,
        student_id: int,
        name: str,
        enrollment_year: int,
        city: str,
    ) -> int:
        """
        Look up the surrogate key for a student.
        If the student does not exist in the dimension, create a new row.
        Returns the surrogate key.
        """
        if student_id in self._student_id_to_key:
            return self._student_id_to_key[student_id]

        key = self._student_key_counter
        self._student_key_counter += 1

        self._students[key] = DimStudent(
            student_key=key,
            student_id=student_id,
            student_name=name,
            enrollment_year=enrollment_year,
            city_name=city,
            effective_date=date(2020, 1, 1),
            expiry_date=None,
            is_current=True,
        )
        self._student_id_to_key[student_id] = key
        return key

    def _get_or_create_course(
        self,
        course_id: int,
        course_name: str,
        department: str,
        credits: int,
    ) -> int:
        """Look up or create a course dimension row."""
        if course_id in self._course_id_to_key:
            return self._course_id_to_key[course_id]

        key = self._course_key_counter
        self._course_key_counter += 1

        self._courses[key] = DimCourse(
            course_key=key,
            course_id=course_id,
            course_name=course_name,
            department_name=department,
            credits=credits,
        )
        self._course_id_to_key[course_id] = key
        return key

    def _get_date_key(self, d: date) -> int:
        """
        Convert a date to a date_key integer.
        Format: YYYYMMDD. January 15 2024 -> 20240115.
        """
        return int(d.strftime("%Y%m%d"))

    def load_source_record(
        self,
        student_id: int,
        student_name: str,
        enrollment_year: int,
        city: str,
        course_id: int,
        course_name: str,
        department: str,
        credits: int,
        grade_date: date,
        gpa: float | None,
    ) -> None:
        """
        Load one source record into the dimensional model.
        
        This is the core of the ETL load stage:
        1. Look up or create dimension rows
        2. Get surrogate keys
        3. Create fact row using surrogate keys
        """
        student_key = self._get_or_create_student(
            student_id, student_name, enrollment_year, city
        )
        course_key = self._get_or_create_course(
            course_id, course_name, department, credits
        )
        date_key = self._get_date_key(grade_date)

        fact = FactGrade(
            grade_key=self._grade_key_counter,
            student_key=student_key,
            course_key=course_key,
            date_key=date_key,
            gpa=gpa,
            credit_hours=credits,
            is_passing=gpa is not None and gpa >= 5.0,
        )
        self._grade_key_counter += 1
        self._facts.append(fact)

    def get_summary(self) -> dict:
        return {
            "students": len(self._students),
            "courses": len(self._courses),
            "facts": len(self._facts),
        }

    def query_avg_gpa_by_department(self) -> dict[str, float]:
        """
        Answer: what is the average GPA per department?
        
        In production this would be a SQL query against the warehouse.
        Here we implement it in Python to demonstrate the model works.
        """
        department_gpas: dict[str, list[float]] = {}

        for fact in self._facts:
            if fact.gpa is None:
                continue
            course = self._courses[fact.course_key]
            dept = course.department_name
            if dept not in department_gpas:
                department_gpas[dept] = []
            department_gpas[dept].append(fact.gpa)

        return {
            dept: round(sum(gpas) / len(gpas), 2)
            for dept, gpas in department_gpas.items()
        }


loader = DimensionalModelLoader()

source_records = [
    (1, "Arjun Mehta",  2022, "Mumbai",    101, "Database Systems", "Computer Science", 4, date(2023, 5, 15), 8.5),
    (1, "Arjun Mehta",  2022, "Mumbai",    102, "Data Structures",  "Computer Science", 4, date(2023, 5, 20), 7.2),
    (2, "Priya Sharma", 2022, "Delhi",     101, "Database Systems", "Computer Science", 4, date(2023, 5, 15), 9.1),
    (2, "Priya Sharma", 2022, "Delhi",     103, "Machine Learning", "Computer Science", 3, date(2023, 5, 22), 9.4),
    (3, "Rohan Verma",  2023, "Mumbai",    104, "Linear Algebra",   "Mathematics",      3, date(2023, 5, 18), 7.8),
    (4, "Sneha Iyer",   2021, "Chennai",   103, "Machine Learning", "Computer Science", 3, date(2023, 5, 22), 8.9),
    (4, "Sneha Iyer",   2021, "Chennai",   105, "Business Analytics","Management",      3, date(2023, 5, 25), None),
]

for record in source_records:
    loader.load_source_record(*record)

print("Dimensional model loaded:")
print(f"  {loader.get_summary()}")

print("\nAverage GPA by department:")
for dept, avg in loader.query_avg_gpa_by_department().items():
    print(f"  {dept}: {avg}")
```

The `*record` in the function call unpacks the tuple into individual arguments. A tuple `(1, "Arjun", ...)` becomes `load_source_record(1, "Arjun", ...)`. This works when the tuple's contents exactly match the function's parameter order.

Run this. Look at how many unique students and courses are created versus how many fact rows exist. Notice that Arjun appears in two fact rows but only one student dimension row. The dimension has been deduped correctly.

---

### The snowflake schema vs star schema decision in practice

Now that you understand both, here is the decision in plain terms.

In a data warehouse today — BigQuery, Snowflake, Redshift — storage costs roughly $20-30 per terabyte per month. A completely denormalized dimension table where "Computer Science" is repeated on every course row adds perhaps a few megabytes of storage for thousands of courses. That costs fractions of a cent.

The query complexity that normalization adds, however, is real. Every additional join is cognitive overhead for analysts, additional complexity for BI tools, and potentially additional query cost in compute-based pricing models.

Use star schema. The denormalization cost is negligible. The simplicity gain is significant. When you encounter a genuinely massive dimension table (hundreds of millions of rows) where denormalization becomes expensive, consider snowflaking only that specific dimension.

---

### Exercises — Lesson 3

1. Build a `dim_date` table as a Python function that generates rows for the year 2020 through 2025. Each row must have: date_key (YYYYMMDD integer), full_date, year, quarter, month, month_name, day_of_week (1=Monday, 7=Sunday), day_name, is_weekend, and academic_term (define your own semester logic). Write it to a CSV file and verify by reading back and checking that January 1 2024 has the correct values for all fields.
    
2. Take the `DimensionalModelLoader` from this lesson and add SCD Type 2 support for student city changes. Add a method `update_student_city(student_id, new_city, change_date)` that expires the current student dimension row and creates a new one. After calling it, verify that: the old row has an expiry_date set, the new row has is_current=True, and fact rows created before the change still correctly reference the old city through their student_key.
    
3. Design a star schema for a simple e-commerce system where customers place orders containing products. Define: one fact table for order line items (one row per product per order), and dimension tables for customers, products, and dates. For each table write out the column names and data types as Python dataclasses. Identify the grain of the fact table explicitly. Identify which dimension attributes might need SCD Type 2 treatment and explain why.
    

---

### Answers — Lesson 3

**Exercise 1**

```python
from datetime import date, timedelta
import csv
from pathlib import Path


def build_date_dimension(
    start_year: int,
    end_year: int,
    output_path: str,
) -> int:
    rows = []
    current = date(start_year, 1, 1)
    end = date(end_year, 12, 31)

    while current <= end:
        quarter = (current.month - 1) // 3 + 1

        if current.month in (7, 8, 9, 10):
            term = f"Monsoon {current.year}"
        elif current.month in (11, 12):
            term = f"Winter {current.year}-{current.year + 1}"
        elif current.month in (1, 2):
            term = f"Winter {current.year - 1}-{current.year}"
        else:
            term = f"Summer {current.year}"

        rows.append({
            "date_key": int(current.strftime("%Y%m%d")),
            "full_date": current.isoformat(),
            "year": current.year,
            "quarter": quarter,
            "month": current.month,
            "month_name": current.strftime("%B"),
            "day_of_week": current.isoweekday(),
            "day_name": current.strftime("%A"),
            "is_weekend": current.isoweekday() >= 6,
            "academic_term": term,
        })
        current += timedelta(days=1)

    Path(output_path).parent.mkdir(parents=True, exist_ok=True)
    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=rows[0].keys())
        writer.writeheader()
        writer.writerows(rows)

    return len(rows)


rows = build_date_dimension(2020, 2025, "data/dim_date.csv")
print(f"Generated {rows} date dimension rows")

with open("data/dim_date.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        if row["full_date"] == "2024-01-01":
            print("\nJanuary 1 2024:")
            for key, value in row.items():
                print(f"  {key}: {value}")
            break
```

**Exercise 2**

```python
from dataclasses import dataclass
from datetime import date
from typing import Optional


@dataclass
class DimStudent:
    student_key: int
    student_id: int
    student_name: str
    city_name: str
    effective_date: date
    expiry_date: Optional[date]
    is_current: bool


class SCDType2StudentDimension:

    def __init__(self):
        self._rows: list[DimStudent] = []
        self._key_counter = 1
        self._student_id_to_current_key: dict[int, int] = {}

    def add_student(
        self,
        student_id: int,
        name: str,
        city: str,
        effective_date: date = date(2020, 1, 1),
    ) -> int:
        key = self._key_counter
        self._key_counter += 1
        self._rows.append(DimStudent(
            student_key=key,
            student_id=student_id,
            student_name=name,
            city_name=city,
            effective_date=effective_date,
            expiry_date=None,
            is_current=True,
        ))
        self._student_id_to_current_key[student_id] = key
        return key

    def update_student_city(
        self,
        student_id: int,
        new_city: str,
        change_date: date,
    ) -> int:
        for row in self._rows:
            if row.student_id == student_id and row.is_current:
                row.expiry_date = change_date
                row.is_current = False
                old_name = row.student_name
                break
        else:
            raise KeyError(f"No current row found for student_id {student_id}")

        new_key = self._key_counter
        self._key_counter += 1
        self._rows.append(DimStudent(
            student_key=new_key,
            student_id=student_id,
            student_name=old_name,
            city_name=new_city,
            effective_date=change_date,
            expiry_date=None,
            is_current=True,
        ))
        self._student_id_to_current_key[student_id] = new_key
        return new_key

    def get_current_key(self, student_id: int) -> int:
        return self._student_id_to_current_key[student_id]

    def get_city_for_key(self, student_key: int) -> str:
        for row in self._rows:
            if row.student_key == student_key:
                return row.city_name
        raise KeyError(f"student_key {student_key} not found")

    def print_all(self) -> None:
        for row in self._rows:
            print(
                f"  key={row.student_key} id={row.student_id} "
                f"city={row.city_name} "
                f"effective={row.effective_date} "
                f"expiry={row.expiry_date} "
                f"current={row.is_current}"
            )


dim = SCDType2StudentDimension()
arjun_key_v1 = dim.add_student(101, "Arjun Mehta", "Mumbai")

historical_fact_student_key = arjun_key_v1

print("Before city change:")
dim.print_all()

arjun_key_v2 = dim.update_student_city(101, "Delhi", date(2023, 6, 1))

print("\nAfter city change:")
dim.print_all()

print(f"\nHistorical fact key {historical_fact_student_key} -> city: {dim.get_city_for_key(historical_fact_student_key)}")
print(f"New fact key {arjun_key_v2} -> city: {dim.get_city_for_key(arjun_key_v2)}")
```

**Exercise 3**

```python
from dataclasses import dataclass
from datetime import date
from typing import Optional


# GRAIN: One row per product line item per order.
# Example: Order 500 has 3 products -> 3 fact rows, all with order_key=500.
# This is the atomic grain of an e-commerce transaction.

@dataclass
class FactOrderLineItem:
    """
    One row = one product in one order.
    
    Grain: order line item level.
    Facts: quantity, unit_price, total_price, discount_amount.
    """
    line_item_key: int
    order_key: int
    customer_key: int
    product_key: int
    date_key: int
    quantity: int
    unit_price: float
    total_price: float
    discount_amount: float
    is_returned: bool


@dataclass
class DimCustomer:
    """
    SCD Type 2 candidates:
    - shipping_address: customers move. Historical orders should show
      the address at time of order, not current address.
    - customer_segment: if a customer is reclassified from 'standard'
      to 'premium', historical revenue should still be attributed to
      their segment at the time.
    
    SCD Type 1 candidates:
    - email: if email changes, we want the current email for
      communication. No need for history.
    """
    customer_key: int
    customer_id: int
    customer_name: str
    email: str
    city: str
    country: str
    customer_segment: str
    registration_date: date
    effective_date: date
    expiry_date: Optional[date]
    is_current: bool


@dataclass
class DimProduct:
    """
    SCD Type 2 candidates:
    - price: if a product's price changes, historical sales should
      reflect what the customer actually paid. But wait -- price is
      in the fact table (unit_price), not here. The dimension stores
      the list price for reference.
    - category: if a product is recategorized, historical sales analysis
      by category should reflect the category at time of sale.
    
    SCD Type 1 candidates:
    - product_name: minor name corrections are usually applied globally.
    """
    product_key: int
    product_id: int
    product_name: str
    category: str
    subcategory: str
    brand: str
    list_price: float
    cost_price: float
    is_active: bool
    effective_date: date
    expiry_date: Optional[date]
    is_current: bool


@dataclass
class DimDate:
    date_key: int
    full_date: date
    year: int
    quarter: int
    month: int
    month_name: str
    day_of_week: int
    day_name: str
    is_weekend: bool
    is_holiday: bool
    season: str


print("E-commerce star schema defined.")
print("\nFact table grain: one row per product line item per order")
print("Fact measures: quantity, unit_price, total_price, discount_amount, is_returned")
print("\nDimension tables: customer, product, date")
print("\nSCD Type 2 attributes:")
print("  DimCustomer: shipping_address/city, customer_segment")
print("  DimProduct: category (recategorization affects historical analysis)")
print("\nSCD Type 1 attributes:")
print("  DimCustomer: email (only current email needed)")
print("  DimProduct: product_name (corrections apply globally)")
```

Lesson 3 complete. Tell me when you are ready for Lesson 4.