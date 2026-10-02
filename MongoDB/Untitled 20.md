
# Gap Module 3 — Advanced Aggregation + Advanced Indexing

This fills the most important omissions from **Lessons 5, 6, and 7**.

The structure:

```text
PART A — Advanced Aggregation
Lesson 5 additions
├── $count
├── $replaceRoot / $replaceWith
├── $sortByCount
├── $sample
├── $bucket / $bucketAuto
└── $out / $merge

Lesson 6 additions
├── string expressions
├── date expressions
├── $let
├── $range
├── $mergeObjects
├── $objectToArray / $arrayToObject
└── $setWindowFields

PART B — Advanced Indexing
Lesson 7 additions
├── TTL indexes
├── wildcard indexes
├── text indexes / $text
├── $hint
├── allPlansExecution
├── hideIndex / unhideIndex
└── index intersection
```

---

# PART A — Advanced Aggregation

# 1. `$count` — the aggregation stage

You already know:

```js
db.jobs.countDocuments({
  status: "OPEN"
})
```

But inside an aggregation pipeline, you can use:

```js
{
  $count: "total"
}
```

Example:

```js
db.jobs.aggregate([
  {
    $match: {
      status: "OPEN"
    }
  },
  {
    $count: "total"
  }
])
```

Possible result:

```js
[
  {
    total: 1250
  }
]
```

### Simple meaning

```text
$match
→ choose documents

$count
→ count whatever remains
```

So:

```text
100,000 jobs
   ↓
$match
   ↓
1,250 jobs
   ↓
$count
   ↓
1,250
```

---

# 2. Why `$count` isn't the same as `countDocuments()`

They can produce the same numerical answer, but they exist in different contexts.

### `countDocuments()`

A collection operation:

```js
db.jobs.countDocuments({
  status: "OPEN"
})
```

### `$count`

An aggregation stage:

```js
db.jobs.aggregate([
  { $match: { status: "OPEN" } },
  { $count: "total" }
])
```

The second becomes especially useful when the count is only one part of a more complex aggregation.

For example:

```text
$match
 ↓
$group
 ↓
$project
 ↓
$count
```

---

# 3. `$replaceRoot`

Now we get into document restructuring.

Suppose:

```js
{
  _id: 1,
  user: {
    name: "Alex",
    email: "alex@example.com"
  }
}
```

You want the result to become:

```js
{
  name: "Alex",
  email: "alex@example.com"
}
```

You can use:

```js
{
  $replaceRoot: {
    newRoot: "$user"
  }
}
```

### Simple meaning

> Replace the **entire current document** with another document.

Think:

```text
current document
       ↓
choose embedded document
       ↓
make that embedded document the new root
```

---

# 4. Why would you need `$replaceRoot`?

Aggregation often creates intermediate structures.

Suppose:

```js
{
  company: {
    name: "Example",
    city: "Pune"
  },

  metrics: {
    jobs: 20,
    applications: 500
  }
}
```

You might want:

```js
{
  name: "Example",
  city: "Pune",
  jobs: 20,
  applications: 500
}
```

You can combine `$mergeObjects` with `$replaceRoot`.

Example:

```js
{
  $replaceRoot: {
    newRoot: {
      $mergeObjects: [
        "$company",
        "$metrics"
      ]
    }
  }
}
```

Now you've transformed the structure.

---

# 5. `$replaceWith`

MongoDB also provides:

```js
{
  $replaceWith: "$user"
}
```

This is the newer, more expression-oriented form of the same general idea.

Mental model:

```text
$replaceRoot
=
replace current root document

$replaceWith
=
expression-based version of the same concept
```

You don't need to memorize both immediately.

Just recognize:

> **Sometimes aggregation needs to completely reshape a document.**

---

# 6. `$sortByCount`

Suppose you want:

> How many applications are in each status?

You could write:

```js
db.applications.aggregate([
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  },
  {
    $sort: {
      count: -1
    }
  }
])
```

There is a shorthand:

```js
{
  $sortByCount: "$status"
}
```

Result conceptually:

```js
[
  { _id: "APPLIED", count: 1200 },
  { _id: "SCREENING", count: 600 },
  { _id: "INTERVIEW", count: 200 },
  { _id: "HIRED", count: 50 }
]
```

### Simple meaning

> **Group by this expression and sort groups by count descending.**

It's basically a convenience stage.

---

# 7. `$sortByCount` vs `$group + $sort`

### Explicit:

```text
$group
→ count
$sort
→ highest first
```

### Shorthand:

```text
$sortByCount
```

When teaching yourself aggregation, I recommend understanding the explicit form first.

Use `$sortByCount` when the shorter expression makes the pipeline clearer.

---

# 8. `$sample`

Suppose you want:

> Give me 5 random jobs.

Use:

```js
db.jobs.aggregate([
  {
    $sample: {
      size: 5
    }
  }
])
```

### Simple meaning

> Randomly select documents from the pipeline.

This is useful for things like:

```text
recommendation experiments
random examples
testing
sampling datasets
“discover jobs” UI
```

---

# 9. Don't confuse `$sample` with `$limit`

```js
{
  $limit: 5
}
```

means:

> Take the first 5 according to the pipeline's current ordering.

Whereas:

```js
{
  $sample: {
    size: 5
  }
}
```

means:

> Choose a random sample.

So:

```text
$limit
=
first N

$sample
=
random N
```

---

# 10. `$bucket`

Now we get to a very useful analytical capability.

Suppose you have candidate experience:

```text
0
1
2
3
4
5
...
```

You want:

```text
0–2 years
3–5 years
6–10 years
11+ years
```

This is called **bucketing**.

You can use:

```js
{
  $bucket: {
    groupBy: "$experienceYears",

    boundaries: [
      0,
      3,
      6,
      11
    ],

    default: "11+",

    output: {
      count: {
        $sum: 1
      }
    }
  }
}
```

Conceptually:

```text
0,1,2
→ bucket 0

3,4,5
→ bucket 1

6,7,8,9,10
→ bucket 2

11+
→ default
```

---

# 11. Why `$bucket` exists

Normal grouping asks:

```text
“Group by this exact value.”
```

For example:

```text
2 years → group
3 years → group
4 years → group
```

But `$bucket` asks:

> **Group values into ranges.**

That's extremely useful for analytics.

Examples:

```text
salary ranges
age ranges
experience ranges
response-time ranges
price ranges
```

---

# 12. Example: salary distribution

Suppose salaries are:

```text
500000
700000
900000
1200000
1800000
2500000
```

You might bucket:

```text
0–10L
10–20L
20L+
```

Then your dashboard can show a histogram-like distribution.

That's much more useful than grouping by exact salary.

---

# 13. `$bucketAuto`

Now imagine you **don't know the best boundaries**.

For example, you have:

```text
millions of salaries
```

and want:

```text
5 roughly even buckets
```

You can use:

```js
{
  $bucketAuto: {
    groupBy: "$salary.min",
    buckets: 5
  }
}
```

### Simple meaning

> MongoDB chooses bucket boundaries automatically to create approximately balanced buckets.

This is useful for exploratory data analysis.

---

# 14. `$bucket` vs `$bucketAuto`

### `$bucket`

You decide:

```text
0–10L
10–20L
20–30L
```

### `$bucketAuto`

MongoDB determines suitable boundaries based on the data.

Mental model:

```text
$bucket
=
I know my ranges

$bucketAuto
=
figure out reasonable ranges from the data
```

---

# 15. `$out`

Now we move from:

> “Calculate some results”

to:

> “Store those results somewhere.”

Suppose you run analytics:

```text
jobs
 ↓
aggregation
 ↓
company statistics
```

You can use:

```js
{
  $out: "companyStatistics"
}
```

Conceptually:

```text
source collection
      ↓
aggregation
      ↓
companyStatistics
```

The result is written to the specified collection.

---

# 16. Why would `$out` be useful?

Suppose a dashboard performs an expensive aggregation over:

```text
500 million applications
```

every time someone opens it.

That's potentially wasteful.

Instead:

```text
raw data
 ↓
periodic aggregation
 ↓
summary collection
 ↓
dashboard reads summary
```

Now the dashboard doesn't repeatedly calculate everything from raw data.

This connects to the **Computed Pattern** from Lesson 3.

---

# 17. `$merge`

`$merge` is more flexible.

Instead of simply replacing a target collection with aggregation output, you can merge results into an existing collection according to rules.

Conceptually:

```text
source data
   ↓
aggregation
   ↓
$merge
   ↓
existing summary collection
```

This is useful for:

```text
incremental summaries
materialized views
upserts
periodic analytics
data pipelines
```

---

# 18. `$out` vs `$merge`

Think:

```text
$out
=
write aggregation result to a collection

$merge
=
merge aggregation result into an existing collection
```

`$merge` gives you much more control over how matching documents are handled.

For example, conceptually:

```text
matching key exists
→ update/replace/merge

no matching key
→ insert
```

---

# 19. Why `$merge` is powerful

Imagine:

```text
Applications
   ↓
group by company
   ↓
calculate today's metrics
   ↓
$merge
   ↓
CompanyDailyMetrics
```

Now your analytics pipeline can maintain a separate summary collection.

This is essentially a simple **materialized-view style** architecture.

---

# 20. Important warning: `$out` / `$merge` are write stages

Don't forget:

```text
$match
$group
$project
```

are transforming/reading pipeline stages.

But:

```text
$out
$merge
```

cause persistent changes.

So pipelines containing them deserve much more careful review.

---

# 21. Advanced aggregation: string expressions

Now let's expand Lesson 6.

Suppose:

```js
{
  name: "  Alex Morgan  "
}
```

You may want:

```text
"alex morgan"
```

MongoDB provides expressions such as:

```text
$toLower
$toUpper
$trim
```

