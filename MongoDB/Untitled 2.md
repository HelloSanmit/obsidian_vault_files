
# MongoDB — Lesson 2

## Querying MongoDB Properly

Lesson 1 gave you the basic mental model and CRUD.

Now we're going to make querying much stronger.

The goal isn't to memorize:

```js
$gt
$lt
$in
$or
```

The goal is to look at a requirement like:

> "Find users from Pune who are at least 21, have JavaScript or Java in their skills, but not Python."

and be able to construct the query yourself.

---

# 1. MongoDB queries are built from conditions

Suppose we have:

```js
db.students.find({
    age: { $gte: 21 }
})
```

Read this as:

> Find documents where `age >= 21`.

A query generally looks like:

```js
db.collection.find({
    field: condition
})
```

Examples:

```js
db.students.find({
    age: 22
})
```

```js
db.students.find({
    age: { $gt: 20 }
})
```

```js
db.students.find({
    city: "Pune"
})
```

---

# 2. Multiple conditions = implicit AND

Suppose:

```js
db.students.find({
    age: { $gte: 21 },
    city: "Pune"
})
```

MongoDB interprets this as:

```text
age >= 21
AND
city == Pune
```

A document must satisfy **both**.

For example:

```json
{
    "name": "Rahul",
    "age": 22,
    "city": "Pune"
}
```

matches.

But:

```json
{
    "name": "Amit",
    "age": 19,
    "city": "Pune"
}
```

doesn't match because age fails.

And:

```json
{
    "name": "Priya",
    "age": 22,
    "city": "Mumbai"
}
```

doesn't match because city fails.

---

# 3. Explicit `$and`

You can also write:

```js
db.students.find({
    $and: [
        { age: { $gte: 21 } },
        { city: "Pune" }
    ]
})
```

This means exactly the same thing in this case.

So why do we need `$and`?

Because sometimes you need to express multiple conditions involving the **same field**.

For example:

```js
db.students.find({
    $and: [
        { age: { $gte: 20 } },
        { age: { $lte: 25 } }
    ]
})
```

This means:

```text
20 <= age <= 25
```

However, MongoDB lets you write this more naturally:

```js
db.students.find({
    age: {
        $gte: 20,
        $lte: 25
    }
})
```

Prefer the simpler version when possible.

---

# 4. `$or`

Now suppose:

> Find students who live in Pune OR Mumbai.

```js
db.students.find({
    $or: [
        { city: "Pune" },
        { city: "Mumbai" }
    ]
})
```

A document only needs to satisfy **one** condition.

Another example:

> Find students who are either older than 25 OR enrolled in MCA.

```js
db.students.find({
    $or: [
        { age: { $gt: 25 } },
        { course: "MCA" }
    ]
})
```

---

# 5. AND + OR together

This is where queries start becoming interesting.

Requirement:

> Find BCA students from Pune OR Mumbai.

You could write:

```js
db.students.find({
    course: "BCA",
    $or: [
        { city: "Pune" },
        { city: "Mumbai" }
    ]
})
```

Logically:

```text
course == BCA
AND
(city == Pune OR city == Mumbai)
```

This is a very common pattern.

---

# 6. `$in`

Remember `$in` from Lesson 1.

Instead of:

```js
db.students.find({
    $or: [
        { city: "Pune" },
        { city: "Mumbai" }
    ]
})
```

you can usually write:

```js
db.students.find({
    city: {
        $in: ["Pune", "Mumbai"]
    }
})
```

This means:

> `city` equals one of these values.

For simple equality alternatives, `$in` is generally cleaner.

---

# 7. `$nin`

`$nin` means:

> not in this list

Example:

```js
db.students.find({
    city: {
        $nin: ["Pune", "Mumbai"]
    }
})
```

This finds students whose city isn't Pune or Mumbai.

---

# 8. `$ne`

`$ne` means:

> not equal

```js
db.students.find({
    course: {
        $ne: "BCA"
    }
})
```

This finds documents where `course` isn't `"BCA"`.

There is an important subtlety here:

MongoDB's handling of missing fields can make `$ne` behave differently from what beginners intuitively expect.

For example, a document without a `course` field can also satisfy a `$ne` query.

That's why later we'll carefully distinguish:

```text
field doesn't equal X
```

from:

```text
field exists AND doesn't equal X
```

---

# 9. `$not`

`$not` negates a condition.

For example:

```js
db.students.find({
    age: {
        $not: {
            $gt: 25
        }
    }
})
```

Conceptually:

> age is NOT greater than 25.

But don't reach for `$not` automatically.

Often a more direct operator is clearer.

For example:

```js
age: { $lte: 25 }
```

is easier to understand than:

```js
age: { $not: { $gt: 25 } }
```

---

# 10. Logical operators summary

You should know these four:

```text
$and
$or
$nor
$not
```

### `$and`

Every condition must be true.

### `$or`

At least one condition must be true.

### `$nor`

None of the conditions should be true.

Example:

```js
db.students.find({
    $nor: [
        { city: "Pune" },
        { age: { $lt: 20 } }
    ]
})
```

Conceptually:

```text
NOT(city == Pune)
AND
NOT(age < 20)
```

### `$not`

Negates a condition/operator expression.

---

# 11. Arrays

MongoDB becomes particularly interesting when documents contain arrays.

Consider:

```json
{
    "name": "Rahul",
    "skills": [
        "Java",
        "JavaScript",
        "MongoDB"
    ]
}
```

To find users whose skills contain MongoDB:

```js
db.users.find({
    skills: "MongoDB"
})
```

MongoDB understands that `skills` is an array.

---

# 12. `$in` with arrays

Suppose:

```json
{
    "skills": [
        "Java",
        "JavaScript",
        "MongoDB"
    ]
}
```

You want users who know **Java OR Python**.

You can write:

```js
db.users.find({
    skills: {
        $in: ["Java", "Python"]
    }
})
```

A document matches if the array contains at least one of those values.

---

# 13. `$all`

Now suppose the requirement is:

> Find users who know BOTH Java AND MongoDB.

Use:

```js
db.users.find({
    skills: {
        $all: ["Java", "MongoDB"]
    }
})
```

This is different from `$in`.

### `$in`

```text
at least one
```

### `$all`

```text
every specified value
```

Example:

```text
skills = ["Java", "MongoDB", "Docker"]
```

Query:

```js
$in: ["Java", "Python"]
```

✅ matches because Java exists.

Query:

```js
$all: ["Java", "Python"]
```

❌ doesn't match because Python isn't present.

---

# 14. Arrays of objects

This is where `$elemMatch` becomes important.

Suppose:

```json
{
    "name": "Rahul",
    "projects": [
        {
            "name": "ChatApp",
            "language": "JavaScript",
            "stars": 100
        },
        {
            "name": "JobForge",
            "language": "Java",
            "stars": 50
        }
    ]
}
```

Suppose we want:

> A project whose language is JavaScript AND stars are greater than 80.

We should use:

```js
db.users.find({
    projects: {
        $elemMatch: {
            language: "JavaScript",
            stars: { $gt: 80 }
        }
    }
})
```

This means:

> There exists **one array element** satisfying both conditions.

This distinction is extremely important.

---

# 15. Why `$elemMatch` matters

Consider:

```json
{
    "projects": [
        {
            "language": "JavaScript",
            "stars": 20
        },
        {
            "language": "Java",
            "stars": 100
        }
    ]
}
```

Now consider a query involving:

```js
{
    "projects.language": "JavaScript",
    "projects.stars": { $gt: 80 }
}
```

MongoDB can potentially satisfy those conditions using **different array elements**.

The document has:

```text
JavaScript → yes
stars > 80 → yes
```

but no **single project** has both properties.

`$elemMatch` expresses the actual requirement:

```text
same array element must satisfy both
```

This is one of the most important array-query concepts.

---

# 16. Nested documents

We saw:

```json
{
    "name": "Rahul",
    "address": {
        "city": "Pune",
        "country": "India"
    }
}
```

Query:

```js
db.users.find({
    "address.city": "Pune"
})
```

This is called **dot notation**.

You can go deeper:

```json
{
    "address": {
        "location": {
            "city": "Pune"
        }
    }
}
```

Query:

```js
db.users.find({
    "address.location.city": "Pune"
})
```

---

# 17. Projection

Suppose:

```json
{
    "_id": ObjectId("..."),
    "name": "Rahul",
    "email": "rahul@gmail.com",
    "age": 22,
    "city": "Pune"
}
```

You only want:

```text
name
email
```

You can use projection:

```js
db.users.find(
    {},
    {
        name: 1,
        email: 1
    }
)
```

Result conceptually:

```json
{
    "_id": "...",
    "name": "Rahul",
    "email": "rahul@gmail.com"
}
```

Notice:

**`_id` is included by default.**

You can explicitly exclude it:

```js
db.users.find(
    {},
    {
        _id: 0,
        name: 1,
        email: 1
    }
)
```

