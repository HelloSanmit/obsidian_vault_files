
# MongoDB — Lesson 4: Updates & Array Operations

Lesson 3 was about **how to design your documents**.

Now we move into one of the most practically important MongoDB skills:

> **How do you safely change data that already exists?**

We'll go from basic updates → nested fields → arrays → positional operators → `arrayFilters` → upserts → common mistakes.

---

# 1. The basic update structure

Most MongoDB updates follow this pattern:

```javascript
db.collection.updateOne(
    <filter>,
    <update>
)
```

For example:

```javascript
db.users.updateOne(
    { email: "sanmit@example.com" },
    { $set: { age: 22 } }
)
```

Think of it as:

```text
Which document?
      ↓
{ email: "..." }

What should change?
      ↓
{ $set: { age: 22 } }
```

`updateOne()` updates at most one matching document, while `updateMany()` can update all matching documents. MongoDB's update methods accept update-operator expressions or, in applicable cases, an aggregation pipeline. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.updateone/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

# 2. `$set` — your most important update operator

```javascript
$set
```

means:

> Set this field to this value.

Suppose:

```javascript
{
    name: "Sanmit",
    age: 21,
    city: "Pune"
}
```

Run:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $set: { age: 22 } }
)
```

Result:

```javascript
{
    name: "Sanmit",
    age: 22,
    city: "Pune"
}
```

Notice:

**Only `age` changed.**

Everything else remained.

---

# 3. `$set` can create a field

Suppose:

```javascript
{
    name: "Sanmit",
    age: 22
}
```

Run:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $set: { role: "candidate" } }
)
```

Result:

```javascript
{
    name: "Sanmit",
    age: 22,
    role: "candidate"
}
```

So `$set` means both:

```text
field exists → change it
field doesn't exist → create it
```

---

# 4. Nested fields with `$set`

Suppose:

```javascript
{
    name: "Sanmit",

    profile: {
        city: "Pune",
        bio: "Developer"
    }
}
```

You want to change only the city.

Do:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $set: { "profile.city": "Mumbai" } }
)
```

Result:

```javascript
{
    name: "Sanmit",

    profile: {
        city: "Mumbai",
        bio: "Developer"
    }
}
```

This is **dot notation**, which you already saw in querying.

---

# 5. A dangerous beginner mistake with nested objects

Suppose:

```javascript
{
    profile: {
        city: "Pune",
        bio: "Developer",
        age: 22
    }
}
```

You do:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $set: {
            profile: {
                city: "Mumbai"
            }
        }
    }
)
```

You may be thinking:

> “I'm changing the city.”

But you're actually setting the **whole `profile` field** to this:

```javascript
{
    city: "Mumbai"
}
```

So the previous:

```text
bio
age
```

are gone.

That's why this:

```javascript
$set: {
    "profile.city": "Mumbai"
}
```

is very different from:

```javascript
$set: {
    profile: {
        city: "Mumbai"
    }
}
```

This distinction is extremely important in application code.

---

# 6. `$unset` — remove a field

Suppose:

```javascript
{
    name: "Sanmit",
    age: 22,
    temporaryToken: "abc123"
}
```

Remove the field:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $unset: { temporaryToken: "" } }
)
```

Result:

```javascript
{
    name: "Sanmit",
    age: 22
}
```

The value supplied to `$unset` is generally irrelevant; the operator removes the field.

So:

```javascript
{ $unset: { temporaryToken: "" } }
```

means:

```text
remove temporaryToken
```

---

# 7. `$inc` — increment/decrement

This is extremely useful.

Suppose:

```javascript
{
    name: "Sanmit",
    loginCount: 10
}
```

Do:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $inc: { loginCount: 1 } }
)
```

Result:

```javascript
loginCount: 11
```

You can decrement:

```javascript
{ $inc: { loginCount: -1 } }
```

This is much better than:

```javascript
// Don't think like this
find user
read loginCount
loginCount++
save user
```

because that introduces a read-modify-write pattern that can be problematic under concurrent updates.

Instead:

```javascript
{ $inc: { loginCount: 1 } }
```

