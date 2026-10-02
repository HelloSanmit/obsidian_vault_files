
# MongoDB — Lesson 6: Aggregation Expressions & Advanced Pipelines

Lesson 5 taught you the **main aggregation stages**.

Now we're going one level deeper.

The important distinction is:

```text
Aggregation stage
    ↓
controls the pipeline

Aggregation expression
    ↓
calculates/constructs a value inside a stage
```

For example:

```javascript
{
    $project: {
        finalPrice: {
            $multiply: ["$price", 0.9]
        }
    }
}
```

Here:

```text
$project      → stage
$multiply     → expression
"$price"      → field-path expression
```

MongoDB expressions evaluate to values and can be used inside stages such as `$project`, `$set`, and `$group`. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/?utm_source=chatgpt.com "Aggregation Pipeline - Database Manual - MongoDB Docs"))

---

# 1. `$cond` — if/else inside MongoDB

Suppose products have:

```javascript
{
    name: "Keyboard",
    price: 2500
}
```

You want:

```text
price >= 3000 → "Expensive"
otherwise     → "Affordable"
```

Use:

```javascript
db.products.aggregate([
    {
        $project: {
            name: 1,
            price: 1,

            category: {
                $cond: {
                    if: { $gte: ["$price", 3000] },
                    then: "Expensive",
                    else: "Affordable"
                }
            }
        }
    }
])
```

For `2500`:

```javascript
{
    name: "Keyboard",
    price: 2500,
    category: "Affordable"
}
```

The important mental model:

```text
$cond
  ↓
if condition
   ├── true  → then
   └── false → else
```

---

# 2. `$cond` can use the shorter syntax

You can also write:

```javascript
{
    $cond: [
        { $gte: ["$price", 3000] },
        "Expensive",
        "Affordable"
    ]
}
```

Same idea.

I recommend learning the object form first because it is easier to read when expressions become complicated.

---

# 3. `$ifNull`

Real databases often contain missing or `null` values.

Suppose:

```javascript
{
    name: "Ishita",
    phone: null
}
```

You want:

```text
phone → "Not provided"
```

Use:

```javascript
db.users.aggregate([
    {
        $project: {
            name: 1,

            phone: {
                $ifNull: [
                    "$phone",
                    "Not provided"
                ]
            }
        }
    }
])
```

Result:

```javascript
{
    name: "Ishita",
    phone: "Not provided"
}
```

MongoDB's `$ifNull` provides a fallback when the input expression evaluates to `null` or refers to a missing field. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/expressions/?utm_source=chatgpt.com "Expressions - Database Manual - MongoDB Docs"))

This becomes extremely useful for API responses.

---

# 4. Combining expressions

Expressions can be nested.

Suppose:

```javascript
{
    price: 5000,
    discount: 20
}
```

You want:

```text
finalPrice = price - (price × discount / 100)
```

You can write:

```javascript
{
    $project: {
        finalPrice: {
            $subtract: [
                "$price",
                {
                    $multiply: [
                        "$price",
                        {
                            $divide: [
                                "$discount",
                                100
                            ]
                        }
                    ]
                }
            ]
        }
    }
}
```

MongoDB expressions can be composed recursively like this. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/?utm_source=chatgpt.com "Aggregation Pipeline - Database Manual - MongoDB Docs"))

This is basically:

```text
expression
   ↓
contains another expression
   ↓
contains another expression
```

---

# 5. `$and`, `$or`, `$not` inside expressions

You've already learned:

```text
$and
$or
$not
```

as query operators.

Aggregation expressions have logical operators too.

For example:

```javascript
{
    $and: [
        { $gte: ["$score", 70] },
        { $eq: ["$status", "active"] }
    ]
}
```

This means:

```text
score >= 70
AND
status == active
```

A common use is inside `$cond`:

```javascript
{
    $cond: {
        if: {
            $and: [
                { $gte: ["$score", 70] },
                { $eq: ["$status", "active"] }
            ]
        },
        then: "Eligible",
        else: "Not Eligible"
    }
}
```