Result:

```json
{
    "name": "Rahul",
    "email": "rahul@gmail.com"
}
```

---

# 18. Projection rule

A common beginner mistake is mixing inclusion and exclusion:

```js
{
    name: 1,
    email: 1,
    age: 0
}
```

Generally, MongoDB projection uses either:

```text
inclusion
```

or:

```text
exclusion
```

with `_id` being the notable exception.

So:

```js
{
    name: 1,
    email: 1,
    _id: 0
}
```

is valid.

But don't mix arbitrary inclusion and exclusion.

---

# 19. Sorting

Suppose you want students ordered by age.

Ascending:

```js
db.students.find().sort({
    age: 1
})
```

Descending:

```js
db.students.find().sort({
    age: -1
})
```

Remember:

```text
1  → ascending
-1 → descending
```

You can sort by multiple fields:

```js
db.students.find().sort({
    age: -1,
    name: 1
})
```

Meaning:

```text
first: age descending
then: name ascending for equal ages
```

---

# 20. Limit

Get only five documents:

```js
db.students.find().limit(5)
```

---

# 21. Skip

Skip the first ten:

```js
db.students.find().skip(10)
```

Combine them:

```js
db.students.find()
    .skip(10)
    .limit(10)
```

Conceptually:

```text
documents 1–10   → skipped
documents 11–20  → returned
```

This gives us the basic form of offset pagination.

---

# 22. Pagination

Suppose your API has:

```text
?page=3&limit=10
```

Then:

```js
const skip = (page - 1) * limit;
```

For:

```text
page = 3
limit = 10
```

we get:

```text
skip = (3 - 1) × 10
     = 20
```

MongoDB:

```js
db.products.find()
    .skip(20)
    .limit(10)
```

This is easy to implement.

But there is a problem.

For extremely large datasets:

```text
skip(1000000)
```

can become inefficient.

Later we'll learn **cursor/range-based pagination**, which is much more suitable for large datasets.

---

# 23. Query chaining

MongoDB queries can be composed:

```js
db.products.find({
    category: "Electronics",
    price: { $gte: 10000 }
})
.sort({
    price: -1
})
.limit(10)
```

Read this as:

> Find electronics costing at least ₹10,000, sort by price descending, return the first 10.

This kind of query should start becoming natural to you.

---

# Exercise Section

Now we're going to increase difficulty.

Don't send answers. **Solve mentally/on paper, then immediately check the answers.**

---

# MCQs — Round 2

### Q1 — Difficult

Given:

```json
{
    "name": "A",
    "age": 22,
    "city": "Pune"
}
```

and:

```json
{
    "name": "B",
    "age": 19,
    "city": "Pune"
}
```

What does this query return?

```js
db.users.find({
    age: { $gte: 20 },
    city: "Pune"
})
```

A. A only  
B. B only  
C. Both  
D. Neither

---

### Q2

Which query means:

> city is Pune OR Mumbai

A.

```js
{ city: { $in: ["Pune", "Mumbai"] } }
```

B.

```js
{ city: { $all: ["Pune", "Mumbai"] } }
```

C.

```js
{ city: { $nin: ["Pune", "Mumbai"] } }
```

D.

```js
{ city: { $ne: ["Pune", "Mumbai"] } }
```

---

### Q3 — Difficult

A document contains:

```json
{
    "skills": ["Java", "MongoDB", "Docker"]
}
```

Which query requires **both Java AND Docker** to be present?

A.

```js
{ skills: { $in: ["Java", "Docker"] } }
```

B.

```js
{ skills: { $all: ["Java", "Docker"] } }
```

C.

```js
{ skills: { $nin: ["Java", "Docker"] } }
```

D.

```js
{ skills: ["Java", "Docker"] }
```

---

### Q4 — Very Important

Consider:

```json
{
    "projects": [
        { "language": "Java", "stars": 20 },
        { "language": "Python", "stars": 100 }
    ]
}
```

You want:

> Find users who have a single project where language is Java AND stars > 50.

Which is the correct concept?

A. `$in`

B. `$all`

C. `$elemMatch`

D. `$nin`

---

### Q5

What does:

```js
db.users.find(
    {},
    {
        name: 1,
        email: 1
    }
)
```

primarily accomplish?

A. Sorts users by name.

B. Returns only matching documents where name/email are modified.

C. Projects the selected fields in the returned documents.

D. Creates indexes on name and email.

---

### Q6 — Tricky

Which statement about this projection is most accurate?

