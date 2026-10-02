
# MongoDB — Lesson 3: Data Modeling

This is one of the **most important MongoDB lessons** because MongoDB design is not primarily about translating an SQL schema into collections.

The central idea is:

> **In MongoDB, you design your data around how your application will use it.**

MongoDB explicitly recommends considering application workload and access patterns when deciding whether related data should be embedded or referenced. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/?utm_source=chatgpt.com "Data Modeling in MongoDB - Database Manual - MongoDB Docs"))

---

# 1. First: what does “data modeling” actually mean?

Suppose you're building a job portal.

You have:

```text
User
Job
Application
Company
Candidate Profile
```

Data modeling means deciding:

- What should be a document?
    
- What should be a collection?
    
- Which information should live inside another document?
    
- Which information should be stored separately?
    
- How should documents be connected?
    
- What queries should be fast?
    
- How much duplication is acceptable?
    

In SQL, you often start by thinking:

```text
tables → primary keys → foreign keys → joins
```

In MongoDB, a better starting point is:

```text
What will my application frequently read together?
What will my application frequently update together?
How large can the related data become?
How frequently does the relationship change?
```

That shift in thinking is extremely important.

---

# 2. SQL normalization vs MongoDB modeling

Imagine this SQL design:

```text
users
-----
id
name
email
phone

addresses
---------
id
user_id
city
country
```

The address is separate because relational databases are good at joining related tables.

MongoDB could instead store:

```javascript
{
  _id: ObjectId("..."),
  name: "Sanmit",
  email: "sanmit@example.com",
  phone: "9999999999",

  address: {
    city: "Pune",
    country: "India"
  }
}
```

Now user + address can be retrieved with one document read.

This is called **embedding**.

MongoDB supports both embedding and referencing; the right choice depends on the application's access patterns. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/embedding/?utm_source=chatgpt.com "Embedded Data in Your MongoDB Schema - Database Manual - MongoDB Docs"))

So:

> MongoDB does **not** mean “never normalize.”

And:

> MongoDB does **not** mean “put everything inside one giant document.”

Both extremes can be bad.

---

# 3. The two fundamental strategies

There are really two big choices.

## A. Embed

Put related information inside the same document.

```javascript
{
  name: "Sanmit",

  address: {
    city: "Pune",
    country: "India"
  }
}
```

Or:

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

---

## B. Reference

Store related information in another document and keep an ID/reference.

```javascript
{
  _id: ObjectId("user1"),
  name: "Sanmit"
}
```

Another collection:

```javascript
{
  _id: ObjectId("job1"),
  title: "Backend Developer",
  recruiterId: ObjectId("user1")
}
```

The two documents are related by the ID.

MongoDB's documentation describes these as the two primary approaches to representing relationships. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/?utm_source=chatgpt.com "Data Modeling in MongoDB - Database Manual - MongoDB Docs"))

---

# 4. The most important question: “Will this data be read together?”

Suppose you have:

```javascript
{
  name: "Sanmit",
  address: {
    city: "Pune",
    country: "India"
  }
}
```

Your application frequently displays:

```text
Name
Email
City
Country
```

Embedding is natural because these pieces of information belong together and are frequently retrieved together.

MongoDB specifically notes that embedding can let an application retrieve related information in a single database operation. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/embedding/?utm_source=chatgpt.com "Embedded Data in Your MongoDB Schema - Database Manual - MongoDB Docs"))

Now imagine instead:

```text
User
   ↓
10 million transactions
```

Putting millions of transactions inside the user's document would obviously be terrible.

So the first mental rule is:

> **Frequently accessed together + bounded relationship → embedding becomes attractive.**

---

# 5. Relationships

Before deciding embed/reference, identify the relationship.

There are three major relationship types:

```text
1. One-to-One
2. One-to-Many
3. Many-to-Many
```

MongoDB itself uses these relationship categories when discussing schema design. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/?utm_source=chatgpt.com "Data Modeling in MongoDB - Database Manual - MongoDB Docs"))

Let's understand them properly.

---

# 6. One-to-One

One entity is associated with exactly one other entity.

Example:

```text
User → Profile
```

One user has one profile.

You could embed:

```javascript
{
  _id: ObjectId("u1"),
  username: "sanmit",

  profile: {
    firstName: "Sanmit",
    lastName: "X",
    bio: "Backend developer",
    city: "Pune"
  }
}
```

