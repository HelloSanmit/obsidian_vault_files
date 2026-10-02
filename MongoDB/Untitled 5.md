
# MongoDB — Lesson 7: Indexes & Query Performance

This is one of the **most important MongoDB lessons for backend development**.

Up to now, you've learned how to ask MongoDB for data.

Now we're asking:

> **How does MongoDB find that data efficiently?**

Imagine a collection containing:

```text
10 documents
100 documents
10,000 documents
10,000,000 documents
```

A query that feels instant at 100 documents can become painfully slow at 10 million if MongoDB has to inspect every document.

That's where **indexes** come in.

MongoDB can use indexes to avoid scanning the entire collection and can also use index ordering to satisfy some sorts efficiently. ([MongoDB](https://www.mongodb.com/docs/v8.0/indexes/?utm_source=chatgpt.com "Indexes - Database Manual v8.0 - MongoDB Docs"))

---

# 1. What is an index?

Think about a book.

Without an index, to find:

```text
"MongoDB"
```

you might read:

```text
page 1
page 2
page 3
...
page 800
```

That's basically a **collection scan**.

With a book index:

```text
MongoDB → page 437
```

you jump much closer to the answer.

MongoDB indexes work on the same general idea:

```text
WITHOUT INDEX

Collection
│
├── Document 1 → check
├── Document 2 → check
├── Document 3 → check
├── ...
└── Document 10,000,000 → check
```

versus:

```text
WITH INDEX

Index
│
└── relevant key range
          ↓
       documents
```

The first case is represented in query plans by `COLLSCAN`; an index scan appears as `IXSCAN`. ([MongoDB](https://www.mongodb.com/docs/manual/reference/explain-results/?utm_source=chatgpt.com "Explain Results - Database Manual - MongoDB Docs"))

---

# 2. Why not index every field?

This is your first important trade-off.

Indexes make **reads** faster, but they aren't free.

When you insert or update a document, MongoDB may also have to update the relevant indexes. Indexes also consume storage and memory.

So:

```text
More indexes
    ↓
potentially faster reads
    +
more write/storage/maintenance cost
```

MongoDB's indexing guidance therefore recommends indexes based on actual query workload rather than indexing every field. ([MongoDB](https://www.mongodb.com/docs/manual/core/query-optimization/?utm_source=chatgpt.com "Query Optimization - Database Manual - MongoDB Docs"))

This is why you'll eventually hear:

> **Index for your queries, not for your schema.**

---

# 3. The default `_id` index

MongoDB automatically creates a unique index on `_id` when a collection is created.

So:

```javascript
db.users.find({
    _id: ObjectId("...")
})
```

already has an index available.

The `_id` index cannot be dropped independently. ([MongoDB](https://www.mongodb.com/docs/v8.0/indexes/?utm_source=chatgpt.com "Indexes - Database Manual v8.0 - MongoDB Docs"))

That's one reason `_id` lookups are so fundamental.

---

# 4. Creating a basic index

Suppose:

```text
users
```

contains:

```javascript
{
    name: "Aarav",
    email: "aarav@example.com",
    age: 24
}
```

You frequently run:

```javascript
db.users.find({
    email: "aarav@example.com"
})
```

Create:

```javascript
db.users.createIndex({
    email: 1
})
```

The `1` means ascending index order.

You can also use:

```javascript
db.users.createIndex({
    email: -1
})
```

for descending order.

For a single-field equality lookup, the direction usually doesn't matter much; direction becomes especially relevant when considering sorting and compound indexes. ([MongoDB](https://www.mongodb.com/docs/manual/core/query-optimization/?utm_source=chatgpt.com "Query Optimization - Database Manual - MongoDB Docs"))

---

# 5. What does the index actually contain?

Conceptually:

```text
users

Document A
email = "aarav@example.com"

Document B
email = "meera@example.com"

Document C
email = "riya@example.com"
```

Index:

```text
"aarav@example.com" → Document A
"meera@example.com" → Document B
"riya@example.com"  → Document C
```

The index maintains keys in an ordered structure; MongoDB's documentation describes compound indexes in terms of a B-tree storing values according to the specified key order. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

You don't need to implement the tree yourself.

Your job is to understand:

```text
index
  ↓
organized keys
  ↓
efficient navigation to relevant records
```

---

# 6. `COLLSCAN` vs `IXSCAN`

This is something I want you to recognize instantly.

### `COLLSCAN`

```text
COLL = collection
SCAN = scan it
```

MongoDB examines documents in the collection.

### `IXSCAN`

```text
IX = index
SCAN = scan index
```

MongoDB is scanning an index.

MongoDB's `explain` output uses `COLLSCAN` when the query planner chooses a collection scan and `IXSCAN` when it chooses an index scan. ([MongoDB](https://www.mongodb.com/docs/manual/reference/explain-results/?utm_source=chatgpt.com "Explain Results - Database Manual - MongoDB Docs"))

---

# 7. How do you check?

Use:

```javascript
db.users.explain("executionStats").find({
    email: "aarav@example.com"
})
```

This gives you information about the chosen plan and actual execution statistics.

MongoDB documents `executionStats` as including metrics such as:

```text
nReturned
executionTimeMillis
totalKeysExamined
totalDocsExamined
```

among other details. ([MongoDB](https://www.mongodb.com/docs/manual/reference/explain-results/?utm_source=chatgpt.com "Explain Results - Database Manual - MongoDB Docs"))

---

# 8. The numbers you should care about

Suppose you run:

```javascript
db.users.explain("executionStats").find({
    email: "aarav@example.com"
})
```

and get roughly:

```text
nReturned = 1
totalDocsExamined = 1
```

That's excellent.

Now imagine:

```text
nReturned = 1
totalDocsExamined = 2,000,000
```

That's very different.

MongoDB defines `totalDocsExamined` as the number of documents examined during execution, while `totalKeysExamined` is the number of index entries examined. ([MongoDB](https://www.mongodb.com/docs/manual/reference/explain-results/?utm_source=chatgpt.com "Explain Results - Database Manual - MongoDB Docs"))

---

# 9. Why `nReturned` alone is misleading

Suppose:

```text
nReturned = 10
```

That doesn't tell you whether MongoDB had to inspect:

```text
10 documents
```

or:

```text
5,000,000 documents
```

That's why you need:

```text
nReturned
totalDocsExamined
totalKeysExamined
```

together.

A useful intuition:

```text
nReturned
    ↓
what I got

totalDocsExamined
    ↓
how much document work happened

totalKeysExamined
    ↓
how much index work happened
```

---

# 10. Simple index example

Without index:

```javascript
db.users.explain("executionStats").find({
    email: "aarav@example.com"
})
```

might conceptually look like:

```text
COLLSCAN
docsExamined: 1,000,000
nReturned: 1
```

With:

```javascript
db.users.createIndex({
    email: 1
})
```

it might become:

```text
IXSCAN
keysExamined: 1
docsExamined: 1
nReturned: 1
```

The exact numbers depend on the data and query planner; the point is understanding what the metrics mean, not memorizing any particular result. MongoDB recommends using `explain` to verify whether an index is actually being used. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/analyze-query-plan/?utm_source=chatgpt.com "Interpret Explain Plan Results - Database Manual - MongoDB Docs"))

---

# 11. Unique indexes

Now something extremely useful for real applications.

Suppose every user must have a unique email.

Create:

```javascript
db.users.createIndex(
    { email: 1 },
    { unique: true }
)
```

Now MongoDB rejects inserts/updates that would create duplicate indexed key values. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.createindexes/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

This is more than a performance feature.

It's a **data-integrity constraint**.

That's important.

An index can therefore provide:

```text
1. performance
2. uniqueness enforcement
```

---

# 12. Why application-only uniqueness is dangerous

Imagine your backend does:

```text
1. Check whether email exists
2. If not, insert user
```

Two requests arrive simultaneously:

```text
Request A → check → doesn't exist
Request B → check → doesn't exist
```

Both might attempt insertion.

A unique index:

```javascript
{
    unique: true
}
```

lets the database enforce the uniqueness constraint.

So your architecture becomes:

```text
Application validation
        +
Database constraint
```

That's much stronger than relying on application logic alone.

---

# 13. `getIndexes()`

You can inspect the indexes on a collection:

```javascript
db.users.getIndexes()
```

You may see something conceptually like:

```javascript
[
    {
        name: "_id_",
        key: {
            _id: 1
        }
    },

    {
        name: "email_1",
        key: {
            email: 1
        }
    }
]
```

MongoDB documents `getIndexes()` as the way to retrieve the index definitions/names for a collection. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/drop-index/?utm_source=chatgpt.com "Drop an Index - Database Manual - MongoDB Docs"))

---

# 14. Dropping an index

You can remove an index:

```javascript
db.users.dropIndex("email_1")
```

You can't drop the default `_id` index independently. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/drop-index/?utm_source=chatgpt.com "Drop an Index - Database Manual - MongoDB Docs"))

In production, don't casually delete an index.

An index you consider “unnecessary” may be supporting another query.

---

# 15. Compound indexes

Now we're getting to the really important part.

Suppose JobForge frequently executes:

```javascript
db.jobs.find({
    location: "Pune",
    status: "open"
})
```

You could create:

```javascript
db.jobs.createIndex({
    location: 1,
    status: 1
})
```

This is a **compound index**.

MongoDB compound indexes contain multiple fields, and **field order matters**. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

---

# 16. Why field order matters

Suppose the index is:

```text
{ location: 1, status: 1 }
```

Conceptually it is ordered like:

```text
Pune
    open
    closed

Mumbai
    open
    closed

Delhi
    open
    closed
```

More abstractly:

```text
location
   ↓
status
```

MongoDB sorts by the first indexed field first, then the second within the first-field groups. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

Therefore:

```text
{ location, status }
```

is not equivalent to:

```text
{ status, location }
```

for all query patterns.

---

# 17. Index prefixes

Suppose you have:

```javascript
{
    location: 1,
    status: 1,
    salary: -1
}
```

The index prefixes are:

```text
{ location: 1 }

{ location: 1, status: 1 }

{ location: 1, status: 1, salary: -1 }
```

MongoDB can use compound indexes for queries involving these prefix combinations. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

So this index can naturally support:

```javascript
{ location: "Pune" }
```

and:

```javascript
{
    location: "Pune",
    status: "open"
}
```

and:

```javascript
{
    location: "Pune",
    status: "open",
    salary: { $gte: 800000 }
}
```

But a query on only:

```javascript
{ status: "open" }
```

does not correspond to an index prefix.

That's a crucial point.

---

# 18. The common misconception

People often think:

> “If my compound index contains a field, queries on that field automatically use it efficiently.”

Not necessarily.

With:

```text
{ location: 1, status: 1 }
```

the index is strongly organized around:

```text
location → status
```

not:

```text
status
```

alone.

MongoDB specifically describes this prefix behavior and notes that a query can use a non-prefix field in some cases but may have to scan more index entries and be less efficient. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

---

# 19. Compound indexes and sorting

Suppose you run:

```javascript
db.jobs.find({
    location: "Pune"
}).sort({
    salary: -1
})
```

You might want:

```javascript
{
    location: 1,
    salary: -1
}
```

Why?

Because:

```text
location = equality
salary   = sort
```

This brings us to one of MongoDB's most important index-design heuristics.

---

# 20. ESR — Equality, Sort, Range

MongoDB's official **ESR guideline** is:

```text
E = Equality
S = Sort
R = Range
```

For many compound-index workloads:

```text
Equality fields
       ↓
Sort fields
       ↓
Range fields
```

This is a guideline rather than a universal law. MongoDB notes that when a range predicate is highly selective, an **ERS** ordering can sometimes be better: Equality → Range → Sort. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

---

# 21. Equality

Example:

```javascript
{
    status: "open"
}
```

That's equality.

Another:

```javascript
{
    location: "Pune"
}
```

Also equality.

MongoDB says equality fields should come first in the ESR approach. Multiple equality fields can be arranged in either order relative to one another, although selectivity still matters. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

---

# 22. Sort

Example:

```javascript
.sort({
    salary: -1
})
```

That's the sort component.

So if your query is:

```javascript
db.jobs.find({
    location: "Pune"
}).sort({
    salary: -1
})
```

a natural candidate is:

```javascript
{
    location: 1,
    salary: -1
}
```

The equality field comes first, then the sort field. MongoDB uses this general arrangement to help queries avoid an in-memory sort when the index can satisfy the requested order. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

---

# 23. Range

Examples:

```javascript
{
    salary: { $gte: 800000 }
}
```

or:

```javascript
{
    age: { $lt: 30 }
}
```

These are range predicates.

MongoDB classifies inequality operators such as `$ne` and `$nin` as range operators for this guideline. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

---

# 24. Example of ESR

Query:

```javascript
db.jobs.find({
    location: "Pune",
    status: "open",
    salary: {
        $gte: 800000
    }
}).sort({
    createdAt: -1
})
```

Potential index:

```javascript
db.jobs.createIndex({
    location: 1,
    status: 1,
    createdAt: -1,
    salary: 1
})
```

Mental classification:

```text
location      → Equality
status        → Equality
createdAt     → Sort
salary        → Range
```

Therefore:

```text
E E S R
```

This follows the ESR structure.

However, MongoDB explicitly says to consider **ERS** when the range predicate is sufficiently selective and reducing the number of records before sorting is more beneficial. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

---

# 25. Why “selectivity” matters

Suppose:

```text
1,000,000 jobs
```

Query:

```text
status = "open"
```

matches:

```text
900,000 jobs
```

That's not very selective.

Now:

```text
jobId = "ABC123"
```

matches:

```text
1 job
```

That's highly selective.

MongoDB defines query selectivity in terms of how effectively the predicate filters the collection; highly selective predicates generally make index use more effective. ([MongoDB](https://www.mongodb.com/docs/manual/core/query-optimization/?utm_source=chatgpt.com "Query Optimization - Database Manual - MongoDB Docs"))

So:

```text
high selectivity
    ↓
small candidate set
    ↓
index can be especially valuable
```

---

# 26. `$ne` and `$nin`

Remember Lesson 2?

```javascript
{
    age: { $ne: 25 }
}
```

and:

```javascript
{
    role: {
        $nin: ["admin", "recruiter"]
    }
}
```

These may look like highly specific conditions.

But they often match **a large portion of the collection**.

MongoDB specifically notes that `$ne` and `$nin` are often not very selective and that an index does not necessarily make such queries faster. ([MongoDB](https://www.mongodb.com/docs/manual/core/query-optimization/?utm_source=chatgpt.com "Query Optimization - Database Manual - MongoDB Docs"))

Very important:

> **Having an index does not guarantee a faster query.**

The query planner decides whether using it is worthwhile.

---

# 27. The query planner

MongoDB has a **query optimizer/planner**.

When you send a query, MongoDB evaluates possible plans and chooses a winning plan.

`explain()` can show the query planner's information and, with `executionStats`, statistics from executing the winning plan. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.explain/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

So your application does not simply say:

```text
"Use index X."
```

MongoDB normally decides.

You can inspect that decision.

---

# 28. `explain("executionStats")`

This is a command I want you to become comfortable using:

```javascript
db.jobs
    .explain("executionStats")
    .find({
        location: "Pune",
        status: "open"
    })
```

Look for:

```text
winningPlan
stage
nReturned
totalKeysExamined
totalDocsExamined
executionTimeMillis
```

MongoDB's explain documentation describes these fields and the `COLLSCAN`/`IXSCAN` stages. ([MongoDB](https://www.mongodb.com/docs/manual/reference/explain-results/?utm_source=chatgpt.com "Explain Results - Database Manual - MongoDB Docs"))

---

# 29. What is a “good” explain result?

There's no universal magic number.

But suppose:

```text
nReturned = 10
totalDocsExamined = 10
```

That's generally a nice sign.

Compare:

```text
nReturned = 10
totalDocsExamined = 500,000
```

Now the database is doing much more work than the number of results suggests.

You should investigate.

This is why performance tuning should be based on actual measurements rather than:

> “I created an index, so the query must be fast now.”

---

# 30. Covered queries

A more advanced concept.

Suppose your query is:

```javascript
db.users.find(
    { email: "aarav@example.com" },
    { _id: 0, email: 1 }
)
```

and your index contains the fields necessary to answer both the filter and projection.

In some situations MongoDB can answer the query directly from the index without fetching the full documents.

This is called a **covered query**.

Conceptually:

```text
Index
 ├── email
 └── enough projected data
       ↓
result
```

rather than:

```text
Index
   ↓
find document
   ↓
fetch from collection
```

This is an optimization concept worth knowing, although it is not something you should design every query around.

---

# 31. Multikey indexes

Now connect this to arrays from Lessons 3–6.

Suppose:

```javascript
{
    name: "Riya",
    skills: [
        "Java",
        "MongoDB",
        "Redis"
    ]
}
```

You create:

```javascript
db.candidates.createIndex({
    skills: 1
})
```

MongoDB automatically treats an index on an array field as a **multikey index**. It creates index entries corresponding to the array's values. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/?utm_source=chatgpt.com "Multikey Indexes - Database Manual - MongoDB Docs"))

Now:

```javascript
db.candidates.find({
    skills: "MongoDB"
})
```

can benefit from that index.

---

# 32. Why is it called “multikey”?

Because one document can generate **multiple index entries**.

For:

```javascript
skills: [
    "Java",
    "MongoDB",
    "Redis"
]
```

conceptually:

```text
Java    → document A
MongoDB → document A
Redis   → document A
```

One document:

```text
A
```

produced multiple keys.

That's the idea behind multikey indexes. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/?utm_source=chatgpt.com "Multikey Indexes - Database Manual - MongoDB Docs"))

---

# 33. Multikey index on embedded documents

Suppose:

```javascript
{
    stock: [
        {
            size: "small",
            quantity: 8
        },
        {
            size: "large",
            quantity: 10
        }
    ]
}
```

You can index:

```javascript
db.products.createIndex({
    "stock.quantity": 1
})
```

MongoDB creates a multikey index when the indexed path goes through an array. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/create-multikey-index-embedded/?utm_source=chatgpt.com "Create an Index on an Embedded Field in an Array - Database Manual - MongoDB Docs"))

This connects directly to the nested-document work you did earlier.

---

# 34. Important multikey restriction

MongoDB has a significant restriction for compound multikey indexes:

> A document cannot have more than one indexed field whose value is an array for that compound multikey index. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/?utm_source=chatgpt.com "Multikey Indexes - Database Manual - MongoDB Docs"))

You don't need to memorize every edge case yet.

Just remember:

```text
arrays + compound indexes
        ↓
special rules exist
```

We'll revisit this when we discuss more complicated schema design.

---

# 35. Partial indexes

Here's a useful real-world feature.

Suppose your `users` collection has:

```text
10 million users
```

but only:

```text
500,000
```

are active.

And your important queries are always:

```javascript
{
    status: "active"
}
```

You could create a partial index:

```javascript
db.users.createIndex(
    { email: 1 },
    {
        partialFilterExpression: {
            status: "active"
        }
    }
)
```

Now only documents satisfying the filter are indexed.

MongoDB describes partial indexes as indexing only documents matching the specified filter expression, reducing index storage and maintenance costs. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/index-partial/?utm_source=chatgpt.com "Partial Indexes - Database Manual v8.0 - MongoDB Docs"))

---

# 36. Why partial indexes are interesting

Without partial index:

```text
10,000,000 users
       ↓
email index
       ↓
all 10M potentially represented
```

Partial:

```text
10,000,000 users
       ↓
only active users
       ↓
500,000 indexed
```

That's a powerful optimization when the workload naturally focuses on a subset.

There is one important condition:

> MongoDB cannot use the partial index for a query if doing so could produce an incomplete result set.

The query generally needs to include conditions compatible with the partial filter. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/index-partial/?utm_source=chatgpt.com "Partial Indexes - Database Manual v8.0 - MongoDB Docs"))

---

# 37. Partial vs sparse

You may encounter:

```text
sparse index
```

A sparse index only includes documents where the indexed field exists.

For example:

```javascript
db.users.createIndex(
    { phone: 1 },
    { sparse: true }
)
```

MongoDB notes that partial indexes can often provide sparse-like behavior with more control, because partial filters can express conditions beyond simple field existence. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/index-sparse/?utm_source=chatgpt.com "Sparse Indexes - Database Manual v8.0 - MongoDB Docs"))