---

# 6. `$switch`

Imagine you have:

```text
score >= 90 → A
score >= 80 → B
score >= 70 → C
otherwise   → F
```

Using nested `$cond` would become ugly.

MongoDB provides `$switch`.

```javascript
db.students.aggregate([
    {
        $project: {
            name: 1,

            grade: {
                $switch: {
                    branches: [
                        {
                            case: { $gte: ["$score", 90] },
                            then: "A"
                        },
                        {
                            case: { $gte: ["$score", 80] },
                            then: "B"
                        },
                        {
                            case: { $gte: ["$score", 70] },
                            then: "C"
                        }
                    ],

                    default: "F"
                }
            }
        }
    }
])
```

Mental model:

```text
$switch
    ↓
case 1 → result
case 2 → result
case 3 → result
default
```

Very useful for classification.

---

# 7. `$map` — transform every array element

Now we get into the more powerful array expressions.

Suppose:

```javascript
{
    name: "Arjun",

    scores: [70, 80, 90]
}
```

You want to add 5 to every score.

You could unwind and regroup, but that's unnecessary.

Use `$map`:

```javascript
db.students.aggregate([
    {
        $project: {
            name: 1,

            updatedScores: {
                $map: {
                    input: "$scores",
                    as: "score",
                    in: {
                        $add: ["$$score", 5]
                    }
                }
            }
        }
    }
])
```

Result:

```javascript
{
    name: "Arjun",
    updatedScores: [75, 85, 95]
}
```

`$map` applies an expression to each array element and returns the transformed array. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/map/?msockid=15ecb796ec436d943b1fa17ced9d6ceb&utm_source=chatgpt.com "$map (expression operator) - Database Manual - MongoDB Docs"))

---

# 8. What is `$$score`?

This is important.

Earlier:

```javascript
"$price"
```

meant:

> field called `price` from the current document.

But:

```javascript
"$$score"
```

means:

> variable called `score`.

We defined it here:

```javascript
as: "score"
```

So:

```text
input → [70,80,90]

as: "score"

iteration 1:
$$score = 70

iteration 2:
$$score = 80

iteration 3:
$$score = 90
```

MongoDB aggregation variables are referenced using the `$$variableName` syntax. ([MongoDB](https://www.mongodb.com/docs/manual/reference/aggregation-variables/?utm_source=chatgpt.com "Variables in Aggregation Expressions - Database Manual - MongoDB Docs"))

---

# 9. `$map` with objects

This gets more realistic.

Suppose:

```javascript
{
    name: "Arjun",

    products: [
        {
            name: "Keyboard",
            price: 2000
        },
        {
            name: "Mouse",
            price: 1000
        }
    ]
}
```

You want to calculate a 10% discounted price for every product.

```javascript
db.orders.aggregate([
    {
        $project: {
            discountedProducts: {
                $map: {
                    input: "$products",
                    as: "product",
                    in: {
                        name: "$$product.name",

                        discountedPrice: {
                            $multiply: [
                                "$$product.price",
                                0.9
                            ]
                        }
                    }
                }
            }
        }
    }
])
```

Result:

```javascript
{
    discountedProducts: [
        {
            name: "Keyboard",
            discountedPrice: 1800
        },
        {
            name: "Mouse",
            discountedPrice: 900
        }
    ]
}
```

This is much more useful than merely knowing that `$map` exists.

---

# 10. `$filter` — keep selected array elements

`$map` means:

> Transform every element.

`$filter` means:

> Keep only elements satisfying a condition.

Suppose:

```javascript
{
    scores: [45, 80, 65, 90]
}
```

You want scores >= 70:

```javascript
db.students.aggregate([
    {
        $project: {
            passingScores: {
                $filter: {
                    input: "$scores",
                    as: "score",
                    cond: {
                        $gte: ["$$score", 70]
                    }
                }
            }
        }
    }
])
```

Result:

```javascript
{
    passingScores: [80, 90]
}
```

`$filter` returns a subset of an array containing the elements whose condition evaluates to true, preserving their original order. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/filter/?utm_source=chatgpt.com "$filter (expression operator) - Database Manual - MongoDB Docs"))