expresses the actual database operation:

> increment this value by one.

---

# 8. Why `$inc` matters in real systems

Imagine 100 people like a post.

You want:

```text
likes = likes + 1
```

MongoDB can express that directly:

```javascript
db.posts.updateOne(
    { _id: postId },
    { $inc: { likeCount: 1 } }
)
```

This idea becomes especially useful when we later discuss **atomicity and concurrency**.

---

# 9. `$min` and `$max`

These are less commonly used by beginners but worth knowing.

Suppose:

```javascript
{
    name: "Sanmit",
    lowestScore: 50
}
```

You want to store a new low score:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $min: { lowestScore: 40 } }
)
```

Result:

```text
40
```

But:

```javascript
{ $min: { lowestScore: 60 } }
```

would leave it at:

```text
40
```

because 40 is already lower.

Similarly:

```javascript
$max
```

keeps the larger value.

These are useful for tracking things like:

```text
highest score
lowest latency
maximum temperature
minimum price
```

---

# 10. `$rename`

You can rename a field:

```javascript
db.users.updateMany(
    {},
    { $rename: { "username": "userName" } }
)
```

Conceptually:

```text
username
   ↓
userName
```

This is useful for schema migrations.

---

# 11. Updating multiple fields at once

You can combine operators:

```javascript
db.users.updateOne(
    { email: "sanmit@example.com" },
    {
        $set: {
            city: "Pune",
            role: "candidate"
        },
        $inc: {
            profileViews: 1
        },
        $unset: {
            temporaryToken: ""
        }
    }
)
```

One update operation can therefore express several changes.

---

# 12. `updateOne()` vs `updateMany()`

Suppose:

```javascript
{
    role: "candidate"
}
```

appears in 500 users.

This:

```javascript
db.users.updateOne(
    { role: "candidate" },
    { $set: { accountType: "standard" } }
)
```

updates only **one matching document**.

This:

```javascript
db.users.updateMany(
    { role: "candidate" },
    { $set: { accountType: "standard" } }
)
```

updates **all matching documents**.

Be very careful with `updateMany()`.

A weak filter can modify your entire collection.

---

# 13. Always mentally test your filter

Before executing:

```javascript
db.users.updateMany(
    { role: "candidate" },
    { $set: { verified: true } }
)
```

first run:

```javascript
db.users.find({ role: "candidate" })
```

You should know:

```text
How many documents?
Which documents?
```

before performing a mass update.

This is basic database hygiene.

---

# 14. `replaceOne()` is VERY different

This is a major concept.

Suppose you have:

```javascript
{
    _id: 1,
    name: "Sanmit",
    age: 22,
    city: "Pune"
}
```

You run:

```javascript
db.users.replaceOne(
    { _id: 1 },
    {
        name: "Sanmit",
        age: 23
    }
)
```

You haven't just changed age.

You replaced the document contents.

Now conceptually:

```javascript
{
    _id: 1,
    name: "Sanmit",
    age: 23
}
```

`city` is gone.

MongoDB documents `replaceOne()` as replacing the entire document contents except for the immutable `_id`; the replacement document cannot contain update operators. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.replaceone/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

# 15. `updateOne()` vs `replaceOne()`

Remember this:

### Partial modification

```javascript
updateOne(
    { _id: 1 },
    { $set: { age: 23 } }
)
```

means:

```text
"Change this field."
```

### Full replacement

```javascript
replaceOne(
    { _id: 1 },
    {
        name: "Sanmit",
        age: 23
    }
)
```

means:

```text
"Replace this document with this new document."
```

For most normal application updates, you'll use update operators.

---

# 16. Now the interesting part: arrays

Suppose:

```javascript
{
    name: "Sanmit",

    skills: [
        "Java",
        "React",
        "MongoDB"
    ]
}
```

MongoDB provides several array-specific operators.

The important ones are:

```text
$push
$addToSet
$pop
$pull
$pullAll
$each
$position
$slice
```

MongoDB documents these as its core array update operators. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update-array/?utm_source=chatgpt.com "Array Update Operators - Database Manual - MongoDB Docs"))

---

# 17. `$push`

Add an element to an array.

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $push: {
            skills: "Redis"
        }
    }
)
```

