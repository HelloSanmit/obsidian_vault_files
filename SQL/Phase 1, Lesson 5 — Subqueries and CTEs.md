
Understood. Let me be transparent about what happened.

Lessons 5, 6, and 7 were compressed. The core concepts are correct and complete, but the depth and the way I built each concept step by step was reduced compared to Lessons 1 through 4. Lessons 1 through 4 had better pacing — I explained the why before the what, used analogies, flagged traps, and built concepts on top of each other deliberately. Lessons 5, 6, and 7 leaned more toward reference-style explanations.

Given that you said concept depth is what matters most to you, I am going to redo Lessons 5, 6, and 7 properly. Same standard as Lessons 1 through 4. Exercises included at the end with answers, you attempt them whenever you want.

One clarification on the LeetCode point — noted. You will do those separately in another conversation. I will still mention relevant problem numbers at the end of the phase as a reference list, but I will not treat them as a blocker.

Starting Lesson 5 now, properly.

---

## Phase 1, Lesson 5 — Subqueries and CTEs

---

### Why subqueries exist — the problem they solve

Everything you have written so far in SQL has been a single, flat query. But real analytical questions are often layered. The answer to one question becomes the input to another question.

Consider this: find all students whose GPA is above the average GPA of the entire student body.

You already know how to compute the average GPA:

```sql
SELECT AVG(gpa) FROM students WHERE gpa IS NOT NULL;
```

And you know how to filter students above a certain GPA:

```sql
SELECT name, gpa FROM students WHERE gpa > 8.22;
```

But hardcoding 8.22 is wrong. The average changes as data changes. What you need is to make the result of the first query feed into the second query automatically. That is exactly what a subquery does.

---

### Subquery in WHERE — the most common pattern

```sql
SELECT name, gpa
FROM students
WHERE gpa > (SELECT AVG(gpa) FROM students WHERE gpa IS NOT NULL);
```

The database executes the inner query first — `SELECT AVG(gpa) FROM students WHERE gpa IS NOT NULL` — gets back a single number, then uses that number in the WHERE clause of the outer query as if you had typed it literally.

The inner query here returns exactly one value — one row, one column. This is called a **scalar subquery**. The outer query's WHERE clause treats it like any other single value. You can use it with =, >, <, and all other comparison operators.

What happens if the inner query returns more than one row when used with = ? The database throws an error. A scalar subquery must return exactly one row and one column. If there is any chance it returns multiple rows, use IN instead of =, which we cover next.

---

### Subquery returning a list — using IN

Sometimes the inner query returns multiple values, and you want to check whether each outer row's value is in that list.

```sql
SELECT name
FROM students
WHERE student_id IN (
    SELECT student_id
    FROM enrollments
    WHERE course_id = 101
);
```

The inner query returns a list of student IDs — everyone enrolled in course 101. The outer query then finds students whose student_id is anywhere in that list.

This is functionally similar to a JOIN, but reads differently. The JOIN version expresses "combine these two tables." The IN version expresses "find students who satisfy this condition about enrollments." For simple cases like this, both work. The IN version is sometimes clearer about intent.

The NOT IN version:

```sql
SELECT name
FROM students
WHERE student_id NOT IN (
    SELECT student_id
    FROM enrollments
    WHERE course_id = 101
);
```

This looks simple. It has a dangerous flaw that you must know about permanently.

---

### The NULL trap in NOT IN — one of the most important things in this entire phase

NOT IN breaks completely and silently when the subquery returns even one NULL value.

Here is why. NOT IN checks: is this value not equal to any value in the list? For each value in the list, the database evaluates `outer_value <> list_value`. If any comparison is UNKNOWN (which happens when list_value is NULL, because anything compared to NULL is UNKNOWN), the entire NOT IN condition becomes UNKNOWN rather than TRUE. Rows where the condition is UNKNOWN are filtered out, just like FALSE rows.

So if the subquery returns even a single NULL alongside legitimate values, NOT IN returns zero rows for the entire outer query. No error. No warning. Just silently wrong results.

In your dataset, student_id in enrollments cannot be NULL because of the foreign key constraint. So this specific query is safe. But in real-world data engineering, subqueries often touch columns without foreign key constraints, and NULLs sneak in. The habit to build now: when using NOT IN, always add `WHERE column IS NOT NULL` inside the subquery, or better, switch to NOT EXISTS which handles NULLs correctly by design.

---

### EXISTS and NOT EXISTS — the NULL-safe alternative

EXISTS does not check values. It checks whether the subquery returns at least one row.

```sql
SELECT name
FROM students s
WHERE EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
);
```

`SELECT 1` inside the subquery means "I do not care about the actual values, just tell me if any row matches." EXISTS returns TRUE the moment it finds the first matching row and stops looking. This makes it efficient — it does not need to count or collect all matches.

