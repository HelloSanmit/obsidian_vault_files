
Got it. I won't use that name in examples again. I'll use varied random names from now on.

# MongoDB — Lesson 5: Aggregation Framework

This is a major step up from the CRUD/query lessons.

So far, you've mostly asked MongoDB:

> “Give me documents matching these conditions.”

Aggregation lets you ask:

> **“Take a collection of documents, transform them, group them, calculate things, combine data, and produce a new result.”**

MongoDB implements this through **aggregation pipelines**, where documents pass through stages sequentially. The output of one stage becomes the input of the next. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/?utm_source=chatgpt.com "Aggregation Pipeline - Database Manual - MongoDB Docs"))

---

# 1. Why do we even need aggregation?

Imagine this collection:

```javascript
orders
```

with documents like:

```javascript
{
    customer: "Aarav",
    city: "Pune",
    amount: 2500,
    status: "delivered"
}
```

and:

```javascript
{
    customer: "Meera",
    city: "Mumbai",
    amount: 1800,
    status: "delivered"
}
```

and:

```javascript
{
    customer: "Rohan",
    city: "Pune",
    amount: 4200,
    status: "cancelled"
}
```

Suppose you ask:

> What is the total delivered revenue from Pune?

A basic `find()` query cannot really perform this complete calculation.

You need to:

```text
filter
   ↓
group
   ↓
calculate
   ↓
return result
```

That's an aggregation pipeline.

---

# 2. The basic structure

```javascript
db.orders.aggregate([
    { stage1 },
    { stage2 },
    { stage3 }
])
```

For example:

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "delivered"
        }
    },

    {
        $group: {
            _id: "$city",
            totalRevenue: {
                $sum: "$amount"
            }
        }
    }
])
```

Conceptually:

```text
orders
  ↓
$match
  ↓
only delivered orders
  ↓
$group
  ↓
one result per city
```

Aggregation normally **does not modify the source collection**. The normal `aggregate()` result is a computed result; stages such as `$merge` or `$out` are used when you intentionally want to write pipeline output to a collection. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/?utm_source=chatgpt.com "Aggregation Pipeline - Database Manual - MongoDB Docs"))

---

# 3. Think of aggregation as a data-processing pipeline

This mental model is extremely important.

Suppose we start with:

```text
100,000 documents
```

Pipeline:

```text
100,000
   ↓
$match
   ↓
20,000
   ↓
$unwind
   ↓
80,000
   ↓
$group
   ↓
500
   ↓
$sort
   ↓
500
   ↓
$limit
   ↓
10
```

Each stage transforms the stream.

That's why it's called a **pipeline**. MongoDB documents this stage-by-stage processing model directly. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/?utm_source=chatgpt.com "Aggregation Pipeline - Database Manual - MongoDB Docs"))

---

# 4. The stages you absolutely need first

Don't try to learn every aggregation operator immediately.

The core stages are:

```text
$match
$project
$set / $addFields
$group
$sort
$limit
$skip
$unwind
$lookup
$facet
```

There are many more stages, but these form the foundation. MongoDB's aggregation-stage reference includes these among the core stages. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

We'll learn them in that order.

---

# 5. `$match`

You already know MongoDB queries.

`$match` is basically:

> **filter the documents entering the pipeline.**

Example:

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "delivered"
        }
    }
])
```

This means:

```text
all orders
    ↓
keep only delivered orders
```

The syntax inside `$match` uses standard MongoDB query predicates. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

---

# 6. `$match` with multiple conditions

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "delivered",
            city: "Pune",
            amount: {
                $gte: 2000
            }
        }
    }
])
```

This means:

```text
status = delivered
AND
city = Pune
AND
amount >= 2000
```

So all the query knowledge from Lessons 1 and 2 carries directly into aggregation.

That's intentional.

Aggregation isn't an entirely different MongoDB language.

It's MongoDB queries + transformations + expressions + stages.

---

# 7. Why `$match` early matters

Suppose you have:

```text
1,000,000 orders
```

but only:

```text
100,000 delivered orders
```

You could do:

```text
1,000,000
   ↓
