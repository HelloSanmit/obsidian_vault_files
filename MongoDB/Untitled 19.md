
# Gap Module 2 — Advanced Querying, Data Modeling Patterns & Update Operators

This module fills the missing pieces from **Lessons 2, 3, and 4**.

We’ll cover:

```text
Lesson 2
├── explicit $eq
├── positional projection $
├── $elemMatch in projection
├── $slice in projection
└── $nor exercise

Lesson 3
├── Bucket Pattern
├── Computed Pattern
├── Outlier Pattern
└── Schema Versioning Pattern

Lesson 4
├── $pullAll
├── $position
├── $sort with $push
├── $setOnInsert
└── $currentDate
```

The important thing is not just learning the syntax. Several of these concepts teach **different ways of thinking about MongoDB**.

---

# PART A — Lesson 2: Advanced Querying

# 1. `$eq`

You have already used equality queries like:

```js
db.users.find({
  age: 25
});
```

MongoDB interprets that as an equality condition.

You can also express it explicitly:

```js
db.users.find({
  age: {
    $eq: 25
  }
});
```

### What does `$eq` mean?

> `$eq` means **equals**.

So:

```js
{ age: 25 }
```

and:

```js
{ age: { $eq: 25 } }
```

express the same basic equality condition.

---

# 2. Why have `$eq` if `{ field: value }` already works?

Because explicit operators become useful when you're building more complex query expressions.

For example:

```js
{
  age: { $eq: 25 }
}
```

can sit conceptually alongside:

```js
{
  age: { $gt: 20 }
}
```

and:

```js
{
  age: { $in: [20, 25, 30] }
}
```

Think:

```text
direct equality
→ { age: 25 }

explicit equality
→ { age: { $eq: 25 } }
```

You don't need to prefer `$eq` for ordinary equality queries.

The important thing is recognizing it when you see it.

---

# 3. `$eq` inside `$expr`

Here's where explicit `$eq` becomes more interesting.

Suppose you want:

> Find jobs where minimum salary equals maximum salary.

You can use:

```js
db.jobs.find({
  $expr: {
    $eq: [
      "$salary.min",
      "$salary.max"
    ]
  }
});
```

Now `$eq` is comparing **two field expressions**.

That is different from:

```js
{
  salary: 50000
}
```

because here we're comparing:

```text
salary.min
against
salary.max
```

This is one reason explicit comparison expressions matter.

---

# 4. Positional projection `$`

You learned the positional `$` update operator:

```js
{
  "courses.$.score": 90
}
```

But there is another use of `$`:

> **Positional projection**

And this is where people often get confused.

Suppose a document is:

```js
{
  name: "Alex",
  courses: [
    { name: "Java", score: 80 },
    { name: "MongoDB", score: 95 },
    { name: "React", score: 88 }
  ]
}
```

Suppose you query:

```js
db.students.find(
  {
    "courses.name": "MongoDB"
  },
  {
    "courses.$": 1
  }
);
```

The projection:

```js
{
  "courses.$": 1
}
```

means roughly:

> Return the array element that matched the query condition.

Instead of receiving all three courses, you can get the matching one.

Conceptually:

```js
{
  name: "Alex",
  courses: [
    {
      name: "MongoDB",
      score: 95
    }
  ]
}
```

---

# 5. Very important: projection `$` vs update `$`

These look similar:

```text
courses.$
```

but they are not the same concept.

### In an update

```js
"courses.$.score"
```

means:

> Update the matching array element.

### In projection

```js
"courses.$": 1
```

means:

> Return the matching array element.

So:

```text
update $
=
which element to modify

projection $
=
which element to return
```

Don't merge them mentally.

---

# 6. Another important limitation

Positional projection is intended to return the relevant matching array element, rather than behaving like:

> “Return every array element satisfying every possible condition.”

When you need more expressive projection behavior, `$elemMatch` can be useful.

---

# 7. `$elemMatch` in a projection

You've already learned `$elemMatch` for querying arrays.

For example:

```js
db.students.find({
  courses: {
    $elemMatch: {
      name: "MongoDB",
      score: { $gte: 90 }
    }
  }
});
```

That means:

> Find students having an array element where both conditions are true on the same element.

But `$elemMatch` can also appear in the **projection**.

Example:

```js
db.students.find(
  {
    name: "Alex"
  },
  {
    courses: {
      $elemMatch: {
        score: { $gte: 90 }
      }
    }
  }
);
```

This means:

> Return only the first array element matching the projection's `$elemMatch` condition.

Conceptually:

```text
full courses
   ↓
projection $elemMatch
   ↓
matching course
```

---

# 8. Query `$elemMatch` vs projection `$elemMatch`

This is worth memorizing conceptually.

### Query

```js
{
  courses: {
    $elemMatch: ...
  }
}
```

asks:

> **Should this document be returned?**

### Projection

```js
{
  courses: {
    $elemMatch: ...
  }
}
```

asks:

> **Which array element should be included in the returned document?**

So:

```text
query $elemMatch
=
document selection

projection $elemMatch
=
result shaping
```

Same operator name, different stage of processing.

---

# 9. Why does this matter?

Imagine:

```js
{
  name: "Alex",
  exams: [
    { subject: "Math", score: 88 },
    { subject: "MongoDB", score: 97 },
    { subject: "React", score: 81 }
  ]
}
```

You want:

> Return the student, but only show one exam with a score of at least 90.

Projection can help:

```js
db.students.find(
  { name: "Alex" },
  {
    exams: {
      $elemMatch: {
        score: { $gte: 90 }
      }
    }
  }
);
```

This is a **data-shaping** operation.

---

# 10. `$slice` in projection

Now imagine:

```js
{
  name: "Alex",
  posts: [
    "Post 1",
    "Post 2",
    "Post 3",
    "Post 4",
    "Post 5"
  ]
}
```

Suppose your API only wants the first two posts.

You can use:

```js
db.users.find(
  { name: "Alex" },
  {
    posts: {
      $slice: 2
    }
  }
);
```

Result conceptually:

```js
{
  name: "Alex",
  posts: [
    "Post 1",
    "Post 2"
  ]
}
```

### What does `$slice` do?

> Return only a portion of an array.

---

# 11. Negative `$slice`

You can also do:

```js
{
  posts: {
    $slice: -2
  }
}
```

That means:

> Return the last two array elements.

So:

```text
$slice: 2
→ first 2

$slice: -2
→ last 2
```

---

# 12. Skip + limit form of `$slice`

You can also use:

```js
{
  posts: {
    $slice: [2, 2]
  }
}
```

Conceptually:

```text
skip 2
take 2
```

So:

```text
Post 1
Post 2
Post 3
Post 4
Post 5

$slice: [2, 2]

→ Post 3
→ Post 4
```

This is useful when an array is reasonably bounded and you only need a section of it.

---

# 13. `$slice` projection vs `$slice` update modifier

Again, the same name appears in two contexts.

### Projection

```js
{
  posts: {
    $slice: 2
  }
}
```

means:

> Return only part of the array.

### Update modifier

```js
{
  $push: {
    posts: {
      $each: newPosts,
      $slice: -10
    }
  }
}
```

means:

> After pushing, keep only a certain portion of the array.

So:

```text
projection $slice
=
control returned data

update $slice
=
control stored array contents
```

---

# 14. `$nor`

You learned that `$nor` means:

> None of the supplied conditions should be true.

For example:

```js
db.jobs.find({
  $nor: [
    { status: "CLOSED" },
    { remote: false }
  ]
});
```

This means:

> Find documents where it is **not true** that status is CLOSED, and it is **not true** that remote is false.

In simpler terms, you're excluding documents satisfying either listed condition.

---

# 15. `$nor` example

Suppose:

```text
Job A:
status = OPEN
remote = true

Job B:
status = CLOSED
remote = true

Job C:
status = OPEN
remote = false
```

Query:

```js
{
  $nor: [
    { status: "CLOSED" },
    { remote: false }
  ]
}
```

Only Job A qualifies.

Why?

```text
Job A
not CLOSED ✓
not remote:false ✓

Job B
CLOSED ✗

Job C
remote:false ✗
```

---

# 16. `$nor` is not the same as `$not`

This is another common confusion.

### `$not`