This is usually very convenient.

### Why?

Because when displaying the user profile:

```text
GET /profile
```

you probably want all of this together.

---

## When would you reference instead?

Suppose the profile is:

- very large
    
- accessed independently
    
- updated frequently
    
- shared by multiple entities
    

Then:

```text
users
profiles
```

with a reference may make more sense.

So relationship type **does not automatically determine** embed vs reference.

---

# 7. One-to-Few

This is a very useful practical case.

Example:

```text
User → 3 addresses
```

A user may have:

```text
Home
Work
Parents' house
```

You can reasonably embed:

```javascript
{
  _id: ObjectId("u1"),

  name: "Sanmit",

  addresses: [
    {
      type: "home",
      city: "Pune"
    },
    {
      type: "work",
      city: "Mumbai"
    }
  ]
}
```

The array is small and bounded.

This is a great embedding candidate.

---

# 8. One-to-Many

Now:

```text
User → many posts
```

For example:

```text
Sanmit
 ├── Post 1
 ├── Post 2
 ├── Post 3
 ├── ...
 └── Post 500
```

There are two possible designs.

### Design A — embed

```javascript
{
  _id: ObjectId("u1"),
  name: "Sanmit",

  posts: [
    {
      title: "MongoDB Lesson",
      content: "..."
    },
    {
      title: "Spring Boot",
      content: "..."
    }
  ]
}
```

Looks convenient.

But now consider 500,000 posts.

The parent document keeps growing.

That brings us to one of the most important MongoDB anti-patterns.

---

# 9. Never blindly create unbounded arrays

MongoDB documents have a **16 MiB maximum BSON document size**. MongoDB also warns that unbounded arrays can create large documents, hurt index performance, and eventually exceed the document-size limit. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/embedding/?utm_source=chatgpt.com "Embedded Data in Your MongoDB Schema - Database Manual - MongoDB Docs"))

Bad:

```javascript
{
  username: "sanmit",

  messages: [
    // potentially millions
  ]
}
```

Bad:

```javascript
{
  productName: "Laptop",

  reviews: [
    // potentially millions
  ]
}
```

Bad:

```javascript
{
  userId: "...",

  notifications: [
    // keeps growing forever
  ]
}
```

This is called the **unbounded array anti-pattern**. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/?utm_source=chatgpt.com "Schema Design Anti-Patterns - Database Manual - MongoDB Docs"))

---

# 10. One-to-Squillions

This is a useful way to think about huge one-to-many relationships.

Example:

```text
Server → millions of log entries
```

Clearly don't do:

```javascript
{
  serverId: "server1",
  logs: [
    ...
  ]
}
```

Instead:

```text
servers
logs
```

Each log references the server:

```javascript
{
  _id: ObjectId("log1"),
  serverId: ObjectId("server1"),
  level: "ERROR",
  message: "Database connection failed",
  timestamp: ISODate("...")
}
```

That's a classic reason to use references.

---

# 11. One-to-Many: where should the reference go?

This is an important design idea.

Suppose:

```text
Author → Books
```

Instead of putting:

```javascript
{
  authorId: "...",
  books: [
    "...",
    "...",
    "..."
  ]
}
```

you can store the reference on the many side:

```javascript
{
  _id: ObjectId("book1"),
  title: "MongoDB Internals",
  authorId: ObjectId("author1")
}
```

Now every book knows its author.

This works particularly well when the “many” side is large.

---

# 12. Many-to-Many

Example:

```text
Student ↔ Course
```

One student:

```text
Java
MongoDB
Machine Learning
```

One course:

```text
Sanmit
Rahul
Aditi
```

Both sides contain many relationships.

You could model this using arrays of references:

```javascript
{
  _id: ObjectId("student1"),
  name: "Sanmit",
  courses: [
    ObjectId("course1"),
    ObjectId("course2")
  ]
}
```

and:

```javascript
{
  _id: ObjectId("course1"),
  name: "MongoDB",
  students: [
    ObjectId("student1"),
    ObjectId("student2")
  ]
}
```

But now notice the problem.

If Sanmit leaves the course, you may need to update both documents.

This creates a **consistency problem**.