For example:

```js
{
  $set: {
    normalizedName: {
      $toLower: {
        $trim: {
          input: "$name"
        }
      }
    }
  }
}
```

Result:

```text
"alex morgan"
```

---

# 22. `$toLower`

### Meaning

> Convert a string to lowercase.

```js
{
  $toLower: "$name"
}
```

Example:

```text
"Alex Morgan"
→
"alex morgan"
```

Useful for:

```text
normalization
case-insensitive comparison strategies
reporting
search preprocessing
```

---

# 23. `$toUpper`

Opposite:

```js
{
  $toUpper: "$country"
}
```

Example:

```text
"india"
→
"INDIA"
```

Simple.

---

# 24. `$trim`

Removes unwanted characters around a string.

Simple example:

```js
{
  $trim: {
    input: "$name"
  }
}
```

Conceptually:

```text
"   Alex   "
→
"Alex"
```

Useful when data has inconsistent whitespace.

---

# 25. `$split`

Suppose:

```text
skills = "Java, MongoDB, Spring Boot"
```

You can turn it into:

```js
[
  "Java",
  " MongoDB",
  " Spring Boot"
]
```

using:

```js
{
  $split: [
    "$skills",
    ","
  ]
}
```

### Meaning

> Split a string into an array using a delimiter.

---

# 26. `$regexMatch`

Suppose you want:

> Does this email look like it belongs to a particular domain?

You can use:

```js
{
  $regexMatch: {
    input: "$email",
    regex: "@example\\.com$"
  }
}
```

It returns a Boolean-like result.

### Meaning

> Test whether a string matches a regular expression.

Important:

`$regexMatch` is an **aggregation expression**.

That's different from using `$regex` as a normal query operator.

---

# 27. String query vs string aggregation

You now have:

```text
$regex
```

for querying documents, versus:

```text
$regexMatch
```

for computing a Boolean inside an aggregation expression.

Again:

```text
query operator
≠
aggregation expression
```

---

# 28. Date expressions

Dates are one of the most important real-world aggregation topics.

Suppose:

```js
{
  createdAt: ISODate("2026-09-20T10:30:00Z")
}
```

You might want:

```text
year
month
day
```

MongoDB provides date expressions for this.

---

# 29. `$dateToString`

Suppose you want:

```text
2026-09
```

from a date.

Conceptually:

```js
{
  $dateToString: {
    format: "%Y-%m",
    date: "$createdAt"
  }
}
```

Result:

```text
"2026-09"
```

Useful for grouping:

```text
applications by month
jobs by month
signups by month
```

---

# 30. `$dateDiff`

Suppose an application was submitted at:

```text
Monday
```

and interviewed at:

```text
Thursday
```

You want:

```text
3 days
```

Use:

```js
{
  $dateDiff: {
    startDate: "$appliedAt",
    endDate: "$interviewAt",
    unit: "day"
  }
}
```

### Meaning

> Calculate the difference between two dates in a specified unit.

Possible units include things like:

```text
year
month
day
hour
minute
second
```

depending on the operation.

---

# 31. `$dateAdd`

Suppose:

```text
application submitted
```

and you want:

> 7 days later.

Conceptually:

```js
{
  $dateAdd: {
    startDate: "$appliedAt",
    unit: "day",
    amount: 7
  }
}
```

### Meaning

> Add a specified amount of time to a date.

---

# 32. `$dateFromString`

Suppose legacy data stores:

```text
"2026-09-20"
```

as a string.

You can convert it to a proper MongoDB date in an aggregation using `$dateFromString`.

This is useful when cleaning/migrating legacy data.

The bigger lesson is:

> **Proper date types are more useful than date strings for database operations.**

---

# 33. `$let`

Now something more advanced.

Suppose an expression repeats a complicated calculation.

Instead of repeating it:

```js
{
  $let: {
    vars: {
      total: {
        $add: [
          "$salary.min",
          "$salary.max"
        ]
      }
    },
    in: {
      $divide: [
        "$$total",
        2
      ]
    }
  }
}
```

### What is `$let`?

> Define local variables inside an aggregation expression.

The syntax:

```text
vars
 ↓
define local values

in
 ↓
use those values
```

Note the `$$`:

```js
"$$total"
```

means:

> Refer to the local variable named `total`.

---

# 34. `$let` mental model

Think JavaScript:

```js
const total = min + max;
return total / 2;
```

MongoDB:

```text
$let
 ├── vars
 └── in
```

It's basically local variable binding inside aggregation expressions.

---

# 35. `$range`

Suppose you need:

```text
0,1,2,3,4
```

MongoDB can generate it.

```js
{
  $range: [
    0,
    5
  ]
}
```

Result:

```js
[0, 1, 2, 3, 4]
```

### Meaning

> Generate a sequence of integers.

---

# 36. `$range` is useful when generating data structures

For example:

```text
days 0–6
months 1–12
positions
indexes
```

It becomes especially powerful when combined with:

```text
$map
$reduce
$zip
```

and other array expressions.

---

# 37. `$mergeObjects`

Suppose:

```js
{
  personal: {
    name: "Alex",
    city: "Pune"
  },

  professional: {
    role: "Backend Engineer",
    years: 2
  }
}
```

You want:

```js
{
  name: "Alex",
  city: "Pune",
  role: "Backend Engineer",
  years: 2
}
```

You can use:

```js
{
  $mergeObjects: [
    "$personal",
    "$professional"
  ]
}
```

### Meaning

> Combine multiple objects into one object.

---

# 38. `$mergeObjects` is useful with `$lookup`

This pattern is especially important.

Suppose `$lookup` produces:

```js
{
  company: [
    {
      name: "Example",
      city: "Pune"
    }
  ]
}
```

You may want to merge the company details into your current document.

With:

```text
$unwind
+
$mergeObjects
+
$replaceRoot
```

you can transform:

```text
Job
{
  title,
  companyId,
  company
}
```

into something flatter.

This is advanced aggregation restructuring.

---

# 39. `$objectToArray`

Suppose:

```js
{
  scores: {
    java: 90,
    mongodb: 95,
    react: 85
  }
}
```

You want the dynamic keys as an array.

Use:

```js
{
  $objectToArray: "$scores"
}
```

Result conceptually:

```js
[
  { k: "java", v: 90 },
  { k: "mongodb", v: 95 },
  { k: "react", v: 85 }
]
```

### Meaning

> Convert an object into an array of key/value pairs.

---

# 40. Why would that be useful?

Because aggregation operators work very naturally with arrays.

After:

```text
object
 ↓
objectToArray
 ↓
array
```

you can then:

```text
$filter
$map
$sort
$unwind
```

the values.

This is especially useful with flexible/dynamic schemas.

---

# 41. `$arrayToObject`

The reverse:

```text
array of key/value pairs
 ↓
object
```

Example conceptually:

```js
[
  { k: "java", v: 90 },
  { k: "mongodb", v: 95 }
]
```

becomes:

```js
{
  java: 90,
  mongodb: 95
}
```

using:

```text
$arrayToObject
```

---

# 42. Dynamic-schema example

Suppose your document contains:

```js
{
  metrics: {
    applications: 500,
    interviews: 120,
    hires: 20
  }
}
```

You could turn it into:

```text
[
  { k: "applications", v: 500 },
  { k: "interviews", v: 120 },
  { k: "hires", v: 20 }
]
```

then process dynamically.

This is especially useful when you **don't know the field names in advance**.

---

# 43. `$setWindowFields`

This is one of the most important additions.

It allows analytics over a **window of related documents** without collapsing them into one group.

Suppose application events are:

```text
Jan → 10
Feb → 15
Mar → 20
Apr → 25
```

You might want:

```text
Jan → 10
Feb → 25
Mar → 45
Apr → 70
```

That's a **running total**.

Window functions are designed for this kind of problem.

---

# 44. `$group` vs `$setWindowFields`

This distinction is important.

### `$group`

Collapses documents.

```text
100 applications
 ↓
1 company summary
```

### `$setWindowFields`

Keeps individual documents but calculates information **over neighboring/related documents**.

```text
Application 1 → calculated value
Application 2 → calculated value
Application 3 → calculated value
```

So:

```text
$group
=
collapse

$setWindowFields
=
calculate over a window while retaining rows/documents
```

This is conceptually similar to SQL window functions.

---

# 45. Running total example

Conceptually:

```js
{
  $setWindowFields: {
    partitionBy: "$companyId",

    sortBy: {
      createdAt: 1
    },

    output: {
      runningApplications: {
        $sum: 1,

        window: {
          documents: [
            "unbounded",
            "current"
          ]
        }
      }
    }
  }
}
```

The result could look like:

```text
Application 1 → 1
Application 2 → 2
Application 3 → 3
Application 4 → 4
```

Instead of replacing them with one group.

---

# 46. `partitionBy`

You may not know this term.

`partitionBy` means:

> Split documents into independent groups/windows.

For example:

```text
company A
  → its window

company B
  → separate window

company C
  → separate window
```

So:

```js
partitionBy: "$companyId"
```

means:

> Perform the window calculation independently for each company.

---

# 47. `sortBy`

Inside `$setWindowFields`:

```js
sortBy: {
  createdAt: 1
}
```

defines the order in which the window calculation proceeds.

For a running total:

```text
oldest
→ newer
→ newer
→ newest
```

---

# 48. Why window functions are useful

They are useful for:

```text
running totals
rankings
moving averages
previous/next values
differences between consecutive records
percentiles
time-based analytics
```

This is one of the most important advanced aggregation features that was missing.

---

# 49. Example: previous application

A window function can help answer:

> “What was the previous event's timestamp?”

Then:

```text
current event
-
previous event
=
time between events
```

That can help calculate:

```text
candidate processing time
recruiter response time
workflow delays
```