Notice `s.student_id` inside the subquery — this references the outer query's current row. The subquery is re-evaluated for each row of the outer query using that row's student_id. This is called a **correlated subquery** and we will cover it properly in a moment.

NOT EXISTS is the correct, NULL-safe way to find rows with no match:

```sql
SELECT name
FROM students s
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
);
```

This finds students with no enrollments. It works correctly regardless of NULLs because it never compares values — it only checks for the presence or absence of matching rows. Make NOT EXISTS your default choice over NOT IN for all "find things with no match" queries.

---

### Correlated subqueries — subqueries that reference the outer query

A correlated subquery references a column from the outer query inside the inner query. This means it cannot run independently — it depends on the current row of the outer query.

```sql
SELECT name, gpa, city
FROM students s
WHERE gpa > (
    SELECT AVG(gpa)
    FROM students
    WHERE city = s.city
    AND gpa IS NOT NULL
);
```

This finds students whose GPA is above the average of their own city. For each student row in the outer query, the inner query runs fresh using that student's city. Arjun's row triggers the inner query with city = 'Mumbai'. Priya's row triggers it with city = 'Delhi'.

The conceptual clarity here is excellent — it reads almost like plain English. The performance implication is serious. If the outer query produces one million rows, the inner query runs one million times. On large datasets this is extremely slow.

In modern SQL, correlated subqueries of this type are almost always replaced with window functions, which perform the same computation in a single pass. Understanding the correlated subquery first is important because it makes the window function's purpose immediately obvious when you see it.

---

### Subquery in FROM — derived tables

A subquery can appear in the FROM clause, producing a temporary table that the outer query treats like a regular table.

```sql
SELECT city, avg_gpa
FROM (
    SELECT city, AVG(gpa) AS avg_gpa
    FROM students
    WHERE gpa IS NOT NULL
    GROUP BY city
) AS city_averages
WHERE avg_gpa > 8.0;
```

The inner query computes average GPA per city and produces a temporary result set. The outer query then filters that result set to keep only cities where the average exceeds 8.0.

Why not just use HAVING for this? You could in this case — `HAVING AVG(gpa) > 8.0` would work. But derived tables in FROM become necessary in two important situations.

First, when you need to filter on a window function result. Window functions run after WHERE in the execution order, so you cannot filter on them in WHERE. You must wrap the window function in a derived table or CTE and filter in the outer query. We will see this exactly in Lesson 6.

Second, when the logic is complex enough that breaking it into layers makes it readable. This is where CTEs are superior to derived tables — they give each layer a name.

The alias after the closing parenthesis — `AS city_averages` — is mandatory in PostgreSQL. Every derived table must be named.

---

### CTEs — the right way to write layered queries

A CTE (Common Table Expression) uses the WITH keyword to define a named, temporary result set before the main query.

```sql
WITH city_averages AS (
    SELECT city, AVG(gpa) AS avg_gpa
    FROM students
    WHERE gpa IS NOT NULL
    GROUP BY city
)
SELECT city, avg_gpa
FROM city_averages
WHERE avg_gpa > 8.0;
```

This produces the same result as the derived table version. The difference is structural. The logic lives at the top, named clearly, and the final SELECT reads simply. Compare how easy it is to understand what this query does versus reading a nested subquery from the inside out.

The real power comes from chaining multiple CTEs:

```sql
WITH 
enrolled_students AS (
    SELECT DISTINCT student_id
    FROM enrollments
),
high_gpa AS (
    SELECT student_id, name, gpa
    FROM students
    WHERE gpa > 8.0
),
high_gpa_enrolled AS (
    SELECT h.name, h.gpa
    FROM high_gpa h
    INNER JOIN enrolled_students e ON h.student_id = e.student_id
)
SELECT name, gpa
FROM high_gpa_enrolled
ORDER BY gpa DESC;
```

Three CTEs. Each one builds on the previous. The final SELECT is three words. This is how complex data transformation queries should be written. Each CTE has a name that describes what it contains. Reading this query top to bottom tells you a story. Reading the equivalent nested subquery version requires working from the innermost parenthesis outward, holding the entire structure in your head simultaneously.

In dbt — the transformation tool in Phase 5 — every model file is essentially a CTE chain that gets compiled into SQL and run against your warehouse. The habits you build writing CTEs now will transfer directly.

---

### When to use what — a clear decision framework

**Subquery in WHERE with a scalar result** — when you need one computed value to filter against. Like filtering above the average.

**IN with a subquery** — when you need to check membership in a list derived from another table. Be careful with NOT IN and NULLs.

**EXISTS / NOT EXISTS** — when you need to check for the presence or absence of related rows. Default to NOT EXISTS over NOT IN for negative checks.