$group
   ↓
$match
```

or:

```text
1,000,000
   ↓
$match
   ↓
100,000
   ↓
$group
```

The second approach often reduces the amount of data later stages need to process.

MongoDB's own aggregation examples explicitly use `$match` to reduce documents before subsequent stages such as `$unwind` and `$group`. ([MongoDB](https://www.mongodb.com/docs/manual/aggregation/?utm_source=chatgpt.com "Aggregation Operations - Database Manual - MongoDB Docs"))

We'll return to optimization in the indexing/performance lesson.

---

# 8. `$project`

`$project` reshapes the documents coming through the pipeline.

Suppose:

```javascript
{
    name: "Aarav",
    age: 24,
    email: "aarav@example.com",
    passwordHash: "...",
    city: "Pune"
}
```

You want:

```text
name
city
```

Use:

```javascript
db.users.aggregate([
    {
        $project: {
            name: 1,
            city: 1
        }
    }
])
```

Result:

```javascript
{
    _id: ...,
    name: "Aarav",
    city: "Pune"
}
```

`$project` can include or exclude fields and can also compute new fields. MongoDB describes it as reshaping documents in the pipeline. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

---

# 9. `_id` behavior

Just like ordinary projection, `_id` is included unless explicitly excluded.

So:

```javascript
{
    $project: {
        _id: 0,
        name: 1,
        city: 1
    }
}
```

produces:

```javascript
{
    name: "Aarav",
    city: "Pune"
}
```

This is useful when you are constructing API-friendly output.

---

# 10. `$project` can create computed fields

This is where aggregation becomes much more interesting.

Suppose:

```javascript
{
    product: "Laptop",
    price: 50000,
    quantity: 2
}
```

You can calculate:

```text
total = price × quantity
```

using:

```javascript
db.orders.aggregate([
    {
        $project: {
            product: 1,
            quantity: 1,
            total: {
                $multiply: ["$price", "$quantity"]
            }
        }
    }
])
```

Result:

```javascript
{
    product: "Laptop",
    quantity: 2,
    total: 100000
}
```

Notice the syntax:

```javascript
"$price"
```

means:

> Take the value of the `price` field from the current document.

This `$fieldName` expression syntax is fundamental in aggregation.

---

# 11. `$set` / `$addFields`

MongoDB's `$set` aggregation stage is an alias for `$addFields`. It adds fields while retaining the existing fields. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

Example:

```javascript
db.orders.aggregate([
    {
        $set: {
            total: {
                $multiply: ["$price", "$quantity"]
            }
        }
    }
])
```

Before:

```javascript
{
    product: "Laptop",
    price: 50000,
    quantity: 2
}
```

After:

```javascript
{
    product: "Laptop",
    price: 50000,
    quantity: 2,
    total: 100000
}
```

Compare:

```text
$project → shape/control the output fields