Negates a particular condition/operator.

Example:

```js
{
  age: {
    $not: {
      $gt: 30
    }
  }
}
```

roughly means:

> Age is not greater than 30.

### `$nor`

Combines multiple conditions:

```js
{
  $nor: [
    conditionA,
    conditionB
  ]
}
```

means:

> None of these conditions should match.

Mental model:

```text
$not
=
negate one expression/operator

$nor
=
NOT (conditionA OR conditionB)
```

---

# 17. Lesson 2 is now gap-complete

We filled:

```text
✓ explicit $eq
✓ positional projection $
✓ projection $elemMatch
✓ projection $slice
✓ $nor practical example
```

Now we move into **Lesson 3**, which contains a more interesting set of missing concepts.

---

# PART B — Lesson 3: Advanced Data Modeling Patterns

Your original Lesson 3 taught:

```text
embed vs reference
cardinality
growth
one-to-one
one-to-few
one-to-many
one-to-squillions
many-to-many
subset pattern
```

That's already strong.

We're now adding four design patterns that help when simple embed/reference decisions aren't enough:

```text
Bucket Pattern
Computed Pattern
Outlier Pattern
Schema Versioning Pattern
```

---

# 18. Bucket Pattern

Imagine an application collecting:

```text
user activity
```

every minute:

```text
10:00
10:01
10:02
10:03
...
```

You might store:

```text
one document per event
```

which could eventually produce enormous numbers of documents.

The **Bucket Pattern** groups related records into buckets.

Instead of:

```text
10:00 → document
10:01 → document
10:02 → document
10:03 → document
```

you might have:

```js
{
  sensorId: "S100",
  date: "2026-09-20",
  measurements: [
    { time: "10:00", value: 20 },
    { time: "10:01", value: 21 },
    { time: "10:02", value: 19 }
  ]
}
```

Conceptually:

```text
many small documents/events
        ↓
group into bounded buckets
```

---

# 19. Why use buckets?

Imagine:

```text
1 measurement
=
1 MongoDB document
```

for:

```text
100 million measurements/day
```

That's a massive document count.

Bucketing can reduce the number of documents.

For example:

```text
1 bucket
=
1 hour of measurements
```

Then:

```text
24 buckets
≈
one day per sensor
```

rather than:

```text
1440 documents
=
one minute per day
```

for a minute-level workload.

---

# 20. The most important rule for buckets

Buckets should be **bounded**.

Bad:

```text
one bucket
→ forever
```

That creates the same unbounded-array problem we already learned.

Better:

```text
one bucket
→ 1 hour

or

one bucket
→ 1 day

or

one bucket
→ 1000 events
```

The exact threshold comes from the workload.

Think:

> **Bucket Pattern = controlled grouping, not infinite grouping.**

---

# 21. Bucket Pattern example: Chat analytics

Suppose you don't store raw messages in buckets.

That might be bad because message retrieval often needs:

```text
exact message
sender
conversation
pagination
individual updates
```

But analytics could be bucketed.

For example:

```js
{
  conversationId: ObjectId("..."),
  hour: ISODate("..."),

  stats: {
    messageCount: 152,
    imageCount: 13,
    averageResponseSeconds: 32
  }
}
```

Now you've separated:

```text
operational message data
```

from:

```text
analytics bucket
```

That's often a good design.

---

# 22. Computed Pattern

Suppose your JobForge dashboard repeatedly asks:

> How many applications does this job have?

You could calculate:

```text
count Application documents
```

every single time.

For a huge system, repeated calculation can be expensive.

The **Computed Pattern** says:

> Precompute data that is expensive to calculate repeatedly but changes in a controlled way.

So:

```js
{
  title: "Backend Engineer",

  applicationCount: 1250
}
```

instead of recalculating:

```text
count(Application where jobId = X)
```

every time.

---

# 23. The tradeoff

Now you have:

```text
source data
=
Applications
```

and:

```text
derived data
=
applicationCount
```

Benefits:

```text
fast reads
simple dashboards
less repeated aggregation
```

Cost:

```text
more complicated writes
possible temporary inconsistency
repair/recalculation requirements
```

This is why we discussed:

> **Source of truth vs derived data.**

---

# 24. Computed Pattern example

Suppose:

```text
Application count = 1000
```

Then another application arrives.

You can:

```text
insert Application
+
increment applicationCount
```

with an atomic update:

```js
$inc: {
  applicationCount: 1
}
```

Or perhaps update asynchronously via an event system if eventual consistency is acceptable.

Again:

```text
exactly when should derived data update?
```

is a product/system requirement.

---

# 25. Computed Pattern can be more complicated than a counter

Suppose you repeatedly calculate:

```text
average application processing time
```

You could maintain summary information such as:

```js
{
  completedCount: 2000,
  totalProcessingSeconds: 500000
}
```

Then:

```text
average
=
totalProcessingSeconds / completedCount
```

Instead of repeatedly scanning every historical application.

That's a more advanced use of computed data.

---

# 26. Computed Pattern isn't “store everything”

Don't start adding:

```text
job.averageSalary
job.averageAge
job.averageExperience
job.numberOfApplicants
job.numberOfRejectedApplicants
job.numberOfInterviews
job.numberOfNotifications
...
```

to every document.

Every derived field creates synchronization complexity.

Only precompute values where the read benefit justifies the maintenance cost.

---

# 27. Outlier Pattern

Now consider:

```text
Company A
→ 50 jobs
```

versus:

```text
Company B
→ 50 million jobs
```

Most companies are normal.

One company is an **outlier**.

Suppose you've designed some embedded structure assuming:

```text
small number of related items
```

but one document unexpectedly receives enormous growth.

The **Outlier Pattern** helps handle exceptional large cases separately.

---

# 28. Example

Suppose a social platform stores:

```js
{
  userId: "...",
  followers: [...]
}
```

For most users:

```text
100 followers
```

Fine.

But one celebrity has:

```text
50 million followers
```

Now the embedded array becomes a problem.

The Outlier Pattern suggests keeping normal data in the normal structure while handling exceptionally large values separately.

Conceptually:

```text
normal user
→ followers embedded

outlier user
→ followers split/secondary structure
```

---

# 29. Why not design everything for the worst case?

You could say:

> “Let's always use a separate collection.”

But that might make normal reads unnecessarily complicated.

The Outlier Pattern says:

> Optimize for the normal case while designing a safe escape route for exceptional cases.

This is a very useful systems-design principle.

---

# 30. JobForge Outlier example

Imagine most companies have:

```text
< 10,000 applications
```

but one global employer has:

```text
20 million applications
```

If your design includes an embedded summary structure that assumes small cardinality, that one employer may cause problems.

You can detect:

```text
company is outlier
```

and route unusually large datasets into a dedicated structure/query strategy.

---

# 31. The outlier pattern is really about data distribution

The important question isn't:

> “Can a document be large?”

It's:

> **“What happens when one entity behaves radically differently from the average?”**

This is especially relevant for:

```text
social media
messaging
analytics
logs
events
large employers
popular products
```

---

# 32. Schema Versioning Pattern

Now suppose your application evolves.

Version 1:

```js
{
  location: "Pune"
}
```

Later you need:

```js
{
  location: {
    city: "Pune",
    country: "India",
    remote: true
  }
}
```

If you have millions of old documents, you can't necessarily transform every document instantly.

Add:

```js
{
  schemaVersion: 2
}
```

Now the application can understand which representation it is dealing with.

---

# 33. Why schema versioning matters

Imagine deployment:

```text
old application
+
new application
```

during a rolling deployment.

Some records may still look like:

```text
version 1
```

while new records are:

```text
version 2
```

Your application can temporarily support both.

Conceptually:

```text
read
 ↓
schemaVersion?
 ├── 1 → old interpretation
 └── 2 → new interpretation
```

---

# 34. Safer migration strategy

Instead of:

```text
change schema
↓
immediately convert all 50 million documents
```

you can do:

```text
Step 1
new code can read v1 + v2

Step 2
new writes use v2

Step 3
background migration converts v1 → v2

Step 4
verify

Step 5
remove old-format support
```

This is called an **expand-and-contract style migration strategy**.

That's a much more production-friendly design.

---

# 35. Schema Versioning + validation