Don't spend much time on sparse indexes yet.

For modern application design, **partial indexes are generally the more flexible concept to understand**.

---

# 38. Indexing your ChatApp

Let's revisit the architecture conceptually.

Suppose messages look like:

```javascript
{
    senderId: ObjectId("..."),
    receiverId: ObjectId("..."),
    text: "Hello",
    seen: false,
    createdAt: ISODate("...")
}
```

What queries will your backend probably perform frequently?

Something like:

```text
messages between user A and user B
```

or:

```text
messages for a conversation
sorted newest first
```

Suppose your query is:

```javascript
db.messages.find({
    senderId: senderId,
    receiverId: receiverId
}).sort({
    createdAt: -1
})
```

A possible compound index is:

```javascript
db.messages.createIndex({
    senderId: 1,
    receiverId: 1,
    createdAt: -1
})
```

Mental model:

```text
sender      → equality
receiver    → equality
createdAt   → sort
```

That fits the ESR pattern nicely. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

Notice what we're doing differently from a beginner:

We're not saying:

> “Messages need indexes.”

We're asking:

> **“What exact message queries does the application repeatedly execute?”**

Then we design the index.

That's the professional mindset.

---

# 39. ChatApp unread messages

Suppose you frequently ask:

```javascript
db.messages.find({
    receiverId: someUser,
    seen: false
})
```