**Correlated subquery** — when you understand the performance implications and the data is small enough that it does not matter, or when you are writing it as a conceptual step before converting to a window function.

**Derived table in FROM** — when you need to query the result of an aggregation or window function. Fine for one-off queries.

**CTE** — whenever your query has more than one logical step. Default to CTEs over derived tables for anything you will maintain, share, or build upon. They are easier to debug — you can comment out the final SELECT and add `SELECT * FROM cte_name` to inspect any intermediate result.

---

### Exercises — Lesson 5

Attempt these whenever you want. Answers follow.

1. Find all students whose GPA is above the overall average GPA of the entire student body.
    
2. Find all students who are enrolled in both course 101 and course 102. Use subqueries, not JOINs.
    
3. Find all students who are not enrolled in course 103. Use NOT EXISTS.
    
4. Using a CTE, compute the average GPA per city, then find all cities where the average GPA is above 8.5. Show city name and average GPA.
    
5. Using a derived table in FROM, find the course with the highest number of enrollments. Show course_id and the count.
    
6. Write a correlated subquery that finds all students whose GPA is above the average GPA of their own city. Then rewrite it using a CTE and a JOIN instead. Compare the two queries.
    

---

### Answers — Lesson 5

**Exercise 1**

```sql
SELECT name, gpa
FROM students
WHERE gpa > (
    SELECT AVG(gpa) 
    FROM students 
    WHERE gpa IS NOT NULL
);
```

**Exercise 2**

```sql
SELECT name
FROM students
WHERE student_id IN (
    SELECT student_id FROM enrollments WHERE course_id = 101
)
AND student_id IN (
    SELECT student_id FROM enrollments WHERE course_id = 102
);
```

**Exercise 3**

```sql
SELECT name
FROM students s
WHERE NOT EXISTS (
    SELECT 1
    FROM enrollments e
    WHERE e.student_id = s.student_id
    AND e.course_id = 103
);
```

**Exercise 4**

```sql
WITH city_avg AS (
    SELECT city, ROUND(AVG(gpa), 2) AS avg_gpa
    FROM students
    WHERE gpa IS NOT NULL
    GROUP BY city
)
SELECT city, avg_gpa
FROM city_avg
WHERE avg_gpa > 8.5;
```

**Exercise 5**

```sql
SELECT course_id, enrollment_count
FROM (
    SELECT course_id, COUNT(*) AS enrollment_count
    FROM enrollments
    GROUP BY course_id
) AS counts
WHERE enrollment_count = (
    SELECT MAX(enrollment_count)
    FROM (
        SELECT COUNT(*) AS enrollment_count
        FROM enrollments
        GROUP BY course_id
    ) AS inner_counts
);
```

**Exercise 6 — correlated version**

```sql
SELECT name, city, gpa
FROM students s
WHERE gpa > (
    SELECT AVG(gpa)
    FROM students
    WHERE city = s.city
    AND gpa IS NOT NULL
);
```

**Exercise 6 — CTE version**

```sql
WITH city_avg AS (
    SELECT city, AVG(gpa) AS avg_gpa
    FROM students
    WHERE gpa IS NOT NULL
    GROUP BY city
)
SELECT s.name, s.city, s.gpa
FROM students s
INNER JOIN city_avg c ON s.city = c.city
WHERE s.gpa > c.avg_gpa;
```

The CTE version computes city averages once. The correlated subquery computes the city average once per student row. On 10 rows the difference is invisible. On 10 million rows the CTE version is orders of magnitude faster.

---

## Phase 1, Lesson 6 — Window Functions

---

### The problem that window functions solve

Go back to Exercise 6 in Lesson 5. You needed to compare each student's GPA to the average GPA of their own city. The correlated subquery solved it by running a fresh average computation for every single row — expensive. The CTE version precomputed city averages once and joined them back — better, but required two steps and a JOIN.

Window functions solve this in a single, elegant step. And they do things that neither subqueries nor GROUP BY can do at all — like ranking rows, computing running totals, or accessing the value of a neighboring row.

The fundamental insight: **window functions compute across a set of related rows while keeping every row in the result.** Aggregates collapse rows into groups. Window functions do not collapse anything. Every input row produces exactly one output row, plus the window function's computed value attached to it.

---

### The anatomy of a window function

Every window function has the same structure:

```sql
function_name() OVER (
    PARTITION BY column1
    ORDER BY column2
    ROWS/RANGE BETWEEN ... AND ...
)
```

The OVER clause is what makes any function a window function. Without OVER, AVG is an aggregate that collapses rows. With OVER, AVG is a window function that computes a value per row while keeping all rows.