---

# 11. `$map` vs `$filter`

Memorize this distinction:

```text
$map
   ↓
change every element

$filter
   ↓
select some elements
```

Example:

```text
[10, 20, 30]

$map    → [20, 40, 60]   (transform)
$filter → [20, 30]       (select)
```

This distinction is extremely important.

---

# 12. `$filter` with objects

Suppose:

```javascript
{
    applications: [
        {
            company: "Alpha",
            status: "rejected"
        },
        {
            company: "Beta",
            status: "interview"
        },
        {
            company: "Gamma",
            status: "rejected"
        }
    ]
}
```

You want only applications where:

```text
status = interview
```

Use:

```javascript
{
    $project: {
        interviewApplications: {
            $filter: {
                input: "$applications",
                as: "application",
                cond: {
                    $eq: [
                        "$$application.status",
                        "interview"
                    ]
                }
            }
        }
    }
}
```

Result:

```javascript
{
    interviewApplications: [
        {
            company: "Beta",
            status: "interview"
        }
    ]
}
```

This is directly relevant to JobForge-style APIs.

---

# 13. `$map` + `$filter`

You can combine them.

Suppose:

```javascript
skills:
[
    { name: "Java", level: 8 },
    { name: "React", level: 5 },
    { name: "MongoDB", level: 9 }
]
```

You want:

> Names of skills with level >= 7.

Pipeline:

```javascript
{
    $project: {
        strongSkills: {
            $map: {
                input: {
                    $filter: {
                        input: "$skills",
                        as: "skill",
                        cond: {
                            $gte: [
                                "$$skill.level",
                                7
                            ]
                        }
                    }
                },
                as: "skill",
                in: "$$skill.name"
            }
        }
    }
}
```

Result:

```javascript
{
    strongSkills: [
        "Java",
        "MongoDB"
    ]
}
```

Read it from the inside outward:

```text
skills
 ↓
$filter → level >= 7
 ↓
$map → take name
 ↓
["Java", "MongoDB"]
```

That “inside-out” reading technique is extremely useful with complex expressions.

---

# 14. `$size`

Want the number of elements in an array?

```javascript
{
    $project: {
        skillCount: {
            $size: "$skills"
        }
    }
}
```

For:

```javascript
skills: ["Java", "React", "MongoDB"]
```

result:

```text
skillCount: 3
```

This is useful for things like:

```text
number of skills
number of applications
number of participants
number of items
```

You do need to think about what happens when the field is missing or not an array; expressions generally need appropriate null/type handling.

---

# 15. `$arrayElemAt`

Want an individual array element?

```javascript
{
    $arrayElemAt: [
        "$skills",
        0
    ]
}
```

returns the first element.

For:

```text
["Java", "React", "MongoDB"]
```

you get:

```text
"Java"
```

You can also use:

```javascript
-1
```

for the last element.

---

# 16. `$concat`

String expressions are useful when building API output.

Suppose:

```javascript
{
    firstName: "Riya",
    lastName: "Sharma"
}
```

Create:

```text
fullName = "Riya Sharma"
```

```javascript
{
    $project: {
        fullName: {
            $concat: [
                "$firstName",
                " ",
                "$lastName"
            ]
        }
    }
}
```

Result:

```javascript
{
    fullName: "Riya Sharma"
}
```

There are many string expressions in MongoDB, including substring, case-conversion, trimming, splitting, and replacement operators.

We don't need all of them yet.

---

# 17. Date expressions

Aggregation becomes especially useful for analytics when dates are involved.

Suppose:

```javascript
{
    createdAt: ISODate("2026-09-20T05:30:00Z")
}
```

You can extract:

```text
year
month
day
hour
```

using date expressions such as:

```javascript
$year
$month
$dayOfMonth
$hour
```

For example:

```javascript
{
    $project: {
        year: {
            $year: "$createdAt"
        },

        month: {
            $month: "$createdAt"
        }
    }
}
```

