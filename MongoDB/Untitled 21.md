
# Gap Module 4 — Advanced Validation + Transactions

This module fills the remaining useful gaps from **Lessons 8 and 9**.

We’ll cover:

```text
Lesson 8
├── pattern
├── anyOf
├── oneOf
├── allOf
├── not
├── multipleOf
├── conditional validation using query operators
└── finding existing invalid documents

Lesson 9
├── maxCommitTimeMS
├── retryWrites
└── fully connecting transaction retry behavior
```

And there is one important correction from the previous gap map:

> **`if` / `then` / `else` should NOT be taught as MongoDB `$jsonSchema` keywords.**

MongoDB's `$jsonSchema` implementation is based on JSON Schema Draft 4 with MongoDB-specific extensions and omissions. The supported keyword set includes `anyOf`, `oneOf`, `allOf`, `not`, `pattern`, and `multipleOf`, but `if`/`then`/`else` are not part of the supported subset. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

That's exactly the kind of distinction I want to catch rather than blindly copy from a gap list.

---

# PART A — Lesson 8: Advanced Schema Validation

## 1. Why advanced validation matters

You already learned:

```text
required
properties
bsonType
enum
minimum
maximum
minItems
maxItems
additionalProperties
```

Those handle a lot.

But real data often has relationships like:

> “This field must match a pattern.”

> “The document can follow schema A or schema B.”

> “The document must satisfy all of these sub-rules.”

> “The document must satisfy exactly one of these formats.”

> “This value must be divisible by 10.”

Now we need the more advanced JSON Schema composition tools.

---

# 2. `pattern`

Suppose a candidate has:

```text
phone = "9876543210"
```

and you want the stored value to follow a specific string pattern.

MongoDB JSON Schema supports:

```js
{
  $jsonSchema: {
    bsonType: "object",

    properties: {
      phone: {
        bsonType: "string",
        pattern: "^[0-9]{10}$"
      }
    }
  }
}
```

### What does `pattern` mean?

> The string must match the supplied regular expression.

So:

```text
^[0-9]{10}$
```

roughly means:

```text
start
↓
exactly 10 digits
↓
end
```

MongoDB's JSON Schema implementation supports `pattern` as a string-validation keyword. ([MongoDB](https://www.mongodb.com/es/docs/v7.0/reference/operator/query/jsonSchema/?utm_source=chatgpt.com "$jsonSchema (query predicate operador) - Manual de base de datos v7.0 - MongoDB Docs"))

---

# 3. What is a regular expression?

A regular expression, often called **regex**, is a pattern describing text.

For example:

```text
^[0-9]+$
```

means:

> one or more digits only.

And:

```text
^ENG-[0-9]{4}$
```

means something like:

```text
ENG-1234
```

but not:

```text
ENGINE-1234
```

Don't worry about becoming a regex expert. For MongoDB validation, you mainly need to recognize the concept.

---

# 4. `pattern` is not the same as `$regex`

This is another layer distinction.

### JSON Schema

```js
pattern: "^[0-9]{10}$"
```

is a **validation rule**.

### MongoDB query

```js
{
  phone: {
    $regex: "^[0-9]{10}$"
  }
}
```

is a **query condition**.

So:

```text
pattern
=
“Is the stored value valid?”

$regex
=
“Does this document match my search?”
```

Same regex concept, different job.

---

# 5. `anyOf`

Now we get into schema composition.

Suppose a `contact` field can be:

```text
email contact
OR
phone contact
```

You might define:

```js
{
  $jsonSchema: {
    properties: {
      contact: {
        anyOf: [
          {
            bsonType: "string",
            pattern: "^[^@]+@[^@]+\\.[^@]+$"
          },
          {
            bsonType: "string",
            pattern: "^[0-9]{10}$"
          }
        ]
      }
    }
  }
}
```

### What does `anyOf` mean?

> The value must match **at least one** of the supplied schemas.

Think:

```text
Schema A ✓
OR
Schema B ✓
OR
Schema C ✓

at least one must work
```