Potential index:

```javascript
db.messages.createIndex({
    receiverId: 1,
    seen: 1
})
```

But don't blindly create it.

First ask:

```text
How many messages?
How selective is seen:false?
Do we sort?
Do we paginate?
What does the real query look like?
```

Then verify with:

```javascript
db.messages
    .explain("executionStats")
    .find({
        receiverId: someUser,
        seen: false
    })
```

This is exactly how you should approach performance work.

---

# 40. JobForge indexes

Suppose JobForge supports:

```text
GET /jobs?location=Pune&status=open
```

with:

```text
sort by newest
```

Query:

```javascript
db.jobs.find({
    location: "Pune",
    status: "open"
}).sort({
    createdAt: -1
})
```

Natural index candidate:

```javascript
db.jobs.createIndex({
    location: 1,
    status: 1,
    createdAt: -1
})
```

Classification:

```text
location   → Equality
status     → Equality
createdAt  → Sort
```

Again:

```text
E E S
```

---

# 41. Another JobForge query

Suppose:

```text
GET /jobs?location=Pune&minSalary=800000
sort by salary descending
```

Query:

```javascript
db.jobs.find({
    location: "Pune",
    salary: {
        $gte: 800000
    }
}).sort({
    salary: -1
})
```

Now we have:

```text
location → Equality
salary   → Range + Sort
```