$set     → add/modify fields while keeping the existing document structure
```

---

# 12. `$group` — the heart of aggregation

This is probably the most important stage in today's lesson.

Suppose:

```javascript
{
    city: "Pune",
    amount: 2000
}
```

and:

```javascript
{
    city: "Pune",
    amount: 3000
}
```

and:

```javascript
{
    city: "Mumbai",
    amount: 4000
}
```

You want:

```text
Pune    → 5000
Mumbai  → 4000
```

Use:

```javascript
db.orders.aggregate([
    {
        $group: {
            _id: "$city",
            totalRevenue: {
                $sum: "$amount"
            }
        }
    }
])
```

Result:

```javascript
{
    _id: "Pune",
    totalRevenue: 5000
}
```

```javascript
{
    _id: "Mumbai",
    totalRevenue: 4000
}
```

MongoDB's `$group` groups input documents by an identifier expression and produces one output document per distinct group key. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

---

# 13. Why `_id` inside `$group`?

This is special.

In:

```javascript
$group: {
    _id: "$city"
}
```

`_id` means:

> **What value defines a group?**

So:

```javascript
_id: "$city"
```

means:

```text
same city → same group
```

You could also do:

```javascript
_id: "$status"
```

which gives:

```text
delivered
cancelled
pending
```

as groups.

---

# 14. `$group` is NOT the same `_id` concept as a normal document

This confuses beginners.

Here:

```javascript
{
    _id: ObjectId("...")
}
```

`_id` is the document's identity.

But here:

```javascript
$group: {
    _id: "$city"
}
```

`_id` is:

> **the grouping key of the output document.**

Completely different role.

---

# 15. `$sum`

Let's learn the most important accumulator first.

Count documents:

```javascript
{
    $group: {
        _id: "$city",
        count: {
            $sum: 1
        }
    }
}
```

If Pune has 200 orders:

```javascript
{
    _id: "Pune",
    count: 200
}
```

Why `$sum: 1`?

Because every incoming document contributes:

```text
1
```

So:

```text
document 1 → +1
document 2 → +1
document 3 → +1
...
```

result:

```text
total number of documents
```

This is one of the most useful MongoDB aggregation patterns.

---

# 16. `$sum` a field

```javascript
{
    $group: {
        _id: "$city",

        totalRevenue: {
            $sum: "$amount"
        }
    }
}
```

Now each document contributes its `amount`.

So:

```text
2000
3000
1500
```

becomes:

```text
6500
```

---

# 17. `$avg`

Calculate an average:

```javascript
{
    $group: {
        _id: "$city",

        averageOrderValue: {
            $avg: "$amount"
        }
    }
}
```

For:

```text
2000
3000
4000
```

result:

```text
3000
```

---

# 18. `$min` and `$max`

You can calculate:

```javascript
{
    $group: {
        _id: "$city",

        cheapestOrder: {
            $min: "$amount"
        },

        biggestOrder: {
            $max: "$amount"
        }
    }
}
```

So each city can produce:

```javascript
{
    _id: "Pune",
    cheapestOrder: 500,
    biggestOrder: 25000
}
```

---

# 19. `$first` and `$last`

These exist too:

```javascript
$first
$last
```

But there is a subtlety:

Their meaning depends on the order of the documents reaching the `$group` stage.

So if you care about "first" or "last", you should generally establish a meaningful ordering beforehand, often using `$sort`.

Example:

```javascript
db.orders.aggregate([
    {
        $sort: {
            createdAt: 1
        }
    },

    {
        $group: {
            _id: "$customerId",
            firstOrder: {
                $first: "$$ROOT"
            },
            lastOrder: {
                $last: "$$ROOT"
            }
        }
    }
])
```

Now "first" and "last" have a defined relationship to `createdAt`.

---

# 20. `$push` inside `$group`

Different from the update operator you learned in Lesson 4.

Inside aggregation:

```javascript
$push
```

can collect values into an array.

Suppose:

```text
Pune → orders 1, 2, 3
Mumbai → orders 4, 5
```

You can:

```javascript
{
    $group: {
        _id: "$city",

        orders: {
            $push: "$_id"
        }
    }
}
```

Result:

```javascript
{
    _id: "Pune",
    orders: [
        ObjectId("..."),
        ObjectId("..."),
        ObjectId("...")
    ]
}
```

So don't confuse:

```text
update $push
    → modify an existing document array

aggregation $push
    → build an array in the aggregation result
```

Same operator name, different context.

---

# 21. `$addToSet` inside aggregation

Similarly, you can collect unique values:

```javascript
{
    $group: {
        _id: "$city",

        categories: {
            $addToSet: "$category"
        }
    }
}
```

If five orders contain:

```text
electronics
books
electronics
clothing
books
```

you might get:

```text
[
    "electronics",
    "books",
    "clothing"
]
```

---

# 22. A complete pipeline

Let's build one.

Question:

> Find the total delivered revenue per city, highest first.

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "delivered"
        }
    },

    {
        $group: {
            _id: "$city",

            totalRevenue: {
                $sum: "$amount"
            }
        }
    },

    {
        $sort: {
            totalRevenue: -1
        }
    }
])
```