PARTITION BY divides rows into groups — the function operates independently within each group. ORDER BY inside OVER defines a sequence within each partition for order-sensitive calculations like rankings and running totals. The ROWS/RANGE frame is advanced and we will touch it briefly — it defines exactly which rows within the partition the function considers for each row's calculation.

None of these are mandatory except OVER itself. A bare OVER () with nothing inside means the entire result set is one window.

---

### Starting simple — window aggregate over the entire table

```sql
SELECT 
    name,
    gpa,
    ROUND(AVG(gpa) OVER (), 2) AS overall_avg
FROM students
WHERE gpa IS NOT NULL;
```

The overall average is computed across all rows and attached to every row. Nine rows in the result (Sahil is excluded by WHERE), each showing their own GPA and the same overall average. No GROUP BY, no collapsing, no join needed.

Run this. Notice that the overall_avg column has the same value on every row. That is correct. The window is the entire result set and does not change per row.

---

### PARTITION BY — independent windows per group

```sql
SELECT 
    name,
    city,
    gpa,
    ROUND(AVG(gpa) OVER (PARTITION BY city), 2) AS city_avg
FROM students
WHERE gpa IS NOT NULL;
```

Now the average is computed independently per city. Every Mumbai student shows the Mumbai average. Every Delhi student shows the Delhi average. Different rows show different window function values because their city partitions are different.

Compare this to the correlated subquery from Lesson 5 that did exactly this. The window function expresses the same logic in one line and executes it in a single pass over the data.

---

### ORDER BY inside OVER — running calculations

Adding ORDER BY inside OVER changes the window function's behavior fundamentally. Instead of computing across the entire partition, it computes from the first row of the partition up to and including the current row, in the specified order. This produces running totals, running counts, and running averages.

```sql
SELECT 
    name,
    gpa,
    SUM(gpa) OVER (ORDER BY gpa DESC) AS running_total
FROM students
WHERE gpa IS NOT NULL;
```

The highest GPA student's running total is just their own GPA. The second student's running total is their GPA plus the first. The third is the sum of the first three. And so on.

This is not a quirk — it is the defined behavior. When ORDER BY appears inside OVER without an explicit frame clause, the default frame is "from the first row of the partition to the current row." The current row changes as you move down the result, so the computation changes per row.

---

### ROW_NUMBER, RANK, DENSE_RANK — understanding the differences precisely

These three ranking functions are among the most used window functions in all of data engineering. They look similar and are easy to confuse. The difference matters in production.

```sql
SELECT 
    name,
    gpa,
    ROW_NUMBER() OVER (ORDER BY gpa DESC)  AS row_num,
    RANK()       OVER (ORDER BY gpa DESC)  AS rnk,
    DENSE_RANK() OVER (ORDER BY gpa DESC)  AS dense_rnk
FROM students
WHERE gpa IS NOT NULL;
```

Run this. Look at what happens when two students have the same GPA — in your dataset, check if any students share a GPA value. If not, all three functions produce identical results. To see the difference clearly, understand the rules:

**ROW_NUMBER** — assigns a unique sequential integer to every row. Ties are broken arbitrarily (or by adding more columns to ORDER BY). If Priya and Divya both have 9.10, one gets 1 and the other gets 2 — the database decides which. No two rows ever get the same number.

**RANK** — tied rows get the same rank. The next rank after a tie group skips numbers equal to the size of the tie. If two students tie for rank 2, the next student gets rank 4. Ranks 2 and 3 are both "used" by the tied pair.

**DENSE_RANK** — tied rows get the same rank. The next rank after a tie group is the next consecutive integer, no skipping. If two students tie for rank 2, the next student gets rank 3.

The practical consequence: if you use RANK and filter for `rnk <= 3` hoping to get the top 3 students, you might get 4 students if two of them tie for rank 2. DENSE_RANK with `dense_rnk <= 3` gives you all students within the top 3 distinct GPA values. ROW_NUMBER with `row_num <= 3` gives you exactly 3 students regardless of ties. Which behavior you want depends on the business question.

---

### PARTITION BY with ranking — the most important pattern you will use

```sql
SELECT 
    name,
    city,
    gpa,
    RANK() OVER (PARTITION BY city ORDER BY gpa DESC) AS city_rank
FROM students
WHERE gpa IS NOT NULL;
```

Now ranking is independent per city. Divya Nair is rank 1 in Delhi. Priya Sharma is rank 2 in Delhi. Arjun Mehta is rank 1 in Mumbai. These rankings reset for each city partition.

The pattern that emerges from this — get the top N per group — is one of the most common queries in data engineering:

```sql
WITH city_ranked AS (
    SELECT 
        name,
        city,
        gpa,
        RANK() OVER (PARTITION BY city ORDER BY gpa DESC) AS city_rank
    FROM students
    WHERE gpa IS NOT NULL
)
SELECT name, city, gpa
FROM city_ranked
WHERE city_rank = 1;
```