Suppose MongoDB validation initially expects version 1.

Then you introduce version 2.

You need to think carefully about:

```text
old documents
new writes
migration
validator
```

You shouldn't accidentally create this:

```text
validator requires v2
+
millions of documents still v1
+
old code still running
```

and then discover your deployment cannot operate correctly.

---

# 36. Schema Versioning isn't only for MongoDB structure

It can also be useful for:

```text
API payloads
event formats
cached data
message formats
```

The broader idea is:

> **Data formats evolve. Version them when compatibility matters.**

---

# 37. Combining the four patterns

These patterns can work together.

Imagine:

```text
Analytics system
```

You could have:

```text
Bucket Pattern
→ group time-based raw measurements

Computed Pattern
→ maintain fast summary statistics

Outlier Pattern
→ isolate unusually large entities

Schema Versioning
→ safely evolve stored document structure
```

That is advanced database design.

---

# 38. Lesson 3 gap now complete

We've added:

```text
✓ Bucket Pattern
✓ Computed Pattern
✓ Outlier Pattern
✓ Schema Versioning Pattern
```

Now let's finish the update-related gaps from Lesson 4.

---

# PART C — Lesson 4: Advanced Updates

# 39. `$pullAll`

You've already used:

```js
$pull
```

Example:

```js
db.students.updateOne(
  { _id: 1 },
  {
    $pull: {
      skills: "Java"
    }
  }
);
```

This removes matching values from an array.

But `$pullAll` has a slightly different purpose.

---

# 40. `$pullAll` meaning

Suppose:

```js
{
  skills: [
    "Java",
    "Python",
    "Go",
    "Rust",
    "Python"
  ]
}
```

You want to remove:

```text
Python
Rust
```

You can use:

```js
db.students.updateOne(
  { _id: 1 },
  {
    $pullAll: {
      skills: [
        "Python",
        "Rust"
      ]
    }
  }
);
```

Result:

```js
{
  skills: [
    "Java",
    "Go"
  ]
}
```

---

# 41. `$pull` vs `$pullAll`

### `$pull`

Remove array elements matching a condition/value.

```js
$pull: {
  skills: "Python"
}
```

### `$pullAll`

Remove all occurrences of several exact values.

```js
$pullAll: {
  skills: [
    "Python",
    "Rust"
  ]
}
```

Mental model:

```text
$pull
=
remove elements matching condition

$pullAll
=
remove all occurrences of listed values
```

---

# 42. `$position`

Now we're working with `$push`.

Suppose:

```js
{
  scores: [70, 80, 90]
}
```

You do:

```js
{
  $push: {
    scores: 100
  }
}
```

Result:

```text
[70, 80, 90, 100]
```

Normally, `$push` adds to the end.

But you can control the insertion position using `$position`.

---

# 43. Example `$position`

```js
db.students.updateOne(
  { _id: 1 },
  {
    $push: {
      scores: {
        $each: [95],
        $position: 1
      }
    }
  }
);
```

Before:

```text
[70, 80, 90]
```

After:

```text
[70, 95, 80, 90]
```

Because:

```text
position 1
```

means insert at index 1.

Remember JavaScript/Python-style zero-based indexing:

```text
index 0 → first
index 1 → second
index 2 → third
```

---

# 44. Why `$each` appears here

You might wonder:

> Why can't I just use `$position` directly?

Because `$position` is a modifier used within the `$push` modifier structure, along with `$each`.

Conceptually:

```js
$push: {
  array: {
    $each: [...],
    $position: ...
  }
}
```

That structure gives you additional control over how the pushed elements are inserted.

---

# 45. `$sort` modifier with `$push`

Here's a very useful advanced pattern.

Suppose:

```js
{
  leaderboard: [
    { name: "A", score: 70 },
    { name: "B", score: 90 },
    { name: "C", score: 80 }
  ]
}
```

You want to add:

```js
{ name: "D", score: 95 }
```

and keep the array sorted by score.

You can use:

```js
db.game.updateOne(
  { _id: 1 },
  {
    $push: {
      leaderboard: {
        $each: [
          { name: "D", score: 95 }
        ],
        $sort: {
          score: -1
        }
      }
    }
  }
);
```

