
# Lesson 14 — Part 3: Building a Real Mongoose Backend

Now we move from **“understanding Mongoose”** to:

> **“How would I actually structure a serious Node.js + Express + MongoDB application?”**

This is where we connect Mongoose to software engineering.

We’ll use a small **JobForge-style backend** for examples, because it is more useful to you than another generic `User` example.

---

# 1. First: the architecture

A clean backend can look like this:

```text
src/
│
├── config/
│   └── database.js
│
├── models/
│   ├── User.js
│   ├── Job.js
│   └── Application.js
│
├── controllers/
│   ├── jobController.js
│   └── applicationController.js
│
├── services/
│   ├── jobService.js
│   └── applicationService.js
│
├── routes/
│   ├── jobRoutes.js
│   └── applicationRoutes.js
│
├── middlewares/
│   ├── auth.js
│   └── errorHandler.js
│
├── utils/
│   └── ...
│
└── server.js
```

The important flow is:

```text
HTTP Request
     ↓
Route
     ↓
Middleware
     ↓
Controller
     ↓
Service
     ↓
Mongoose Model
     ↓
MongoDB
```

Each layer has a job.

---

# 2. Why not put everything in the controller?

A beginner might write:

```js
router.post("/jobs", async (req, res) => {
  const job = await Job.create(req.body);

  res.status(201).json(job);
});
```

That's fine for a tiny project.

But soon you get:

```js
router.post("/jobs", async (req, res) => {

  // validate input

  // check authentication

  // check company exists

  // check permission

  // normalize data

  // create job

  // update company's job count

  // emit event

  // log activity

  // handle errors

  // return response
});
```

Now the route is doing everything.

That's how spaghetti code begins.

---

# 3. Controller responsibility

A controller should mostly deal with **HTTP**.

For example:

```js
export async function createJob(req, res, next) {
  try {
    const job = await jobService.createJob({
      userId: req.user.id,
      data: req.body
    });

    res.status(201).json(job);
  } catch (error) {
    next(error);
  }
}
```

Notice:

```text
Controller
   ↓
receives HTTP request
   ↓
calls service
   ↓
returns HTTP response
```

It doesn't need to understand every database detail.

---

# 4. Service responsibility

The service contains business logic.

```js
export async function createJob({ userId, data }) {
  // check whether user can create jobs

  // validate business rules

  // create job

  // return result
}
```

Now the separation is:

```text
Controller
=
HTTP concerns

Service
=
Business concerns

Model
=
Database interaction
```

This becomes incredibly useful when the application gets large.

---

# 5. Model responsibility

Example:

```js
const jobSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: true,
      trim: true
    },

    companyId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "Company",
      required: true
    },

    status: {
      type: String,
      enum: ["OPEN", "CLOSED"],
      default: "OPEN"
    }
  },
  {
    timestamps: true
  }
);
```

The model describes the persistence structure and provides database operations.

```js
const Job = mongoose.model("Job", jobSchema);
```

---

# 6. A new method: `trim`

You may have seen this already, but let's make its meaning explicit.

```js
trim: true
```

for a Mongoose String means Mongoose removes unnecessary whitespace around the string.

For example:

```text
"   Backend Developer   "
```

becomes conceptually:

```text
"Backend Developer"
```

This is useful for user-entered text.

---

# 7. Another important String option: `lowercase`

Example:

```js
email: {
  type: String,
  lowercase: true
}
```

Mongoose can convert:

```text
"USER@EXAMPLE.COM"
```

into:

```text
"user@example.com"
```

This can be useful for normalized values such as email addresses.

But don't blindly normalize every field. Case can matter for some kinds of data.

---

# 8. `select()` — very useful

Suppose you have:

```js
const users = await User.find();
```

and your user documents contain:

```text
name
email
passwordHash
phone
profileImage
createdAt
```

You don't want to send everything to the frontend.

You can use:

```js
const users = await User
  .find()
  .select("name email profileImage");
```

### What does `select()` do?

It controls **which fields are returned**.

Think:

```text
select()
=
“Give me only these fields.”
```

This is called **projection** in MongoDB terminology.

MongoDB can perform projection directly, and Mongoose gives you a convenient API for it.

---

# 9. Excluding fields

You can also exclude:

```js
const users = await User
  .find()
  .select("-passwordHash");
```

The `-` means:

> Don't include this field.

This is extremely important when returning user documents.

---

# 10. Don't return passwords

A common mistake:

```js
res.json(user);
```

What if `user` contains:

```js
{
  email: "...",
  passwordHash: "..."
}
```

Even though the password is hashed, you generally don't want it in the API response.

A good model can even configure sensitive fields to be excluded by default.

For example:

```js
passwordHash: {
  type: String,
  required: true,
  select: false
}
```

Then normal queries don't automatically return it.

When you genuinely need it:

```js
const user = await User
  .findOne({ email })
  .select("+passwordHash");
```

That is very useful for login.

---

# 11. `select: false`

This is a **schema configuration**, not the same as:

```js
.select("-passwordHash")
```

Think:

```text
Schema:
passwordHash hidden by default
```

Then:

```js
.select("+passwordHash")
```

means:

> Explicitly include it for this query.

This is an elegant security-oriented pattern.

---

# 12. `exec()` — a method you should know

You've already seen:

```js
const users = await User.find({
  status: "ACTIVE"
});
```

You can also write:

```js
const users = await User
  .find({
    status: "ACTIVE"
  })
  .exec();
```

### What does `exec()` mean?

Very simply:

> **Execute the Mongoose query and give me the result.**

Mongoose queries can be built incrementally:

```js
const query = User
  .find({ status: "ACTIVE" })
  .sort({ createdAt: -1 })
  .limit(20);
```

Then:

```js
const users = await query.exec();
```

For modern Mongoose code, you'll commonly see both styles:

```js
await User.find(...)
```

and:

```js
await User.find(...).exec()
```

You don't need to use `exec()` everywhere, but you should recognize it.

---

# 13. `exists()` — a useful method

Suppose you only want to know:

> “Does a job with this ID exist?”

Don't necessarily fetch the entire job.

You can use:

```js
const exists = await Job.exists({
  _id: jobId
});
```

The result is truthy when a matching document exists and null-like when none exists.

Conceptually:

```text
find()
=
“Give me the document.”

exists()
=
“Just tell me whether something exists.”
```

This can avoid unnecessary data retrieval.

---

# 14. `countDocuments()`

Suppose you want:

> How many open jobs are there?

You can do:

```js
const count = await Job.countDocuments({
  status: "OPEN"
});
```

It returns a number:

```text
125
```

This is different from fetching all documents and doing:

```js
jobs.length
```

You generally don't want to retrieve 125,000 documents just to count them.

---

# 15. `estimatedDocumentCount()`

Another method you should recognize:

```js
await Job.estimatedDocumentCount();
```

This is for getting an **estimate of the total number of documents in a collection**, without using a filter.

Important distinction:

```text
countDocuments()
=
count documents matching a condition

estimatedDocumentCount()
=
estimate total collection size
```

Use the one that matches the problem.

---

# 16. `distinct()`

Suppose you want all unique job locations:

```js
const locations = await Job.distinct("location");
```

You might get:

```js
[
  "Pune",
  "Mumbai",
  "Delhi",
  "Bengaluru"
]
```

### Meaning

`distinct()` means:

> Give me unique values for this field.

It's useful for things such as:

```text
unique categories
unique locations
unique statuses
unique company IDs
```

But again, don't use it blindly on enormous high-cardinality data when another design would be better.

---

# 17. Sorting

Example:

```js
const jobs = await Job
  .find({ status: "OPEN" })
  .sort({ createdAt: -1 });
```

You already know:

```text
1  = ascending
-1 = descending
```

So:

```js
{ createdAt: -1 }
```

means:

> newest first.

---

# 18. `limit()`

```js
.limit(20)
```

means:

> Return at most 20 documents.

This is essential for APIs.

Never casually write:

```js
Job.find({})
```

for a large collection and return every document.

Imagine:

```text
2 million jobs
```

and someone calls:

```http
GET /jobs
```

Returning all 2 million would be absurd.

---

# 19. `skip()`

```js
.skip(20)
.limit(20)
```

means roughly:

```text
skip first 20
then return next 20
```

This is simple pagination.

For small/moderate result sets, it can be fine.

For huge offsets, cursor-based pagination is often a better design.

We discussed that earlier.

---

# 20. A new built-in JavaScript method: `Array.isArray()`

You'll encounter this constantly in backend code.

```js
Array.isArray(value)
```

asks:

> “Is this value actually an array?”

Example:

```js
Array.isArray(["Java", "MongoDB"]);
```

returns:

```text
true
```

while:

```js
Array.isArray("Java");
```

returns:

```text
false
```

Why is this useful?

Because APIs receive unpredictable input.

For example:

```js
if (!Array.isArray(req.body.skills)) {
  // invalid input
}
```

---

# 21. `ObjectId.isValid()`

Another method worth knowing.

```js
mongoose.Types.ObjectId.isValid(value)
```

It checks whether the supplied value can be interpreted as a valid ObjectId representation.

For example:

```js
if (!mongoose.Types.ObjectId.isValid(jobId)) {
  return res.status(400).json({
    message: "Invalid job ID"
  });
}
```

Why is this useful?

Suppose the request is:

```http
GET /jobs/hello123
```

rather than:

```http
GET /jobs/<valid-id>
```

You can reject malformed IDs before performing the database lookup.

### Important distinction

It checks **whether the value is valid as an ObjectId representation**.

It does **not** mean:

> “This job exists.”

Those are two separate questions.

```text
isValid()
=
Is the ID structurally valid?

findById()
=
Does a document exist with this ID?
```

---

# 22. `findById()` can return `null`

Example:

```js
const job = await Job.findById(jobId);
```

Possible results:

```text
document
```

or:

```text
null
```

So this is important:

```js
if (!job) {
  return res.status(404).json({
    message: "Job not found"
  });
}
```

A very common beginner error is assuming:

```js
const job = await Job.findById(id);
job.title
```

will always work.

It doesn't.

---

# 23. `orFail()`

Mongoose has a useful method:

```js
const job = await Job
  .findById(jobId)
  .orFail();
```

### What does `orFail()` mean?

Roughly:

> “If no document is found, throw an error instead of quietly returning null.”

So:

```text
found
 ↓
return document

not found
 ↓
throw error
```

This can be useful when you have centralized error handling.

For example:

```js
const job = await Job
  .findById(jobId)
  .orFail(new Error("Job not found"));
```

Then your error middleware can convert that into a proper HTTP response.

This can make service code cleaner.

---

# 24. `insertMany()`

Suppose you're importing many jobs:

```js
await Job.insertMany([
  {
    title: "Backend Engineer",
    ...
  },
  {
    title: "Frontend Engineer",
    ...
  },
  {
    title: "ML Engineer",
    ...
  }
]);
```

### Meaning

`insertMany()`:

> Inserts multiple documents in one operation.

Useful for:

```text
data imports
seed data
batch ingestion
migrations
```

It is generally more suitable than doing:

```js
await Job.create(job1);
await Job.create(job2);
await Job.create(job3);
```

for large batches.

---

# 25. `replaceOne()` vs `updateOne()`

This distinction is extremely important.

### `updateOne()`

You modify selected fields:

```js
await Job.updateOne(
  { _id: jobId },
  {
    $set: {
      title: "Senior Backend Engineer"
    }
  }
);
```

The rest of the document remains.

### `replaceOne()`

You essentially replace the document's contents with the replacement document.

Conceptually:

```text
updateOne
=
modify parts

replaceOne
=
replace the document
```

Do not casually use `replaceOne()` when you only intend to modify one field.

---

# 26. `deleteOne()` vs `findOneAndDelete()`

### `deleteOne()`

```js
await Job.deleteOne({
  _id: jobId
});
```

You primarily get the operation result.

### `findOneAndDelete()`

```js
const deletedJob = await Job.findOneAndDelete({
  _id: jobId
});
```

This lets you obtain the deleted document.

Conceptually:

```text
deleteOne()
=
“Delete it.”

findOneAndDelete()
=
“Find it, delete it, and give me the deleted document.”
```

That difference can matter when you need to emit events or write an audit record.

---

# 27. Soft delete

Often you shouldn't physically delete important data immediately.

Instead:

```js
{
  deletedAt: null
}
```

When deleted:

```js
{
  deletedAt: new Date()
}
```

Then your normal queries use:

```js
{
  deletedAt: null
}
```

or an appropriate condition.

This is called:

> **Soft deletion**

Benefits can include:

```text
recovery
auditing
historical records
safer accidental deletion handling
```

But it also means every query needs to account for deleted documents.

That's a tradeoff.

---

# 28. New JavaScript built-in: `new Date()`

You've seen:

```js
new Date()
```

This creates a JavaScript Date object representing the current time.

Example:

```js
const now = new Date();
```

You can store that in MongoDB as a BSON date.

This is different from storing:

```js
"2026-09-20"
```

as an arbitrary string.

For dates you need to query, sort, compare, or aggregate, proper date types are generally much more useful.

---

# 29. Pagination architecture

Let's build a realistic endpoint.

```js
export async function listJobs(req, res, next) {
  try {
    const page = Number(req.query.page ?? 1);
    const limit = Math.min(
      Number(req.query.limit ?? 20),
      100
    );

    const skip = (page - 1) * limit;

    const jobs = await Job
      .find({ status: "OPEN" })
      .select("title companyId location createdAt")
      .sort({ createdAt: -1 })
      .skip(skip)
      .limit(limit)
      .lean();

    res.json({
      page,
      limit,
      jobs
    });
  } catch (error) {
    next(error);
  }
}
```

Let's explain every unfamiliar bit.

---

# 30. `Number()`

```js
Number(req.query.page)
```

HTTP query parameters are strings.

So:

```text
?page=3
```

usually arrives conceptually as:

```js
"3"
```

`Number("3")` converts it to:

```js
3
```

This is a built-in JavaScript function.

---

# 31. `??` — nullish coalescing

This:

```js
req.query.page ?? 1
```

means roughly:

> Use `req.query.page`, unless it is `null` or `undefined`; otherwise use `1`.

So:

```js
undefined ?? 1
```

becomes:

```text
1
```

while:

```js
5 ?? 1
```

becomes:

```text
5
```

This is different from the `||` operator.

That distinction becomes useful when `0`, `false`, or `""` are legitimate values.

---

# 32. `Math.min()`

```js
Math.min(
  Number(req.query.limit ?? 20),
  100
)
```

ensures the limit doesn't exceed 100.

For example:

```text
limit = 20
→ 20

limit = 50
→ 50

limit = 5000
→ 100
```

`Math.min()` simply returns the smallest number you give it.

This is a common defensive-programming technique.

---

# 33. Why limiting API pagination matters

Without a maximum:

```http
GET /jobs?limit=10000000
```

could create unnecessary database and application load.

So:

```text
Client requests 10 million
          ↓
Server says maximum = 100
          ↓
database returns at most 100
```

Never trust client-supplied limits.

---

# 34. But there's a problem with page-number pagination

Imagine:

```text
page=1
page=2
page=3
...
page=100000
```

Then:

```js
.skip(1999980)
```

can become increasingly inefficient depending on the query.

For large systems, cursor-based pagination is often preferable.

---

# 35. Cursor pagination

Suppose the last item from page 1 has:

```text
createdAt = 2026-09-20T10:00:00
_id = ABC
```

The next request can say:

```http
GET /jobs?before=...
```

Then MongoDB can query from the cursor instead of skipping millions of documents.

A conceptual query might look like:

```js
Job.find({
  status: "OPEN",
  createdAt: { $lt: cursorDate }
})
.sort({
  createdAt: -1
})
.limit(20);
```

This needs careful design when timestamps can be identical, which is why `_id` can be used as a tie-breaker.

---

# 36. Stable pagination

Suppose two jobs have exactly the same:

```text
createdAt
```

Sorting only by `createdAt` can create ambiguous ordering.

A more deterministic ordering can be:

```js
.sort({
  createdAt: -1,
  _id: -1
})
```

This gives you:

```text
primary ordering
+
tie breaker
```

This is a small detail, but it's exactly the kind of detail that matters in production.

---

# 37. Repository layer — do we need it?

You may see:

```text
Controller
   ↓
Service
   ↓
Repository
   ↓
Model
```

For example:

```js
async function findOpenJobs(filters) {
  return Job
    .find({
      ...filters,
      status: "OPEN"
    })
    .lean()
    .exec();
}
```

The repository isolates persistence operations.

This can be useful for larger applications.

But don't blindly create:

```text
controller
service
repository
repositoryFactory
repositoryManager
databaseAdapter
```

for a 500-line application.

Architecture should reduce complexity, not manufacture it.

For your projects, I'd start with:

```text
Controller
   ↓
Service
   ↓
Model
```

and introduce a repository layer when it actually gives you value.

---

# 38. Business logic belongs in the service

Consider:

> A candidate cannot apply to the same job twice.

That is not merely a Mongoose schema rule.

It's a **business rule**.

Your service might do:

```js
async function applyToJob({ candidateId, jobId }) {
  // business logic here
}
```

And the database should also have an appropriate unique constraint:

```js
applicationSchema.index(
  {
    candidateId: 1,
    jobId: 1
  },
  {
    unique: true
  }
);
```

This gives you two layers:

```text
Service
=
friendly business handling

Database
=
final consistency enforcement
```

That's strong architecture.

---

# 39. Why both layers?

Suppose:

```text
Request A:
check → no application

Request B:
check → no application

Request A:
insert

Request B:
insert
```

The service-level check alone isn't sufficient under concurrency.

The unique index provides the final protection.

This is one of the biggest lessons from the entire MongoDB course.

---

# 40. Transactions in a service

Suppose applying to a job requires:

```text
1. Create application
2. Increment job application count
3. Create audit record
```

You might need a transaction depending on your consistency requirements.

Conceptually:

```js
async function applyToJob(...) {
  const session = await mongoose.startSession();

  try {
    return await session.withTransaction(async () => {

      // create application
      // increment job count
      // create audit record

    });
  } finally {
    await session.endSession();
  }
}
```

Now the **service** owns the business operation.

The controller doesn't need to understand transaction details.

That's clean separation.

---

# 41. Error middleware

Express can centralize errors.

For example:

```js
function errorHandler(err, req, res, next) {
  console.error(err);

  res.status(500).json({
    message: "Internal server error"
  });
}
```

You already saw:

```js
next(error);
```

### What is `next()`?

`next()` is an Express function that tells the framework:

> “I'm done with this middleware; move to the next middleware.”

And:

```js
next(error)
```

means:

> “There was an error; pass control to the error-handling middleware.”

This is an **Express built-in mechanism**, not a MongoDB feature.

---

# 42. Error translation

In a serious system, your error middleware can classify errors.

Conceptually:

```text
Mongoose ValidationError
       ↓
400

Duplicate key
       ↓
409

Invalid ObjectId
       ↓
400

Not found
       ↓
404

Unexpected database failure
       ↓
500/503
```

This creates a consistent API.

---

# 43. Custom application errors

You can define:

```js
class NotFoundError extends Error {
  constructor(message) {
    super(message);
    this.statusCode = 404;
  }
}
```

Then:

```js
throw new NotFoundError("Job not found");
```

Your error middleware handles it.

This is much cleaner than scattering:

```js
res.status(404)
```

through every service function.

---

# 44. `instanceof` — another JavaScript operator/method-like concept

You'll encounter:

```js
error instanceof NotFoundError
```

It asks roughly:

> “Was this object created from this class/type?”

For example:

```js
error instanceof Error
```

can tell you whether the object is an Error object or derived from one.

Useful for classifying errors.

---

# 45. Validation should happen before the database

Suppose the client sends:

```json
{
  "title": "",
  "salary": "banana"
}
```

You don't want to just send that directly to MongoDB and hope Mongoose catches everything.

A robust architecture can be:

```text
HTTP input
   ↓
request validation
   ↓
business validation
   ↓
Mongoose casting/validation
   ↓
MongoDB
```

Each layer protects a different boundary.

---

# 46. Mongoose schema validation example

```js
salary: {
  type: Number,
  required: true,
  min: 0
}
```

This gives you:

```text
type = Number
required
minimum = 0
```

But maybe your business rule is:

> Salary must be greater than zero for a full-time job.

That belongs to business logic or a more specific validator.

---

# 47. Don't put every rule into the schema

This is important.

Bad:

```text
Mongoose schema
 ├── authentication logic
 ├── permissions
 ├── hiring rules
 ├── workflow rules
 ├── notification rules
 └── database structure
```

Instead:

```text
Controller
 └── HTTP

Service
 └── business rules

Model
 └── persistence structure

Database
 └── critical data constraints
```

---

# 48. Query projection + lean together

A very common read pattern is:

```js
const jobs = await Job
  .find({ status: "OPEN" })
  .select("title companyId location createdAt")
  .sort({ createdAt: -1 })
  .limit(20)
  .lean();
```

This says:

```text
find the relevant jobs
        ↓
return only needed fields
        ↓
sort newest first
        ↓
maximum 20
        ↓
return plain objects
```

That's a very reasonable pattern for many GET APIs.

---

# 49. But don't over-optimize too early

You shouldn't decide:

```text
EVERY QUERY MUST USE LEAN
EVERY QUERY MUST SELECT 3 FIELDS
EVERY QUERY MUST HAVE A SPECIAL INDEX
```

without measuring.

Performance engineering follows:

```text
Measure
 ↓
Identify bottleneck
 ↓
Change
 ↓
Measure again
```

not:

```text
Guess
 ↓
Optimize everything
```

---

# 50. `explain()` — bringing Lesson 7 back

Suppose you have:

```js
Job.find({
  companyId,
  status: "OPEN"
});
```

You can inspect the query plan in MongoDB.

The important question isn't:

> “Does my Mongoose code look clean?”

It's:

> **“What does MongoDB actually do when this query executes?”**

That's where `explain()` matters.

You should be able to move mentally from:

```text
Mongoose
```

to:

```text
MongoDB query
```

to:

```text
query planner
```

to:

```text
index / collection scan
```

---

# 51. This is why ORM/ODM abstraction can be dangerous

A developer writes:

```js
Application
  .find(...)
  .populate(...)
  .populate(...)
  .sort(...)
```

It looks innocent.

But underneath could be:

```text
large collection scan
+
multiple lookups
+
large document hydration
+
sorting
+
huge response
```

The abstraction hides complexity.

Your job is to **see through the abstraction**.

That's one of the most valuable backend skills you're developing.

---

# 52. `bulkWrite()` in a realistic migration

Imagine every old job has:

```text
status = "OPEN"
```

but you introduce:

```text
employmentType
```

and need to backfill:

```text
FULL_TIME
```

You could process in batches and use:

```js
await Job.bulkWrite([
  {
    updateMany: {
      filter: {
        employmentType: { $exists: false }
      },
      update: {
        $set: {
          employmentType: "FULL_TIME"
        }
      }
    }
  }
]);
```

### New method keyword: `$exists`

You already saw lots of query operators, but this one is worth emphasizing.

```js
{
  employmentType: {
    $exists: false
  }
}
```

means:

> Find documents where this field does not exist.

And:

```js
$exists: true
```

means:

> Find documents where the field exists.

---

# 53. A migration isn't just a script

A serious migration needs to consider:

```text
How many documents?
How long will it run?
Can it be restarted?
Can it be interrupted?
Does it need batching?
Is it idempotent?
Will it lock/block anything?
Are indexes affected?
How do we verify success?
```