This finds the highest GPA student from every city. The CTE wraps the window function, and then the outer query filters on the rank. This is why window function results must be filtered in an outer query — the WHERE clause runs before window functions in the execution order, so you cannot write `WHERE city_rank = 1` in the same query that computes city_rank. The CTE or derived table wrapper is not optional here — it is required by the execution order.

You will use this pattern constantly. Deduplication: keep only the most recent version of each record. Prioritization: show only the highest-revenue product per category. Session analysis: find the first event per user session. The pattern is always PARTITION BY the group, ORDER BY the criterion, rank, then filter.

---

### LAG and LEAD — accessing neighboring rows

LAG accesses the value from the previous row in the window. LEAD accesses the value from the next row.

```sql
SELECT 
    student_id,
    name,
    gpa,
    LAG(gpa)  OVER (ORDER BY student_id) AS prev_gpa,
    LEAD(gpa) OVER (ORDER BY student_id) AS next_gpa
FROM students;
```

The first row has no previous row — LAG returns NULL. The last row has no next row — LEAD returns NULL. Everything in the middle shows the adjacent row's value.

The second argument specifies the offset — how many rows back or forward:

```sql
LAG(gpa, 2) OVER (ORDER BY student_id)
```

This looks 2 rows back instead of 1. The third argument is the default value to return instead of NULL when there is no row at that offset:

```sql
LAG(gpa, 1, 0.0) OVER (ORDER BY student_id)
```

Returns 0.0 for the first row instead of NULL.

In data engineering, LAG and LEAD are the primary tools for time-series analysis. Computing day-over-day revenue change, detecting state transitions (a customer going from active to churned), calculating time between events — all of these use LAG or LEAD against a timestamp or sequential ID column.

---

### FIRST_VALUE and LAST_VALUE

```sql
SELECT 
    name,
    city,
    gpa,
    FIRST_VALUE(name) OVER (
        PARTITION BY city 
        ORDER BY gpa DESC
    ) AS best_student_in_city
FROM students
WHERE gpa IS NOT NULL;
```

FIRST_VALUE returns the value from the first row of the window, defined by PARTITION BY and ORDER BY. Every student in Mumbai sees the name of the Mumbai student with the highest GPA on their row. Every Delhi student sees the top Delhi student. Without needing a separate lookup or join.

LAST_VALUE has a subtle behavior that surprises almost everyone. By default, the window frame ends at the current row, not the last row of the partition. So LAST_VALUE returns the current row's own value most of the time, not the actual last row in the partition. To get the true last value, you need to explicitly extend the frame:

```sql
LAST_VALUE(name) OVER (
    PARTITION BY city
    ORDER BY gpa DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` means: consider all rows from the very first to the very last in the partition, regardless of where the current row is. This is the window frame. The default frame when ORDER BY is present is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — which is why LAST_VALUE behaves unexpectedly. Always specify the frame explicitly when using LAST_VALUE.

---

### Exercises — Lesson 6

1. For each student, show their name, GPA, and the overall average GPA of all students. Do not exclude NULLs in the WHERE clause — include all students. Observe what happens to Sahil's row.
    
2. For each student, show their name, city, GPA, and the average GPA of their city.
    
3. Rank all students by GPA descending using all three ranking functions. Then add a fourth column showing what ROW_NUMBER would assign if you also included student_id as a tiebreaker in the ORDER BY.
    
4. Find the top GPA student from each city using the RANK window function with a CTE.
    
5. For each student, show their name, GPA, and the GPA of the student with the immediately lower student_id. Show 0.0 for the first student instead of NULL.
    
6. Show each student's name, city, GPA, and the name of the highest GPA student in their city — without using a JOIN or subquery. Use a window function only.
    
7. Using ROW_NUMBER, show only the 3rd and 4th highest GPA students overall. Handle NULLs by putting them last.
    

---

### Answers — Lesson 6

**Exercise 1**

```sql
SELECT 
    name,
    gpa,
    ROUND(AVG(gpa) OVER (), 2) AS overall_avg
FROM students;
```

Sahil's row shows his GPA as NULL and the overall_avg computed excluding his NULL (AVG ignores NULLs automatically).

**Exercise 2**

```sql
SELECT 
    name,
    city,
    gpa,
    ROUND(AVG(gpa) OVER (PARTITION BY city), 2) AS city_avg
FROM students
WHERE gpa IS NOT NULL;
```

**Exercise 3**

