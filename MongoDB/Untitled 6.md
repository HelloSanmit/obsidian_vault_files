
# MongoDB — Lesson 8: Schema Validation

This is an important lesson because it corrects a very common misunderstanding:

> **MongoDB has a flexible schema. That does not mean your production database should accept completely arbitrary documents.**

By default, MongoDB allows documents in the same collection to have different fields and types. Once your application has a defined structure, MongoDB's **schema validation** can enforce rules such as required fields, BSON types, ranges, allowed values, and nested structures. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/?utm_source=chatgpt.com "Schema Validation - Database Manual - MongoDB Docs"))

And this is where MongoDB becomes much more interesting from a backend-engineering perspective.

---

# 1. The problem schema validation solves

Imagine your application intends every user document to look like:

```javascript
{
    name: "Riya",
    email: "riya@example.com",
    age: 23,
    role: "candidate"
}
```

But your database gradually ends up containing:

```javascript
{
    name: "Riya",
    email: "riya@example.com",
    age: 23,
    role: "candidate"
}
```

```javascript
{
    name: "Kabir",
    email: "kabir@example.com",
    age: "24",
    role: "candidate"
}
```

```javascript
{
    username: "Maya",
    email: "maya@example.com"
}
```

```javascript
{
    name: "Arjun",
    email: 12345,
    role: "candidate"
}
```

MongoDB didn't necessarily stop these documents from entering because its default schema behavior is flexible.

Now your backend has to deal with:

```text
age → sometimes number
age → sometimes string
name → sometimes missing
email → sometimes string
email → sometimes number
```

That leads to:

```text
harder queries
runtime errors
messier application logic
bad analytics
migration problems
```

Schema validation exists to stop accidental structural corruption.

MongoDB describes schema validation as a mechanism for preventing unintended schema changes and data-type errors after an application establishes a defined structure. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/?utm_source=chatgpt.com "Schema Validation - Database Manual - MongoDB Docs"))

---

# 2. Very important: MongoDB is not “schema-less”

I want you to stop using the phrase:

> MongoDB is schema-less.

A better statement is:

> **MongoDB has a flexible schema by default, and schema validation lets you enforce as much structure as your application needs.**

This is a subtle but important distinction.

Think:

```text
SQL
 ↓
schema-first / strongly structured

MongoDB
 ↓
flexible by default
        +
optional validation
        ↓
controlled flexibility
```

MongoDB's documentation explicitly describes its model as flexible schema and notes that validation rules don't necessarily need to cover every field. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/?utm_source=chatgpt.com "Schema Validation - Database Manual - MongoDB Docs"))

---

# 3. Where validation lives

This is one of the most important architectural ideas.

You can have validation at several layers:

```text
Frontend
    ↓
Backend
    ↓
Mongoose / ODM
    ↓
MongoDB
```

For example, your React frontend may reject:

```text
empty email
```

Your Express/Spring backend may reject:

```text
invalid request
```

Your Mongoose schema may define:

```text
email required
```

And MongoDB itself can enforce:

```text
email must be a string
```

These are **different layers**.

They should not be confused.

---

# 4. Application validation vs database validation

Suppose your backend receives:

```javascript
{
    email: "user@example.com"
}
```

Your backend can validate:

```text
Is email present?
Is it syntactically valid?
Is this user allowed to perform this operation?
```

But suppose another piece of code directly accesses MongoDB and writes:

```javascript
{
    email: 12345
}
```

Your backend validation isn't involved.

MongoDB-level validation can protect the collection itself.

So:

```text
Application validation
    ↓
good API behavior

Database validation
    ↓
good stored data
```

A robust system often uses both.

---

# 5. Schema validation is not business logic

Very important.

MongoDB validation is excellent for things like:

```text
email must be string
age must be integer
status must be one of known values
skills must be an array
salary must be non-negative
```

But you generally shouldn't try to put every business rule into the database validator.

For example:

> “A recruiter can only reject an application if they are assigned to that job.”

That's primarily an **authorization/business rule**.

Your application handles that.

So:

```text
Database validator
    ↓
Is the data structurally valid?

Application service
    ↓
Is this operation logically allowed?
```

This distinction will matter enormously when you build Spring Boot.

---

# 6. `$jsonSchema`

The most common way to define MongoDB schema validation is:

```javascript
validator: {
    $jsonSchema: {
        ...
    }
}
```

MongoDB supports the JSON Schema vocabulary for validation, with MongoDB-specific BSON typing and some differences from the generic JSON Schema specification. MongoDB's current documentation states that its implementation is based on JSON Schema draft 4 with MongoDB-specific extensions/omissions. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

The basic shape is:

```javascript
{
    $jsonSchema: {
        bsonType: "object",

        required: [
            "name",
            "email"
        ],

        properties: {
            name: {
                bsonType: "string"
            },

            email: {
                bsonType: "string"
            }
        }
    }
}
```

Let's break every piece down.

---

# 7. `bsonType`

This is MongoDB's version of specifying the underlying BSON type.

Examples:

```javascript
bsonType: "string"
```

```javascript
bsonType: "int"
```

```javascript
bsonType: "long"
```

```javascript
bsonType: "double"
```

```javascript
bsonType: "decimal"
```

```javascript
bsonType: "bool"
```

```javascript
bsonType: "date"
```

```javascript
bsonType: "objectId"
```

```javascript
bsonType: "array"
```

```javascript
bsonType: "object"
```

This is particularly important because MongoDB stores BSON types, not merely abstract JSON types. MongoDB's schema-validation examples use `bsonType` specifically to enforce these types. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 8. Why BSON types matter

Consider:

```javascript
age: 25
```

and:

```javascript
age: "25"
```

They may look similar to a human.

To a database, they're different types.

That's important for:

```text
sorting
comparisons
aggregation
arithmetic
index behavior
application serialization
```

Suppose you execute:

```javascript
{
    $gte: ["$age", 18]
}
```

What should happen when one document stores:

```text
25
```

and another:

```text
"25"
```

You don't want your application to continually wonder what type it received.

Schema validation can enforce consistency.

---

# 9. `required`

Suppose every user must have:

```text
name
email
role
```

Write:

```javascript
required: [
    "name",
    "email",
    "role"
]
```

This means those fields must exist.

For example:

```javascript
{
    name: "Riya",
    email: "riya@example.com",
    role: "candidate"
}
```

passes the structural requirement.

But:

```javascript
{
    name: "Riya",
    email: "riya@example.com"
}
```

fails because:

```text
role
```

is missing.

MongoDB documents `required` as the keyword used to require specified fields in a document. ([MongoDB](https://www.mongodb.com/docs/compass/validation/?utm_source=chatgpt.com "Set Validation Rules for Your Schema - Compass - MongoDB Docs"))

---

# 10. `required` does not mean “must have a non-null value”

This is subtle.

Suppose:

```javascript
required: ["phone"]
```

You might think:

```javascript
{
    phone: null
}
```

should fail automatically.

But `required` is about **presence**.

If you also want to prevent `null`, you need an appropriate type/value restriction.

For example:

```javascript
phone: {
    bsonType: "string"
}
```

Now:

```javascript
phone: "9876543210"
```

is acceptable structurally, while:

```javascript
phone: null
```

does not satisfy the string type.

This distinction is very important:

```text
required
    ↓
field must exist

bsonType
    ↓
field must have acceptable type
```

---

# 11. `properties`

This is where you define rules for specific fields.

Example:

```javascript
properties: {
    name: {
        bsonType: "string"
    },

    age: {
        bsonType: "int"
    },

    email: {
        bsonType: "string"
    }
}
```

Think:

```text
required
    ↓
Which fields must exist?

properties
    ↓
What rules apply to those fields?
```

---

# 12. A complete simple validator

Let's create a real collection.

```javascript
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",

            required: [
                "name",
                "email",
                "age",
                "role"
            ],

            properties: {
                name: {
                    bsonType: "string"
                },

                email: {
                    bsonType: "string"
                },

                age: {
                    bsonType: "int"
                },

                role: {
                    bsonType: "string"
                }
            }
        }
    }
})
```

This says:

```text
document must be an object

name     → required + string
email    → required + string
age      → required + integer
role     → required + string
```

MongoDB supports specifying validation rules during collection creation using the `validator` option. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 13. Test a valid document

```javascript
db.users.insertOne({
    name: "Riya",
    email: "riya@example.com",
    age: 23,
    role: "candidate"
})
```

This satisfies the schema.

---

# 14. Test an invalid document

```javascript
db.users.insertOne({
    name: "Riya",
    email: "riya@example.com",
    age: "23",
    role: "candidate"
})
```

Problem:

```text
age
 ↓
string
```

but schema requires:

```text
int
```

With the default `validationAction: "error"`, MongoDB rejects the write. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/handle-invalid-documents/?utm_source=chatgpt.com "Choose How to Handle Invalid Documents - Database Manual - MongoDB Docs"))

---

# 15. Validation failure is a database error

You may see an error resembling:

```text
MongoServerError: Document failed validation
```

MongoDB can include information about the validation failure, including the validator/operator involved and the field that failed. MongoDB's documentation examples show this detailed validation-failure output. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/specify-allowed-field-values/?utm_source=chatgpt.com "Specify Allowed Field Values - Database Manual - MongoDB Docs"))

Your backend should catch this properly and translate it into an appropriate application/API response.

For example:

```text
MongoDB
   ↓
validation failure
   ↓
backend catches database error
   ↓
HTTP 400 / appropriate response
```

Don't expose raw database internals to users.

---

# 16. `enum`

Now let's control allowed values.

Suppose role can only be:

```text
candidate
recruiter
admin
```

Use:

```javascript
role: {
    bsonType: "string",
    enum: [
        "candidate",
        "recruiter",
        "admin"
    ]
}
```

Now:

```javascript
role: "candidate"
```

passes.

But:

```javascript
role: "canddiate"
```

fails.

MongoDB's `enum` validation restricts a field to one of the specified values. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/specify-allowed-field-values/?utm_source=chatgpt.com "Specify Allowed Field Values - Database Manual - MongoDB Docs"))

This is incredibly useful for fields that behave like application enums.

---

# 17. Why `enum` matters

Without validation:

```javascript
status: "interview"
```

someone could accidentally insert:

```javascript
status: "interveiw"
```

or:

```javascript
status: "Interview"
```

or:

```javascript
status: "INTERVIEW"
```

Now your application has multiple representations of the same logical state.

You could end up writing:

```javascript
if (status === "interview") ...
if (status === "Interview") ...
```

which is horrible.

A controlled enum-like set prevents many such mistakes.

---

# 18. Numeric restrictions

Suppose:

```text
age
```

must be between 18 and 100.

You can write:

```javascript
age: {
    bsonType: "int",
    minimum: 18,
    maximum: 100
}
```

Now:

```text
17 → invalid
18 → valid
35 → valid
100 → valid
101 → invalid
```

MongoDB's `$jsonSchema` supports numeric constraints such as `minimum` and `maximum`. Official MongoDB examples use these to restrict numeric fields. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/schema-validation/bypass-document-validation/?utm_source=chatgpt.com "Bypass Schema Validation - Database Manual v8.0 - MongoDB Docs"))

---

# 19. `exclusiveMinimum` and `exclusiveMaximum`

There is an important distinction between:

```text
minimum
```

and exclusive boundaries.

Conceptually:

```text
minimum = 0
```

can allow zero.

Whereas an exclusive lower boundary says:

```text
must be > 0
```

The exact JSON Schema keyword behavior follows the MongoDB-supported JSON Schema vocabulary.

You won't need this constantly, but you should know these constraints exist when modeling things such as:

```text
price
percentage
rating
coordinates
```

---

# 20. Strings

You can validate string lengths.

For example:

```javascript
username: {
    bsonType: "string",
    minLength: 3,
    maxLength: 30
}
```

Conceptually:

```text
"ab"            → invalid
"abc"           → valid
"abcdefghijkl"  → valid
>30 chars       → invalid
```

This is useful for:

```text
username
short labels
codes
titles
```

You don't need to force a maximum length everywhere. Use constraints where they represent a meaningful domain rule.

---

# 21. Arrays

Suppose:

```javascript
skills: [
    "Java",
    "MongoDB",
    "Redis"
]
```

You can require:

```javascript
skills: {
    bsonType: "array"
}
```

But that's not enough if every element should be a string.

You can also define:

```javascript
skills: {
    bsonType: "array",
    items: {
        bsonType: "string"
    }
}
```

Now:

```javascript
[
    "Java",
    "MongoDB"
]
```

is structurally valid.

But:

```javascript
[
    "Java",
    123,
    true
]
```

is invalid because not every element is a string.

MongoDB's JSON Schema examples use `items` to define validation rules for array elements. ([MongoDB](https://www.mongodb.com/docs/compass/schema/export/?utm_source=chatgpt.com "Export Your Schema - Compass - MongoDB Docs"))