This is exactly where you shouldn't mechanically chant:

```text
ESR always
ESR always
ESR always
```

MongoDB explicitly says the choice between:

```text
ESR
ERS
```

depends on how selective the range is and whether avoiding a sort is more important. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

This is why performance engineering requires measurement.

---

# 42. Sort direction in compound indexes

Suppose:

```javascript
{
    location: 1,
    createdAt: -1
}
```

The direction of the individual keys matters for compound sort support.

MongoDB can traverse an index in forward or reverse directions, so some ascending/descending sort combinations can be supported without creating another index, depending on the requested sort pattern and equality predicates.

The key point for now:

> **Compound index field order and direction are part of the query-design decision.**

MongoDB's compound-index documentation covers these ordering rules. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

We'll practice this heavily later.

---

# 43. Why too many indexes are bad

Imagine:

```text
users
```

with:

```text
20 indexes
```

Every user insertion/update may need to maintain many corresponding index structures.

That means:

```text
more indexes
   ↓
more disk
more maintenance
more write work
more operational complexity
```

MongoDB recommends choosing indexes based on workload and notes indexing has performance/operational costs. ([MongoDB](https://www.mongodb.com/docs/v8.0/indexes/?utm_source=chatgpt.com "Indexes - Database Manual v8.0 - MongoDB Docs"))

So don't create:

```javascript
email
name
age
city
phone
role
createdAt
updatedAt
...
```

just because each field _might_ be queried someday.

---

# 44. Index design process

For a production backend, your thinking should be:

```text
1. Identify important query
        ↓
2. Identify filter fields
        ↓
3. Identify sort fields
        ↓
4. Identify range predicates
        ↓
5. Design candidate index
        ↓
6. Run explain()
        ↓
7. Inspect executionStats
        ↓
8. Test with realistic data
        ↓
9. Keep/remove based on workload
```

That is much more useful than memorizing random index examples.

---

# 45. One mistake you should avoid

Never say:

> “This query is slow, so I'll add an index to every field in it.”

For:

```javascript
{
    location: "Pune",
    status: "open",
    category: "backend"
}
```

you might think:

```text
index location
index status
index category
```

But perhaps the actual workload is best served by:

```javascript
{
    location: 1,
    status: 1,
    category: 1
}
```

Or perhaps another ordering is better.

The right answer depends on the actual query shapes, selectivity, sorts, and workload.

That's why compound indexes and ESR matter.

---

# 46. Exercises

Just four.

## Exercise 1 — Basic

You frequently run:

```javascript
db.users.find({
    email: "riya@example.com"
})
```

What index would you consider?

### Answer

```javascript
db.users.createIndex({
    email: 1
})
```

If emails must be unique:

```javascript
db.users.createIndex(
    { email: 1 },
    { unique: true }
)
```

The second additionally enforces uniqueness. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.createindexes/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

# 47. Exercise 2 — Compound

You frequently run:

```javascript
db.jobs.find({
    location: "Pune",
    status: "open"
}).sort({
    createdAt: -1
})
```

What is a natural index candidate?

### Answer

```javascript
db.jobs.createIndex({
    location: 1,
    status: 1,
    createdAt: -1
})
```

Reason:

```text
location   → Equality
status     → Equality
createdAt  → Sort
```

That follows the general ESR guideline. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/?utm_source=chatgpt.com "The ESR (Equality, Sort, Range) Guideline - Database Manual - MongoDB Docs"))