```sql
SELECT 
    name,
    gpa,
    ROW_NUMBER() OVER (ORDER BY gpa DESC NULLS LAST)                      AS row_num,
    RANK()       OVER (ORDER BY gpa DESC NULLS LAST)                      AS rnk,
    DENSE_RANK() OVER (ORDER BY gpa DESC NULLS LAST)                      AS dense_rnk,
    ROW_NUMBER() OVER (ORDER BY gpa DESC NULLS LAST, student_id ASC)      AS row_num_tiebreak
FROM students;
```

**Exercise 4**

```sql
WITH city_ranked AS (
    SELECT 
        name, city, gpa,
        RANK() OVER (PARTITION BY city ORDER BY gpa DESC) AS rnk
    FROM students
    WHERE gpa IS NOT NULL
)
SELECT name, city, gpa
FROM city_ranked
WHERE rnk = 1;
```

**Exercise 5**

```sql
SELECT 
    name,
    gpa,
    LAG(gpa, 1, 0.0) OVER (ORDER BY student_id) AS prev_student_gpa
FROM students;
```

**Exercise 6**

```sql
SELECT 
    name,
    city,
    gpa,
    FIRST_VALUE(name) OVER (
        PARTITION BY city 
        ORDER BY gpa DESC NULLS LAST
    ) AS top_student_in_city
FROM students
WHERE gpa IS NOT NULL;
```

**Exercise 7**

```sql
WITH ranked AS (
    SELECT 
        name,
        gpa,
        ROW_NUMBER() OVER (ORDER BY gpa DESC NULLS LAST) AS rn
    FROM students
)
SELECT name, gpa, rn
FROM ranked
WHERE rn IN (3, 4);
```

---

## Phase 1, Lesson 7 — Query Optimization and EXPLAIN ANALYZE

---

### Why this lesson belongs in the SQL phase

Most SQL courses teach you to write queries that produce correct results. They stop there. A data engineer needs more — you need queries that produce correct results efficiently, on tables with millions or billions of rows, inside automated pipelines that run every hour.

A query that takes 3 seconds on your 10-row test table can take 6 hours on a production table with 500 million rows if you made the wrong structural choices. The difference between 6 hours and 3 minutes is usually not a faster machine — it is understanding how the database executes your query and removing the bottlenecks.

---

### What happens between writing SQL and seeing results

When you press run in pgAdmin, PostgreSQL does four things before returning a single row.

**Parsing.** Your SQL text is checked for syntax errors and converted into an internal tree structure called a parse tree. If you have a syntax error, it is caught here.

**Rewriting.** If your query references a view (a saved query), the view definition is substituted in. Rules and triggers are also applied at this stage.

**Planning.** This is the most important stage for performance. The query planner examines your query, looks at statistics it has collected about your tables — how many rows exist, how values are distributed, which indexes exist — and generates multiple possible ways to execute the query. It estimates the cost of each approach and selects the cheapest one.

The planner is sophisticated but imperfect. It works from statistics, and statistics can be outdated or misleading for unusual data distributions. Sometimes the planner makes a wrong choice. Understanding the plan lets you diagnose when this happens and fix it.

**Execution.** The chosen plan is executed. Results are returned row by row or in batches.

---

### EXPLAIN — reading the plan before execution

```sql
EXPLAIN
SELECT name FROM students WHERE city = 'Mumbai';
```

EXPLAIN shows you the execution plan without running the query. Useful for checking expensive queries before committing them.

```sql
EXPLAIN ANALYZE
SELECT name FROM students WHERE city = 'Mumbai';
```

EXPLAIN ANALYZE actually runs the query and shows both the planned approach and the real measured execution statistics. This is what you use when diagnosing why a real query is slow. The word ANALYZE here is not the SQL ANALYZE statement — it is a modifier to EXPLAIN that triggers actual execution and measurement.

Run the EXPLAIN ANALYZE version now in pgAdmin and look at the output.

---

### Reading the output — what every term means

The output is a tree of operations. Inner nodes run first and feed results upward to outer nodes. Indentation shows the nesting. Here is a real example structure you will see:

```
Seq Scan on students  
  (cost=0.00..1.10 rows=3 width=36)  
  (actual time=0.012..0.018 rows=3 loops=1)
  Filter: ((city)::text = 'Mumbai'::text)
  Rows Removed by Filter: 7
```

**Seq Scan** — Sequential scan. The database reads every single row in the table from beginning to end, applying the filter as it goes. For a 10-row table this is fine. For a 100-million-row table this means reading 100 million rows to find 3. This is the number one performance problem in production SQL.

**cost=0.00..1.10** — Planner's estimated cost. The first number is startup cost — cost before the first row is returned. The second number is total cost. These are not milliseconds — they are abstract planner units used only for comparison between plans. Lower is better.

**rows=3** — Planner's estimate of how many rows this node will return. If this is wildly different from the actual rows, the planner made a bad estimate, usually because table statistics are outdated or the data distribution is unusual.