Before:

```javascript
skills: [
    "Java",
    "React",
    "MongoDB"
]
```

After:

```javascript
skills: [
    "Java",
    "React",
    "MongoDB",
    "Redis"
]
```

---

# 18. `$push` allows duplicates

Suppose:

```javascript
skills: [
    "Java",
    "React"
]
```

Run:

```javascript
$push: {
    skills: "Java"
}
```

Result:

```javascript
[
    "Java",
    "React",
    "Java"
]
```

That's important.

`$push` does **not** enforce uniqueness.

---

# 19. `$addToSet`

If you want set-like behavior:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $addToSet: {
            skills: "Java"
        }
    }
)
```

If Java already exists:

```text
nothing is added
```

If it doesn't:

```text
Java is added
```

MongoDB documents `$addToSet` as adding an array value only when that value isn't already present. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update-array/?utm_source=chatgpt.com "Array Update Operators - Database Manual - MongoDB Docs"))

This is excellent for:

```text
skills
tags
roles
followers
participants
```

when duplicate values shouldn't exist.

---

# 20. `$push` vs `$addToSet`

Very important:

```text
$push
    ↓
append regardless of duplicate

$addToSet
    ↓
append only if value isn't already present
```

Think:

```text
$push      → List
$addToSet  → Set
```

Not literally Java's `List` and `Set`, but that's a good mental model.

---

# 21. Adding multiple values with `$each`

Suppose:

```javascript
skills: [
    "Java"
]
```

You want:

```text
React
MongoDB
Redis
```

You can do:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $push: {
            skills: {
                $each: [
                    "React",
                    "MongoDB",
                    "Redis"
                ]
            }
        }
    }
)
```

Result:

```javascript
[
    "Java",
    "React",
    "MongoDB",
    "Redis"
]
```

`$each` modifies `$push` or `$addToSet` so multiple values can be added in one update. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update-array/?utm_source=chatgpt.com "Array Update Operators - Database Manual - MongoDB Docs"))

---

# 22. `$addToSet` + `$each`

This is also useful:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $addToSet: {
            skills: {
                $each: [
                    "Java",
                    "Redis",
                    "MongoDB"
                ]
            }
        }
    }
)
```

Existing values won't be duplicated.

---

# 23. `$pop`

Remove from the beginning or end.

```javascript
{
    $pop: {
        skills: 1
    }
}
```

removes the **last** element.

```javascript
{
    $pop: {
        skills: -1
    }
}
```

removes the **first** element.

MongoDB documents `$pop` as removing either the first or last array element. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update-array/?utm_source=chatgpt.com "Array Update Operators - Database Manual - MongoDB Docs"))

---

# 24. `$pull`

This is one of the most useful array operators.

Suppose:

```javascript
skills: [
    "Java",
    "React",
    "MongoDB",
    "Python"
]
```

Remove MongoDB:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $pull: {
            skills: "MongoDB"
        }
    }
)
```

Result:

```javascript
[
    "Java",
    "React",
    "Python"
]
```

---

# 25. `$pull` is condition-based

Suppose:

```javascript
scores: [
    40,
    70,
    90,
    55
]
```

You want to remove every score below 60:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $pull: {
            scores: {
                $lt: 60
            }
        }
    }
)
```

Result:

```javascript
[
    70,
    90
]
```

So:

```text
$pull
  ↓
remove array elements matching a condition
```

MongoDB defines `$pull` as removing all array elements that match a specified query. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update-array/?utm_source=chatgpt.com "Array Update Operators - Database Manual - MongoDB Docs"))

---

# 26. `$pull` with objects

Suppose:

```javascript
{
    name: "Sanmit",

    projects: [
        {
            name: "ChatApp",
            status: "completed"
        },
        {
            name: "JobForge",
            status: "in-progress"
        }
    ]
}
```

Remove every project whose status is completed:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $pull: {
            projects: {
                status: "completed"
            }
        }
    }
)
```