This becomes useful for:

```text
monthly revenue
daily registrations
applications per month
messages per day
```

---

# 18. Example: registrations by month

Suppose:

```text
users
```

contains:

```javascript
{
    name: "Riya",
    createdAt: ISODate("2026-08-10")
}
```

You can group users by year/month:

```javascript
db.users.aggregate([
    {
        $group: {
            _id: {
                year: { $year: "$createdAt" },
                month: { $month: "$createdAt" }
            },

            registrations: {
                $sum: 1
            }
        }
    },

    {
        $sort: {
            "_id.year": 1,
            "_id.month": 1
        }
    }
])
```

Result could look like:

```text
2026-07 → 340
2026-08 → 510
2026-09 → 620
```

Now we're doing actual analytics.

---

# 19. `$expr`

There is another very important concept.

Earlier, ordinary queries looked like:

```javascript
db.products.find({
    price: { $gt: 1000 }
})
```

But what if you want:

> Find products whose `discountedPrice` is less than their `costPrice`.

That's a comparison **between two fields**.

You can use `$expr`:

```javascript
db.products.find({
    $expr: {
        $lt: [
            "$discountedPrice",
            "$costPrice"
        ]
    }
})
```

`$expr` allows aggregation expressions inside query predicates. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/expressions/?utm_source=chatgpt.com "Expressions - Database Manual - MongoDB Docs"))

This is important because it connects:

```text
query language
        +
aggregation expressions
```

---

# 20. `$lookup` — advanced version

You saw the simple form in Lesson 5:

```javascript
{
    $lookup: {
        from: "users",
        localField: "candidateId",
        foreignField: "_id",
        as: "candidate"
    }
}
```

But `$lookup` can also contain its own pipeline.

This is much more powerful.

Suppose:

```text
applications
candidates
```

You want:

> For each application, fetch the candidate but return only their name and skills.

```javascript
db.applications.aggregate([
    {
        $lookup: {
            from: "candidates",

            localField: "candidateId",
            foreignField: "_id",

            pipeline: [
                {
                    $project: {
                        name: 1,
                        skills: 1
                    }
                }
            ],

            as: "candidate"
        }
    }
])
```

Now the foreign collection itself is processed through a pipeline.

MongoDB supports both equality lookups and `$lookup` pipelines, including correlated subqueries using `let` variables. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/?utm_source=chatgpt.com "$lookup (aggregation stage) - Database Manual - MongoDB Docs"))

---

# 21. `$lookup` with `let`

Now suppose an application contains:

```javascript
{
    candidateId: ObjectId("..."),
    requiredExperience: 3
}
```

Imagine we want to find candidate records satisfying a condition involving the current application.

We can expose fields from the current document:

```javascript
let: {
    requiredExperience: "$requiredExperience"
}
```

and reference it inside the foreign pipeline:

```javascript
"$$requiredExperience"
```

Example structure:

```javascript
{
    $lookup: {
        from: "candidates",

        let: {
            requiredExperience: "$requiredExperience"
        },

        pipeline: [
            {
                $match: {
                    $expr: {
                        $gte: [
                            "$experienceYears",
                            "$$requiredExperience"
                        ]
                    }
                }
            }
        ],

        as: "matchingCandidates"
    }
}
```

The mental model:

```text
current application
       ↓
let
       ↓
$$requiredExperience
       ↓
foreign pipeline
```

MongoDB uses `$$variableName` for aggregation variables, including variables declared by `$lookup`'s `let`. A `$match` inside that foreign pipeline uses `$expr` when referencing those variables. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/?utm_source=chatgpt.com "$lookup (aggregation stage) - Database Manual - MongoDB Docs"))

---

# 22. `$facet` — multiple results from one input

Now let's return to JobForge.

Imagine a search API:

```text
GET /jobs?skills=Java&location=Pune
```

The frontend wants:

```text
matching jobs
total number of results
locations
salary ranges
```

You don't want separate processing pipelines starting from scratch conceptually for every result type.