MongoDB's documentation notes that embedding may not be appropriate for complex relationships such as many-to-many relationships, and references or other modeling patterns may be preferable. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/enforce-consistency/embed-data/?utm_source=chatgpt.com "Enforce Data Consistency with Embedding - Database Manual - MongoDB Docs"))

---

# 13. The real trade-off: duplication vs joins

This is where MongoDB starts getting interesting.

Suppose we have:

```text
User
```

and:

```text
Job
```

A job frequently displays:

```text
Recruiter name
Recruiter company
Recruiter profile picture
```

Option 1:

```javascript
{
  title: "Java Developer",
  recruiterId: ObjectId("u1")
}
```

Then the application has to retrieve the recruiter separately.

Option 2:

```javascript
{
  title: "Java Developer",

  recruiter: {
    id: ObjectId("u1"),
    name: "Rahul",
    company: "ABC Technologies"
  }
}
```

Now the job contains a copy of recruiter information.

That's **denormalization**.

---

# 14. “But duplication is bad!”

In traditional database thinking, duplication often sounds dangerous.

But MongoDB deliberately allows denormalization when it improves application access patterns.

Suppose recruiter information is:

```text
name
company
profilePhoto
```

and those values rarely change.

Embedding a copy may be perfectly reasonable.

But now imagine the recruiter changes their company name.

You've potentially got:

```text
Job 1 → old company
Job 2 → old company
Job 3 → old company
Job 4 → old company
...
```

You now have a consistency problem.

MongoDB explicitly discusses this trade-off: duplicated data may need consistency management, while embedding can eliminate certain duplication patterns. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/enforce-consistency/embed-data/?utm_source=chatgpt.com "Enforce Data Consistency with Embedding - Database Manual - MongoDB Docs"))

So:

> **Duplication is not automatically bad. Uncontrolled duplication is bad.**

---

# 15. Think in terms of read frequency and write frequency

This is an extremely useful design heuristic.

Suppose:

```text
Data A changes once a year.
Data B is read 100,000 times/day.
```

Embedding A into B may be reasonable.

But suppose:

```text
Data A changes 100 times/second.
Data B contains a copy of A in 1 million documents.
```

Now updating A becomes painful.

This gives you an important question:

> **How often is this data read, and how often is it updated?**

MongoDB's modeling guidance explicitly weighs query performance, returning related entities together, and update performance when choosing embedding vs references. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/map-relationships/?utm_source=chatgpt.com "Map Schema Relationships - Database Manual - MongoDB Docs"))

---

# 16. Atomicity changes your modeling decision

This is a major concept.

A write involving a **single MongoDB document is atomic**.

Suppose:

```javascript
{
  _id: ObjectId("u1"),

  balance: 1000,

  address: {
    city: "Pune"
  }
}
```

You can update multiple fields within the document as one atomic operation.

This is one reason embedding can be powerful:

```text
Related data
     ↓
same document
     ↓
single atomic update
```

MongoDB supports multi-document transactions when atomicity across multiple documents is required, but its documentation notes that transactions generally carry more cost than single-document writes and should not replace sound schema design. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

So another important rule is:

> **If pieces of data must always change together, keeping them in one document can simplify correctness.**

---

# 17. Example: Order and Order Items

Suppose an online store has:

```text
Order
Order Items
```

An order might contain:

```javascript
{
  _id: ObjectId("order1"),
  userId: ObjectId("user1"),

  items: [
    {
      productId: ObjectId("p1"),
      name: "Keyboard",
      price: 2500,
      quantity: 2
    },

    {
      productId: ObjectId("p2"),
      name: "Mouse",
      price: 1000,
      quantity: 1
    }
  ],

  total: 6000
}
```

Why might embedding be excellent here?

Because an order is naturally viewed as a unit:

```text
Get Order
  ↓
Get all items
  ↓
Calculate/display order
```

And once the order is placed, the historical item information should usually remain stable.

Notice something interesting:

We stored:

```text
productId
name
price
```

instead of only:

```text
productId
```

Why?

Because the product's current price may change later.

Suppose:

```text
Keyboard price today = ₹2500
```

and tomorrow:

```text
Keyboard price = ₹3000
```

The old order still needs to say:

```text
₹2500
```

So duplication here is actually useful.

This is a very important real-world MongoDB principle:

> **Sometimes duplicated historical data is intentional because the old state matters.**