---

# 48. Exercise 3 — Explain

You run:

```javascript
db.users
    .explain("executionStats")
    .find({
        email: "riya@example.com"
    })
```

and see:

```text
nReturned: 1
totalDocsExamined: 1,000,000
```

What does that tell you?

### Answer

MongoDB returned only one document, but examined about one million documents.

That strongly suggests the query is doing substantial unnecessary scanning. You should inspect the winning plan and consider whether an appropriate index is missing or whether some other query-planning issue exists. `explain("executionStats")` is specifically intended to expose these execution characteristics. ([MongoDB](https://www.mongodb.com/docs/manual/reference/explain-results/?utm_source=chatgpt.com "Explain Results - Database Manual - MongoDB Docs"))

---

# 49. Exercise 4 — Difficult conceptual

You have:

```javascript
db.jobs.createIndex({
    location: 1,
    status: 1,
    salary: -1
})
```

Which query is naturally aligned with the index prefix?

### A

```javascript
db.jobs.find({
    status: "open"
})
```

### B

```javascript
db.jobs.find({
    location: "Pune"
})
```

### C

```javascript
db.jobs.find({
    salary: { $gte: 800000 }
})
```

### D

```javascript
db.jobs.find({
    status: "open",
    salary: { $gte: 800000 }
})
```

### Answer

**B** is the clearest prefix-aligned query.

The index begins:

```text
location
```

so:

```javascript
{ location: "Pune" }
```

matches the first index key.

A query using only `status` does not match the index's prefix. MongoDB does have some cases where a compound index can still be used for non-prefix fields, but that does not make it equivalent to having the desired fields as the index prefix, and efficiency can differ substantially. ([MongoDB](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/?utm_source=chatgpt.com "Compound Indexes - Database Manual - MongoDB Docs"))

---

# 50. Final mental model

This is the picture I want you to keep:

```text
                    QUERY
                      │
                      ↓
                Query Planner
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
      Collection Scan          Index
       (COLLSCAN)                │
          │                      ↓
          │                   IXSCAN
          │                      │
          └───────────┬──────────┘
                      ↓
                  Documents
                      ↓
                    Result
```

And your job as a backend developer is to understand:

```text
Which queries matter?
        ↓
Which index supports them?
        ↓
Does the planner actually use it?
        ↓
Does explain() show efficient execution?
```

---

# 51. What you should know after Lesson 7

You don't need to memorize every MongoDB index type yet.

You should understand these extremely well:

```text
Index
  → speeds appropriate reads, at a cost

COLLSCAN
  → collection scan

IXSCAN
  → index scan

createIndex()
  → create an index

unique
  → enforce uniqueness

single-field index
  → one indexed field

compound index
  → multiple indexed fields

index prefix
  → beginning portion of compound index

multikey index
  → index over array data

explain("executionStats")
  → inspect actual query execution

totalDocsExamined
  → documents examined

totalKeysExamined
  → index entries examined

ESR
  → Equality → Sort → Range guideline
```

The most important habit is this:

> **Never assume an index improved a query. Measure it with `explain()` using realistic data.**

MongoDB's own documentation recommends `explain` for determining whether queries use indexes and how many keys/documents they examine. ([MongoDB](https://www.mongodb.com/docs/manual/tutorial/analyze-query-plan/?utm_source=chatgpt.com "Interpret Explain Plan Results - Database Manual - MongoDB Docs"))

### Next lesson: Schema Validation

We'll return to something you learned in Lesson 3:

> “MongoDB is flexible-schema, but that doesn't mean anything goes.”

We'll cover **JSON Schema validation, required fields, BSON types, enum-like restrictions, validation levels/actions, `$jsonSchema`, and how to enforce sane structure without giving up MongoDB's flexibility**. This will also connect directly to designing your ChatApp/JobForge database safely.