Read this like English:

```text
1. Keep delivered orders
2. Group by city
3. Sum amount
4. Sort highest to lowest
```

This way of **reading pipelines as a sentence** is one of the best ways to learn aggregation.

---

# 23. `$sort`

Same idea as ordinary querying:

```javascript
{
    $sort: {
        totalRevenue: -1
    }
}
```

means descending.

```javascript
1
```

means ascending.

MongoDB's aggregation `$sort` stage changes ordering but does not change the contents of the documents. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

---

# 24. `$limit`

Suppose:

```javascript
{
    $limit: 5
}
```

Only the first five pipeline results continue.

For example:

```text
top 5 cities by revenue
```

would typically be:

```javascript
[
    { $group: ... },
    { $sort: { totalRevenue: -1 } },
    { $limit: 5 }
]
```

The order matters.

This:

```text
group → sort → limit
```

means top 5.

Whereas:

```text
group → limit → sort
```

could produce a very different result.

That's a recurring theme:

> **Aggregation stage order matters.**

---

# 25. `$skip`

You've already seen `skip()` in queries.

Aggregation has:

```javascript
{
    $skip: 20
}
```

which skips the first 20 documents in the pipeline stream.

Often used with:

```text
$sort
$skip
$limit
```

for pagination.

But remember our earlier warning: large offset pagination can become increasingly expensive, and aggregation doesn't magically remove that limitation.

---

# 26. `$unwind` — one of the most important concepts

This is where aggregation starts feeling different.

Suppose:

```javascript
{
    name: "Aarav",
    skills: [
        "Java",
        "MongoDB",
        "Redis"
    ]
}
```

`$unwind` takes the array and creates one pipeline document per element. MongoDB defines `$unwind` exactly this way: it deconstructs an array field and outputs a document for each array element. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

```javascript
{
    $unwind: "$skills"
}
```

Output conceptually becomes:

```javascript
{
    name: "Aarav",
    skills: "Java"
}
```

```javascript
{
    name: "Aarav",
    skills: "MongoDB"
}
```

```javascript
{
    name: "Aarav",
    skills: "Redis"
}
```

One document became three pipeline documents.

---

# 27. Why would we want that?

Suppose you have:

```text
candidate → skills[]
```

and want:

> How many candidates know each skill?

You can't easily `$group` individual array elements until you break the array apart.

Pipeline:

```javascript
db.candidates.aggregate([
    {
        $unwind: "$skills"
    },

    {
        $group: {
            _id: "$skills",
            candidateCount: {
                $sum: 1
            }
        }
    }
])
```

Now:

```text
Java      → 1200
MongoDB   → 700
Redis     → 450
Python    → 900
```

This pattern is incredibly common.

---

# 28. `$unwind` mental model

Remember:

```text
Before:

Document
 └── array
      ├── A
      ├── B
      └── C

After $unwind:

Document → A
Document → B
Document → C
```

This one idea will make a huge part of aggregation understandable.

---

# 29. `$lookup`

Now we connect Lesson 3's **references** with aggregation.

Suppose:

```text
orders
customers
```

An order has:

```javascript
{
    customerId: ObjectId("c1"),
    amount: 3000
}
```

Customer:

```javascript
{
    _id: ObjectId("c1"),
    name: "Meera"
}
```

You want:

```text
order + customer information
```

MongoDB can use `$lookup`.

```javascript
db.orders.aggregate([
    {
        $lookup: {
            from: "customers",
            localField: "customerId",
            foreignField: "_id",
            as: "customer"
        }
    }
])
```

The result adds:

```javascript
customer: [
    {
        _id: ObjectId("c1"),
        name: "Meera"
    }
]
```

`$lookup` performs a left outer join with another collection in the same database and adds the matching documents as an array field. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/?utm_source=chatgpt.com "$lookup (aggregation stage) - Database Manual - MongoDB Docs"))

---

# 30. Why does `$lookup` return an array?

Even if there's logically one customer, MongoDB represents the lookup result as an array:

```javascript
customer: [
    {
        ...
    }
]
```

because the lookup can match multiple documents.

If the relationship is known to be one-to-one and you want a single object, you can follow it with:

```javascript
{
    $unwind: "$customer"
}
```

Now:

```javascript
customer: {
    _id: ...,
    name: "Meera"
}
```

So a very common pattern is:

```text
$lookup
   ↓
$unwind
```

---

# 31. `$lookup` is not “free”

This matters for your future backend work.

MongoDB notes that `$lookup` performance depends on the operation and indexing of the foreign field; unindexed lookups or expensive correlated operations on large collections can become slow. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/?utm_source=chatgpt.com "$lookup (aggregation stage) - Database Manual - MongoDB Docs"))

So:

```text
MongoDB supports joins
```

does **not** mean:

```text
"Forget about schema design and join everything."
```

That brings us back to Lesson 3.

---

# 32. `$facet`

This is a very useful later-stage concept.

Suppose a job-search API needs to return:

```text
jobs
total count
salary distribution
location counts
```

from the same filtered dataset.