---

# 18. ChatApp: let's model your project properly

This is where I want you to connect this lesson to your existing project.

Your ChatApp has:

```text
User
Message
```

A message essentially contains:

```text
sender
receiver
text
image
seen
timestamp
```

A message could be:

```javascript
{
  _id: ObjectId("m1"),

  senderId: ObjectId("u1"),
  receiverId: ObjectId("u2"),

  text: "Hello",

  image: null,

  seen: false,

  createdAt: ISODate("...")
}
```

Notice what we **should not** do:

```javascript
{
  sender: {
    name: "...",
    email: "...",
    profilePic: "..."
  },

  receiver: {
    name: "...",
    email: "...",
    profilePic: "..."
  }
}
```

inside every message.

Why?

Imagine 1 million messages.

You would duplicate:

```text
sender name
sender email
sender profile picture
receiver name
receiver email
...
```

over and over.

That becomes extremely wasteful.

Instead:

```text
users
messages
```

with:

```text
messages.senderId
messages.receiverId
```

This is a good example where **referencing is more appropriate** because messages are numerous and user information can change independently.

---

# 19. What about conversations?

Suppose later your ChatApp supports:

```text
1-to-1 chat
group chat
group members
admins
last message
unread count
```

Now you might introduce:

```text
conversations
```

Example:

```javascript
{
  _id: ObjectId("c1"),

  type: "group",

  participants: [
    ObjectId("u1"),
    ObjectId("u2"),
    ObjectId("u3")
  ],

  name: "Backend Team",

  createdAt: ISODate("...")
}
```

Then:

```text
messages
    ↓
conversationId
```

This gives you:

```text
Conversation
      ↓
many messages
      ↓
each message references conversation
```

Now think about scale.

You would **not** put all messages inside:

```javascript
conversation.messages = [...]
```

because that's an unbounded array problem. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/?utm_source=chatgpt.com "Schema Design Anti-Patterns - Database Manual - MongoDB Docs"))

---

# 20. JobForge: let's apply the same thinking

Your future project could have:

```text
User
CandidateProfile
Job
Application
Company
```

A natural design might be:

```text
users
candidateProfiles
jobs
applications
companies
```

### User

```javascript
{
  _id: ObjectId("u1"),
  name: "Sanmit",
  email: "sanmit@example.com",
  role: "candidate"
}
```

### Candidate profile

```javascript
{
  _id: ObjectId("p1"),
  userId: ObjectId("u1"),

  skills: [
    "Java",
    "Spring Boot",
    "React",
    "MongoDB"
  ],

  experienceYears: 1,

  education: {
    degree: "BCA",
    university: "..."
  }
}
```

### Job

```javascript
{
  _id: ObjectId("j1"),

  title: "Java Backend Developer",
  companyId: ObjectId("c1"),

  skills: [
    "Java",
    "Spring Boot",
    "PostgreSQL"
  ],

  location: "Pune",

  salaryMin: 600000,
  salaryMax: 900000
}
```

### Application

```javascript
{
  _id: ObjectId("a1"),

  jobId: ObjectId("j1"),
  candidateId: ObjectId("u1"),

  status: "interview",

  appliedAt: ISODate("...")
}
```

Notice the pattern:

```text
User
  ↑
candidateId
  │
Application
  │
jobId
  ↓
Job
```

Applications are their own entities because they have their own important state:

```text
status
appliedAt
resume
interview
rejection reason
notes
```

They aren't merely properties of a user or a job.

That's an important modeling concept:

> **Sometimes the relationship itself becomes an entity.**

---

# 21. Many-to-many in JobForge

Consider:

```text
Candidate ↔ Skill
```

A candidate can have many skills.

A skill can belong to many candidates.

You could store candidate skills as strings:

```javascript
skills: [
  "Java",
  "Spring Boot",
  "MongoDB"
]
```

This can be perfectly fine if skills are simple attributes.

But suppose later you need a proper:

```text
Skill
-----
_id
name
category
popularity
```

Then a separate collection might make sense.

This demonstrates another important idea:

> **Don't create a separate collection just because something has a name. Create it when it behaves like an entity that needs independent querying, updating, relationships, or lifecycle.**

---

# 22. Don't over-normalize MongoDB

A beginner often learns:

```text
Good database = many separate collections
```

and ends up with something ridiculous like:

```text
users
user_names
user_emails
user_addresses
user_skills
user_phone_numbers
```

Then retrieving one user requires many operations.

MongoDB's model exists partly to avoid unnecessary fragmentation of closely related data.

MongoDB recommends considering whether related data is accessed together and notes that embedding can avoid frequent `$lookup` operations. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/embedding/?utm_source=chatgpt.com "Embedded Data in Your MongoDB Schema - Database Manual - MongoDB Docs"))

---

# 23. Don't over-embed either

The opposite beginner mistake is:

```text
One giant document containing everything.
```

For example:

```javascript
{
  user: {
    profile: {...},
    posts: [...],
    messages: [...],
    notifications: [...],
    applications: [...],
    jobs: [...],
    transactions: [...]
  }
}
```

This becomes terrible.

Why?

```text
Huge document
     ↓
Large reads
Large updates
Growing arrays
More memory
Possible 16 MiB limit
Poor scalability
```

MongoDB identifies excessively large documents and unbounded arrays as schema anti-patterns. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/?utm_source=chatgpt.com "Schema Design Anti-Patterns - Database Manual - MongoDB Docs"))

---

# 24. The most useful decision framework

When deciding:

```text
Embed or Reference?
```

ask these questions in roughly this order.

### Question 1 — Are the two things usually read together?

```text
YES → embedding becomes attractive
NO  → reference becomes more attractive
```

### Question 2 — Is the amount of related data bounded?

```text
YES → embedding can work
NO  → be careful
```

### Question 3 — Does the child data grow continuously?

```text
YES → reference is often safer
```

### Question 4 — Does the related data change independently?

```text
YES → reference becomes more attractive
```

### Question 5 — Must they update together atomically?

```text
YES → embedding becomes attractive
```

### Question 6 — Will duplication be expensive or difficult to keep consistent?

```text
YES → reference may be better
```

### Question 7 — Do you often query the child independently?

```text
YES → separate collection/reference becomes more attractive
```

This is not a rigid formula.

It's a way of thinking.

---

# 25. A very important concept: access patterns

Consider two designs.

### Design A

```text
users
messages
```

Query:

```text
Give me all messages between A and B
```

This is natural.

### Design B

```text
users
{
  messages: [...]
}
```

Query:

```text
Give me last 20 messages between A and B
```

Now you've embedded a potentially huge amount of data inside users.

That's a poor model for the workload.

This is why MongoDB recommends beginning data modeling from the application's workload and relationships rather than blindly copying relational schemas. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/map-relationships/?utm_source=chatgpt.com "Map Schema Relationships - Database Manual - MongoDB Docs"))

---

# 26. Schema design is really about queries

This is probably the **single most important lesson of Lesson 3**.

Imagine someone asks:

> "Should I embed comments inside posts?"

There is no universal answer.

You need to ask:

```text
How many comments can a post have?
Are comments usually displayed with the post?
Do we need to query comments independently?
Can comments become huge?
How often are comments updated?
Do we paginate comments?
```

Then decide.

So:

```text
Bad mindset:
"What is the theoretically pure schema?"

Better mindset:
"What schema makes the important operations efficient and correct?"
```

---

# 27. A practical example: Instagram-like post

Suppose:

```text
Post
 ├── author
 ├── caption
 ├── image
 ├── likes
 └── comments
```

A naive design:

```javascript
{
  authorId: ObjectId("u1"),
  caption: "Hello",

  likes: [
    ObjectId("u1"),
    ObjectId("u2"),
    ...
  ],

  comments: [
    {...},
    {...},
    {...},
    ...
  ]
}
```

This might work for tiny numbers.

But:

```text
likes → potentially millions
comments → potentially millions
```

Now we are back to unbounded arrays.

A scalable design might separate:

```text
posts
comments
likes
```

or use specific schema patterns depending on the workload.

MongoDB specifically recommends bounding arrays through approaches such as subsetting or references when arrays can become very large. ([MongoDB](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/unbounded-arrays/?utm_source=chatgpt.com "Avoid Unbounded Arrays - Database Manual v8.0 - MongoDB Docs"))

---

# 28. The “subset” idea

Suppose a product has:

```text
50,000 reviews
```

but your page only shows:

```text
latest 5 reviews
```