Result conceptually:

```js
[
  { name: "D", score: 95 },
  { name: "B", score: 90 },
  { name: "C", score: 80 },
  { name: "A", score: 70 }
]
```

---

# 46. Why is `$sort` useful?

Suppose your array represents:

```text
top players
top products
recent actions
priority items
```

You can maintain a bounded sorted array without:

```text
read document
↓
sort in application
↓
modify
↓
save
```

Instead, MongoDB can perform the update transformation.

---

# 47. `$sort` doesn't mean “sort the whole collection”

Important:

```js
$push: {
  leaderboard: {
    $each: [...],
    $sort: {...}
  }
}
```

sorts the **array being modified**.

It does not sort your MongoDB collection.

Think:

```text
$sort in aggregation
=
sort pipeline documents

$sort inside $push
=
sort the array after pushing
```

Different context.

---

# 48. `$slice` can be combined with `$sort`

Here's where this gets powerful.

Suppose you only want the:

> Top 10 players.

You can do:

```js
db.game.updateOne(
  { _id: 1 },
  {
    $push: {
      leaderboard: {
        $each: [
          { name: "D", score: 95 }
        ],
        $sort: {
          score: -1
        },
        $slice: 10
      }
    }
  }
);
```

Conceptually:

```text
push
 ↓
sort descending
 ↓
keep first 10
```

Now the array stays bounded.

That's a very nice MongoDB pattern.

---

# 49. Why bounded arrays are so powerful

This:

```text
top 10 players
```

is a **bounded dataset**.

So embedding is appropriate.

You don't need:

```text
millions of leaderboard entries
```

inside the same document.

Instead:

```text
document
 └── top 10
```

is safe and predictable.

This connects Lesson 3 directly to Lesson 4.

---

# 50. `$setOnInsert`

Now we move into upsert.

Suppose you do:

```js
db.settings.updateOne(
  {
    userId: 123
  },
  {
    $set: {
      theme: "dark"
    }
  },
  {
    upsert: true
  }
);
```

If the document doesn't exist, MongoDB creates it.

But perhaps you want:

```text
theme
→ change every time

createdAt
→ only set when first created
```

That's where `$setOnInsert` comes in.

---

# 51. Example `$setOnInsert`

```js
db.settings.updateOne(
  {
    userId: 123
  },
  {
    $set: {
      theme: "dark"
    },

    $setOnInsert: {
      createdAt: new Date()
    }
  },
  {
    upsert: true
  }
);
```

### Existing document

```text
$set
✓

$setOnInsert
✗
```

### New upserted document

```text
$set
✓

$setOnInsert
✓
```

This is extremely useful.

---

# 52. Why is it better than `$set: { createdAt: ... }`?

Imagine the document already exists:

```text
createdAt = 2025-01-01
```

If every update did:

```js
$set: {
  createdAt: new Date()
}
```

you would accidentally change the creation timestamp.

But:

```js
$setOnInsert: {
  createdAt: new Date()
}
```

preserves it after the initial insert.

That's exactly what we want.

---

# 53. Upsert lifecycle

Think:

```text
Filter:
userId = 123

Does document exist?
        │
   ┌────┴────┐
   │         │
  YES        NO
   │         │
 $set      $set
            +
       $setOnInsert
```

This pattern appears constantly in production applications.

---

# 54. `$currentDate`

Suppose you want MongoDB itself to set:

```text
updatedAt
```

to the current date.

You can use:

```js
db.users.updateOne(
  { _id: 1 },
  {
    $currentDate: {
      updatedAt: true
    }
  }
);
```

### Meaning

> Set this field to the current database date/time.

---

# 55. Why use `$currentDate`?

Suppose you have:

```text
Application server
MongoDB server
```

and you want the database operation itself to determine the update timestamp.

Then:

```js
$currentDate
```

makes MongoDB the source for that timestamp.

This can be useful in certain consistency/design scenarios.

---

# 56. `$currentDate` with timestamp

MongoDB can also produce a BSON timestamp in appropriate contexts.

But for ordinary application fields like:

```text
createdAt
updatedAt
lastSeenAt
```