`$facet` lets multiple sub-pipelines operate on the same incoming set of documents. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

Example:

```javascript
db.jobs.aggregate([
    {
        $match: {
            skills: "Java",
            location: "Pune"
        }
    },

    {
        $facet: {

            jobs: [
                { $sort: { createdAt: -1 } },
                { $limit: 20 }
            ],

            totalCount: [
                { $count: "count" }
            ],

            salaryStats: [
                {
                    $group: {
                        _id: null,
                        averageSalary: {
                            $avg: "$salary"
                        },

                        maximumSalary: {
                            $max: "$salary"
                        }
                    }
                }
            ]
        }
    }
])
```

Conceptually:

```text
                   matching jobs
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
           jobs      totalCount   salaryStats
```

This is extremely relevant to search/filtering systems.

---

# 23. `$facet` mental model

Think:

```text
one input
    ↓
 ┌──┴───────────────┐
 ↓        ↓         ↓
A        B          C
↓        ↓          ↓
result   result     result
```

So instead of one pipeline producing one kind of output, `$facet` creates multiple views of the same input.

---

# 24. A very useful complete JobForge-style pipeline

Imagine jobs look like:

```javascript
{
    title: "Backend Engineer",
    location: "Pune",
    skills: [
        "Java",
        "Spring Boot",
        "MongoDB"
    ],
    salary: 900000,
    status: "open"
}
```

We want:

> Open Java jobs in Pune, sorted by salary, showing the first 10, plus total count and average salary.

```javascript
db.jobs.aggregate([
    {
        $match: {
            status: "open",
            location: "Pune",
            skills: "Java"
        }
    },

    {
        $facet: {
            jobs: [
                {
                    $sort: {
                        salary: -1
                    }
                },

                {
                    $limit: 10
                },

                {
                    $project: {
                        _id: 1,
                        title: 1,
                        salary: 1,
                        skills: 1
                    }
                }
            ],

            total: [
                {
                    $count: "count"
                }
            ],

            salaryStats: [
                {
                    $group: {
                        _id: null,

                        average: {
                            $avg: "$salary"
                        },

                        maximum: {
                            $max: "$salary"
                        }
                    }
                }
            ]
        }
    }
])
```

Now you should be able to mentally parse the whole thing:

```text
$match
 ↓
only relevant jobs

$facet
 ├── jobs
 │    ├── sort
 │    ├── limit
 │    └── project
 │
 ├── total
 │    └── count
 │
 └── salaryStats
      └── group + calculations
```

That's a genuinely useful backend aggregation.

---

# 25. `$reduce` — the advanced array operation

Now for one of the more difficult concepts.

Suppose:

```javascript
{
    prices: [100, 200, 300]
}
```

You want:

```text
600
```

You could use `$sum` in some contexts, but `$reduce` teaches a much more general concept:

> Iterate through an array while maintaining an accumulated value.

MongoDB defines `$reduce` as applying an expression to each array element and combining the elements into a single value. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/reduce/?utm_source=chatgpt.com "$reduce (expression operator) - Database Manual - MongoDB Docs"))

Example:

```javascript
{
    $project: {
        total: {
            $reduce: {
                input: "$prices",

                initialValue: 0,

                in: {
                    $add: [
                        "$$value",
                        "$$this"
                    ]
                }
            }
        }
    }
}
```

Understand these two variables:

```text
$$value → accumulated result so far

$$this  → current array element
```

So:

```text
initialValue = 0

element 100:
value = 0 + 100 = 100

element 200:
value = 100 + 200 = 300

element 300:
value = 300 + 300 = 600
```

Result:

```text
600
```

---

# 26. `$reduce` mental model

In JavaScript you'd mentally write:

```javascript
let total = 0;

for (const price of prices) {
    total += price;
}
```

MongoDB `$reduce` is essentially expressing that sort of computation within the aggregation expression system.

So:

```text
$map
    → transform

$filter
    → select

$reduce
    → accumulate
```

That's a very useful trio.

---

# 27. `$map`, `$filter`, `$reduce`

Memorize this:

```text
ARRAY
│
├── $map
│      ↓
│   transform every item
│
├── $filter
│      ↓
│   keep matching items
│
└── $reduce
       ↓
    combine into one result
```

Example:

```text
[1,2,3,4,5]

$map
→ [2,4,6,8,10]

$filter (>3)
→ [4,5]

$reduce (sum)
→ 15
```

That mental model is more important than syntax.

---

# 28. Do not confuse `$unwind` with `$map`

This is an important distinction.

### `$unwind`

```text
one document
   ↓
many pipeline documents
```

Example:

```javascript
{
    skills: ["Java", "MongoDB"]
}
```

becomes:

```text
document → Java
document → MongoDB
```

### `$map`

```text
one document
   ↓
one document
```

The array remains an array.

```javascript
{
    skills: ["Java", "MongoDB"]
}
```

might become:

```javascript
{
    skills: ["JAVA", "MONGODB"]
}
```

So:

```text
$unwind → affects pipeline document count

$map    → transforms array inside the document
```

Very different.

---

# 29. When should you use `$map` vs `$unwind`?

Suppose you want:

> Convert all skill names to uppercase.

Use `$map`.

```text
skills
 ↓
$map
 ↓
new skills array
```

But suppose you want:

> Count how many candidates have each skill.

Use `$unwind`.

```text
candidate
 ↓
skills
 ↓
$unwind
 ↓
one pipeline document per skill
 ↓
$group
```

So ask yourself:

> **Do I need to transform the array, or do I need to turn its elements into separate pipeline records?**

That's the key distinction.

---

# 30. Exercises

A small set, but these are intentionally more challenging.

## Exercise 1

Given:

```javascript
{
    name: "Riya",
    scores: [40, 75, 90, 55]
}
```

Create a new field:

```text
passingScores
```

containing only scores >= 60.

### Answer

```javascript
db.students.aggregate([
    {
        $project: {
            name: 1,

            passingScores: {
                $filter: {
                    input: "$scores",
                    as: "score",
                    cond: {
                        $gte: ["$$score", 60]
                    }
                }
            }
        }
    }
])
```

---

# 31. Exercise 2

Given:

```javascript
{
    name: "Riya",
    scores: [40, 75, 90, 55]
}
```

Create:

```text
adjustedScores
```

where every score is increased by 5.

### Answer

```javascript
db.students.aggregate([
    {
        $project: {
            name: 1,

            adjustedScores: {
                $map: {
                    input: "$scores",
                    as: "score",
                    in: {
                        $add: ["$$score", 5]
                    }
                }
            }
        }
    }
])
```

---

# 32. Exercise 3

Given:

```javascript
{
    name: "Riya",
    scores: [40, 75, 90, 55]
}
```

Calculate the total of all scores using `$reduce`.

### Answer

```javascript
db.students.aggregate([
    {
        $project: {
            name: 1,

            totalScore: {
                $reduce: {
                    input: "$scores",
                    initialValue: 0,

                    in: {
                        $add: [
                            "$$value",
                            "$$this"
                        ]
                    }
                }
            }
        }
    }
])
```

Result:

```text
260
```

---

# 33. Exercise 4 — harder

Given:

```javascript
{
    skills: [
        { name: "Java", level: 8 },
        { name: "React", level: 5 },
        { name: "MongoDB", level: 9 },
        { name: "Redis", level: 4 }
    ]
}
```

Return only:

```text
["Java", "MongoDB"]
```

### Answer

Use:

```text
$filter
   ↓
level >= 7

then $map
   ↓
extract name
```

```javascript
db.candidates.aggregate([
    {
        $project: {
            strongSkills: {
                $map: {
                    input: {
                        $filter: {
                            input: "$skills",
                            as: "skill",
                            cond: {
                                $gte: [
                                    "$$skill.level",
                                    7
                                ]
                            }
                        }
                    },

                    as: "skill",

                    in: "$$skill.name"
                }
            }
        }
    }
])
```

---

# 34. Difficult conceptual question

Which is the best description?