`$facet` lets you run multiple aggregation pipelines over the same input documents. MongoDB documents `$facet` for exactly this kind of multi-faceted processing. ([MongoDB](https://www.mongodb.com/docs/manual/reference/mql/aggregation-stages/?utm_source=chatgpt.com "Aggregation Stages - Database Manual - MongoDB Docs"))

Conceptually:

```javascript
{
    $facet: {
        jobs: [
            { $sort: { createdAt: -1 } },
            { $limit: 20 }
        ],

        total: [
            { $count: "count" }
        ],

        locations: [
            {
                $group: {
                    _id: "$location",
                    count: { $sum: 1 }
                }
            }
        ]
    }
}
```

This is highly relevant to your future **JobForge** project.

We won't go deeply into `$facet` today; it deserves its own treatment later.

---

# 33. Aggregation expressions

Here's an important distinction.

Stages:

```text
$match
$group
$project
$sort
```

operate as pipeline stages.

Inside those stages, you use **expressions/operators** such as:

```text
$sum
$avg
$multiply
$divide
$cond
$ifNull
$concat
$size
```

So conceptually:

```text
Pipeline
│
├── stages
│    ├── $match
│    ├── $group
│    └── $project
│
└── expressions inside stages
     ├── $sum
     ├── $multiply
     ├── $cond
     └── ...
```

This distinction becomes important as pipelines get more advanced.

---

# 34. Example: calculate discounted price

Suppose:

```javascript
{
    product: "Monitor",
    price: 20000,
    discountPercent: 10
}
```

We can calculate:

```javascript
{
    $project: {
        product: 1,

        finalPrice: {
            $multiply: [
                "$price",
                {
                    $subtract: [
                        1,
                        {
                            $divide: [
                                "$discountPercent",
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

Conceptually:

```text
discount = 10 / 100 = 0.10

1 - 0.10 = 0.90

20000 × 0.90 = 18000
```

This is where aggregation starts becoming a small data-processing language.

---

# 35. Aggregation vs application-side JavaScript

Suppose you need:

> Calculate total sales by city for 5 million orders.

You could theoretically:

```text
MongoDB → fetch 5 million documents
       ↓
Node.js
       ↓
loop
       ↓
calculate
```

That is generally a poor approach.

Aggregation allows the database to perform the filtering, grouping, and calculation:

```text
MongoDB
   ↓
$match
$group
$sort
   ↓
small result
   ↓
Node.js
```

MongoDB explicitly describes aggregation operations as a way to process groups of documents and perform analytics within the database rather than moving all the data elsewhere. ([MongoDB](https://www.mongodb.com/docs/manual/aggregation/?utm_source=chatgpt.com "Aggregation Operations - Database Manual - MongoDB Docs"))

---

# 36. A complete real-world example

Suppose:

```javascript
orders
```

contains:

```javascript
{
    customerId: ObjectId("..."),
    city: "Pune",
    status: "delivered",
    amount: 2500,
    items: [
        { product: "Keyboard", quantity: 1 },
        { product: "Mouse", quantity: 2 }
    ]
}
```

Question:

> Find the top three cities by delivered revenue.

Pipeline:

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "delivered"
        }
    },

    {
        $group: {
            _id: "$city",
            revenue: {
                $sum: "$amount"
            }
        }
    },

    {
        $sort: {
            revenue: -1
        }
    },

    {
        $limit: 3
    }
])
```

Read it:

```text
delivered orders
      ↓
group by city
      ↓
sum revenue
      ↓
highest first
      ↓
top 3
```

That is the basic aggregation mindset.

---

# 37. Another example: most popular skills

Collection:

```javascript
candidates
```

Document:

```javascript
{
    name: "Meera",
    skills: [
        "Java",
        "Spring Boot",
        "MongoDB"
    ]
}
```

Pipeline:

```javascript
db.candidates.aggregate([
    {
        $unwind: "$skills"
    },

    {
        $group: {
            _id: "$skills",
            count: {
                $sum: 1
            }
        }
    },

    {
        $sort: {
            count: -1
        }
    },

    {
        $limit: 10
    }
])
```

This answers:

> Which ten skills occur most frequently among candidates?

That is the sort of query you could actually build into JobForge analytics.

---

# 38. Pipeline order: a crucial skill

Consider:

```javascript
[
    { $match: ... },
    { $unwind: ... },
    { $group: ... },
    { $sort: ... },
    { $limit: ... }
]
```

Each stage changes what the next stage sees.

For example:

```text
$match → $unwind
```

means you only unwind the matched documents.

But:

```text
$unwind → $match
```

means you're unwinding everything before filtering.

That may produce a dramatically different amount of intermediate data.

So don't think of a pipeline as:

```text
"a list of independent commands"
```

Think:

```text
output of Stage 1
          ↓
input to Stage 2
          ↓
output of Stage 2
          ↓
input to Stage 3
```

---

# 39. Aggregation pipeline naming convention

When pipelines get complicated, format them like this:

```javascript
db.orders.aggregate([
    // 1. Filter
    {
        $match: {
            status: "delivered"
        }
    },

    // 2. Group
    {
        $group: {
            _id: "$city",
            revenue: { $sum: "$amount" }
        }
    },

    // 3. Sort
    {
        $sort: {
            revenue: -1
        }
    },

    // 4. Top 5
    {
        $limit: 5
    }
])
```

This makes debugging much easier.

---

# 40. One subtle issue: intermediate document growth

Aggregation isn't exempt from MongoDB's BSON document-size rules.

Returned aggregation documents are subject to MongoDB's 16 MiB BSON document limit. MongoDB also limits a pipeline to 1000 stages, and memory-heavy stages have configurable disk-use behavior depending on deployment/version. ([MongoDB](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-limits/?utm_source=chatgpt.com "Aggregation Pipeline Limits - Database Manual - MongoDB Docs"))

For now, don't worry about memorizing the limits.

Just understand:

> **Aggregation can itself create large intermediate/results, so the shape of your pipeline matters.**

---

# 41. Exercises

Just a few, because the concept matters more than doing 30 syntax questions.

## Exercise 1 — Easy

Collection:

```javascript
products
```

```javascript
{
    name: "Keyboard",
    price: 2500,
    category: "electronics"
}
```

Question:

> Return only products belonging to `electronics`.

### Answer

```javascript
db.products.aggregate([
    {
        $match: {
            category: "electronics"
        }
    }
])
```

---

# 42. Exercise 2 — Medium

Collection:

```text
orders
```

Each document:

```javascript
{
    city: "Pune",
    amount: 2000
}
```

Question:

> Calculate total order value for each city.

### Answer

```javascript
db.orders.aggregate([
    {
        $group: {
            _id: "$city",
            total: {
                $sum: "$amount"
            }
        }
    }
])
```

---

# 43. Exercise 3 — Medium

Collection:

```javascript
candidates
```

```javascript
{
    name: "Meera",
    skills: [
        "Java",
        "MongoDB",
        "Redis"
    ]
}
```

Question:

> Find how many candidates have each skill.

### Answer

```javascript
db.candidates.aggregate([
    {
        $unwind: "$skills"
    },

    {
        $group: {
            _id: "$skills",
            count: {
                $sum: 1
            }
        }
    }
])
```

The important part is understanding **why `$unwind` comes first**.

Without it, `$group` sees the entire array as a value rather than individual skills.

---

# 44. Exercise 4 — Hard

Collection:

```javascript
orders
```

Each order has:

```javascript
{
    city: "Pune",
    status: "delivered",
    amount: 5000
}
```

Question:

> Return the top 3 cities by delivered revenue.

### Answer

```javascript
db.orders.aggregate([
    {
        $match: {
            status: "delivered"
        }
    },

    {
        $group: {
            _id: "$city",
            revenue: {
                $sum: "$amount"
            }
        }
    },

    {
        $sort: {
            revenue: -1
        }
    },

    {
        $limit: 3
    }
])
```

The sequence:

```text
$match
→ $group
→ $sort
→ $limit
```

is more important to understand than memorizing the syntax.

---

# 45. One difficult conceptual question

Which statement is correct?

**A.** `$group` modifies the original documents.

**B.** `$unwind` combines many documents into one.

**C.** `$lookup` can combine related documents from another collection, returning matches in an array field.

**D.** `$match` can only be used at the end of a pipeline.

### Answer

**C.**

`$lookup` performs a left outer join and adds matching foreign documents as an array. `$match` can appear wherever appropriate in a pipeline and is often placed early to reduce the amount of data later stages process. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/?utm_source=chatgpt.com "$lookup (aggregation stage) - Database Manual - MongoDB Docs"))