a BSON Date is usually what you conceptually want.

So the common form you'll encounter is:

```js
$currentDate: {
  updatedAt: true
}
```

---

# 57. `$currentDate` vs `new Date()`

### Application-generated

```js
{
  $set: {
    updatedAt: new Date()
  }
}
```

Node creates the date.

### Database-generated

```js
{
  $currentDate: {
    updatedAt: true
  }
}
```

MongoDB generates the current date as part of the update.

Neither is universally “better.”

The question is:

> **Which system should be authoritative for this timestamp?**

---

# 58. `$setOnInsert` + `$currentDate`

You can combine these concepts.

For example:

```js
db.settings.updateOne(
  { userId: 123 },

  {
    $set: {
      theme: "dark"
    },

    $setOnInsert: {
      createdAt: new Date()
    },

    $currentDate: {
      updatedAt: true
    }
  },

  {
    upsert: true
  }
);
```

Conceptually:

```text
first creation
→ createdAt + updatedAt

later update
→ only updatedAt changes
```

This is a very useful pattern.

---

# 59. One subtle issue: time semantics

Suppose you need:

```text
updatedAt
```

to represent:

> “When MongoDB processed the update.”

Then `$currentDate` is conceptually attractive.

Suppose you need:

> “Timestamp generated by the application event.”

Then application-side time may be appropriate.

Again, architecture follows semantics.

---

# 60. Combining `$push`, `$sort`, `$slice`

Let's create a realistic example.

Suppose:

```js
{
  name: "Game",
  topScores: [
    { player: "A", score: 900 },
    { player: "B", score: 850 },
    { player: "C", score: 800 }
  ]
}
```

New player:

```js
{
  player: "D",
  score: 920
}
```

We want:

```text
add
+
sort descending
+
keep top 3
```

Use:

```js
db.games.updateOne(
  { _id: 1 },
  {
    $push: {
      topScores: {
        $each: [
          {
            player: "D",
            score: 920
          }
        ],
        $sort: {
          score: -1
        },
        $slice: 3
      }
    }
  }
);
```

Result:

```js
[
  { player: "D", score: 920 },
  { player: "A", score: 900 },
  { player: "B", score: 850 }
]
```

Notice the array never grows beyond three elements.

That's excellent MongoDB modeling.

---

# 61. This single example connects three lessons

### Lesson 3

Bounded arrays are safe to embed.

### Lesson 4

`$push + $sort + $slice`.

### Performance

No need to retrieve and sort the array in Node.

That's exactly how the separate lessons should eventually connect.

---

# 62. Another useful update pattern

Suppose a user's recent searches should contain only the latest 10 searches.

You can do something conceptually like:

```js
db.users.updateOne(
  { _id: userId },
  {
    $push: {
      recentSearches: {
        $each: ["spring boot mongodb"],
        $position: 0,
        $slice: 10
      }
    }
  }
);
```

Let's unpack it:

```text
$each
→ add the new item

$position: 0
→ put it at the front

$slice: 10
→ retain only first 10
```

So the array stays:

```text
bounded
+
newest first
```

That's a very useful real-world pattern.

---

# 63. `$pullAll` with duplicates

Suppose:

```js
{
  tags: [
    "Java",
    "MongoDB",
    "Java",
    "Spring"
  ]
}
```

Then:

```js
{
  $pullAll: {
    tags: ["Java"]
  }
}
```

removes all matching occurrences.

Result:

```js
{
  tags: [
    "MongoDB",
    "Spring"
  ]
}
```

That's one reason `$pullAll` exists.

---

# 64. Choosing `$pull` vs `$pullAll`

Suppose you want:

> Remove every tag equal to `"Java"`.

Use:

```text
$pull
```

is natural.

Suppose you want:

> Remove `"Java"`, `"Python"`, and `"Go"`.

Then:

```text
$pullAll
```

expresses the intent directly.

So it's partly about readability too.

---

# 65. One important limitation

Don't assume these array modifiers make enormous arrays safe.

For example:

```text
$push + $sort + $slice
```

is excellent for:

```text
top 10
recent 20
latest 50
```

But not:

```text
top 10 million
```