You don't necessarily need to embed all 50,000.

You could keep a small subset inside the product:

```javascript
{
  _id: ObjectId("p1"),

  name: "Keyboard",

  recentReviews: [
    {...},
    {...},
    {...}
  ]
}
```

and maintain all reviews separately:

```text
reviews
```

This is called the **subset pattern**.

Conceptually:

```text
Large dataset
     ↓
keep frequently needed subset embedded
     +
keep complete dataset separately
```

This is one of the ways MongoDB addresses large/unbounded arrays. ([MongoDB](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/unbounded-arrays/?utm_source=chatgpt.com "Avoid Unbounded Arrays - Database Manual v8.0 - MongoDB Docs"))

---

# 29. Embedding and “data locality”

Embedding has another advantage that isn't obvious at first.

Suppose:

```javascript
{
  name: "Sanmit",

  address: {
    city: "Pune"
  },

  education: {
    degree: "BCA"
  }
}
```

All of this belongs to the same document.

When MongoDB retrieves the user, related information is located together.

Conceptually:

```text
Disk/data locality
       ↓
same document
       ↓
related data retrieved together
```

That's one reason embedding can reduce the need for extra lookups. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/embedding/?utm_source=chatgpt.com "Embedded Data in Your MongoDB Schema - Database Manual - MongoDB Docs"))

---

# 30. A subtle but important rule: “ownership”

Ask:

> Does this piece of data belong exclusively to the parent?

Example:

```text
User
 └── address
```

Usually yes.

So embedding makes sense.

But:

```text
User
 └── Job
```

No.

A job isn't owned exclusively by one user in the same sense. A job can exist independently, be searched independently, and have many applications.

So it makes sense as a separate entity.

---

# 31. Another subtle rule: lifecycle

Ask:

> Does this data have an independent lifecycle?

Example:

```text
Application
```

has:

```text
created
submitted
reviewed
interviewed
accepted/rejected
```

It clearly has its own lifecycle.

Therefore:

```text
applications
```

deserves to be its own collection/entity.

Compare that with:

```text
User.address
```

which may simply be part of user information.

This is a powerful modeling heuristic.

---

# 32. The three big axes you should remember

Almost every embedding/reference decision can be understood through these:

### 1. Cardinality

How many related documents?

```text
one
few
many
millions
```

### 2. Access pattern

Are they:

```text
read together?
read independently?
updated together?
```

### 3. Growth

Will the relationship:

```text
stay small?
grow indefinitely?
```

If you understand those three, MongoDB data modeling starts becoming much easier.

---

# 33. A simple mental model

Use this:

```text
                    RELATED DATA
                         │
             ┌───────────┴───────────┐
             │                       │
        Read together?          Read separately?
             │                       │
            YES                      YES
             │                       │
       Small/bounded?          Independent lifecycle?
          /       \                 /        \
        YES       NO              YES        NO
         │         │               │          │
       EMBED     CAREFUL        REFERENCE   Analyze workload
```

This is not a law.

It is a decision aid.

---

# 34. Five common MongoDB modeling mistakes

### Mistake 1: “MongoDB is schema-less, so design doesn't matter.”

Wrong.

MongoDB's flexible schema does not mean your application should have a random structure.

Bad schema design can still create major performance and consistency problems. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/?utm_source=chatgpt.com "Schema Design Anti-Patterns - Database Manual - MongoDB Docs"))

---

### Mistake 2: Embedding everything

```text
User
 ├── posts
 ├── messages
 ├── notifications
 ├── applications
 └── transactions
```

Eventually the document becomes enormous.

---

### Mistake 3: Referencing everything

```text
user
 └── profileId

profile
 └── addressId

address
 └── cityId

city
 └── countryId
```

Now a simple profile page becomes unnecessarily complicated.

---

### Mistake 4: Ignoring growth

```javascript
comments: [...]
```

might look fine today.

But ask:

```text
What happens when there are 10 million?
```

Always think about growth.

---

### Mistake 5: Designing before thinking about queries

A schema can look beautiful on paper and still perform badly for the application's actual workload.

MongoDB recommends explicitly identifying application workload and mapping relationships before making these design decisions. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/map-relationships/?utm_source=chatgpt.com "Map Schema Relationships - Database Manual - MongoDB Docs"))