This is where database engineering becomes much more than CRUD.

---

# 54. `bulkWrite()` operations

`bulkWrite()` can contain different operation types.

For example:

```js
[
  {
    insertOne: {...}
  },
  {
    updateOne: {...}
  },
  {
    updateMany: {...}
  },
  {
    deleteOne: {...}
  },
  {
    replaceOne: {...}
  }
]
```

So think of it as:

> **A batch of different database operations.**

Very useful for large-scale data work.

---

# 55. Ordered vs unordered bulk operations

You may encounter:

```js
{
  ordered: false
}
```

This means the server doesn't have to process the batch strictly one operation after another in the supplied order, subject to the driver's/server's processing behavior.

Why would that be useful?

For large independent batches, one failing operation doesn't necessarily need to prevent unrelated operations from being processed.

But whether ordered or unordered is appropriate depends on dependencies between operations.

---

# 56. `updateMany()` can be extremely powerful

Example:

```js
await Job.updateMany(
  {
    status: "OPEN",
    expiresAt: { $lt: new Date() }
  },
  {
    $set: {
      status: "CLOSED"
    }
  }
);
```

This means:

```text
Find expired open jobs
       ↓
set them all to CLOSED
```

MongoDB performs the set-based operation.

You don't have to:

```text
fetch 100,000 jobs
↓
loop in Node
↓
save each
```

That would often be unnecessarily expensive.

---

# 57. New JavaScript method: `Array.map()`

You will encounter this constantly in backend code.

```js
const names = users.map(user => user.name);
```

It means:

> Transform every item in an array into something else.

Example:

```text
users
 ↓
user.name
 ↓
names
```

If:

```js
[
  { name: "A" },
  { name: "B" },
  { name: "C" }
]
```

then:

```js
users.map(user => user.name)
```

produces:

```js
["A", "B", "C"]
```

This isn't a MongoDB method. It's a JavaScript array method.

I will keep explicitly identifying these distinctions.

---

# 58. JavaScript `filter()`

Another one:

```js
const activeUsers = users.filter(
  user => user.status === "ACTIVE"
);
```

It means:

> Keep only the array elements that satisfy a condition.

So:

```text
map
=
transform

filter
=
keep some
```

Notice how similar this is to MongoDB aggregation expressions.

---

# 59. Do not confuse JavaScript `.filter()` with MongoDB `$filter`

JavaScript:

```js
users.filter(...)
```

runs in:

```text
Node.js
```

MongoDB:

```js
{
  $filter: ...
}
```

runs in:

```text
MongoDB aggregation engine
```

Same conceptual idea, different execution environment.

This distinction is extremely important for performance.

If you pull 500,000 records into Node and then use JavaScript `.filter()`, that's very different from filtering them efficiently in MongoDB.

---

# 60. The “where should this happen?” question

For every operation, ask:

```text
Should this happen in:
1. Browser?
2. Node.js?
3. MongoDB?
```

For example:

### Formatting a display name

Probably:

```text
Node/frontend
```

### Filtering 5 million database records

Usually:

```text
MongoDB
```

### Business rule

Usually:

```text
Node service
```

### Database uniqueness

Definitely:

```text
MongoDB index
```

This question alone will improve your backend architecture considerably.

---

# 61. A complete request example

Imagine:

```http
POST /api/jobs
Authorization: Bearer ...
```

Body:

```json
{
  "title": "Backend Engineer",
  "location": "Pune"
}
```

Flow:

```text
Request
 ↓
Express Router
 ↓
auth middleware
 ↓
request validation
 ↓
JobController
 ↓
JobService
 ↓
check authorization
 ↓
create Job document
 ↓
Mongoose
 ↓
MongoDB
 ↓
result
 ↓
Service
 ↓
Controller
 ↓
JSON response
```

That is the complete architecture you should visualize.

---

# 62. What happens when something goes wrong?

Suppose the database has a unique constraint and creation fails.

```text
MongoDB
   ↓
duplicate key error
   ↓
MongoDB driver
   ↓
Mongoose
   ↓
Service
   ↓
Controller
   ↓
next(error)
   ↓
Error middleware
   ↓
409 response
```

The error isn't supposed to randomly leak through the system.

Each layer participates in handling it.

---

# 63. Real JobForge example

Imagine:

```text
POST /applications
```

The service might conceptually do:

```text
1. Validate candidate
2. Validate job
3. Check whether job accepts applications
4. Ensure candidate isn't already applied
5. Create application
6. Emit/update relevant state
7. Return application
```

Now ask:

> Which parts belong where?

A reasonable split:

```text
Controller
→ HTTP input/output

Service
→ business logic

Mongoose
→ persistence interaction

MongoDB
→ indexes / atomicity / constraints
```

That's exactly the skill we're building.

---

# 64. One subtle architecture problem: returning Mongoose documents directly

You may see:

```js
res.json(user);
```