because the document still has to obey MongoDB's document-size constraints and your workload still has to be efficient.

The pattern works because the **resulting array is intentionally bounded**.

---

# 66. New methods/operators from this module

Here's your revised mental dictionary.

### Query

```text
$eq
```

> Equals.

### Projection

```text
courses.$
```

> Return the matching array element.

```text
$elemMatch
```

> Choose an array element satisfying a condition in projection.

```text
$slice
```

> Return only part of an array.

### Data modeling

```text
Bucket Pattern
```

> Group many related records into bounded buckets.

```text
Computed Pattern
```

> Precompute expensive-to-calculate values.

```text
Outlier Pattern
```

> Handle unusually large entities separately.

```text
Schema Versioning
```

> Allow data formats to evolve safely.

### Updates

```text
$pullAll
```

> Remove all occurrences of several specified values.

```text
$position
```

> Control where `$push` inserts new elements.

```text
$sort
```

> Sort an array after `$push`.

```text
$setOnInsert
```

> Apply a field only when an upsert creates a new document.

```text
$currentDate
```

> Set a field to the current database date/time.

---

# 67. What I want you to notice

There is a deeper pattern behind these additions.

We started with:

```text
“Which operator does what?”
```

But now you're learning:

```text
How do I shape returned data?
How do I keep arrays bounded?
How do I avoid unnecessary application-side processing?
How do I precompute expensive information?
How do I handle exceptional data growth?
How do I evolve schemas safely?
How do I make upserts preserve creation metadata?
```

That's why these additions are worth learning.

---

# 68. Mini challenge

Suppose you have:

```js
{
  userId: 101,

  recentSearches: [
    "Java",
    "MongoDB",
    "React",
    "Docker"
  ]
}
```

Requirements:

> A new search should be inserted at the front, and only the latest 5 searches should remain.

The solution is conceptually:

```js
$push: {
  recentSearches: {
    $each: [newSearch],
    $position: 0,
    $slice: 5
  }
}
```

You should now be able to explain every part.

---

# 69. Mini challenge

A leaderboard must:

```text
1. add new score
2. sort highest first
3. retain top 10
```

Answer:

```js
$push: {
  leaderboard: {
    $each: [newScore],
    $sort: {
      score: -1
    },
    $slice: 10
  }
}
```

Again:

```text
push
→ sort
→ bound
```

---

# 70. Mini challenge

You maintain user settings.

Requirement:

```text
theme can change
createdAt should only exist/set during first creation
updatedAt changes every time
```

A good conceptual update:

```js
{
  $set: {
    theme: "dark"
  },

  $setOnInsert: {
    createdAt: new Date()
  },

  $currentDate: {
    updatedAt: true
  }
}
```

with:

```js
{
  upsert: true
}
```

This is exactly the kind of small but powerful MongoDB pattern that appears in real applications.

---

# 71. The lesson-order map is now improving

Your course now effectively has:

```text
Lesson 1
CRUD + counting + distinct + compound operations + cursors
        ↓
Lesson 2
Queries + advanced projections
        ↓
Lesson 3
Data modeling + advanced modeling patterns
        ↓
Lesson 4
Updates + advanced array/upsert operations
```

That makes the progression much more complete.

---

# 72. Current gap status

After today's module:

```text
Lesson 1 gaps
✓ filled

Lesson 2 gaps
✓ filled

Lesson 3 gaps
✓ filled

Lesson 4 gaps
✓ filled
```

So we **do not need to revisit those lessons again** unless we encounter a concept later that naturally builds on them.

The next gap module will cover **Lessons 5–7 together**:

```text
Advanced Aggregation
├── $count
├── $replaceRoot / $replaceWith
├── $bucket / $bucketAuto
├── $sortByCount
├── $sample
├── $out / $merge
├── string/date operators
├── $let
├── $mergeObjects
├── $range
├── $objectToArray / $arrayToObject
└── $setWindowFields

Advanced Indexing
├── TTL
├── wildcard
├── text/search
├── $hint
├── allPlansExecution
├── hideIndex
└── index intersection
```

That will be the **largest gap module**, because aggregation and indexing are two areas where the missing concepts actually add substantial new capabilities rather than just a few extra methods.