MongoDB's supported JSON Schema keywords include `anyOf`. ([MongoDB](https://www.mongodb.com/es/docs/v7.0/reference/operator/query/jsonSchema/?utm_source=chatgpt.com "$jsonSchema (query predicate operador) - Manual de base de datos v7.0 - MongoDB Docs"))

---

# 6. `oneOf`

Now:

```js
{
  oneOf: [
    schemaA,
    schemaB
  ]
}
```

means:

> Exactly one of the supplied schemas must match.

This sounds similar to `anyOf`, but the distinction matters.

### `anyOf`

```text
A
OR
B

A and B can both match.
```

### `oneOf`

```text
Exactly one matches.
```

So:

```text
anyOf
=
one or more

oneOf
=
exactly one
```

MongoDB supports `oneOf` in its JSON Schema subset. ([MongoDB](https://www.mongodb.com/es/docs/v7.0/reference/operator/query/jsonSchema/?utm_source=chatgpt.com "$jsonSchema (query predicate operador) - Manual de base de datos v7.0 - MongoDB Docs"))

---

# 7. Why `oneOf` can be tricky

Imagine:

```js
oneOf: [
  {
    bsonType: "number"
  },
  {
    bsonType: "number"
  }
]
```

Any number matches **both** schemas.

Therefore:

```text
matches A
+
matches B
=
2 matches
```

and `oneOf` fails because it requires exactly one.

This is an important logical distinction.

---

# 8. `allOf`

Now:

```js
{
  allOf: [
    schemaA,
    schemaB
  ]
}
```

means:

> The value must satisfy **every** supplied schema.

Think:

```text
A ✓
AND
B ✓
AND
C ✓
```

All must hold.

MongoDB supports `allOf` in its JSON Schema implementation. ([MongoDB](https://www.mongodb.com/es/docs/v7.0/reference/operator/query/jsonSchema/?utm_source=chatgpt.com "$jsonSchema (query predicate operador) - Manual de base de datos v7.0 - MongoDB Docs"))

---

# 9. A useful `allOf` example

Suppose:

```text
username
```

must:

```text
be a string
AND
have minimum length
AND
match a pattern
```

Conceptually:

```js
{
  allOf: [
    {
      bsonType: "string"
    },
    {
      minLength: 5
    },
    {
      pattern: "^[a-zA-Z0-9_]+$"
    }
  ]
}
```

Every constraint must hold.

---

# 10. `anyOf` vs `oneOf` vs `allOf`

This is worth putting into one picture:

```text
ANYOF
A OR B OR C
→ at least one

ONEOF
A OR B OR C
→ exactly one

ALLOF
A AND B AND C
→ all
```

This is essentially logical composition of schemas.

---

# 11. `not`

Now the opposite concept.

```js
{
  not: {
    bsonType: "string"
  }
}
```

means:

> The value must **not** satisfy the supplied schema.

MongoDB's JSON Schema subset supports `not`. ([MongoDB](https://www.mongodb.com/es/docs/v7.0/reference/operator/query/jsonSchema/?utm_source=chatgpt.com "$jsonSchema (query predicate operador) - Manual de base de datos v7.0 - MongoDB Docs"))

So:

```text
string
→ invalid

number
→ potentially valid
```

---

# 12. Why is `not` useful?

Suppose you explicitly want to reject a particular representation.

For example, perhaps a field must not be a string because your application requires a number:

```js
{
  not: {
    bsonType: "string"
  }
}
```

Though in many practical situations:

```js
bsonType: "number"
```

is clearer.

So don't use `not` just because you can.

---

# 13. `multipleOf`

Suppose:

```text
quantity
```

must be in multiples of 5:

```text
5
10
15
20
...
```

You can use:

```js
{
  multipleOf: 5
}
```

### Meaning

> The number must be evenly divisible by 5.

MongoDB supports `multipleOf` in its JSON Schema subset. ([MongoDB](https://www.mongodb.com/es/docs/v7.0/reference/operator/query/jsonSchema/?utm_source=chatgpt.com "$jsonSchema (query predicate operador) - Manual de base de datos v7.0 - MongoDB Docs"))

---

# 14. Example

```js
{
  $jsonSchema: {
    bsonType: "object",

    properties: {
      quantity: {
        bsonType: "int",
        multipleOf: 5
      }
    }
  }
}
```

Then:

```text
10 → valid
15 → valid
25 → valid

12 → invalid
17 → invalid
```

This is a small feature, but it can be useful for:

```text
batch sizes
pack quantities
pricing units
inventory increments
```

---

# 15. Conditional validation — the correct MongoDB approach

This is where I want to correct the earlier gap list.

A general JSON Schema tutorial may teach:

```text
if
then
else
```

But that's not part of MongoDB's supported `$jsonSchema` subset.

So how do we implement a condition like:

> If `employmentType = FULL_TIME`, `salary` must exist.

MongoDB can combine `$expr` or query operators with `$jsonSchema` in the collection validator. MongoDB explicitly supports combining JSON Schema with query-operator validation. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 16. Example: conditional validation with `$or`

Suppose:

```text
FULL_TIME
→ salary required

INTERNSHIP
→ salary optional
```

Conceptually, you can build validation logic around:

```text
status is not FULL_TIME
OR
salary exists
```

For example, using query operators within a validator:

```js
{
  $or: [
    {
      employmentType: {
        $ne: "FULL_TIME"
      }
    },
    {
      salary: {
        $exists: true
      }
    }
  ]
}
```

Combined with your `$jsonSchema` rules, you can enforce the conditional business/data constraint.

The idea is more important than memorizing this exact validator:

```text
conditional rule
=
query-logic validation

structural rule
=
$jsonSchema
```

---

# 17. `$expr` for cross-field rules

Suppose:

```text
salary.min
salary.max
```

must satisfy:

```text
salary.min <= salary.max
```

JSON Schema by itself isn't ideal for comparing two field values.

You can use:

```js
{
  $expr: {
    $lte: [
      "$salary.min",
      "$salary.max"
    ]
  }
}
```

Now the database compares the values.

This is the same `$expr` concept you learned in earlier lessons.

---

# 18. Think of validation as two categories

This is now even clearer.

### Structural validation

```text
field exists
field type
length
pattern
enum
array size
```

Use:

```text
$jsonSchema
```

### Relational/cross-field validation

```text
min <= max
endDate > startDate
if type=A, field B required
```

Use appropriate:

```text
query operators
$expr
$and
$or
```

or application/business logic when the rule belongs there.

---

# 19. Finding invalid documents

This is a particularly valuable missing concept.

Suppose you've added a new validator.

What about the **old data**?

MongoDB doesn't magically rewrite existing documents just because you created a validator.

There may be existing invalid records.

MongoDB explicitly documents querying for documents that match or don't match a schema using `$jsonSchema` in query conditions. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/use-json-schema-query-conditions/?utm_source=chatgpt.com "Query for and Modify Valid or Invalid Documents - Database Manual - MongoDB Docs"))

---

# 20. Define a schema expression

For example:

```js
const studentSchema = {
  $jsonSchema: {
    bsonType: "object",

    required: [
      "name",
      "age"
    ],

    properties: {
      name: {
        bsonType: "string"
      },

      age: {
        bsonType: "int",
        minimum: 18
      }
    }
  }
};
```

Now you can query:

```js
db.students.find(studentSchema)
```

to find documents matching the schema.

MongoDB documents this pattern explicitly. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/use-json-schema-query-conditions/?utm_source=chatgpt.com "Query for and Modify Valid or Invalid Documents - Database Manual - MongoDB Docs"))

---

# 21. Finding invalid documents

Here's the interesting part:

```js
db.students.find({
  $nor: [
    studentSchema
  ]
})
```

Conceptually:

```text
studentSchema
   ↓
which documents match?
   ↓
NOR / NOT
   ↓
which documents don't match?
```

MongoDB documents this exact approach for locating non-conforming documents. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/use-json-schema-query-conditions/?utm_source=chatgpt.com "Query for and Modify Valid or Invalid Documents - Database Manual - MongoDB Docs"))

This is extremely useful during migrations.

---

# 22. Why is this powerful?

Imagine your collection has:

```text
10 million documents
```

You introduce:

```text
new validation rule
```

Before enforcing it strictly, you can inspect:

```text
which existing documents violate it?
```

Then:

```text
find invalid
 ↓
repair
 ↓
verify
 ↓
enable strict validation
```

That's production-quality schema management.

---

# 23. Schema validation migration workflow

A mature process:

```text
Existing data
      ↓
design new schema
      ↓
find non-conforming documents
      ↓
repair/migrate
      ↓
verify
      ↓
enable validator
      ↓
monitor
```

Not:

```text
enable validator
↓
hope existing data is okay
```

This is one of the most practical additions to Lesson 8.

---

# 24. `additionalProperties: false` revisited

Remember this?

```js
additionalProperties: false
```

means:

> Reject properties not explicitly described.

But MongoDB automatically has:

```text
_id
```

so if you use:

```text
additionalProperties: false
```

you need to account for `_id` in the schema.

MongoDB's current documentation specifically warns about this behavior. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/json-schema-tips/?utm_source=chatgpt.com "Tips for JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 25. Why strict schemas can surprise you

Suppose:

```js
{
  $jsonSchema: {
    properties: {
      name: {
        bsonType: "string"
      }
    },

    additionalProperties: false
  }
}
```

A document containing:

```text
_id
name
```

can fail because:

```text
_id
```

wasn't allowed by your explicit property definitions.

So:

> Strict validation needs to be designed with MongoDB's automatically generated fields in mind.

---

# 26. Missing vs `null`

Another subtle validation point.

These are different:

```js
{
  bio: null
}
```

and:

```js
{
  // bio missing
}
```

If you say:

```js
bio: {
  bsonType: "string"
}
```

you're not necessarily saying:

> `bio` must exist.

You need:

```js
required: ["bio"]
```

if existence is required.

MongoDB's documentation explicitly distinguishes missing fields from fields whose value is `null`. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/json-schema-tips/?utm_source=chatgpt.com "Tips for JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 27. If `null` is legitimate

You can say:

```js
bio: {
  bsonType: [
    "string",
    "null"
  ]
}
```

Now:

```text
string → valid
null → valid
```

But:

```text
number → invalid
```

This is important for real APIs where fields can intentionally be empty.

---

# 28. A realistic JobForge validator

Let's combine several concepts.

Suppose:

```js
{
  $jsonSchema: {
    bsonType: "object",

    required: [
      "title",
      "status",
      "employmentType"
    ],

    properties: {
      title: {
        bsonType: "string",
        minLength: 3,
        maxLength: 120
      },

      status: {
        bsonType: "string",
        enum: [
          "DRAFT",
          "OPEN",
          "CLOSED",
          "EXPIRED"
        ]
      },

      employmentType: {
        bsonType: "string",
        enum: [
          "FULL_TIME",
          "PART_TIME",
          "INTERNSHIP"
        ]
      },

      skills: {
        bsonType: "array",
        minItems: 1,
        maxItems: 30,
        uniqueItems: true
      }
    }
  }
}
```

Now we've built a meaningful database-level structure.

---

# 29. Add a conditional rule

We can combine that with query logic such as:

```text
If employmentType = FULL_TIME
→ salary should exist
```

Now the validator conceptually becomes:

```text
$and
 ├── $jsonSchema
 └── conditional query rule
```

This gives you:

```text
structure
+
cross-field rules
```

MongoDB explicitly supports combining `$jsonSchema` with query-operator validation. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 30. When not to put the rule into MongoDB

Suppose:

> A recruiter cannot lower the salary after an offer has been accepted.

That's not merely a document-shape rule.

It depends on:

```text
workflow state
actor permissions
business history
```

That belongs primarily in:

```text
application/business logic
```

Database validation should protect **data integrity**, not become the entire business engine.

---

# PART B — Lesson 9: Advanced Transactions

Your original Lesson 9 is already strong.

We're adding three things:

```text
maxCommitTimeMS
retryWrites
full transaction retry mental model
```

---

# 31. `maxCommitTimeMS`

Suppose you have:

```js
session.startTransaction({
  maxCommitTimeMS: 1000
});
```

### What does it mean?

It places a limit on how long the **commit action** can run before timing out.

MongoDB's current Node.js driver documentation lists `maxCommitTimeMS` as a transaction option controlling how long the commit action may run, in milliseconds. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

So:

```text
maxCommitTimeMS: 1000
```

means roughly:

> Don't let the transaction's commit operation run beyond about one second.

Important:

> This is not the same thing as “the entire transaction must finish within one second.”

It specifically concerns the commit action.

---

# 32. Why does commit deserve its own timeout?

Because:

```text
transaction work
```

and:

```text
commit
```

are not conceptually identical.

You might have:

```text
operations execute
      ↓
commit
      ↓
replication/acknowledgement work
```

A system can therefore expose separate control over commit behavior.

---

# 33. `retryWrites`

You may see a connection URI/configuration containing:

```text
retryWrites=true
```

The Node.js driver currently enables retryable writes by default. The option controls whether supported write operations can be retried by the driver after certain retryable failures. ([MongoDB](https://www.mongodb.com/docs/drivers/node/current/connect/connection-options/?utm_source=chatgpt.com "Specify Connection Options - Node.js Driver - MongoDB Docs"))

### Simple meaning

> Allow the driver to retry certain write operations when a retryable failure occurs.

---

# 34. Is `retryWrites` the same as transaction retry?

No.

This is extremely important.

```text
retryable write
≠
transaction retry
```

For transactions, MongoDB distinguishes errors such as:

```text
TransientTransactionError
UnknownTransactionCommitResult
```

and transaction handling has its own retry semantics. The convenient `withTransaction()` API incorporates retry logic for these transaction error categories. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 35. Why doesn't `retryWrites` simply solve transactions?

Because a transaction is a **multi-operation unit**.

Suppose:

```text
transaction
 ├── operation A
 ├── operation B
 └── operation C
```

If the transaction becomes transiently invalid:

```text
retry the transaction
```

is fundamentally different from:

```text
retry operation B
```

You need to preserve the transaction's semantics.

MongoDB's transaction documentation explicitly notes that individual writes within a transaction aren't simply made retryable by `retryWrites`; a transaction marked `TransientTransactionError` may require retrying the transaction as a whole. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

---

# 36. `TransientTransactionError`

Suppose:

```text
transaction running
      ↓
primary steps down
```

You might receive:

```text
TransientTransactionError
```

### Meaning

Roughly:

> The transaction encountered a transient condition and the transaction should be retried.

The convenient transaction API handles this retry logic for you. The core API requires you to implement the retry structure yourself. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 37. `UnknownTransactionCommitResult`

Now a different situation.

Imagine:

```text
transaction operations succeed
      ↓
commit sent
      ↓
network problem
```

Your application doesn't know:

```text
Did commit succeed?
```

You might receive:

```text
UnknownTransactionCommitResult
```

This means:

> The client can't confidently determine whether the commit completed.

That's fundamentally different from:

```text
transaction definitely failed
```

---

# 38. Why retry the commit?

Suppose:

```text
commit request
 ↓
network lost
```

If you simply restart the entire business operation, you could create unintended duplicates or side effects.

Transaction retry handling therefore distinguishes:

```text
retry entire transaction
```

from:

```text
retry commit
```

The Node.js driver's convenient transaction API handles these categories automatically according to its documented retry behavior. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 39. Core API mental model

Conceptually:

```text
START TRANSACTION
      ↓
operations
      ↓
commit
```

If:

```text
TransientTransactionError
```

then:

```text
restart whole transaction
```

If:

```text
UnknownTransactionCommitResult
```

then:

```text
retry commit
```

until success or another non-retryable error occurs, following the driver's transaction guidance. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 40. Convenient API

The easier pattern is:

```js
await session.withTransaction(async () => {

  await collection1.updateOne(
    filter1,
    update1,
    { session }
  );

  await collection2.insertOne(
    document2,
    { session }
  );

});
```

The driver provides automatic transaction error handling for the documented transient/commit error categories. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 41. What does `withTransaction()` return?

This is a useful method detail.

If your callback returns:

```js
return {
  applicationId
};
```

then `withTransaction()` can return that callback value as well.

The current Node.js driver documentation explicitly states that the convenient transaction API returns the callback's return value. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

So conceptually:

```js
const result = await session.withTransaction(async () => {
  return { success: true };
});
```

gives you:

```js
{
  success: true
}
```

This can make service code cleaner.

---

# 42. Every operation needs the session

This remains one of the most important transaction rules.

Suppose:

```js
await User.updateOne(..., { session });
```

but:

```js
await Audit.insertOne(...); // forgot session
```

Then the second operation isn't participating in the same transaction.

The MongoDB Node.js driver documentation explicitly emphasizes passing the session to each operation that should belong to the transaction. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

Mental rule:

```text
transaction
 ↓
operation 1 → session
operation 2 → session
operation 3 → session
```

---

# 43. Transaction + validation

Now combine today's Lesson 8 and 9 concepts.

Suppose you want:

```text
Create Application
+
Update Job count
```

Inside a transaction:

```text
Operation 1
→ Mongoose validation

Operation 2
→ atomic update

Commit
```

If validation fails before commit:

```text
transaction doesn't commit
```

Your application then translates the error appropriately.

This is where your layers interact:

```text
API validation
 ↓
Mongoose validation
 ↓
MongoDB operation
 ↓
transaction
```

---

# 44. Transaction + unique index

Imagine:

```text
candidateId + jobId
```

has a unique index.

Two application requests arrive simultaneously.

Both begin transactions.

Both attempt:

```text
insert application
```

The unique index remains the final consistency mechanism.

A transaction doesn't magically eliminate uniqueness requirements.

This is another important lesson:

```text
transaction
+
constraints
```

often work together.

---

# 45. Transaction + retry

Now imagine:

```text
transaction attempt 1
      ↓
TransientTransactionError
```

If retried:

```text
transaction attempt 2
      ↓
success
      ↓
commit
```

So the **business operation must be safe to repeat at the transaction level**.

That is one reason transaction callbacks shouldn't have uncontrolled external side effects.

For example, don't casually do:

```text
send email
```

inside the transaction callback and assume a retry won't send it twice.

---

# 46. Transaction callbacks should be deterministic

A useful mental rule:

> The transaction callback may be executed again.

Therefore inside it, prefer:

```text
database operations
```

that are safe under retry semantics.

Be extremely careful with:

```text
external API calls
email sending
payment operations
random non-repeatable behavior
```

This is not just stylistic advice. It follows directly from the driver's documented transaction retry behavior. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 47. Read/write concerns inside transactions

Your Lesson 9 already covered these.

The current Node.js driver supports configuring:

```text
readConcern
writeConcern
readPreference
maxCommitTimeMS
```

for transactions, and transaction-level settings override inherited client settings where specified. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

So your conceptual hierarchy is:

```text
MongoClient defaults
       ↓
database
       ↓
collection
       ↓
transaction
```

with the more specific configuration taking precedence where supported. ([MongoDB](https://www.mongodb.com/docs/drivers/node/current/crud/configure/?utm_source=chatgpt.com "Configure CRUD Operations - Node.js Driver - MongoDB Docs"))

---

# 48. What does `retryWrites=true` actually NOT mean?

It does **not** mean:

```text
“my application is exactly-once”
```

It does **not** mean:

```text
“all database operations can safely repeat”
```

It does **not** mean:

```text
“business side effects are automatically duplicated safely”
```

It means the driver has retryable-write behavior for operations that qualify.

That distinction is very important.

---

# 49. A real retry scenario

Suppose:

```text
POST /applications
```

Your application performs a supported write.

Network problem occurs after the server may have processed it.

The driver may retry.

If you have:

```text
unique(candidateId, jobId)
```

the database constraint helps prevent duplicate applications.

This is an example of multiple layers cooperating:

```text
retryable write
+
database uniqueness
+
application error handling
```

---

# 50. New methods/options from today's lesson

### Validation

```text
pattern
```

> String must match a regular expression.

```text
anyOf
```

> At least one schema must match.

```text
oneOf
```

> Exactly one schema must match.

```text
allOf
```

> Every schema must match.

```text
not
```

> The value must not match the supplied schema.

```text
multipleOf
```

> Number must be evenly divisible by the specified value.

---

### Transactions

```text
maxCommitTimeMS
```

> Limit how long transaction commit may run.

```text
retryWrites
```

> Enable/disable driver retry behavior for supported retryable writes.

```text
withTransaction()
```

> Execute transaction work with driver-managed transaction handling/retries.

---

# 51. One new JavaScript concept worth knowing: `Number.isInteger()`

You saw this earlier, but let's explicitly connect it to validation.

```js
Number.isInteger(value)
```

asks:

> Is this JavaScript value an integer?

Example:

```js
Number.isInteger(10)
```

returns:

```text
true
```

while:

```js
Number.isInteger(10.5)
```

returns:

```text
false
```

This is useful when validating pagination, counts, IDs represented numerically, and other API parameters.

---

# 52. One subtle JavaScript concept: `async` callbacks

You may see:

```js
await session.withTransaction(async () => {
  await ...
});
```

Why `async`?

Because the callback performs asynchronous database operations.

An `async` function automatically returns a Promise.

You don't need to understand every Promise internal detail yet, but keep this mental model:

```text
async function
→ function can use await
→ returns a Promise
```

Then:

```js
await
```

means:

> Wait for that Promise to settle before continuing.

This is JavaScript, not MongoDB.

---

# 53. Current Lesson 8 status

Your validation curriculum now includes:

```text
✓ bsonType
✓ required
✓ properties
✓ enum
✓ minimum / maximum
✓ minItems / maxItems
✓ uniqueItems
✓ nested validation
✓ arrays
✓ additionalProperties
✓ validationLevel
✓ validationAction
✓ collMod
✓ bypassDocumentValidation
✓ $expr
✓ pattern
✓ anyOf
✓ oneOf
✓ allOf
✓ not
✓ multipleOf
✓ conditional validation with query operators
✓ finding invalid existing documents
✓ validation migration workflow
✓ null vs missing
```

One deliberately corrected item:

```text
✗ if / then / else as MongoDB $jsonSchema
```

because that is not part of MongoDB's supported `$jsonSchema` keyword subset. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 54. Current Lesson 9 status

You now have:

```text
✓ single-document atomicity
✓ race conditions
✓ conditional updates
✓ optimistic concurrency
✓ $inc vs read-modify-write
✓ sessions
✓ transactions
✓ Core API
✓ Convenient API
✓ TransientTransactionError
✓ UnknownTransactionCommitResult
✓ read concern
✓ write concern
✓ read preference
✓ transaction session propagation
✓ maxCommitTimeMS
✓ retryWrites
✓ transaction retry mental model
✓ why external side effects are dangerous inside retryable callbacks
```

So Lessons **8 and 9 are now gap-complete**.

---

# 55. The deeper lesson from today's material

There's a recurring hierarchy here:

```text
                    DATA
                     ↓
             ┌───────┴────────┐
             ↓                ↓
        STRUCTURE         RELATIONSHIPS
             ↓                ↓
        $jsonSchema         $expr
             ↓                ↓
             └───────┬────────┘
                     ↓
                 MongoDB
                     ↓
                Transaction
                     ↓
              Atomic outcome
                     ↓
            Application logic
```

And then:

```text
Application retry
        ↓
database retry behavior
        ↓
transaction retry behavior
        ↓
idempotency
        ↓
business correctness
```

This is the kind of layered thinking that makes advanced MongoDB much easier to understand.

---

# Gap-fill progress

We're now here:

```text
Lessons 1–4   ✓
Lessons 5–7   ✓
Lessons 8–9   ✓
```

Remaining:

```text
Lessons 10–12
→ operational architecture commands

Lesson 13
→ change-stream edge cases

Lesson 14
→ discriminators
→ plugins
→ deeper Node.js Driver
→ advanced Mongoose behavior
```

The next module should cover **Lessons 10–12 together**, because replica-set administration and sharding administration naturally belong together: you'll learn the practical commands and operational workflow behind the replication/sharding theory you already studied.