Sometimes this is perfectly fine.

But in larger systems, you may want an explicit response shape:

```js
res.json({
  id: user._id,
  name: user.name,
  email: user.email
});
```

Why?

Because your database model and public API model don't necessarily have to be identical.

This separation becomes valuable when your database evolves.

---

# 65. DTO idea

You may hear:

> DTO — Data Transfer Object

A DTO is basically:

> A defined shape for data entering or leaving a layer/system.

For example:

```js
{
  title,
  location,
  salary
}
```

might be your create-job request DTO.

While your MongoDB document might contain:

```text
title
location
salary
companyId
createdAt
updatedAt
status
internalFlags
```

You don't necessarily want the client to control or see everything.

This is a major concept we'll use later in your Spring Boot backend too.

---

# 66. Never trust the frontend

Suppose the frontend sends:

```json
{
  "role": "ADMIN"
}
```

and your database model supports:

```text
role
```

Should you simply do:

```js
User.create(req.body)
```

No.

Because the client is not authoritative.

A user could manually send:

```json
{
  "role": "ADMIN"
}
```

even if your UI never exposes that option.

Your backend must construct trusted data.

For example:

```js
User.create({
  name: req.body.name,
  email: req.body.email,
  role: "USER"
});
```

or otherwise derive role through authorized server-side logic.

---

# 67. Mass assignment problem

This issue is worth remembering.

Bad:

```js
User.findByIdAndUpdate(
  userId,
  req.body
);
```

The client might update fields you never intended to expose.

Potentially:

```json
{
  "name": "New Name",
  "role": "ADMIN",
  "isVerified": true
}
```

A safer approach is to explicitly whitelist fields:

```js
const allowedUpdates = {
  name: req.body.name,
  bio: req.body.bio,
  profileImage: req.body.profileImage
};
```

Then:

```js
await User.updateOne(
  { _id: userId },
  { $set: allowedUpdates }
);
```

This is an application-security concept.

---

# 68. One more built-in: `Object.keys()`

You'll often need:

```js
Object.keys(req.body)
```

It returns the object's own enumerable property names.

Example:

```js
Object.keys({
  name: "A",
  age: 22
});
```

produces:

```js
["name", "age"]
```

It can be useful when validating or filtering incoming objects.

---

# 69. Another useful built-in: `JSON.stringify()`

You've probably seen:

```js
JSON.stringify(data)
```

It converts a JavaScript value into a JSON string.

Example:

```js
JSON.stringify({
  name: "A"
});
```

becomes conceptually:

```text
'{"name":"A"}'
```

This is useful for:

```text
logging
HTTP payloads
serialization
debugging
```

But don't unnecessarily stringify objects just to log them; structured logging is usually better in production systems.

---

# 70. The backend discipline I want you to develop

For every database operation, ask:

### Question 1

**What data am I trying to access?**

### Question 2

**What is the access pattern?**

### Question 3

**Can this operation be atomic?**

### Question 4

**Does the database need an index or constraint?**

### Question 5

**Should this happen in MongoDB or Node?**

### Question 6

**Can concurrent requests break this?**

### Question 7

**What happens when the database fails?**

### Question 8

**Am I returning sensitive/internal data?**

That is production thinking.

---

# 71. Let's dissect a bad endpoint

Consider:

```js
app.post("/applications", async (req, res) => {
  try {
    const existing = await Application.findOne({
      candidateId: req.body.candidateId,
      jobId: req.body.jobId
    });

    if (existing) {
      return res.status(409).json({
        message: "Already applied"
      });
    }

    const application = await Application.create(req.body);

    const job = await Job.findById(req.body.jobId);

    job.applicationCount++;

    await job.save();

    res.status(201).json(application);
  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
});
```

Looks reasonable.

But now you can identify many issues.

---

# 72. Problem 1 — client controls too much

```js
Application.create(req.body)
```

Dangerous.

The client might send fields it shouldn't control:

```json
{
  "candidateId": "...",
  "jobId": "...",
  "status": "HIRED",
  "approvedBy": "..."
}
```

The server should construct the trusted document.

---

# 73. Problem 2 — race condition

```text
find existing
 ↓
nothing
 ↓
create
```

Two requests can race.

Solution:

```text
unique compound index
+
service-level check
```

---

# 74. Problem 3 — non-atomic multi-document update

You create the application:

```text
success
```

Then increment job count:

```text
failure
```

Now:

```text
Application exists
but
job count wasn't updated
```

Depending on your requirements, that may need a transaction or a different consistency strategy.

---

# 75. Problem 4 — read-modify-write counter

```js
job.applicationCount++;
await job.save();
```

Potential concurrency issue.

Better:

```js
await Job.updateOne(
  { _id: jobId },
  { $inc: { applicationCount: 1 } }
);
```

or use the operation inside a suitable transaction if multiple changes must succeed together.