This is a powerful analytics pattern.

---

# 50. Advanced aggregation mental map

You now have:

```text
FILTER
$match

SHAPE
$project
$set
$replaceRoot
$replaceWith

GROUP
$group
$count
$sortByCount
$bucket
$bucketAuto

ARRAYS
$unwind
$map
$filter
$reduce
$range

OBJECTS
$mergeObjects
$objectToArray
$arrayToObject

STRINGS
$toLower
$toUpper
$trim
$split
$regexMatch

DATES
$dateToString
$dateFromString
$dateDiff
$dateAdd

WINDOW ANALYTICS
$setWindowFields

OUTPUT
$out
$merge

SAMPLING
$sample
```

That's now a much stronger aggregation toolkit.

---

# PART B — Advanced Indexing

Now we move to Lesson 7.

---

# 51. TTL indexes

You already saw TTL briefly in Lesson 15.

Now let's learn it as an index concept.

TTL stands for:

> **Time To Live**

A TTL index lets MongoDB automatically remove documents after a time threshold.

Example:

```js
db.sessions.createIndex(
  {
    expiresAt: 1
  },
  {
    expireAfterSeconds: 0
  }
)
```

Suppose:

```js
{
  userId: 123,
  expiresAt: ISODate("2026-09-20T18:00:00Z")
}
```

Once the expiry condition is reached, MongoDB's TTL process can remove the document.

---

# 52. What is `expireAfterSeconds: 0`?

This is an important pattern.

It means:

> Treat the date stored in the indexed field as the expiration time.

So:

```text
expiresAt
=
actual expiration moment
```

Another style is:

```text
createdAt
+
expireAfterSeconds
```

meaning:

> Expire the document some number of seconds after its date.

---

# 53. TTL use cases

Excellent candidates:

```text
password reset tokens
email verification tokens
temporary sessions
temporary logs
ephemeral cache records
temporary processing state
```

Not good candidates:

```text
important application records
financial history
permanent user profiles
audit records that must be retained
```

unless your retention requirements explicitly permit it.

---

# 54. TTL is not a scheduler

Very important.

Suppose:

```text
expiresAt = 10:00:00
```

Don't think:

```text
10:00:00 exactly
→ instant deletion
```

TTL cleanup is performed by MongoDB's background mechanism, so deletion is not a precise real-time timer.

Therefore TTL should not be used for:

> “At exactly 10:00:00, execute business logic.”

It is a **data-expiration mechanism**.

---

# 55. Wildcard indexes

MongoDB sometimes deals with documents containing unpredictable fields.

Example:

```js
{
  name: "Alex",
  metadata: {
    language: "English",
    preferredEditor: "VS Code",
    favoriteTheme: "Dark"
  }
}
```

Another document:

```js
{
  name: "Maria",
  metadata: {
    timezone: "IST",
    device: "Laptop",
    framework: "Spring"
  }
}
```

If users can create arbitrary fields under `metadata`, indexing every possible field isn't practical.

A wildcard index can help with dynamic field structures.

Conceptually:

```js
db.users.createIndex({
  "metadata.$**": 1
})
```

### Simple meaning

> Index dynamically occurring fields under that path.

---

# 56. Why wildcard indexes shouldn't be your default

Because:

```text
flexibility
```

has costs.

A wildcard index may be less precise than a carefully designed targeted index.

So:

```text
known stable query
→ regular index

dynamic/unpredictable field structure
→ wildcard may be useful
```

Again:

> Flexible schema does not mean “index everything dynamically.”

---

# 57. Text indexes

MongoDB has a traditional text-search capability using text indexes.

Conceptually:

```js
db.jobs.createIndex({
  title: "text",
  description: "text"
})
```

Then:

```js
db.jobs.find({
  $text: {
    $search: "spring boot backend"
  }
})
```

### What does `$text` mean?

> Perform text search over fields covered by a text index.

This is different from:

```js
{ title: "Backend Engineer" }
```

which is ordinary equality matching.

---

# 58. Text search vs normal index

A normal index is great for queries like:

```text
companyId = X
status = OPEN
salary >= Y
createdAt sorted
```

Text search is designed for:

```text
“backend spring boot developer”
```

where you're searching words/terms within text.

So:

```text
B-tree-style query index
=
structured fields

text index
=
word-based text search
```

---

# 59. `$text` isn't the same as `$regex`

Suppose:

```js
{
  title: {
    $regex: "backend",
    $options: "i"
  }
}
```

This is pattern matching.

While:

```js
{
  $text: {
    $search: "backend engineer"
  }
}
```

is MongoDB's text-search mechanism.

They're different tools.

---

# 60. Search architecture decision

At an application-design level, think:

```text
exact structured filtering
→ normal indexes

basic database text search
→ text index / $text

advanced relevance/search experience
→ dedicated search capability
```

For systems that use Atlas, **Atlas Search** is worth learning separately as a production search architecture.

Don't force ordinary indexes to solve every search problem.