**width=36** — Estimated average width of each output row in bytes.

**actual time=0.012..0.018** — Real execution time in milliseconds. First number is time to first row. Second is total time. This is what you actually care about.

**loops=1** — How many times this node executed. In a nested loop join, an inner node might execute once per outer row — so loops would be the number of outer rows.

**Rows Removed by Filter: 7** — 7 rows were read and discarded by the WHERE clause. Only 3 matched.

---

### The scan types — understanding what the planner chose and why

**Seq Scan** — reads every row. The only option when no usable index exists or when the query is not selective enough to justify an index.

**Index Scan** — uses a B-tree index to jump directly to matching rows, then fetches the full row from the table. Fast for selective queries — when your WHERE clause matches a small fraction of rows.

**Index Only Scan** — uses an index and does not need to fetch the full row at all, because every column you need is already in the index. The fastest scan type.

**Bitmap Index Scan + Bitmap Heap Scan** — used when a moderate number of rows match. The index finds all matching row locations, builds a bitmap in memory, sorts them by physical location, then fetches all matching rows in physical order. More efficient than many individual Index Scans when many rows match.

Run these two queries and compare the plan for each:

```sql
EXPLAIN ANALYZE SELECT name FROM students WHERE student_id = 3;
EXPLAIN ANALYZE SELECT name FROM students WHERE city = 'Mumbai';
```

The first should use an Index Scan — student_id is a primary key and has an automatic index. The second will likely use a Seq Scan — no index on city yet.

---

### Indexes — the most important performance tool

An index is a separate data structure maintained alongside the table. Its purpose is to allow the database to find rows matching a condition without reading every row in the table.

The analogy to a book index is exact. A book with 1000 pages and no index forces you to read every page to find mentions of "normalization." A book index lists topics alphabetically and tells you exactly which pages to turn to. A database index stores column values in a sorted structure with pointers to the actual row locations on disk.

The default and most common index type is a B-tree — a balanced tree structure where values are stored in sorted order. Finding a specific value is O(log n) — dramatically faster than the O(n) sequential scan for large tables.

Primary key columns automatically get a B-tree index. Foreign key columns do not in PostgreSQL — you must create them manually. You almost always should, because joins on foreign keys without indexes degrade to nested loop scans.

Create an index on city and observe the difference:

```sql
CREATE INDEX idx_students_city ON students(city);
```

Now run EXPLAIN ANALYZE on the city = 'Mumbai' query again. On your 10-row table the planner may still choose a Seq Scan. This is correct — for tiny tables, the overhead of using an index exceeds the cost of just reading 10 rows. On a large table the planner would switch to an Index Scan. The planner is making the right decision in both cases.

---

### When indexes help and when they actively hurt

**Indexes help when:**

- Your WHERE clause is selective — it matches a small percentage of rows
- You frequently join on that column
- You frequently sort by that column and can use an Index Scan to avoid a sort operation

**Indexes hurt when:**

- You are inserting, updating, or deleting large numbers of rows — every write must also update all indexes. A table with 10 indexes takes roughly 10x longer to load data into
- The query returns most of the table anyway — an index on a column with only two distinct values (like a boolean) rarely helps because each value matches half the table
- The indexed column is inside a function in the WHERE clause — `WHERE UPPER(city) = 'MUMBAI'` cannot use an index on city because the index stores raw values, not uppercased values

**In data engineering specifically:** when loading millions of rows into a table as part of a pipeline, the standard pattern is to drop indexes before the load and rebuild them afterward. The rebuild is faster than maintaining the index row by row during the insert.

---

### Common performance problems and their solutions

**Seq Scan on a large table with a selective filter** — add an index on the filter column.

**N+1 queries** — running one query to get a list, then running a separate query per item in application code. 1000 items means 1001 database round trips. In a pipeline this is catastrophic. Solution: use a single JOIN or IN clause.

**Function on an indexed column in WHERE**

```sql
-- Cannot use index on city
WHERE UPPER(city) = 'MUMBAI'

-- Can use index on city
WHERE city = 'Mumbai'
```

Store data consistently so you do not need to transform it at query time. Or create a functional index: `CREATE INDEX ON students(UPPER(city))`.

**Implicit type mismatch**

```sql
-- student_id is INT, '3' is text -- may prevent index use
WHERE student_id = '3'

-- Correct -- types match
WHERE student_id = 3
```

Always match the literal type to the column type.

**Cartesian product from a missing JOIN condition** — joining two tables with no ON clause or a condition that is always true produces every combination of rows. On large tables this produces billions of rows and crashes pipelines. Always verify your JOIN conditions are correct before running on large data.