---

# 35. Exercises

Keeping this lighter than the previous lessons.

## Exercise 1 — Design decision

You have:

```text
User
 └── phoneNumbers
```

A user can have at most 3 phone numbers.

Would you embed or reference?

### Answer

**Embed.**

Reason:

```text
small
bounded
belongs to user
usually retrieved with user
```

Example:

```javascript
{
  name: "Sanmit",
  phoneNumbers: [
    "+91...",
    "+91..."
  ]
}
```

---

## Exercise 2 — Design decision

You have:

```text
User → Notifications
```

A user may receive millions of notifications over several years.

Embed or reference?

### Answer

**Reference/separate collection.**

The relationship is potentially unbounded.

Do not create:

```javascript
notifications: [...]
```

inside the user document. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/?utm_source=chatgpt.com "Schema Design Anti-Patterns - Database Manual - MongoDB Docs"))

---

## Exercise 3 — Concept

Which is generally more important when designing a MongoDB schema?

A. Making every entity fully normalized  
B. Making the schema look similar to an ER diagram  
C. Optimizing around actual application access patterns  
D. Minimizing collection count at all costs

### Answer

**C**

MongoDB's data-modeling guidance explicitly emphasizes workload/access patterns in choosing how data should be modeled. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/map-relationships/?utm_source=chatgpt.com "Map Schema Relationships - Database Manual - MongoDB Docs"))

---

## Exercise 4 — JobForge

You have:

```text
Candidate
Candidate's applications
```

Each application contains:

```text
jobId
status
appliedAt
```

Should applications be an array inside the candidate document?

### Answer

For a real job portal, I'd model **applications as a separate collection**.

Why?

```text
Candidate
   ↓
many applications
   ↓
potentially large
   ↓
applications have their own lifecycle/state
```

Example:

```javascript
{
  _id: ObjectId("a1"),
  candidateId: ObjectId("u1"),
  jobId: ObjectId("j1"),
  status: "interview",
  appliedAt: ISODate("...")
}
```

---

# 36. One final challenge

You are designing an e-commerce product.

```text
Product
```

has:

```text
name
price
description
reviews
```

A product can have **millions of reviews**.

But the homepage only displays:

```text
average rating
latest 3 reviews
```

How would you model it?

Think for a moment.

### A sensible design

```text
products
reviews
```

And in the product:

```javascript
{
  name: "Keyboard",
  averageRating: 4.6,

  recentReviews: [
    {...},
    {...},
    {...}
  ]
}
```

while the complete reviews remain in:

```text
reviews
```

This combines:

```text
embedding
+
referencing
+
bounded subset
```

That is much closer to how serious MongoDB schema design works than simply saying “MongoDB = embed everything.” ([MongoDB](https://www.mongodb.com/docs/v8.0/data-modeling/design-antipatterns/unbounded-arrays/?utm_source=chatgpt.com "Avoid Unbounded Arrays - Database Manual v8.0 - MongoDB Docs"))

---

# 37. What you should remember from Lesson 3

Don't memorize 50 rules. Remember these:

```text
MongoDB schema design starts with workload.
```

```text
Embed when related data naturally belongs together
and is small/bounded and commonly accessed together.
```

```text
Reference when related data is large, independently managed,
independently queried, or continuously growing.
```

```text
Avoid unbounded arrays.
```

```text
Duplication is sometimes intentional.
```

```text
Single-document atomicity can be a major modeling advantage.
```

```text
Don't blindly normalize.
Don't blindly denormalize.
```

And the biggest one:

> **You don't design a MongoDB database first and then figure out your queries. You think about the application's important queries and operations, then design the database around them.**

That principle will become even more important when we reach **indexes, aggregation, and performance**. MongoDB's own modeling process is built around workload, relationships, and the resulting access patterns. ([MongoDB](https://www.mongodb.com/docs/manual/data-modeling/schema-design-process/map-relationships/?utm_source=chatgpt.com "Map Schema Relationships - Database Manual - MongoDB Docs"))

### Lesson 3 complete.

**Next: Lesson 4 — Updates and Array Operations.**  
We'll go much deeper into `$set`, `$unset`, `$inc`, `$push`, `$pull`, `$addToSet`, positional operators, updating nested arrays, array filters, and the difference between replacing a document and updating specific fields.