```js
{
    name: 1,
    email: 1,
    _id: 0
}
```

A. Invalid because inclusion and exclusion can never coexist.

B. Valid because `_id` can be explicitly excluded while other fields are included.

C. Valid only if `_id` is a string.

D. It deletes `_id` from the database.

---

### Q7

What does:

```js
.sort({ age: -1 })
```

do?

A. Sorts age ascending.

B. Sorts age descending.

C. Filters age less than zero.

D. Removes age.

---

### Q8 — Difficult

Suppose:

```text
page = 4
limit = 25
```

Using offset pagination, what is the skip value?

A. 25  
B. 50  
C. 75  
D. 100

---

### Q9 — Very Difficult

Suppose:

```json
{
    "name": "A",
    "age": 25
}
```

and:

```json
{
    "name": "B"
}
```

What is the important caveat with:

```js
db.users.find({
    age: { $ne: 25 }
})
```

A. Only B can match because A has age 25.

B. Neither can match.

C. A does not match, but B may also match because the `age` field is absent.

D. Only A can match.

---

### Q10 — Difficult

Which is generally preferable when expressing:

> age is between 20 and 30 inclusive

A.

```js
{
    $or: [
        { age: { $gte: 20 } },
        { age: { $lte: 30 } }
    ]
}
```

B.

```js
{
    age: {
        $gte: 20,
        $lte: 30
    }
}
```

C.

```js
{
    age: {
        $in: [20, 30]
    }
}
```

D.

```js
{
    age: {
        $all: [20, 30]
    }
}
```

---

## MCQ Answers

**Q1 → A**

Both conditions must be true. A satisfies both.

**Q2 → A**

`$in` means one of these values.

**Q3 → B**

`$all` requires all specified array values.

**Q4 → C**

`$elemMatch` makes the conditions apply to the **same array element**.

**Q5 → C**

Projection controls which fields are returned.

**Q6 → B**

`_id` is the special exception to the inclusion/exclusion rule.

**Q7 → B**

`-1` = descending.

**Q8 → C**

```text
skip = (page - 1) × limit
     = 3 × 25
     = 75
```

**Q9 → C**

This is a very important MongoDB subtlety: `$ne` can match documents where the field doesn't exist.

**Q10 → B**

That's the cleanest representation:

```js
age: {
    $gte: 20,
    $lte: 30
}
```

---

# Coding Exercises

We'll use:

```js
db.users.insertMany([
    {
        name: "Rahul",
        age: 22,
        city: "Pune",
        skills: ["Java", "JavaScript", "MongoDB"],
        role: "developer"
    },
    {
        name: "Priya",
        age: 25,
        city: "Mumbai",
        skills: ["Python", "Machine Learning", "SQL"],
        role: "developer"
    },
    {
        name: "Amit",
        age: 19,
        city: "Pune",
        skills: ["JavaScript", "React"],
        role: "student"
    },
    {
        name: "Sneha",
        age: 28,
        city: "Delhi",
        skills: ["Java", "Spring", "Docker"],
        role: "developer"
    },
    {
        name: "Vikram",
        age: 31,
        city: "Pune",
        skills: ["Go", "Docker", "Kubernetes"],
        role: "developer"
    }
])
```

---

# Basic

### B1

Find all developers.

### B2

Find users older than 25.

### B3

Find users from Pune.

### B4

Return only `name` and `city`, excluding `_id`.

### B5

Sort users by age from youngest to oldest.

---

## Answers

### B1

```js
db.users.find({
    role: "developer"
})
```

### B2

```js
db.users.find({
    age: { $gt: 25 }
})
```

### B3

```js
db.users.find({
    city: "Pune"
})
```

### B4

```js
db.users.find(
    {},
    {
        _id: 0,
        name: 1,
        city: 1
    }
)
```

### B5

```js
db.users.find().sort({
    age: 1
})
```

---

# Medium

### M1

Find developers who live in Pune.

### M2

Find users who live in Pune OR Mumbai.

### M3

Find users who know Java OR Python.

### M4

Find users who know BOTH Java and Docker.

### M5

Find users aged between 20 and 30 inclusive.

### M6

Return the three oldest users.

---

## Answers

### M1

```js
db.users.find({
    role: "developer",
    city: "Pune"
})
```

### M2

```js
db.users.find({
    city: {
        $in: ["Pune", "Mumbai"]
    }
})
```

### M3

```js
db.users.find({
    skills: {
        $in: ["Java", "Python"]
    }
})
```

### M4