**Outdated statistics** — the planner's estimates depend on statistics collected by the ANALYZE command. PostgreSQL runs this automatically in the background, but sometimes statistics fall behind, especially after large data loads. If the planner is making obviously wrong choices about row counts, run:

```sql
ANALYZE students;
```

This refreshes the statistics and gives the planner accurate information.

---

### Exercises — Lesson 7

1. Run EXPLAIN ANALYZE on a query that filters by student_id. Identify every term in the output — what scan type, what the cost numbers mean, what actual time means.
    
2. Run EXPLAIN ANALYZE on a query that filters by city before and after creating an index on city. Note what changes between the two plans. Note whether the planner uses the index on a 10-row table and explain why or why not.
    
3. Run EXPLAIN ANALYZE on a three-table JOIN query. Identify what join types PostgreSQL chose — Hash Join, Nested Loop, or Merge Join. Look up what each join type means in terms of how the database executes it.
    
4. Deliberately write a query with a function on an indexed column in WHERE — use UPPER(name). Run EXPLAIN ANALYZE. Confirm it uses a Seq Scan. Then create a functional index and run again.
    
5. Write a query that has a bug that would cause severe performance problems on a large table but runs fine on your 10-row table. Identify what the bug is and how you would fix it. There are multiple valid answers — think about everything covered in this lesson.
    

---

### Answers — Lesson 7

**Exercise 1**

```sql
EXPLAIN ANALYZE
SELECT name FROM students WHERE student_id = 3;
```

You will see Index Scan using the primary key index. cost=startup..total. actual time=milliseconds to first row..total. loops=1 because this node runs once. rows=1 because one student has student_id=3.

**Exercise 2**

```sql
-- Before index
EXPLAIN ANALYZE SELECT name FROM students WHERE city = 'Mumbai';

CREATE INDEX idx_students_city ON students(city);

-- After index
EXPLAIN ANALYZE SELECT name FROM students WHERE city = 'Mumbai';
```

On 10 rows: likely Seq Scan both times. The planner correctly decides that scanning 10 rows is cheaper than the overhead of an index lookup. This is correct planner behavior. On millions of rows it would switch to Index Scan.

**Exercise 3**

```sql
EXPLAIN ANALYZE
SELECT s.name, c.course_name, e.grade
FROM students s
INNER JOIN enrollments e ON s.student_id = e.student_id
INNER JOIN courses c ON e.course_id = c.course_id;
```

On small tables PostgreSQL typically chooses Hash Join — one table is hashed into memory, the other is scanned and each row is looked up in the hash. Nested Loop is chosen when one table is small and the inner table has an index on the join column.

**Exercise 4**

```sql
-- Seq Scan expected -- UPPER prevents index use
EXPLAIN ANALYZE SELECT name FROM students WHERE UPPER(name) LIKE 'A%';

-- Create functional index
CREATE INDEX idx_students_upper_name ON students(UPPER(name));

-- Now may use index
EXPLAIN ANALYZE SELECT name FROM students WHERE UPPER(name) LIKE 'A%';
```

Note: LIKE with a leading non-wildcard character can use a B-tree index. LIKE '%something' cannot.

**Exercise 5**  
Multiple valid answers. Examples:

- A query with `WHERE student_id = '3'` — type mismatch, may prevent index use at scale
- A query joining students to enrollments with no ON clause — produces a cross join, 10 × 14 = 140 rows on this data, billions on large tables
- A correlated subquery that runs the inner query once per outer row — on millions of rows this becomes millions of inner queries

---

## Phase 1 complete

You now have a thorough foundation in SQL at the level a data engineer needs. Here is everything covered across the seven lessons:

The relational model — why it exists, what problems it solves, how tables, rows, columns, and keys work. SQL execution order — FROM, WHERE, GROUP BY, HAVING, SELECT, ORDER BY, LIMIT — and why it matters for every query you write. SELECT, WHERE, all filtering operators, NULL semantics and why `= NULL` never works. Aggregations, GROUP BY, HAVING, the distinction between COUNT(*) and COUNT(column). All four JOIN types, row multiplication, the LEFT JOIN IS NULL pattern for finding missing relationships. Subqueries in WHERE, FROM, and as correlated queries. EXISTS and NOT EXISTS and when to use them over IN and NOT IN. CTEs for writing layered, readable, maintainable queries. Window functions — OVER, PARTITION BY, ORDER BY inside OVER — the execution model. ROW_NUMBER, RANK, DENSE_RANK and their precise differences. LAG, LEAD, FIRST_VALUE for accessing neighboring and extreme rows. EXPLAIN ANALYZE — reading execution plans, understanding scan types, join types, cost estimates. Indexes — how they work, when they help, when they hurt, functional indexes.

When you are ready, tell me and we move to Phase 2 — Python for Data Engineering.