---

# 76. Problem 5 — HTTP and business logic mixed

The route handles everything.

We should move business behavior into:

```text
ApplicationService
```

---

# 77. Problem 6 — error handling is too generic

Everything becomes:

```text
500 Internal Server Error
```

Even:

```text
invalid ID
duplicate application
validation error
```

A proper error system distinguishes them.

---

# 78. Problem 7 — multiple unnecessary queries

We may already know:

```text
candidate exists?
job exists?
application exists?
```

Some of these checks can potentially be combined or redesigned depending on the workflow.

Performance begins with good access-pattern design.

---

# 79. A much better conceptual design

```text
POST /applications
        ↓
Auth middleware
        ↓
Request validation
        ↓
ApplicationController
        ↓
ApplicationService
        ↓
 ┌──────────────────────────┐
 │ business rules           │
 │ authorization            │
 │ duplicate handling       │
 │ transaction if required  │
 └──────────────────────────┘
        ↓
Mongoose
        ↓
MongoDB
```

This architecture is much easier to maintain.

---

# 80. And now something important about your career

You're learning MongoDB right now.

But the deeper objective is not:

> “Become the person who knows 200 MongoDB methods.”

It is:

> **Become the backend developer who understands what the database is doing.**

Knowing:

```js
User.find()
```

is beginner knowledge.

Knowing:

```text
Why is this query slow?
Which index does it need?
Can this race?
Should this be atomic?
Should I use aggregation?
Should I use populate?
Should I denormalize?
Does this require a transaction?
What happens under concurrent requests?
What happens when MongoDB becomes unavailable?
```

is backend engineering.

---

# 81. New methods/functions from this section

Keep this small reference for today:

|Method / function|Simple meaning|
|---|---|
|`.select()`|Choose which fields come back|
|`.exec()`|Explicitly execute a Mongoose query|
|`.exists()`|Check whether a matching document exists|
|`.countDocuments()`|Count documents matching a filter|
|`.estimatedDocumentCount()`|Estimate total documents in a collection|
|`.distinct()`|Return unique values of a field|
|`.orFail()`|Throw if no matching document is found|
|`.insertMany()`|Insert multiple documents|
|`.replaceOne()`|Replace an entire matching document|
|`.findOneAndDelete()`|Find, delete, and return the deleted document|
|`.updateMany()`|Update many matching documents|
|`ObjectId.isValid()`|Check whether an ID has valid ObjectId form|
|`Array.isArray()`|Check whether a value is an array|
|`Number()`|Convert a value to a number|
|`Math.min()`|Return the smallest number|
|`Object.keys()`|Get an object's property names|
|`JSON.stringify()`|Convert a JS value to a JSON string|
|`new Date()`|Create a Date object for the current time|
|`.map()`|Transform every array item|
|`.filter()`|Keep array items matching a condition|
|`instanceof`|Check an object's class/type relationship|
|`??`|Use a fallback for `null`/`undefined`|

These are not things you need to memorize like vocabulary. The important part is recognizing them when you see real code.

---

# 82. Your new Mongoose mental model

At the beginning of Lesson 14, you had:

```text
Mongoose
=
MongoDB helper for Node
```

Now I want you to think:

```text
                    NODE.JS
                       │
             ┌─────────┴─────────┐
             │                   │
        Application            Mongoose
          logic                   │
             │              MongoDB Driver
             │                   │
             └──────────┬────────┘
                        ↓
                    MongoDB
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
       Queries       Indexes      Transactions
          ↓             ↓             ↓
      Aggregation   Replication    Sharding
                        │
                   Storage Engine
```

Mongoose sits in the middle, but **MongoDB's real capabilities still matter**.

---

# 83. Lesson 14 progress

We're now well into the production side:

```text
✓ Mongoose architecture
✓ Schema / Model / Document
✓ Casting
✓ Validation
✓ Middleware
✓ Change tracking
✓ save vs update
✓ Atomic updates
✓ Upserts
✓ arrayFilters
✓ bulkWrite
✓ aggregation
✓ populate
✓ lean
✓ sessions
✓ transactions
✓ connection pooling
✓ timeouts
✓ retries
✓ idempotency
✓ error handling
✓ pagination
✓ cursor pagination
✓ projection
✓ soft deletion
✓ service/controller architecture
✓ API/database boundaries
✓ mass-assignment risks
✓ realistic MongoDB backend design
```

There is still one particularly valuable portion of Lesson 14 left: **advanced Mongoose behavior and performance** — especially `populate()` vs `$lookup` in real scenarios, `bulkWrite()` optimization, middleware edge cases, sessions, transaction retries, connection-pool tuning, query timeouts, and how to diagnose a slow Mongoose endpoint from HTTP request all the way down to MongoDB.

That is the part that will make this lesson feel much closer to **professional backend development** than a normal Mongoose course.