---

# 22. `minItems` and `maxItems`

You can also control array length.

For example:

```javascript
skills: {
    bsonType: "array",
    minItems: 1,
    maxItems: 20,

    items: {
        bsonType: "string"
    }
}
```

Now your application guarantees:

```text
at least 1 skill
at most 20 skills
every skill → string
```

This is especially useful because Lesson 3 taught you to think about **bounded vs unbounded arrays**.

Notice how the lessons connect:

```text
Lesson 3
    ↓
Don't blindly create giant arrays

Lesson 8
    ↓
You can actually enforce reasonable bounds
```

---

# 23. Nested objects

Suppose your user contains:

```javascript
address: {
    city: "Pune",
    country: "India"
}
```

You can validate the nested object:

```javascript
address: {
    bsonType: "object",

    required: [
        "city",
        "country"
    ],

    properties: {
        city: {
            bsonType: "string"
        },

        country: {
            bsonType: "string"
        }
    }
}
```

This creates validation recursively.

You can therefore build a schema like:

```text
user
├── name → string
├── email → string
├── age → int
├── role → enum
└── address → object
      ├── city → string
      └── country → string
```

That is one of the strengths of JSON Schema-based validation.

---

# 24. Nested arrays of objects

This is where things become much more realistic.

Suppose:

```javascript
skills: [
    {
        name: "Java",
        level: 8
    },

    {
        name: "MongoDB",
        level: 7
    }
]
```

You can validate the entire structure:

```javascript
skills: {
    bsonType: "array",

    items: {
        bsonType: "object",

        required: [
            "name",
            "level"
        ],

        properties: {
            name: {
                bsonType: "string"
            },

            level: {
                bsonType: "int",
                minimum: 1,
                maximum: 10
            }
        }
    }
}
```

Now MongoDB understands:

```text
skills
 ↓
must be array
 ↓
each element must be object
 ↓
each object must contain name + level
 ↓
name must be string
 ↓
level must be int from 1–10
```

That's a proper nested document schema.

---

# 25. `additionalProperties`

This one is extremely important.

Suppose your schema defines:

```javascript
properties: {
    name: {...},
    email: {...},
    age: {...}
}
```

By default, that **does not necessarily mean other fields are forbidden**.

MongoDB's schema validation is flexible: the rules can cover only the fields you care about. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/?utm_source=chatgpt.com "Schema Validation - Database Manual - MongoDB Docs"))

If you want to prohibit undeclared fields, JSON Schema provides:

```javascript
additionalProperties: false
```

Then only properties explicitly defined in the schema are accepted.

---

# 26. Why `additionalProperties: false` can be dangerous

Suppose:

```javascript
{
    $jsonSchema: {
        bsonType: "object",

        properties: {
            name: {
                bsonType: "string"
            }
        },

        additionalProperties: false
    }
}
```

A MongoDB document automatically contains:

```text
_id
```

So MongoDB's documentation warns that when using `additionalProperties: false`, you need to account for `_id` in the schema's `properties`; otherwise documents can be rejected because `_id` is not listed. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/json-schema-tips/?utm_source=chatgpt.com "Tips for JSON Schema Validation - Database Manual - MongoDB Docs"))

This is a classic gotcha.

For example:

```javascript
properties: {
    _id: {
        bsonType: "objectId"
    },

    name: {
        bsonType: "string"
    }
},

additionalProperties: false
```

Now `_id` is explicitly permitted.

---

# 27. Should you always use `additionalProperties: false`?

No.

This is one of those things beginners tend to overdo.

Suppose your application intentionally stores metadata:

```javascript
{
    name: "Riya",
    email: "...",

    metadata: {
        source: "import",
        campaign: "summer",
        ...
    }
}
```

A completely closed schema could make future evolution painful.

Remember what MongoDB's flexible schema is useful for.

A sensible strategy is often:

```text
strict validation for important core fields
+
flexibility where flexibility is intentional
```

Not:

```text
reject every unknown field everywhere
```

---

# 28. Validation with query operators

This is one of the most interesting advanced features.

MongoDB validation does not have to use only `$jsonSchema`.

You can also build validators using MongoDB query expressions/operators. The MongoDB documentation states that validators can use query operators, with certain exclusions such as `$near`, `$nearSphere`, `$text`, and `$where`. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/collmod/?utm_source=chatgpt.com "collMod (database command) - Database Manual - MongoDB Docs"))

For example:

```javascript
validator: {
    $or: [
        {
            name: {
                $type: "string"
            }
        },

        {
            username: {
                $type: "string"
            }
        }
    ]
}
```

This says:

```text
name is string
OR
username is string
```

That is beyond the simple “shape validation” idea.

---

# 29. `$jsonSchema` + query expressions

These can also be combined.

Suppose a document has:

```javascript
{
    price: 1000,
    discountedPrice: 800
}
```

You want:

```text
discountedPrice < price
```

This isn't merely a type rule.

It's a relationship between two fields.

MongoDB's documentation gives examples where a validator combines `$jsonSchema` with `$expr` to enforce exactly this sort of semantic relationship. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

Conceptually:

```javascript
validator: {
    $and: [
        {
            $jsonSchema: {
                ...
            }
        },

        {
            $expr: {
                $lt: [
                    "$discountedPrice",
                    "$price"
                ]
            }
        }
    ]
}
```

Now you have:

```text
structural validation
        +
cross-field validation
```

This is significantly more powerful.

---

# 30. Structural vs semantic validation

This distinction is worth learning.

### Structural

```text
email → string
age → int
skills → array
role → one of allowed values
```

This is what `$jsonSchema` is naturally good at.

### Semantic

```text
discountedPrice < price
endDate >= startDate
balance >= 0
```

These involve relationships between values.

MongoDB validators can combine JSON Schema with query expressions to handle such rules. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 31. `validationAction`

Now suppose a document violates your validator.

What should MongoDB do?

There are currently three documented validation actions in modern MongoDB:

```text
error
warn
errorAndLog
```

`error` rejects the write, `warn` allows it but logs the violation, and MongoDB's documentation notes that `errorAndLog` was introduced in MongoDB 8.1 and both rejects the write and logs the violation. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/handle-invalid-documents/?utm_source=chatgpt.com "Choose How to Handle Invalid Documents - Database Manual - MongoDB Docs"))

Let's understand why these exist.

---

# 32. `validationAction: "error"`

This is the default.