```js
db.users.find({
    skills: {
        $all: ["Java", "Docker"]
    }
})
```

### M5

```js
db.users.find({
    age: {
        $gte: 20,
        $lte: 30
    }
})
```

### M6

```js
db.users.find()
    .sort({ age: -1 })
    .limit(3)
```

---

# Hard

### H1

Find developers who:

- are at least 25 years old
    
- live in Pune
    
- know Docker
    

### H2

Find users who:

- live in Pune or Delhi
    
- are younger than 30
    
- know either Java or JavaScript
    

### H3

Return:

```text
name
age
skills
```

for users who know Docker, sorted by age descending.

Don't return `_id`.

### H4

Suppose documents have:

```json
{
    "name": "Rahul",
    "projects": [
        {
            "name": "ChatApp",
            "language": "JavaScript",
            "stars": 120
        },
        {
            "name": "JobForge",
            "language": "Java",
            "stars": 50
        }
    ]
}
```

Find users who have **one project** where:

```text
language = JavaScript
AND
stars > 100
```

### H5 — Interview-style

Find users who:

- are developers
    
- are NOT from Mumbai
    
- are at least 21
    
- know Docker OR MongoDB
    

Try to construct this without using unnecessary `$and`.

---

# Hard Answers

### H1

```js
db.users.find({
    role: "developer",
    age: { $gte: 25 },
    city: "Pune",
    skills: "Docker"
})
```

---

### H2

```js
db.users.find({
    city: { $in: ["Pune", "Delhi"] },
    age: { $lt: 30 },
    skills: { $in: ["Java", "JavaScript"] }
})
```

Notice how we don't need an explicit `$and`.

MongoDB implicitly ANDs these field conditions.

---

### H3

```js
db.users.find(
    {
        skills: "Docker"
    },
    {
        _id: 0,
        name: 1,
        age: 1,
        skills: 1
    }
).sort({
    age: -1
})
```

---

### H4

```js
db.users.find({
    projects: {
        $elemMatch: {
            language: "JavaScript",
            stars: { $gt: 100 }
        }
    }
})
```

The critical concept is:

```text
$elemMatch
```

because both conditions must apply to the **same project**.

---

### H5

```js
db.users.find({
    role: "developer",
    city: { $ne: "Mumbai" },
    age: { $gte: 21 },
    skills: { $in: ["Docker", "MongoDB"] }
})
```

That's a very realistic MongoDB query.

---

# One important challenge

Before finishing today's lesson, look at this requirement:

> "Give me the first 20 developers from Pune, sorted by newest registration date, but only return their name, email and profile picture."

If your document looks like:

```json
{
    "_id": "...",
    "name": "Rahul",
    "email": "rahul@gmail.com",
    "profilePicture": "...",
    "role": "developer",
    "city": "Pune",
    "createdAt": "..."
}
```

you should now be able to construct:

```js
db.users.find(
    {
        role: "developer",
        city: "Pune"
    },
    {
        _id: 0,
        name: 1,
        email: 1,
        profilePicture: 1
    }
)
.sort({
    createdAt: -1
})
.limit(20)
```

That's already the basic shape of a real API query.

---

# What you should know now

You should be comfortable with:

```text
Queries
│
├── equality
├── $gt / $gte / $lt / $lte
├── $ne
├── $in / $nin
│
├── Logical
│   ├── $and
│   ├── $or
│   ├── $nor
│   └── $not
│
├── Arrays
│   ├── direct matching
│   ├── $in
│   ├── $all
│   └── $elemMatch
│
├── Nested documents
│   └── dot notation
│
└── Result shaping
    ├── projection
    ├── sort
    ├── limit
    └── skip
```

But **don't start worrying about indexes yet**.

The next lesson should be:

# Lesson 3 — MongoDB Data Modeling

This is where MongoDB starts becoming genuinely interesting.

We'll tackle the question:

> **"I have Users, Messages, Jobs, Applications, Comments, Likes, etc. How do I decide what should be embedded inside a document and what should be stored as a reference?"**

We'll use examples from **ChatApp and JobForge**, and cover:

- embedding
    
- referencing
    
- one-to-one
    
- one-to-many
    
- many-to-many
    
- cardinality
    
- document size
    
- duplication
    
- normalization vs denormalization
    
- atomic updates
    
- schema design
    
- real-world modeling decisions
    

Then we'll start looking at **why some MongoDB schemas are fast and others become disasters**.

That's the point where you'll move from _writing MongoDB queries_ to actually **designing MongoDB databases**.