---

# 61. `$hint`

MongoDB has a query planner that normally chooses indexes automatically.

But sometimes you want to tell it:

> **Use this index for this operation.**

That's what a hint is for.

Conceptually:

```js
db.jobs.find({
  companyId: 123,
  status: "OPEN"
})
.hint({
  companyId: 1,
  status: 1
})
```

The exact syntax differs by shell/driver context, but the concept is:

```text
hint()
=
explicitly tell MongoDB which index to use
```

---

# 62. Why would you ever use `$hint`?

Because you're diagnosing or controlling a specific workload.

For example:

```text
Index A
Index B
```

and you want to compare execution behavior.

You can test different candidate indexes intentionally.

But:

> **Don't blindly hard-code hints into normal application queries.**

A future data distribution or index change may make the forced hint worse.

---

# 63. Automatic planner vs manual hint

Think:

```text
normal
→ MongoDB chooses

hint
→ developer overrides that choice
```

So hinting is a powerful tool, but also dangerous if used carelessly.

---

# 64. `explain("allPlansExecution")`

You've already used:

```text
explain("executionStats")
```

which tells you about the winning execution plan.

Now:

```text
explain("allPlansExecution")
```

gives you more information about the candidate plans considered during planning/execution.

The interesting question becomes:

> **Why did MongoDB choose this plan?**

rather than merely:

> What did the winning plan do?

---

# 65. Why rejected plans matter

Suppose MongoDB considered:

```text
Plan A → index on companyId
Plan B → index on status
Plan C → compound index
```

and selected:

```text
Plan C
```

Investigating candidate-plan behavior can help when a query behaves unexpectedly.

This is especially useful in serious query-performance debugging.

---

# 66. `executionStats` vs `allPlansExecution`

Mental model:

```text
executionStats
=
focus on the winning plan's execution

allPlansExecution
=
show more detail about competing candidate plans
```

This belongs in your advanced performance toolbox.

---

# 67. `hideIndex()`

Now an excellent production feature.

Suppose you have:

```text
Index A
Index B
Index C
```

and you're wondering:

> Is Index C actually necessary?

Instead of immediately doing:

```text
dropIndex
```

you can hide it.

Conceptually:

```js
db.jobs.hideIndex("index_name")
```

### Meaning

> Temporarily make the index unavailable to the query planner without immediately deleting it.

This lets you test:

```text
How does the workload behave without this index?
```

---

# 68. Why hiding is safer than immediate deletion

Suppose you delete:

```text
Index C
```

and production performance gets worse.

Now you have to recreate it.

With hiding:

```text
Index C
   ↓
hidden
   ↓
observe workload
```

If performance becomes bad:

```text
unhide
```

and restore its availability to the planner.

This is a beautiful example of safe production experimentation.

---

# 69. `unhideIndex()`

The opposite:

> Make the hidden index visible to the query planner again.

Conceptually:

```js
db.jobs.unhideIndex("index_name")
```

So:

```text
hideIndex()
=
test without planner using it

unhideIndex()
=
restore normal availability
```

---

# 70. Index deletion vs index hiding

Think:

```text
drop
=
destroy

hide
=
test
```

In production, testing before destructive removal can be extremely valuable.

---

# 71. `dropIndexes()`

You already know:

```text
dropIndex()
```

for one index.

There is also:

```text
dropIndexes()
```

which removes indexes from a collection according to its operation semantics.

The important lesson isn't the syntax.

It's:

> **Index lifecycle is part of database operations.**

Indexes are infrastructure objects that can become outdated as workloads evolve.

---

# 72. Index intersection

Now something more conceptual.

Suppose you have:

```text
Index A:
companyId

Index B:
status
```

and your query is:

```js
{
  companyId: 123,
  status: "OPEN"
}
```

MongoDB may in some circumstances combine indexes to answer a query.

This is called:

> **Index intersection**

Conceptually:

```text
Index A
   ↓
matching company IDs

Index B
   ↓
matching OPEN jobs

       ↓
combine
       ↓
result
```

---

# 73. Why index intersection doesn't mean “don't create compound indexes”

This is important.

Suppose your workload consistently uses:

```text
companyId
+
status
+
createdAt sort
```

A well-designed compound index can be much more suitable than hoping MongoDB intersects separate indexes.

So:

```text
intersection
=
possible fallback/optimization

compound index
=
deliberate workload design
```

Don't use intersection as an excuse for poor index design.

---

# 74. A common beginner mistake

Developer says:

> “I have an index on companyId and an index on status, so I don't need a compound index.”

Not necessarily.

Suppose your actual query is:

```js
Job.find({
  companyId,
  status
})
.sort({
  createdAt: -1
})
.limit(20)
```

A compound index aligned to the workload may be much more appropriate.

Again:

> **Indexes should follow query patterns.**

---

# 75. TTL + application design

Let's connect TTL to our earlier JobForge examples.

Suppose you create:

```text
PasswordResetToken
```

document:

```js
{
  userId,
  tokenHash,
  expiresAt
}
```

TTL index:

```text
expiresAt
```

Now MongoDB automatically removes expired tokens eventually.

But authentication logic still needs to verify:

```text
token hasn't expired
```

at the time it is used.

Why?

Because TTL deletion isn't a real-time authorization mechanism.

That's an important security distinction.

---

# 76. Wildcard + schema design

Suppose a product has arbitrary technical properties:

```js
{
  name: "Laptop",
  attributes: {
    ram: "32GB",
    battery: "80Wh",
    screenSize: "15.6",
    ports: 4
  }
}
```

Another product:

```js
{
  name: "Camera",
  attributes: {
    sensor: "APS-C",
    lensMount: "X",
    megapixels: 26
  }
}
```

A wildcard index can be useful if queries across arbitrary `attributes.*` fields are a major workload.

But if your application really only searches:

```text
ram
screenSize
price
```

then targeted indexes are more predictable.

Again:

```text
dynamic workload
→ wildcard candidate

known workload
→ targeted index
```

---

# 77. Search + index design

Suppose JobForge supports:

```text
“Spring Boot Developer”
```

plus:

```text
location = Pune
status = OPEN
remote = true
```

This isn't one single “index problem.”

You potentially have:

```text
text search
+
structured filtering
+
sorting
```

This should be treated as a search architecture problem.

That is why the course needed at least an introduction to text search.

---

# 78. Advanced aggregation + indexes

Here's a subtle point.

Suppose:

```js
db.jobs.aggregate([
  {
    $match: {
      companyId: 123
    }
  },
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  }
])
```

Your `$match` comes first.

A suitable index on:

```text
companyId
```

can potentially reduce the amount of data MongoDB has to process before the grouping stage.

So:

```text
indexes
```

and:

```text
aggregation
```

are not separate worlds.

They work together.

---

# 79. Advanced aggregation + `$sort`

Suppose:

```js
[
  {
    $match: {
      companyId: 123
    }
  },
  {
    $sort: {
      createdAt: -1
    }
  },
  {
    $limit: 20
  }
]
```

A suitable index can potentially help MongoDB efficiently execute the filter+sort combination.

This is the same principle you've already learned:

> **Pipeline order affects performance.**

And now:

> **Indexes can affect how efficiently the pipeline runs.**

---

# 80. `$sortByCount` + large datasets

Suppose:

```text
1 billion applications
```

and you use:

```js
{
  $sortByCount: "$status"
}
```

This is logically correct.

But correctness doesn't guarantee cheap execution.

You still need to think:

```text
how many input documents?
memory?
disk spill?
execution time?
```

Advanced MongoDB means understanding that:

> **Every aggregation operation has a computational cost.**

---

# 81. `$bucket` + dashboard

Suppose recruiter dashboard wants:

```text
0–2 years: 120 candidates
3–5 years: 300 candidates
6–10 years: 80 candidates
10+ years: 30 candidates
```

This is exactly where:

```text
$bucket
```

shines.

You don't need to manually calculate ranges in Node.

Let the database perform the data grouping.

---

# 82. `$setWindowFields` + recruiter analytics

Suppose recruiter dashboard wants:

> Candidate applications received over time, plus running total.

Pipeline idea:

```text
Applications
 ↓
group by date
 ↓
sort chronologically
 ↓
window calculation
 ↓
running total
```

This is much more sophisticated analytics than:

```text
find()
→ loop in JavaScript
```

That's why window functions belong in advanced aggregation.

---

# 83. `$merge` + Computed Pattern

Now connect two earlier lessons.

Lesson 3:

```text
Computed Pattern
```

Lesson 5:

```text
$merge
```

Together:

```text
Raw Applications
       ↓
Aggregation
       ↓
daily company metrics
       ↓
$merge
       ↓
CompanyDailyMetrics
```

Now you've built a basic **materialized analytics pipeline**.

This is a real production architecture pattern.

---

# 84. `$out` + migration

Similarly:

```text
old collection
 ↓
aggregation/transformation
 ↓
$out
 ↓
new collection
```

can be useful for certain migration/transformation workflows.

You could conceptually:

```text
users_v1
 ↓ transform
users_v2
```

then validate the new data before switching application reads.

This should be carefully planned for production because `$out` changes persistent data.

---

# 85. `allPlansExecution` + hidden indexes

These two advanced features even work together conceptually.

Imagine you're unsure whether Index A is still valuable.

You can:

```text
1. inspect query plan
2. hide Index A
3. test workload
4. inspect performance
5. unhide or remove
```

That's far better than:

```text
1. delete index
2. production breaks
3. panic
```

This is what **production database engineering** looks like.

---

# 86. Advanced indexing mental map

You now have:

```text
NORMAL
single
compound
multikey
unique
partial

SPECIAL
TTL
wildcard
text

QUERY CONTROL
hint
explain
allPlansExecution

INDEX LIFECYCLE
hide
unhide
drop

PLANNER
index intersection
```