```javascript
validationAction: "error"
```

Invalid document:

```text
WRITE
 ↓
validation
 ↓
FAIL
```

This is what you normally want for an established production schema.

MongoDB rejects inserts/updates that violate the validation criteria. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/handle-invalid-documents/?utm_source=chatgpt.com "Choose How to Handle Invalid Documents - Database Manual - MongoDB Docs"))

---

# 33. `validationAction: "warn"`

Now:

```javascript
validationAction: "warn"
```

means:

```text
invalid document
      ↓
write still happens
      ↓
validation violation logged
```

MongoDB allows the operation but records the violation in the MongoDB log. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/handle-invalid-documents/?utm_source=chatgpt.com "Choose How to Handle Invalid Documents - Database Manual - MongoDB Docs"))

Why would you ever want this?

Migration.

Imagine an old collection contains:

```text
2 million documents
```

and you discover:

```text
email should always be a string
```

but some old documents aren't valid.

Immediately switching to hard rejection could complicate migration.

So you can first use:

```text
warn
```

to observe violations.

---

# 34. `validationAction: "errorAndLog"`

Modern MongoDB versions add:

```javascript
validationAction: "errorAndLog"
```

This does both:

```text
reject invalid write
+
log validation failure
```

MongoDB's current docs mark this option as new in version 8.1. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/handle-invalid-documents/?utm_source=chatgpt.com "Choose How to Handle Invalid Documents - Database Manual - MongoDB Docs"))

This is worth knowing because documentation and examples written before MongoDB 8.1 won't show it.

---

# 35. `validationLevel`

Now another concept:

```text
How broadly should MongoDB apply the rules?
```

MongoDB provides:

```text
off
strict
moderate
```

Current documentation describes `strict` as the default, `moderate` as applying to inserts and updates on existing valid documents while not requiring updates to already-invalid existing documents to pass validation, and `off` as disabling validation for inserts and updates. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-level/?utm_source=chatgpt.com "Specify Validation Level for Existing Documents - Database Manual - MongoDB Docs"))

---

# 36. `strict`

This is:

```javascript
validationLevel: "strict"
```

Conceptually:

```text
every insert
+
every update
 ↓
validation
```

This is the normal long-term production setting for a clean schema.

---

# 37. `moderate`

This is more subtle.

Suppose your database contains:

```text
Document A → valid
Document B → already invalid
```

You add a new validator.

With:

```javascript
validationLevel: "moderate"
```

the validator applies to:

```text
new documents
+
updates to documents that currently satisfy the rules
```

but existing invalid documents can continue to be updated without being required to satisfy the new schema.

MongoDB documents this specifically as a migration-friendly behavior. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-level/?utm_source=chatgpt.com "Specify Validation Level for Existing Documents - Database Manual - MongoDB Docs"))

---

# 38. Why `moderate` exists

Imagine this:

```text
Old database
↓
1,000,000 documents
↓
50,000 don't satisfy new schema
```

You don't necessarily want:

```text
new schema
    ↓
suddenly every old invalid document becomes impossible to update
```

Instead:

```text
moderate
 ↓
new writes must be valid
valid old documents must stay valid
invalid old documents can be migrated gradually
```

That makes schema evolution much easier.

---

# 39. `off`

```javascript
validationLevel: "off"
```

means validation isn't applied to inserts or updates.

You might temporarily use this during certain controlled administrative operations, but you generally shouldn't leave important collections without validation unless the application's design genuinely calls for it. MongoDB documents `off` as disabling validation for inserts and updates. ([MongoDB](https://www.mongodb.com/docs/v7.0/reference/command/create/?utm_source=chatgpt.com "create (database command) - Database Manual v7.0 - MongoDB Docs"))

---

# 40. `validationLevel` vs `validationAction`

Do not mix these up.

### `validationLevel`

```text
WHO/WHICH WRITES get validated?
```

### `validationAction`

```text
WHAT HAPPENS when validation fails?
```

Mental model:

```text
validationLevel
    ↓
scope

validationAction
    ↓
response
```

Example:

```javascript
validationLevel: "moderate",
validationAction: "warn"
```

means:

```text
apply rules moderately
+
allow failures but log them
```

---

# 41. Changing validation after collection creation

You don't have to destroy and recreate a collection.

MongoDB provides `collMod` to modify collection options, including the validator, `validationLevel`, and `validationAction`. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/collmod/?utm_source=chatgpt.com "collMod (database command) - Database Manual - MongoDB Docs"))

Example:

```javascript
db.runCommand({
    collMod: "users",

    validator: {
        $jsonSchema: {
            bsonType: "object",

            required: [
                "name",
                "email"
            ],

            properties: {
                name: {
                    bsonType: "string"
                },

                email: {
                    bsonType: "string"
                }
            }
        }
    },

    validationLevel: "strict",
    validationAction: "error"
})
```

This is how schema evolution is handled operationally.

---

# 42. Existing documents are NOT magically rewritten

This is very important.

Suppose you already have:

```text
1,000,000 existing documents
```

Then you add a validator.

MongoDB does **not** automatically go through all those documents and repair them.

MongoDB's documentation notes that existing documents do not undergo validation checks merely because a validator was added; they encounter validation when subsequent updates are performed, depending on validation level. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/collmod/?utm_source=chatgpt.com "collMod (database command) - Database Manual - MongoDB Docs"))

This is why schema migration is a real engineering task.

---

# 43. Finding invalid existing documents

MongoDB supports using `$jsonSchema` in query conditions to find documents that do or don't match a schema. This is useful when you add validation to an existing collection and need to identify bad records. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/use-json-schema-query-conditions/?utm_source=chatgpt.com "Query for and Modify Valid or Invalid Documents - Database Manual - MongoDB Docs"))

Conceptually:

```javascript
db.users.find({
    $nor: [
        {
            $jsonSchema: {
                ...
            }
        }
    ]
})
```

This means:

> Find documents that do **not** satisfy this schema.

You can then:

```text
find invalid records
       ↓
inspect them
       ↓
clean/migrate them
       ↓
tighten validation
```

That's a realistic production workflow.

---

# 44. `db.collection.validate()`

MongoDB also provides collection validation tooling.

For example:

```javascript
db.accounts.validate()
```

MongoDB's documentation uses this to verify compliance when working with schema validation and reports non-compliant documents. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-polymorphic-collections/?utm_source=chatgpt.com "Specify Validation for Polymorphic Collections - Database Manual - MongoDB Docs"))

This is useful during:

```text
migrations
troubleshooting
schema changes
data cleanup
```

---

# 45. Bypassing validation

Sometimes you genuinely need to write a document that doesn't satisfy the validator.

MongoDB provides:

```text
bypassDocumentValidation
```

on supported operations.

For example, an administrative restore/migration operation may use this when old data doesn't satisfy a newly introduced validation rule. MongoDB documents bypassing validation as an explicit per-operation option rather than a permanent property of the collection. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/schema-validation/bypass-document-validation/?utm_source=chatgpt.com "Bypass Schema Validation - Database Manual v8.0 - MongoDB Docs"))

Conceptually:

```text
normal application write
    ↓
validation enforced

controlled migration/restore
    ↓
possibly bypass validation
```

This should be treated as a privileged administrative capability, not a convenient way to avoid fixing application bugs.

When access control is enabled, bypassing validation requires the corresponding `bypassDocumentValidation` privilege; MongoDB documents built-in roles such as `dbAdmin` and `restore` as providing that action. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/schema-validation/bypass-document-validation/?utm_source=chatgpt.com "Bypass Schema Validation - Database Manual v8.0 - MongoDB Docs"))

---

# 46. Why “bypass” can be dangerous

Suppose your validator guarantees:

```text
age → int
```

Then someone does:

```text
bypassDocumentValidation: true
```

and inserts:

```javascript
{
    age: "twenty"
}
```

Now you've reintroduced the exact problem the validator was designed to prevent.

So think:

```text
bypass validation
    =
controlled exception
```

not:

```text
normal application behavior
```

---

# 47. Polymorphic collections

Now we're reaching an advanced but very useful MongoDB concept.

Suppose one collection contains multiple document types:

```text
accounts
```

and it stores:

```text
customer
account
```

documents with different shapes.

You don't necessarily have to abandon schema validation.

MongoDB supports **polymorphic collection validation**, where different document types can have different schemas, commonly distinguished using a field like:

```javascript
docType: "customer"
```

or:

```javascript
docType: "account"
```

MongoDB documents using `oneOf` to allow multiple schemas in a collection. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-polymorphic-collections/?utm_source=chatgpt.com "Specify Validation for Polymorphic Collections - Database Manual - MongoDB Docs"))

---

# 48. Example polymorphic schema

Customer:

```javascript
{
    docType: "customer",
    customerId: "C101",
    name: "Riya"
}
```

Account:

```javascript
{
    docType: "account",
    accountNumber: "A500",
    accountType: "savings",
    balance: 50000
}
```

Now:

```javascript
$jsonSchema: {
    oneOf: [
        customerSchema,
        accountSchema
    ]
}
```

This means:

```text
document
   ↓
matches customer schema
OR
matches account schema
```

MongoDB explicitly documents `oneOf` for this kind of polymorphic validation. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-polymorphic-collections/?utm_source=chatgpt.com "Specify Validation for Polymorphic Collections - Database Manual - MongoDB Docs"))

---

# 49. Why this matters for you

Because MongoDB's flexibility doesn't mean every collection needs:

```text
one document shape only
```

You can intentionally model:

```text
same collection
+
multiple known variants
+
strong validation for each variant
```

This can be useful for:

```text
events
notifications
different message types
content types
workflow objects
```

But don't use polymorphism merely to avoid properly separating entities.

---

# 50. MongoDB validation vs Mongoose validation

This is **extremely relevant to your ChatApp**.

You used Mongoose.

Mongoose can have:

```javascript
const userSchema = new mongoose.Schema({
    name: {
        type: String,
        required: true
    },

    email: {
        type: String,
        required: true
    }
});
```

That's validation at the **ODM/application layer**.

MongoDB's:

```javascript
$jsonSchema
```

is validation at the **database layer**.

These are not the same thing.

---

# 51. Why Mongoose validation isn't enough

Imagine:

```text
Node.js application
       ↓
Mongoose
       ↓
MongoDB
```

Mongoose validation works when the document passes through Mongoose in the relevant operation.

But imagine another service:

```text
Python service
       ↓
MongoDB
```

or:

```text
MongoDB Compass
       ↓
MongoDB
```

or:

```text
admin script
       ↓
MongoDB
```

Mongoose isn't involved.

MongoDB-level validation can enforce structural rules regardless of which client performs the write.

That's one reason database validation exists.

---

# 52. But MongoDB validation doesn't replace Mongoose

You also don't want:

```text
MongoDB validator = everything
```

Mongoose/application validation can provide:

```text
developer-friendly model validation
custom application behavior
casting
middleware
application-level constraints
```

MongoDB can provide:

```text
database-level structural guarantee
```

A mature application may use both.

---

# 53. Where validation should live

A useful mental model:

```text
                 USER REQUEST
                      ↓
               React/frontend
                      ↓
               basic UX validation
                      ↓
                Backend API
                      ↓
             business validation
                      ↓
                Mongoose
                      ↓
          model/application validation
                      ↓
                 MongoDB
                      ↓
           database schema validation
```

Each layer has a different job.

---

# 54. Don't duplicate every rule everywhere

You might be tempted to write:

```text
Frontend:
email string

Backend:
email string

Mongoose:
email string

MongoDB:
email string
```

That's not necessarily bad for critical guarantees, but you shouldn't mindlessly duplicate every rule at every layer.

For example:

```text
MongoDB:
BSON type is string
```

is a strong database invariant.

Frontend:

```text
email input is non-empty
```

is a UX concern.

Backend:

```text
email is not already registered
```

is application/database logic involving uniqueness.

Different responsibilities.

---

# 55. Validation + indexes

Now connect Lesson 7 and Lesson 8.

Suppose:

```text
email must be unique
```

You might write:

```text
schema validation
    +
unique index
```

But these do different things.

Schema validation:

```text
email must be string
```

Unique index:

```text
no duplicate email values
```

So:

```text
validator
    ↓
shape/type/value constraints

index
    ↓
lookup performance + uniqueness
```

Don't confuse them.

---

# 56. A proper ChatApp user validator

Let's design something realistic.

Suppose a user is:

```javascript
{
    username: "riya24",
    email: "riya@example.com",
    passwordHash: "...",
    profilePic: "...",
    bio: "...",
    isOnline: false,
    createdAt: ISODate("...")
}
```

A reasonable MongoDB validator might enforce:

```text
username     → string
email        → string
passwordHash → string
bio          → string
isOnline     → boolean
createdAt    → date
```

But you might leave some optional fields optional:

```text
profilePic
bio
```

That is a good illustration of controlled flexibility.

---

# 57. Example ChatApp validator

```javascript
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",

            required: [
                "username",
                "email",
                "passwordHash",
                "isOnline",
                "createdAt"
            ],

            properties: {

                username: {
                    bsonType: "string",
                    minLength: 3,
                    maxLength: 30
                },

                email: {
                    bsonType: "string"
                },

                passwordHash: {
                    bsonType: "string"
                },

                profilePic: {
                    bsonType: "string"
                },

                bio: {
                    bsonType: "string",
                    maxLength: 500
                },

                isOnline: {
                    bsonType: "bool"
                },

                createdAt: {
                    bsonType: "date"
                }
            }
        }
    }
})
```

This doesn't attempt to validate every aspect of the application.

It establishes a sensible database contract.

---

# 58. Message validator

Your ChatApp message concept is even more interesting.

Suppose:

```javascript
{
    senderId: ObjectId("..."),
    receiverId: ObjectId("..."),
    text: "Hello",
    image: "https://...",
    seen: false,
    createdAt: ISODate("...")
}
```

You want:

```text
senderId    → ObjectId
receiverId  → ObjectId
text        → string, optional
image       → string, optional
seen        → boolean
createdAt   → date
```

But there is a semantic rule:

> A message should contain either text or image content.

That cannot be expressed as cleanly by just saying:

```text
text optional
image optional
```

because then this is technically structurally valid:

```javascript
{
    senderId: ...,
    receiverId: ...,
    seen: false,
    createdAt: ...
}
```

but semantically it's a useless message.

This is where combining JSON Schema with query expressions can become useful.

---

# 59. Structural + semantic message validation

Conceptually:

```text
$jsonSchema
    ↓
types are correct

$expr / logical query rules
    ↓
at least one content field exists
```

You could construct a validator combining `$and` with `$jsonSchema` and additional query conditions.

MongoDB explicitly supports validators composed from `$jsonSchema` and query operators. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

This is a great example of why schema validation is more than:

> “Is email a string?”

---

# 60. But should MongoDB enforce everything?

Not necessarily.

For example:

```text
"message cannot contain banned content"
```

doesn't belong naturally in a simple schema validator.

Likewise:

```text
"receiver must be an existing user"
```

is a referential/business rule.

MongoDB validation isn't a foreign-key system in the relational sense.

Your application may need to enforce:

```text
receiverId exists
receiver hasn't blocked sender
sender is authenticated
```

These are application-level concerns.

---

# 61. Validation is about invariants

This is probably the most important conceptual insight of today's lesson.

An **invariant** is something that should always be true.

Examples:

```text
age is an integer
role is one of known values
createdAt is a date
rating is 1–5
```

These are database invariants.

Your validator says:

```text
"MongoDB: don't let the collection violate these truths."
```

That's a very powerful way to think about schema validation.

---

# 62. Database invariants vs application rules

Compare:

### Database invariant

```text
rating must be between 1 and 5
```

### Application rule

```text
only customers who purchased the product can review it
```

The first is naturally expressed as a data constraint.

The second requires knowledge of other records and application context.

So:

```text
Invariant about document
    → schema validation

Rule about behavior/context
    → application/service layer
```

This distinction will help you enormously in backend architecture.

---

# 63. Schema validation and schema evolution

Imagine Version 1:

```javascript
{
    name,
    email
}
```

Version 2 introduces:

```javascript
{
    name,
    email,
    phone
}
```

You have several options.

You could:

```text
make phone optional
```

while migrating old records.

Then later:

```text
phone becomes required
```

once everything has been migrated.

That might involve:

```text
1. Add relaxed validator
2. Find invalid/old documents
3. Backfill phone where possible
4. Verify collection
5. Tighten validator
```

MongoDB's `moderate` validation level and query-based identification of invalid documents are particularly useful in this kind of migration workflow. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/use-json-schema-query-conditions/?utm_source=chatgpt.com "Query for and Modify Valid or Invalid Documents - Database Manual - MongoDB Docs"))

---

# 64. A realistic migration strategy

Suppose `phone` is becoming mandatory.

Current data:

```text
1,000,000 users
```

and:

```text
150,000 missing phone
```

Don't immediately do:

```javascript
required: ["phone"]
validationLevel: "strict"
```

without understanding the existing data.

A safer process:

```text
Existing collection
       ↓
inspect invalid documents
       ↓
estimate migration
       ↓
use moderate/warn if appropriate
       ↓
backfill/clean
       ↓
verify
       ↓
switch to strict/error
```

MongoDB's documentation explicitly discusses `moderate` and `warn` as mechanisms useful when existing data doesn't yet conform to newly introduced rules. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-level/?utm_source=chatgpt.com "Specify Validation Level for Existing Documents - Database Manual - MongoDB Docs"))

---

# 65. A subtle point: updates are validated too

Schema validation isn't only about:

```javascript
insertOne()
```

It also matters when you update an existing document.

For example, suppose this is valid:

```javascript
{
    age: 25
}
```

and validator says:

```text
age must be int
```

Then:

```javascript
db.users.updateOne(
    { ... },
    {
        $set: {
            age: "twenty-five"
        }
    }
)
```

can be rejected by the validator.

MongoDB explicitly states that schema validation applies to inserts and updates according to the configured validation level. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/collmod/?utm_source=chatgpt.com "collMod (database command) - Database Manual - MongoDB Docs"))

This matters enormously because otherwise an application could insert good documents and then corrupt them during updates.

---

# 66. Validation does not automatically validate unchanged old fields in the way you might intuitively imagine

This one is subtle enough that you should understand the operational model rather than rely on a simplistic slogan.

Suppose:

```text
existing document = valid
```

and you update:

```javascript
$set: {
    lastLogin: ...
}
```

MongoDB evaluates the resulting document against the validator according to the collection's validation level.

So an update can't simply turn a valid document into an invalid one under normal strict validation.

That's why validation is useful for preserving invariants over time.

---

# 67. Why this matters for `$set`

Remember Lesson 4:

```javascript
$set: {
    "profile.city": "Pune"
}
```

Now imagine:

```text
profile.city must be string
```

Schema validation can guarantee that your update doesn't accidentally turn it into:

```javascript
profile: {
    city: 12345
}
```

So Lesson 4 + Lesson 8 combine:

```text
update operators
       +
schema validation
       ↓
safe document mutations
```

---

# 68. Validation and `$push`

Likewise:

```javascript
$push: {
    skills: "MongoDB"
}
```

If the schema says:

```javascript
skills: {
    bsonType: "array",
    items: {
        bsonType: "string"
    }
}
```

then accidentally pushing:

```javascript
$push: {
    skills: 123
}
```

can result in validation failure.

That's a beautiful example of the database protecting your data model from an application bug.

---

# 69. What MongoDB validation does NOT do

This is just as important as what it does.

Schema validation generally doesn't replace:

```text
authentication
authorization
business workflows
foreign-key semantics
complex cross-document consistency
password hashing
API validation
rate limiting
```

For example:

```text
password must be string
```

is not the same thing as:

```text
password must be securely hashed
```

Your validator should not be confused with your security architecture.

---

# 70. A strong production architecture

For your eventual backend projects, think:

```text
                    API Request
                        ↓
                  Controller/API
                        ↓
                DTO/request validation
                        ↓
                   Service layer
                        ↓
        business rules / authorization
                        ↓
             Repository / Mongoose
                        ↓
                  MongoDB driver
                        ↓
             MongoDB schema validation
                        ↓
                     Storage
```

And separately:

```text
MongoDB indexes
    ↓
query performance
+
unique constraints
```

This is a much more mature backend mental model than:

```text
"MongoDB stores JSON."
```

---

# 71. A complete JobForge-style validator

Let's design something closer to your future project.

Suppose a job looks like:

```javascript
{
    title: "Backend Engineer",

    companyId: ObjectId("..."),

    location: "Pune",

    salaryMin: 700000,
    salaryMax: 1000000,

    status: "open",

    skills: [
        "Java",
        "Spring Boot",
        "MongoDB"
    ],

    createdAt: ISODate("...")
}
```

We might want:

```text
title       → required string
companyId   → required ObjectId
location    → required string
salaryMin   → integer >= 0
salaryMax   → integer >= 0
status      → open/closed/draft
skills      → array of strings
createdAt   → date
```

A validator might look like:

```javascript
db.createCollection("jobs", {
    validator: {
        $jsonSchema: {

            bsonType: "object",

            required: [
                "title",
                "companyId",
                "location",
                "salaryMin",
                "salaryMax",
                "status",
                "skills",
                "createdAt"
            ],

            properties: {

                title: {
                    bsonType: "string",
                    minLength: 2,
                    maxLength: 150
                },

                companyId: {
                    bsonType: "objectId"
                },

                location: {
                    bsonType: "string"
                },

                salaryMin: {
                    bsonType: "int",
                    minimum: 0
                },

                salaryMax: {
                    bsonType: "int",
                    minimum: 0
                },

                status: {
                    enum: [
                        "draft",
                        "open",
                        "closed"
                    ]
                },

                skills: {
                    bsonType: "array",

                    minItems: 1,
                    maxItems: 30,

                    items: {
                        bsonType: "string"
                    }
                },

                createdAt: {
                    bsonType: "date"
                }
            }
        }
    }
})
```

Then you can add a semantic validator such as:

```text
salaryMax >= salaryMin
```

using an additional expression if your design requires database-level enforcement of that invariant. MongoDB supports combining `$jsonSchema` with query-expression validation. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 72. Notice something important about this design

We didn't validate:

```text
company exists
```

That's because:

```text
companyId → ObjectId
```

only validates the **type**.

It doesn't mean the referenced document actually exists.

That's an important difference between:

```text
type constraint
```

and:

```text
referential integrity
```

MongoDB does not automatically turn `companyId` into a relational foreign key simply because you validate it as an ObjectId.

Your application or a different consistency strategy has to handle that relationship.

---

# 73. Schema validation and references

This ties directly back to Lesson 3.

Suppose:

```javascript
application: {
    candidateId: ObjectId("..."),
    jobId: ObjectId("...")
}
```

Validator can guarantee:

```text
candidateId → objectId
jobId       → objectId
```

But not automatically:

```text
that candidate exists
that job exists
```

So:

```text
MongoDB validation
    ↓
reference shape is valid

Application/database logic
    ↓
reference relationship is valid
```

This distinction is very important for MongoDB because references are application-level relationships rather than relational foreign keys.

---

# 74. Validation doesn't replace indexes

Suppose:

```text
email must be string
email must be unique
```

Your validator can handle:

```text
string
```

but uniqueness should normally be enforced by:

```javascript
db.users.createIndex(
    { email: 1 },
    { unique: true }
)
```

So your architecture could be:

```text
Schema validator:
email → string

Unique index:
email → unique

Backend:
email format/business checks
```

Three different concerns.

---

# 75. Common beginner mistakes

## Mistake 1: “MongoDB has no schema.”

Not really.

MongoDB has flexible schema by default and supports explicit schema validation. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/?utm_source=chatgpt.com "Schema Validation - Database Manual - MongoDB Docs"))

---

## Mistake 2: Making every field required

Don't automatically do:

```javascript
required: [
    "everyPossibleField"
]
```

Optional fields are often legitimate.

MongoDB explicitly notes that validation does not need to cover every field unless your application requires it. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/?utm_source=chatgpt.com "Schema Validation - Database Manual - MongoDB Docs"))

---

## Mistake 3: Using validation for authorization

This:

```text
receiverId must be ObjectId
```

is validation.

This:

```text
only receiver can mark message as seen
```

is authorization/business logic.

Don't mix them.

---

## Mistake 4: Using `additionalProperties: false` blindly

It can make schema evolution unnecessarily painful, and you must account for `_id`. MongoDB specifically calls this out. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/json-schema-tips/?utm_source=chatgpt.com "Tips for JSON Schema Validation - Database Manual - MongoDB Docs"))

---

## Mistake 5: Adding strict validation to dirty historical data without a migration plan

You may have existing invalid documents.

MongoDB's `moderate`, `warn`, and schema-based queries can help you migrate incrementally. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/use-json-schema-query-conditions/?utm_source=chatgpt.com "Query for and Modify Valid or Invalid Documents - Database Manual - MongoDB Docs"))

---

## Mistake 6: Assuming validation checks old documents immediately

Adding a validator doesn't automatically rewrite or repair existing documents. Existing records encounter validation when affected by writes according to the configured level. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/collmod/?utm_source=chatgpt.com "collMod (database command) - Database Manual - MongoDB Docs"))

---

# 76. The complete mental model

Think of MongoDB validation like this:

```text
                     COLLECTION
                         │
                         ↓
                   VALIDATION RULES
                         │
             ┌───────────┴───────────┐
             ↓                       ↓
        $jsonSchema              Query rules
             │                       │
             ↓                       ↓
        Structure                Semantics
             │                       │
      ┌──────┼──────┐                │
      ↓      ↓      ↓                ↓
   required type  range          cross-field
      │             │                │
      └─────────────┴────────────────┘
                         ↓
                  validation result
                         │
                ┌────────┴────────┐
                ↓                 ↓
              valid             invalid
                │                 │
                ↓                 ↓
              WRITE       error / warn / errorAndLog
```

And separately:

```text
validationLevel
    ↓
when/which documents are checked

validationAction
    ↓
what happens when they fail
```

---

# 77. What I want you to be able to design now

Given:

```javascript
{
    username: "riya24",

    age: 23,

    role: "candidate",

    skills: [
        "Java",
        "MongoDB"
    ],

    address: {
        city: "Pune",
        country: "India"
    }
}
```

you should now be able to reason:

```text
username
    → required string

age
    → integer + range

role
    → enum

skills
    → array
    → each element string
    → bounded array

address
    → object
    → nested rules
```

That is the actual skill this lesson is trying to build.

---

# 78. Exercises

A small set, but these are much more conceptual than syntax drills.

## Exercise 1 — Design a validator

Create validation for:

```javascript
{
    username: "arjun_dev",
    age: 24,
    role: "candidate"
}
```

Rules:

```text
username → string, 3–20 chars
age      → integer, 18–100
role     → candidate/recruiter/admin
```

### Answer

```javascript
{
    $jsonSchema: {
        bsonType: "object",

        required: [
            "username",
            "age",
            "role"
        ],

        properties: {
            username: {
                bsonType: "string",
                minLength: 3,
                maxLength: 20
            },

            age: {
                bsonType: "int",
                minimum: 18,
                maximum: 100
            },

            role: {
                enum: [
                    "candidate",
                    "recruiter",
                    "admin"
                ]
            }
        }
    }
}
```

---

# 79. Exercise 2 — Find the problem

You have:

```javascript
skills: {
    bsonType: "array"
}
```

Does this guarantee that all skills are strings?

### Answer

No.

It only says:

```text
skills itself must be an array
```

To constrain every element:

```javascript
skills: {
    bsonType: "array",

    items: {
        bsonType: "string"
    }
}
```

MongoDB's JSON Schema uses `items` to define rules for array elements. ([MongoDB](https://www.mongodb.com/docs/compass/schema/export/?utm_source=chatgpt.com "Export Your Schema - Compass - MongoDB Docs"))

---

# 80. Exercise 3 — Conceptual

What is the difference?

```text
validationLevel
```

vs

```text
validationAction
```

### Answer

```text
validationLevel
    → determines which inserts/updates are subject to validation

validationAction
    → determines what happens when a document violates validation
```

For example:

```javascript
validationLevel: "moderate",
validationAction: "warn"
```

means:

```text
moderate validation scope
+
allow violations but log them
```

MongoDB documents these as separate collection-validation controls. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-level/?utm_source=chatgpt.com "Specify Validation Level for Existing Documents - Database Manual - MongoDB Docs"))

---

# 81. Exercise 4 — Difficult

You have:

```javascript
{
    salaryMin: 800000,
    salaryMax: 500000
}
```

Both fields individually satisfy:

```text
integer
minimum = 0
```

But the document is still logically invalid.

Why can't simple `minimum`/`maximum` validation solve this?

### Answer

Because the rule is:

```text
salaryMax >= salaryMin
```

That compares **two fields**.

It's not a property of either field independently.

You therefore need an additional expression/query condition, such as using `$expr`, combined with `$jsonSchema` if you want the database validator to enforce it. MongoDB's official schema-validation documentation demonstrates combining `$jsonSchema` with `$expr` for cross-field rules. ([MongoDB](https://www.mongodb.com/docs/manual/core/schema-validation/specify-json-schema/?utm_source=chatgpt.com "Specify JSON Schema Validation - Database Manual - MongoDB Docs"))

---

# 82. Difficult conceptual question

Which statement is correct?

**A.** MongoDB schema validation makes Mongoose validation unnecessary.

**B.** `required` means the field must contain a non-null string.

**C.** A unique index and schema validation solve exactly the same problem.

**D.** Schema validation can enforce document structure/types, while unique indexes can enforce uniqueness; application logic still handles many business and authorization rules.

### Answer

**D**

That's the architecture you should remember.

---

# 83. The bigger lesson

Look at what we've built across the last few lessons:

```text
Lesson 3
    ↓
Data Modeling
    ↓
What should documents look like?

Lesson 4
    ↓
Updates
    ↓
How do I safely change them?

Lesson 5–6
    ↓
Aggregation
    ↓
How do I transform/analyze them?

Lesson 7
    ↓
Indexes
    ↓
How do I make important queries efficient?

Lesson 8
    ↓
Schema Validation
    ↓
How do I prevent bad data from entering/staying in the database?
```

So MongoDB isn't just:

```text
insert()
find()
update()
delete()
```

You're now learning the actual database-engineering layer:

```text
             MongoDB
                │
    ┌───────────┼────────────┐
    ↓           ↓            ↓
Data Model    Indexes    Validation
    ↓           ↓            ↓
structure   performance   integrity
```

That combination is what makes your future backend projects much stronger.

---

# 84. One final principle

There is a useful progression here:

> **Application code should prevent invalid operations.**

> **Database constraints should prevent invalid stored state.**

> **Indexes should make important access patterns efficient.**

And these are different jobs.

For your ChatApp, for example:

```text
Authentication
    → backend/security

"Only receiver can mark message seen"
    → authorization/business logic

seen must be boolean
    → schema validation

senderId/receiverId lookup
    → indexes

messages between two users
    → data model + compound index
```

That's the kind of thinking I want you to develop, rather than simply collecting MongoDB commands.

---

### Lesson 8 complete.

**Next lesson: MongoDB Transactions & Concurrency.**

This one is particularly important because we'll answer questions like:

```text
What does "atomic" actually mean?

Why is one-document atomicity special?

What happens when two requests update the same document simultaneously?

What are race conditions?

When do you need a transaction?

Why can transactions be expensive?

How do sessions work?

What are read/write concerns?

How does MongoDB prevent inconsistent multi-document operations?
```

We'll connect all of that back to **ChatApp messaging**, **unread counts**, and **JobForge applications**, so you understand not just how transactions work, but **when a backend developer should and should not use one**.