Now:

```javascript
projects: [
    {
        name: "JobForge",
        status: "in-progress"
    }
]
```

---

# 27. `$push` modifiers

`$push` gets interesting when combined with:

```text
$each
$position
$slice
$sort
```

MongoDB documents these as modifiers for array updates. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update-array/?utm_source=chatgpt.com "Array Update Operators - Database Manual - MongoDB Docs"))

Example:

```javascript
{
    $push: {
        recentMessages: {
            $each: [
                { text: "Hello" },
                { text: "Hi" }
            ],

            $slice: -10
        }
    }
}
```

Conceptually:

```text
add messages
      ↓
keep only last 10
```

This can be useful for **bounded arrays**.

---

# 28. `$slice` is particularly important for your modeling skills

Remember Lesson 3?

We said:

> Avoid unbounded arrays.

Suppose you intentionally maintain:

```javascript
recentMessages: [...]
```

and only ever need the most recent 20.

You can enforce the size during the push:

```javascript
$push: {
    recentMessages: {
        $each: [newMessage],
        $slice: -20
    }
}
```

Now the array remains bounded.

But this does **not** mean `$slice` magically makes giant-array modeling safe. You still need to think about workload and whether this data belongs embedded at all.

---

# 29. Now: updating an object inside an array

This is where MongoDB becomes much more interesting.

Suppose:

```javascript
{
    name: "Sanmit",

    courses: [
        {
            name: "MongoDB",
            completed: false
        },
        {
            name: "Java",
            completed: true
        }
    ]
}
```

You want:

```text
MongoDB.completed = true
```

How do we find the correct array element?

This brings us to the **positional `$` operator**.

---

# 30. Positional `$` operator

You can do:

```javascript
db.users.updateOne(
    {
        name: "Sanmit",
        "courses.name": "MongoDB"
    },
    {
        $set: {
            "courses.$.completed": true
        }
    }
)
```

The `$` represents the matching array element.

So:

```text
courses.$
    ↓
the first array element matching the query
```

MongoDB documents `$` as a placeholder for the first array element matching the query, and the array field must appear in the query document. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional/?utm_source=chatgpt.com "$ (update operator) - Database Manual - MongoDB Docs"))

---

# 31. Understand the `$` mentally

Given:

```javascript
courses: [
    { name: "MongoDB", completed: false },
    { name: "Java", completed: true },
    { name: "React", completed: false }
]
```

Query:

```javascript
"courses.name": "MongoDB"
```

means:

```text
Find an array element where name = MongoDB
```

Then:

```javascript
"courses.$.completed"
```

means:

```text
Modify that matching element's completed field.
```

Result:

```javascript
courses: [
    { name: "MongoDB", completed: true },
    { name: "Java", completed: true },
    { name: "React", completed: false }
]
```

---

# 32. `$` only represents the first matching element

This is important.

Suppose:

```javascript
tags: [
    "Java",
    "Backend",
    "Java"
]
```

The positional `$` isn't a mechanism for saying “all matching elements.”

For updating **all** elements, MongoDB provides `$[]`. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional/?utm_source=chatgpt.com "$ (update operator) - Database Manual - MongoDB Docs"))

---

# 33. `$[]` — update every array element

Suppose:

```javascript
scores: [
    10,
    20,
    30
]
```

You want to add 5 to every score:

```javascript
db.students.updateOne(
    { _id: 1 },
    {
        $inc: {
            "scores.$[]": 5
        }
    }
)
```

Result:

```javascript
[
    15,
    25,
    35
]
```

`$[]` means:

> all array elements.

MongoDB documents `$[]` as the all-positional operator. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional-all/?utm_source=chatgpt.com "$[] (update operator) - Database Manual - MongoDB Docs"))

---

# 34. `$[identifier]` — update selected elements

Now we get to the most powerful array-update concept.

Suppose:

```javascript
scores: [
    40,
    80,
    90,
    50
]
```

You want to add 5 only to scores below 60.

Use:

```javascript
db.students.updateOne(
    { _id: 1 },
    {
        $inc: {
            "scores.$[score]": 5
        }
    },
    {
        arrayFilters: [
            {
                score: { $lt: 60 }
            }
        ]
    }
)
```

Result:

```javascript
[
    45,
    80,
    90,
    55
]
```

The filtered positional operator `$[identifier]` works with `arrayFilters` to update all array elements satisfying the specified condition. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional-filtered/?utm_source=chatgpt.com "$[<identifier>] (update operator) - Database Manual - MongoDB Docs"))

---

# 35. `$`, `$[]`, `$[x]` — memorize this difference

This is worth memorizing.

```text
$         → first matching array element

$[]       → every array element

$[x]      → every array element matching arrayFilters
```

Think:

```text
$       = one
$[]     = all
$[x]    = selected
```

That mental shortcut will save you a lot of confusion.

---

# 36. `arrayFilters` with objects

Suppose:

```javascript
{
    courses: [
        {
            name: "MongoDB",
            score: 50
        },
        {
            name: "Java",
            score: 80
        },
        {
            name: "React",
            score: 60
        }
    ]
}
```

You want to add 10 to every course whose score is below 70.

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $inc: {
            "courses.$[course].score": 10
        }
    },
    {
        arrayFilters: [
            {
                "course.score": { $lt: 70 }
            }
        ]
    }
)
```

Result:

```javascript
courses: [
    {
        name: "MongoDB",
        score: 60
    },
    {
        name: "Java",
        score: 80
    },
    {
        name: "React",
        score: 70
    }
]
```

This is one of the places where MongoDB's update language becomes considerably more powerful than simple CRUD.

---

# 37. Nested arrays

You can even have:

```javascript
{
    students: [
        {
            name: "A",
            subjects: [
                { name: "Math", score: 50 },
                { name: "CS", score: 80 }
            ]
        },
        {
            name: "B",
            subjects: [
                { name: "Math", score: 60 },
                { name: "CS", score: 90 }
            ]
        }
    ]
}
```

Now imagine:

> Increase every CS score below 70 by 5.

You can combine `$[]` and `$[identifier]`:

```javascript
db.school.updateMany(
    {},
    {
        $inc: {
            "students.$[].subjects.$[subject].score": 5
        }
    },
    {
        arrayFilters: [
            {
                "subject.name": "CS",
                "subject.score": { $lt: 70 }
            }
        ]
    }
)
```

MongoDB supports `$[]` and `$[identifier]` together for nested arrays. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional-all/?utm_source=chatgpt.com "$[] (update operator) - Database Manual - MongoDB Docs"))

Don't worry if this looks complicated.

You don't need to memorize this syntax immediately.

Just understand **what problem each operator solves**.

---

# 38. Upsert

Another important concept:

```javascript
upsert: true
```

means:

```text
If document exists → update it.

If document doesn't exist → create it.
```

Example:

```javascript
db.users.updateOne(
    { email: "sanmit@example.com" },
    {
        $set: {
            name: "Sanmit",
            role: "candidate"
        }
    },
    {
        upsert: true
    }
)
```

If no user with that email exists, MongoDB inserts a document based on the operation.

MongoDB documents `upsert` as causing an update operation to insert a document when no matching document exists. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.updateone/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

# 39. Why `upsert` is useful

Imagine your app receives:

```text
userId = 123
```

and wants to store:

```text
lastSeen
```

You could say:

```text
find user
if exists → update
else → insert
```

That's two conceptual steps.

With upsert:

```javascript
updateOne(
    { userId: 123 },
    { $set: { lastSeen: new Date() } },
    { upsert: true }
)
```

you express:

> “Make sure this record has this state.”

Very useful for things like:

```text
settings
counters
preferences
cache-like records
metadata
```

But don't blindly use upsert everywhere.

The filter should represent a logically unique record, and unique indexes are important when duplicate upserts would be problematic. MongoDB's documentation specifically recommends unique indexing to help avoid multiple upserts for the same logical record. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.updateone/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

# 40. Important result: `matchedCount` vs `modifiedCount`

When you execute an update, you'll typically get information such as:

```javascript
{
    acknowledged: true,
    matchedCount: 1,
    modifiedCount: 1
}
```

Understand the difference.

### `matchedCount`

How many documents matched your filter?

### `modifiedCount`

How many documents were actually changed?

For example:

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    { $set: { age: 22 } }
)
```