That is a much more complete index toolbox.

---

# 87. New methods/operators from this gap module

Here is your reference list.

### Aggregation stages

```text
$count
```

> Count documents currently flowing through the pipeline.

```text
$replaceRoot
$replaceWith
```

> Replace the current document with another document/expression.

```text
$sortByCount
```

> Group by an expression and sort by count.

```text
$sample
```

> Randomly sample documents.

```text
$bucket
```

> Group values into explicitly defined ranges.

```text
$bucketAuto
```

> Automatically create approximate data ranges.

```text
$out
```

> Write aggregation results to a collection.

```text
$merge
```

> Merge aggregation results into a target collection.

---

### Aggregation expressions

```text
$toLower
```

> Convert text to lowercase.

```text
$toUpper
```

> Convert text to uppercase.

```text
$trim
```

> Remove surrounding unwanted whitespace/characters.

```text
$split
```

> Split a string into an array.

```text
$regexMatch
```

> Test whether a string matches a regex.

```text
$dateToString
```

> Format a date as text.

```text
$dateFromString
```

> Convert a string into a date.

```text
$dateDiff
```

> Calculate the difference between dates.

```text
$dateAdd
```

> Add time to a date.

```text
$let
```

> Define local variables inside an expression.

```text
$range
```

> Generate an integer sequence.

```text
$mergeObjects
```

> Combine objects.

```text
$objectToArray
```

> Convert object fields into key/value array entries.

```text
$arrayToObject
```

> Convert key/value entries back into an object.

```text
$setWindowFields
```

> Perform calculations over related ordered documents while retaining them.

---

### Index concepts

```text
TTL
```

> Automatically expire documents.

```text
wildcard index
```

> Index dynamic/unpredictable fields.

```text
text index
```

> Support MongoDB's basic text-search capability.

```text
$text
```

> Search indexed text.

```text
$hint / hint()
```

> Tell MongoDB which index to use.

```text
explain("allPlansExecution")
```

> Inspect candidate query plans in greater detail.

```text
hideIndex()
```

> Hide an index from normal planner consideration without immediately deleting it.

```text
unhideIndex()
```

> Make a hidden index available again.

```text
index intersection
```

> Combine multiple indexes for some query plans.

---

# 88. One important JavaScript distinction

Some of today's things are **not JavaScript functions**.

For example:

```text
$bucket
$mergeObjects
$setWindowFields
$count
```

are MongoDB aggregation operators.

Whereas:

```text
Math.min()
Number.isInteger()
Array.map()
```

are JavaScript APIs.

And:

```text
.populate()
.lean()
.exec()
```

are Mongoose APIs.

Keep this three-layer model:

```text
JavaScript
    ↓
Mongoose / Driver
    ↓
MongoDB
```

That distinction will become increasingly important when we get into the Node driver.

---

# 89. The three biggest conceptual additions today

Forget the syntax for a moment.

There are three major ideas.

## 1. Aggregation can reshape data dramatically

You can now:

```text
filter
group
bucket
sample
reshape
calculate windows
write results
```

So MongoDB can act as a substantial **data-processing engine**, not merely a document lookup system.

---

## 2. Indexes are operational infrastructure

They're not just:

```text
“things that make queries faster.”
```

They're also:

```text
planner inputs
production tuning tools
lifecycle-managed infrastructure
search structures
expiration mechanisms
```

---

## 3. Performance engineering is experimental

You now have a workflow:

```text
explain
 ↓
understand plans
 ↓
hide index / hint / test
 ↓
measure
 ↓
keep or revert
```

That is far better than guessing.

---

# 90. Where these concepts belong in the original course

Your lesson map is now:

```text
Lesson 5
Aggregation Fundamentals
   +
$count
$replaceRoot
$replaceWith
$sortByCount
$sample
$bucket
$bucketAuto
$out
$merge

Lesson 6
Advanced Aggregation
   +
string operators
date operators
$let
$range
$mergeObjects
$objectToArray
$arrayToObject
$setWindowFields

Lesson 7
Indexes & Query Performance
   +
TTL
wildcard
text indexes
$text
$hint
allPlansExecution
hideIndex
unhideIndex
index intersection
```

That's exactly where these concepts should live.

---

# 91. Current gap status

We're now at:

```text
Lessons 1–4 gaps
✓ complete

Lessons 5–7 gaps
✓ complete
```

So the remaining high-value gaps are:

```text
Lesson 8
Advanced JSON Schema validation

Lesson 9
Small transaction additions

Lessons 10–12
Operational replica-set + sharding commands

Lesson 13
Change-stream edge cases

Lesson 14
Advanced Mongoose + MongoDB Node Driver
```

After those, we'll have essentially finished the **entire gap-fill curriculum**.

And there is one especially important module among those remaining: **Lesson 14 + the Node.js MongoDB Driver**, because that will tie together everything you've learned at the actual application/driver level rather than just through Mongoose.