---

# 46. The aggregation mental model

I want you to leave this lesson with this picture:

```text
                 AGGREGATION
                      │
                      ↓
                input documents
                      │
              ┌───────┴───────┐
              ↓               ↓
           $match          filter
              ↓
           $unwind        expand arrays
              ↓
           $group         combine documents
              ↓
         $project/$set    reshape/calculate
              ↓
           $sort          order
              ↓
           $limit         restrict
              ↓
          $lookup         join
              ↓
             result
```

Not every pipeline uses every stage.

---

# 47. What matters most from Lesson 5

You should now understand these concepts:

```text
aggregate()
    ↓
pipeline

$match
    ↓
filter documents

$project
    ↓
shape output

$set
    ↓
add/compute fields

$group
    ↓
group documents + calculate

$sum / $avg / $min / $max
    ↓
aggregation accumulators

$unwind
    ↓
turn array elements into separate pipeline documents

$sort
    ↓
order results

$limit
    ↓
take first N results

$lookup
    ↓
join related collection
```

And the most important conceptual distinction:

> **A normal query asks MongoDB to find documents. An aggregation pipeline asks MongoDB to process documents.**

That distinction is the foundation for everything that follows.

### Next lesson: MongoDB Aggregation — Part 2

We'll go substantially deeper into **aggregation expressions**, especially `$cond`, `$ifNull`, `$map`, `$filter`, `$reduce`, string/date operators, `$lookup` with pipelines, `$facet`, and more realistic API/reporting queries. That's where aggregation starts becoming genuinely powerful rather than just “group and count.”