Suppose Sanmit already has:

```text
age = 22
```

Then you might have:

```text
matchedCount = 1
modifiedCount = 0
```

because the document matched but its stored value didn't actually change.

---

# 41. Real ChatApp example

Let's connect this directly to your project.

Suppose message:

```javascript
{
    _id: ObjectId("m1"),
    senderId: ObjectId("u1"),
    receiverId: ObjectId("u2"),
    text: "Hello",
    seen: false
}
```

When receiver reads it:

```javascript
db.messages.updateOne(
    { _id: ObjectId("m1") },
    { $set: { seen: true } }
)
```

Simple.

But your authorization logic is **not** a MongoDB concern alone.

Your backend should first establish:

```text
Is u2 actually the receiver of m1?
```

Then perform the update.

This is an important full-stack distinction:

```text
MongoDB:
"How do I update this?"

Application:
"Is this user allowed to update it?"
```

Database syntax does not automatically give you business authorization.

---

# 42. Real ChatApp unread count example

Suppose you maintain:

```javascript
unreadCount: 4
```

A new incoming message:

```javascript
db.users.updateOne(
    { _id: receiverId },
    {
        $inc: {
            unreadCount: 1
        }
    }
)
```

When messages are read:

```javascript
db.users.updateOne(
    { _id: receiverId },
    {
        $set: {
            unreadCount: 0
        }
    }
)
```

Now you can see how Lesson 4 connects directly to your application.

---

# 43. Real JobForge example

Suppose a candidate has:

```javascript
{
    _id: ObjectId("u1"),

    skills: [
        "Java",
        "Spring Boot",
        "React"
    ]
}
```

Add MongoDB:

```javascript
db.users.updateOne(
    { _id: ObjectId("u1") },
    {
        $addToSet: {
            skills: "MongoDB"
        }
    }
)
```

Remove React:

```javascript
db.users.updateOne(
    { _id: ObjectId("u1") },
    {
        $pull: {
            skills: "React"
        }
    }
)
```

This is a realistic use of the operators we just learned.

---

# 44. Common mistakes

## Mistake 1 — accidentally replacing nested object

Bad:

```javascript
$set: {
    profile: {
        city: "Pune"
    }
}
```

when you intended only to change city.

Better:

```javascript
$set: {
    "profile.city": "Pune"
}
```

---

## Mistake 2 — using `$push` when duplicates aren't allowed

Bad:

```javascript
$push: {
    skills: "Java"
}
```

when Java may already exist.

Use:

```javascript
$addToSet
```

when uniqueness is desired.

---

## Mistake 3 — using `$` expecting all matches

```javascript
"courses.$.completed"
```

targets the positional match represented by `$`, not every matching element.

For all:

```javascript
"courses.$[].completed"
```

For conditionally selected elements:

```javascript
"courses.$[course].completed"
```

with:

```javascript
arrayFilters
```

([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional/?utm_source=chatgpt.com "$ (update operator) - Database Manual - MongoDB Docs"))

---

## Mistake 4 — dangerous `updateMany()`

This:

```javascript
db.users.updateMany(
    {},
    { $set: { verified: true } }
)
```

means:

> update every user.

An empty filter means all documents match.

Never casually run mass updates.

---

## Mistake 5 — using `$` with upsert

MongoDB specifically warns against using the positional `$` operator with upsert because there may be no matching array element when insertion occurs. ([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional/?utm_source=chatgpt.com "$ (update operator) - Database Manual - MongoDB Docs"))

---

# 45. The update-operator mental map

Don't memorize syntax randomly.

Think in categories:

```text
FIELD OPERATIONS
│
├── $set       → assign
├── $unset     → remove
├── $rename    → rename
│
NUMERIC
│
├── $inc       → increase/decrease
├── $min       → keep smaller
└── $max       → keep larger

ARRAY
│
├── $push      → append
├── $addToSet  → append uniquely
├── $pop       → remove first/last
├── $pull      → remove matching elements
└── $pullAll   → remove specified values

ARRAY POSITION
│
├── $          → first matching
├── $[]        → every element
└── $[x]       → filtered elements
```

That is the framework I want you to have in your head.

---

# 46. A few exercises

Keeping the exercise load deliberately small.

## Exercise 1 — Basic

Given:

```javascript
{
    name: "Sanmit",
    age: 21,
    city: "Pune"
}
```

Change:

```text
age → 22
city → Mumbai
```

### Answer

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $set: {
            age: 22,
            city: "Mumbai"
        }
    }
)
```

---

# 47. Exercise 2 — Array

Given:

```javascript
{
    name: "Sanmit",
    skills: [
        "Java",
        "React",
        "MongoDB"
    ]
}
```

Add:

```text
Redis
```

but don't add it if it already exists.

### Answer

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $addToSet: {
            skills: "Redis"
        }
    }
)
```

---

# 48. Exercise 3 — Medium

Given:

```javascript
{
    name: "Sanmit",

    courses: [
        { name: "MongoDB", completed: false },
        { name: "Java", completed: true },
        { name: "React", completed: false }
    ]
}
```

Set MongoDB's `completed` to `true`.

### Answer

```javascript
db.users.updateOne(
    {
        name: "Sanmit",
        "courses.name": "MongoDB"
    },
    {
        $set: {
            "courses.$.completed": true
        }
    }
)
```

---

# 49. Exercise 4 — Hard

Given:

```javascript
{
    name: "Sanmit",

    courses: [
        { name: "MongoDB", score: 50 },
        { name: "Java", score: 90 },
        { name: "React", score: 60 }
    ]
}
```

Increase the score of **every course whose score is below 70 by 10**.

### Answer

```javascript
db.users.updateOne(
    { name: "Sanmit" },
    {
        $inc: {
            "courses.$[course].score": 10
        }
    },
    {
        arrayFilters: [
            {
                "course.score": { $lt: 70 }
            }
        ]
    }
)
```

Result:

```javascript
{
    courses: [
        { name: "MongoDB", score: 60 },
        { name: "Java", score: 90 },
        { name: "React", score: 70 }
    ]
}
```

---

# 50. One conceptual question

Which statement is correct?

**A.** `$set` always replaces the entire document.

**B.** `$push` prevents duplicate array elements.

**C.** `$` updates the first matching array element, `$[]` targets all array elements, and `$[x]` targets elements matching `arrayFilters`.

**D.** `replaceOne()` only changes specified fields.

### Answer

**C**

And remember:

```text
updateOne + $set
    ↓
partial update

replaceOne
    ↓
document replacement
```

([MongoDB](https://www.mongodb.com/docs/manual/reference/operator/update/positional/?utm_source=chatgpt.com "$ (update operator) - Database Manual - MongoDB Docs"))

---

# Lesson 4 takeaway

You should now be comfortable thinking:

```text
"I need to change one field"
        ↓
$set

"I need to delete a field"
        ↓
$unset

"I need to increase a number"
        ↓
$inc

"I need to append to an array"
        ↓
$push

"I need to append without duplicates"
        ↓
$addToSet

"I need to remove matching array elements"
        ↓
$pull

"I need to update one matching array element"
        ↓
$

"I need to update every array element"
        ↓
$[]

"I need to update only array elements satisfying a condition"
        ↓
$[identifier] + arrayFilters
```

And there's one distinction I especially want you to remember:

```text
updateOne()
    ≠
replaceOne()
```

`updateOne()` with operators changes selected parts of a document; `replaceOne()` replaces the document contents. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.replaceone/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

**Next lesson: MongoDB Aggregation Framework.**

That's where MongoDB starts becoming much more powerful: `$match`, `$group`, `$project`, `$sort`, `$unwind`, `$lookup`, `$facet`, pipelines, and eventually how to build real analytics/API queries without trying to do everything in JavaScript.