**A.** `$map` converts array elements into separate pipeline documents.

**B.** `$unwind` transforms each array element but keeps the original pipeline-document count.

**C.** `$filter` returns a subset of an array based on a condition, while `$map` transforms each element.

**D.** `$reduce` always returns an array.

### Answer

**C**

And:

```text
$unwind → can increase pipeline document count
$map    → returns an array
$filter → returns an array
$reduce → produces one accumulated value
```

MongoDB's documentation defines `$map`, `$filter`, and `$reduce` accordingly. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/map/?msockid=15ecb796ec436d943b1fa17ced9d6ceb&utm_source=chatgpt.com "$map (expression operator) - Database Manual - MongoDB Docs"))

---

# 35. The complete aggregation picture so far

You now have:

```text
                   AGGREGATION
                        │
                        ↓
                    STAGES
                        │
        ┌───────────────┼────────────────┐
        ↓               ↓                ↓
     $match           $group           $lookup
        ↓               ↓                ↓
     filter         aggregate         join
        │
        ├── $project → reshape
        ├── $set → add/modify
        ├── $sort → order
        ├── $limit → restrict
        ├── $skip → skip
        ├── $unwind → expand arrays
        └── $facet → multiple pipelines
```

And inside those stages:

```text
                  EXPRESSIONS
                       │
     ┌─────────────────┼─────────────────┐
     ↓                 ↓                 ↓
  $cond             $map             $filter
  $switch           $reduce          $ifNull
  $add              $multiply        $divide
  $concat           $size            $arrayElemAt
  date operators    comparison       logical operators
```

That's a much better mental model than memorizing individual commands.

---

# 36. One thing I really want you to notice

We started with:

```javascript
db.users.find({
    role: "candidate"
})
```

and now we're capable of constructing something like:

```javascript
db.jobs.aggregate([
    {
        $match: {
            status: "open",
            location: "Pune"
        }
    },

    {
        $set: {
            skillCount: {
                $size: "$skills"
            }
        }
    },

    {
        $facet: {
            jobs: [
                {
                    $sort: {
                        salary: -1
                    }
                },
                {
                    $limit: 10
                }
            ],

            total: [
                {
                    $count: "count"
                }
            ],

            salary: [
                {
                    $group: {
                        _id: null,
                        average: {
                            $avg: "$salary"
                        }
                    }
                }
            ]
        }
    }
])
```

That's already approaching the kind of database work you'll encounter in a serious backend.

You don't need to memorize this entire pipeline.

You need to understand **how to construct it from smaller pieces**.

---

# 37. Lesson 6 takeaway

The important concepts are:

```text
$cond
    → if/else

$ifNull
    → fallback for null/missing values

$switch
    → multiple conditional branches

$map
    → transform every array element

$filter
    → select array elements

$reduce
    → accumulate an array into one result

$lookup + pipeline
    → sophisticated joins

$facet
    → multiple analyses from the same input

$expr
    → use aggregation expressions in queries
```

And remember these three:

> **`$map` transforms.**

> **`$filter` selects.**

> **`$reduce` accumulates.**

One more important distinction:

> **`$unwind` changes the number of pipeline documents; `$map` changes the contents of an array.**

MongoDB's aggregation system is built around composing stages and expressions, which is why increasingly complex pipelines can be constructed from these smaller concepts. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/?utm_source=chatgpt.com "Aggregation Pipeline - Database Manual - MongoDB Docs"))

### Next lesson: MongoDB Indexes & Query Performance

We'll leave most new aggregation syntax behind and learn something that is **absolutely essential for becoming a good backend developer**:

```text
How MongoDB actually finds documents
        ↓
indexes
        ↓
COLLSCAN vs IXSCAN
        ↓
single-field indexes
        ↓
compound indexes
        ↓
unique indexes
        ↓
multikey indexes
        ↓
index prefixes
        ↓
ESR rule
        ↓
explain()
        ↓
query performance
```

That lesson will be considerably more important for your actual ChatApp/JobForge backend than memorizing another 30 aggregation operators.