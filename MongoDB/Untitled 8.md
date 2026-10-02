
# MongoDB — Lesson 9: Transactions & Concurrency

This is one of the lessons that separates:

```text
"I know MongoDB CRUD"
```

from:

```text
"I understand how a database behaves when a real application has
multiple requests happening at the same time."
```

The central problem is **concurrency**.

Your backend may have:

```text
User A → request 1
User B → request 2
Admin  → request 3
Background worker → request 4
```

all hitting MongoDB simultaneously.

You need to understand:

- atomicity
    
- race conditions
    
- lost updates
    
- isolation
    
- transactions
    
- sessions
    
- commit / abort
    
- optimistic concurrency
    
- read concern
    
- write concern
    
- transaction retries
    
- when transactions are unnecessary
    

MongoDB gives you **single-document atomicity by default**, and supports multi-document ACID transactions when atomicity across multiple documents or collections is actually required. MongoDB also explicitly recommends using good schema design to minimize the need for multi-document transactions. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 1. Start with the word: atomicity

You've already encountered this in Lesson 3, but now we're going much deeper.

Suppose this document exists:

```javascript
{
    _id: 1,
    balance: 5000,
    status: "active"
}
```

You execute:

```javascript
db.accounts.updateOne(
    { _id: 1 },
    {
        $set: {
            balance: 4500,
            status: "active"
        }
    }
)
```

That update is atomic at the **document level**.

You don't get a weird state where MongoDB permanently commits:

```text
balance = 4500
```

but fails halfway through the same document and permanently leaves the other field half-updated.

MongoDB guarantees that writes to a single document are atomic, even when the operation modifies multiple fields. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 2. Why single-document atomicity is such a big deal

Imagine this document:

```javascript
{
    product: "Laptop",
    stock: 10,
    sold: 20
}
```

You sell one laptop.

You want:

```text
stock → 9
sold  → 21
```

Both fields are in the same document.

You can do:

```javascript
db.products.updateOne(
    { _id: productId, stock: { $gt: 0 } },
    {
        $inc: {
            stock: -1,
            sold: 1
        }
    }
)
```

One document.

One atomic update.

This is often preferable to splitting stock and sales into unrelated documents and immediately reaching for a transaction.

MongoDB specifically emphasizes that good schema design and embedding can eliminate the need for many distributed transactions. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 3. Atomicity is not the same as isolation

These are related, but different.

### Atomicity

> All changes in the unit succeed, or none do.

### Isolation

> Concurrent operations shouldn't observe an inappropriate partial/intermediate state.

### Consistency

> The database moves from one valid state to another according to its guarantees and constraints.

### Durability

> Once a transaction is durably committed according to the configured write concern, its committed data has the corresponding persistence guarantees.

These four ideas are the familiar **ACID** properties.

MongoDB supports ACID multi-document transactions, with the exact durability/visibility characteristics determined in part by transaction read/write concerns. ([MongoDB](https://www.mongodb.com/docs/v7.0/core/transactions/?utm_source=chatgpt.com "Transactions - Database Manual v7.0 - MongoDB Docs"))

---

# 4. Before transactions: understand race conditions

Consider:

```javascript
{
    _id: 1,
    stock: 1
}
```

Two customers simultaneously try to buy it.

Your naive application might do:

```text
Request A:
read stock → 1

Request B:
read stock → 1
```

Now both think:

```text
"I can buy it."
```

Then:

```text
A → stock = 0
B → stock = 0
```

Your application just processed two purchases even though only one item existed.

This is a **race condition**.

---

# 5. The classic bad pattern

This is dangerous:

```javascript
// Step 1
const product = await db.products.findOne({
    _id: productId
});

// Step 2
if (product.stock > 0) {
    // Step 3
    await db.products.updateOne(
        { _id: productId },
        {
            $set: {
                stock: product.stock - 1
            }
        }
    );
}
```

Why?

Because between:

```text
READ
```

and:

```text
WRITE
```

another request can modify the document.

This is called a **read-modify-write race**.

---

# 6. The better solution: put the condition in the update

Instead:

```javascript
const result = await db.products.updateOne(
    {
        _id: productId,
        stock: { $gt: 0 }
    },
    {
        $inc: {
            stock: -1
        }
    }
);
```

Now the database operation means:

> “Decrease stock only if this product currently has stock greater than zero.”

That condition and update happen as one atomic single-document operation.

MongoDB explicitly recommends including the expected current value in the update filter when you need to prevent conflicting concurrent updates. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 7. Why this works

Suppose:

```text
stock = 1
```

Two requests arrive.

MongoDB processes the competing atomic updates against the document.

One succeeds:

```text
stock → 0
```

The other now no longer satisfies:

```javascript
stock: { $gt: 0 }
```

so it doesn't decrement the stock.

This is an extremely useful pattern:

```text
expected state
     +
atomic update
     ↓
safe concurrency behavior
```

---

# 8. `matchedCount` becomes very useful here

Suppose:

```javascript
const result = await db.products.updateOne(
    {
        _id: productId,
        stock: { $gt: 0 }
    },
    {
        $inc: {
            stock: -1
        }
    }
);
```

Then:

```javascript
result.matchedCount
```

may be:

```text
1
```

meaning the condition matched and the update happened.

Or:

```text
0
```

meaning:

```text
no document satisfied the condition
```

The backend can then return something like:

```text
"Product is out of stock"
```

This is a simple form of **optimistic concurrency control**.

---

# 9. Optimistic concurrency control

The idea is:

```text
"I assume nobody changed this between my expected state
and my update, but I will verify the expected state during the write."
```

For example:

```javascript
db.accounts.updateOne(
    {
        _id: accountId,
        version: 7
    },
    {
        $set: {
            balance: 8000
        },

        $inc: {
            version: 1
        }
    }
)
```

If another request already changed:

```text
version = 8
```

your update no longer matches.

That lets you detect a conflict.

---

# 10. Version-based optimistic concurrency

This is a powerful pattern for documents that are edited by multiple actors.

Example:

```javascript
{
    _id: 1,
    title: "Backend Engineer",
    salary: 800000,
    version: 3
}
```

Client reads:

```text
version = 3
```

Another client updates the job:

```text
version = 4
```

First client now tries:

```javascript
db.jobs.updateOne(
    {
        _id: 1,
        version: 3
    },
    {
        $set: {
            salary: 850000
        },

        $inc: {
            version: 1
        }
    }
)
```

Result:

```text
matchedCount = 0
```

The application knows:

> “The record changed after I read it.”

That is much safer than silently overwriting someone else's change.

---

# 11. `$inc` vs `$set` under concurrency

This distinction is very important.

Suppose:

```javascript
{
    likes: 10
}
```

Two clients run:

```javascript
$set: {
    likes: 11
}
```

Both started from:

```text
10
```

You could lose one increment.

But:

```javascript
$inc: {
    likes: 1
}
```

from both clients can safely produce:

```text
12
```

because each operation expresses a relative change rather than overwriting the same absolute value.

MongoDB's atomicity documentation uses this exact distinction: concurrent `$inc` operations accumulate, whereas competing `$set` operations based on stale values can overwrite one another. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 12. Now the big question: when do we need a transaction?

Suppose you have:

```text
Account A
Account B
```

and want to transfer:

```text
₹1000
```

You must perform:

```text
A balance -1000
B balance +1000
```

If only the first update succeeds:

```text
A = 4000
B = 5000
```

you've destroyed the intended invariant:

```text
money disappeared
```

Those two documents need to change as one logical operation.

This is an excellent use case for a transaction.

---

# 13. Without a transaction

Your backend might do:

```text
Update A
   ↓
success

Update B
   ↓
failure
```

Now:

```text
A changed
B didn't
```

That's the exact class of problem multi-document transactions solve.

MongoDB supports transactions across multiple documents, collections, databases, and, in supported deployments, shards. ([MongoDB](https://www.mongodb.com/docs/v7.0/core/transactions/?utm_source=chatgpt.com "Transactions - Database Manual v7.0 - MongoDB Docs"))

---

# 14. Transaction mental model

Think:

```text
START
  ↓
Operation 1
  ↓
Operation 2
  ↓
Operation 3
  ↓
COMMIT
```

If everything succeeds:

```text
COMMIT
 ↓
all changes become part of the committed transaction
```

If something fails:

```text
ABORT
 ↓
all transaction changes are discarded
```

MongoDB states that committed transaction changes become visible as a committed unit, while aborted transaction changes are discarded. ([MongoDB](https://www.mongodb.com/docs/v7.0/core/transactions/?utm_source=chatgpt.com "Transactions - Database Manual v7.0 - MongoDB Docs"))

---

# 15. A transaction does NOT mean “every query is locked”

This is a common misconception.

A transaction isn't:

```text
"I'm locking the entire database."
```

MongoDB's concurrency behavior is much more sophisticated.

Transactions provide transactional atomicity/isolation semantics while MongoDB still handles concurrent operations through its normal concurrency-control machinery.

For certain lock acquisition situations, transactions can wait only within configured limits before aborting; MongoDB documents a default transaction lock-request waiting behavior and associated configuration. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-production-consideration//?utm_source=chatgpt.com "Production Considerations - Database Manual - MongoDB Docs"))

The key lesson:

> **Transaction ≠ giant database lock.**

---

# 16. Sessions

Transactions are associated with a **session**.

Think:

```text
Client
  ↓
Session
  ↓
Transaction
  ↓
multiple operations
```

In the Node.js driver, you create a `ClientSession` from a `MongoClient` and associate transaction operations with that session. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 17. Basic transaction flow in `mongosh`

A simplified structure:

```javascript
const session = db.getMongo().startSession();

session.startTransaction();

try {
    // Operations using the session

    session.commitTransaction();
} catch (error) {
    session.abortTransaction();
} finally {
    session.endSession();
}
```

Conceptually:

```text
startSession
     ↓
startTransaction
     ↓
operations
     ↓
commit / abort
     ↓
endSession
```

MongoDB's session API exposes these transaction lifecycle methods. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/session.withtransaction/?utm_source=chatgpt.com "Session - Database Manual - MongoDB Docs"))

---

# 18. In a transaction, the operations must belong to the session

This is critical.

Conceptually:

```javascript
await users.updateOne(
    { _id: userId },
    { $set: { ... } },
    { session }
);
```

The `{ session }` tells MongoDB:

> This operation belongs to this transaction/session.

If one operation accidentally executes without the session, it isn't part of that transaction.

The Node.js driver documentation explicitly says you must pass the session to each operation that you want included in the transaction. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 19. Node.js transaction — Core API

Since your ChatApp uses Node.js, you should understand this.

```javascript
const session = client.startSession();

try {
    session.startTransaction();

    await db.collection("accounts").updateOne(
        { _id: fromId },
        {
            $inc: {
                balance: -1000
            }
        },
        { session }
    );

    await db.collection("accounts").updateOne(
        { _id: toId },
        {
            $inc: {
                balance: 1000
            }
        },
        { session }
    );

    await session.commitTransaction();

} catch (error) {

    await session.abortTransaction();
    throw error;

} finally {

    await session.endSession();
}
```

This is the basic structure to understand.

The Node.js driver docs call this the **Core API**, where you explicitly manage the transaction lifecycle. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 20. Convenient Transaction API

The Node.js driver also provides:

```javascript
withSession()
```

and:

```javascript
withTransaction()
```

The convenient API handles transaction lifecycle and includes retry behavior for certain transaction errors. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

Conceptually:

```javascript
await client.withSession(async (session) => {

    return session.withTransaction(async () => {

        await operation1({ session });

        await operation2({ session });

        await operation3({ session });
    });
});
```

The exact callback signature depends on the driver version/API being used; the important thing is understanding that the convenience API wraps the transaction lifecycle.

---

# 21. Why `withTransaction()` is attractive

The driver can handle important transaction error cases, including:

```text
TransientTransactionError
UnknownTransactionCommitResult
```

The official MongoDB application transaction documentation says the callback API automatically retries the transaction as a whole for eligible transient errors and retries commit when the commit result is unknown. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

That's one reason it is preferable to manually reinvent transaction retry behavior without understanding MongoDB's error labels.

---

# 22. `TransientTransactionError`

Imagine your transaction is running and:

```text
primary steps down
network issue
transient server condition
```

Your transaction may fail with an error carrying:

```text
TransientTransactionError
```

This does **not** necessarily mean:

> “Your business operation is permanently impossible.”

It means:

> “This transaction attempt encountered a transient error; the transaction may need to be retried.”

The official MongoDB transaction documentation describes this retry semantics explicitly. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

---

# 23. `UnknownTransactionCommitResult`

Another interesting scenario:

```text
You send COMMIT
       ↓
network problem
       ↓
client doesn't know whether server committed
```

You now have an awkward state:

```text
Did it commit?
```

The driver can receive an error labeled:

```text
UnknownTransactionCommitResult
```

The correct approach can be to retry the commit rather than blindly execute the entire business operation again.

MongoDB's driver/application documentation specifically distinguishes retrying the commit from retrying the entire transaction. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

---

# 24. Don't blindly retry the business operation

This is an important engineering point.

Suppose your transaction means:

```text
charge customer
create order
reduce inventory
```

If you don't know whether commit succeeded, blindly running the entire business workflow from the beginning could create duplicate side effects outside the transaction.

MongoDB's transaction APIs are designed around retrying the **transaction or commit at the appropriate layer**, and driver APIs provide logic for these error categories. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

The broader lesson:

> **Database retry semantics are part of transaction design.**

---

# 25. Snapshot and isolation

Now let's discuss what your transaction reads.

Suppose a transaction performs:

```text
Read A
Read B
```

while other clients are modifying the database.

A transaction's read concern affects what data it can observe and what consistency guarantees it gets.

MongoDB supports transaction read concerns including:

```text
local
majority
snapshot
```

with transaction-specific behavior documented by MongoDB. ([MongoDB](https://www.mongodb.com/docs/manual/reference/read-concern/?utm_source=chatgpt.com "Read Concern - Database Manual - MongoDB Docs"))

---

# 26. `local` read concern

Conceptually:

```text
"Give me data from this node without requiring
that it be majority committed."
```

This can offer lower consistency guarantees.

MongoDB documents `local` as returning data from the queried instance without guaranteeing that the data has been written to a majority; such data can potentially be rolled back. ([MongoDB](https://www.mongodb.com/docs/manual/reference/read-concern/?utm_source=chatgpt.com "Read Concern - Database Manual - MongoDB Docs"))

You don't need to obsess over this for your first MongoDB application.

Just understand:

```text
read concern
    ↓
how much consistency/visibility assurance do I require?
```

---

# 27. `majority` read concern

This asks for data that has been committed to a majority of the replica set members.

Conceptually:

```text
local
    ↓
"I saw it on this node."

majority
    ↓
"I saw data that's majority committed."
```

MongoDB documents `"majority"` read concern as returning data committed to a majority of the replica set members. ([MongoDB](https://www.mongodb.com/docs/manual/reference/read-concern/?utm_source=chatgpt.com "Read Concern - Database Manual - MongoDB Docs"))

---

# 28. `snapshot`

This is particularly interesting for transactions.

The idea is:

> Read from a consistent snapshot of data.

MongoDB documents `"snapshot"` as providing snapshot semantics in supported transaction contexts, with its full guarantees depending on the transaction's write concern and deployment. ([MongoDB](https://www.mongodb.com/docs/manual/reference/read-concern/?utm_source=chatgpt.com "Read Concern - Database Manual - MongoDB Docs"))

For multi-document transactional workflows where you need a consistent view while performing related reads, snapshot semantics are conceptually very important.

---

# 29. Write concern

Read concern answers:

```text
"How should reads be guaranteed?"
```

Write concern asks:

```text
"How much acknowledgment/durability do I require from writes?"
```

For example:

```javascript
{ w: "majority" }
```

means the write should be acknowledged after the required majority condition.

MongoDB's write concern documentation defines `w` as the requested number of acknowledgments, with `"majority"` being a special majority-based setting; `j: true` can additionally request journal acknowledgment. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/mongo.getwriteconcern/?utm_source=chatgpt.com "Mongo() (mongosh method) - Database Manual - MongoDB Docs"))

---

# 30. `w: 1` vs `w: "majority"`

Conceptually:

```text
w: 1
    ↓
primary acknowledges

w: "majority"
    ↓
majority-based acknowledgment
```

The exact deployment and journaling details matter, but the broad trade-off is:

```text
stronger durability/replication assurance
        ↕
latency / availability considerations
```

MongoDB's default global write-concern behavior is deployment/configuration dependent; current MongoDB documentation notes that the default global write concern for replica sets and sharded clusters is `majority`. ([MongoDB](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com "Write Concern - Database Manual v8.0 - MongoDB Docs"))

---

# 31. `j: true`

You may see:

```javascript
{
    w: "majority",
    j: true
}
```

`j: true` asks MongoDB to acknowledge after the write has been written to the on-disk journal.

MongoDB documents `j` as requesting acknowledgment that the write has been written to the on-disk journal. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/mongo.getwriteconcern/?utm_source=chatgpt.com "Mongo() (mongosh method) - Database Manual - MongoDB Docs"))

Again, don't memorize combinations yet.

Understand the concept:

```text
write concern
    ↓
how much acknowledgment/durability do I require?
```

---

# 32. Read concern + write concern

These work together.

For a distributed application, you may care about:

```text
What data am I allowed to see?
+
When is a write considered sufficiently acknowledged?
```

MongoDB's causal-consistency documentation shows how different read/write concern combinations provide different guarantees, including monotonic reads/writes and read-your-own-writes. ([MongoDB](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/?utm_source=chatgpt.com "Causal Consistency and Read and Write Concerns - Database Manual - MongoDB Docs"))

This is getting into distributed-systems territory.

That's intentional.

---

# 33. Sessions aren't only for transactions

This is another subtle point.

A `ClientSession` is a broader concept than:

```text
"transaction object"
```

Sessions can also provide causal consistency and related operation context.

MongoDB documents **causally consistent sessions** as a mechanism for obtaining ordering guarantees across operations. ([MongoDB](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/?utm_source=chatgpt.com "Causal Consistency and Read and Write Concerns - Database Manual - MongoDB Docs"))

For this lesson, though, associate:

```text
session
    ↓
context that can carry a transaction
```

and later we'll revisit sessions in the distributed-systems portion.

---

# 34. A real JobForge transaction

Suppose a candidate applies to a job.

You want to:

```text
1. Create Application
2. Increment job applicationCount
```

Could this need a transaction?

Potentially yes **if your business invariant requires both changes to commit together**.

Without a transaction:

```text
create application → success

increment count → failure
```

Now:

```text
Application exists
count incorrect
```

But before reaching for a transaction, ask:

> Do I actually need this count to be perfectly synchronized at write time?

You could instead calculate application counts using aggregation or maintain a derived counter asynchronously.

This is exactly why good data modeling can eliminate unnecessary transactions.

MongoDB explicitly recommends schema design that minimizes the need for distributed transactions. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 35. Transaction vs derived data

Suppose:

```text
jobs.applicationCount
```

is merely a convenience for fast display.

You could decide:

```text
applications = source of truth
applicationCount = derived data
```

Then if the count temporarily lags:

```text
applications → correct
count        → eventually corrected
```

may be acceptable.

In that architecture, you might not need a transaction.

But if:

```text
"job must not accept more than N applications"
```

depends on the counter being exact at the moment of acceptance, the consistency requirements become much stronger.

This is a **business decision**, not merely a MongoDB syntax decision.

---

# 36. Real ChatApp example

Suppose a message is sent.

You want:

```text
1. Insert message
2. Update conversation metadata
3. Update unread count
```

Should those always be one transaction?

Not necessarily.

Ask:

```text
Does unreadCount have to be perfectly synchronized?
Can it be derived?
Can it be eventually consistent?
Can the UI calculate it another way?
```

For example, you might use:

```text
messages
    ↓
source of truth

unreadCount
    ↓
derived optimization
```

Then you may deliberately avoid a multi-document transaction.

But if your business rules require all three changes to be atomic, a transaction becomes much more justified.

---

# 37. This leads to an extremely important rule

> **Do not use transactions merely because multiple database operations exist.**

Use them when:

> **those operations must succeed or fail as one logical unit.**

That's the key.

Three unrelated writes:

```text
write A
write B
write C
```

do not automatically require a transaction.

But:

```text
debit A
credit B
```

clearly does if you need atomic money movement.

---

# 38. One-document transaction vs multi-document transaction

Suppose you have:

```javascript
{
    customer: "Aarav",

    balance: 1000,

    transactions: [...]
}
```

If your schema keeps the relevant state within a single document, you may be able to make the operation atomic without a multi-document transaction.

Compare:

```text
embedded model
   ↓
one atomic document write
```

versus:

```text
normalized model
   ↓
multiple documents
   ↓
possibly transaction
```

This is precisely why schema design affects transaction requirements. MongoDB's documentation explicitly highlights this relationship. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 39. Transaction performance

Transactions aren't free.

MongoDB explicitly notes that multi-document/distributed transactions generally have greater performance cost than single-document writes. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

Why?

Conceptually, the database has more work to do:

```text
track transaction state
manage multiple operations
maintain isolation
prepare commit
coordinate visibility
possibly coordinate across shards
handle retries/failures
```

So:

```text
single atomic update
```

is often simpler and cheaper than:

```text
multi-document transaction
```

when both can correctly implement your business requirement.

---

# 40. Transactions have operational limits

MongoDB documents several production considerations, including transaction lifetime limits and lock acquisition behavior. The default transaction lifetime is under one minute unless configured differently, and transactions can be aborted due to resource/operational constraints. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-production-consideration//?utm_source=chatgpt.com "Production Considerations - Database Manual - MongoDB Docs"))

The practical lesson:

> **Keep transactions short.**

Don't do this:

```text
START TRANSACTION
   ↓
call external API
   ↓
wait 15 seconds
   ↓
run computation
   ↓
wait for another service
   ↓
commit
```

A transaction should contain the necessary database operations, not an entire business workflow.

---

# 41. Never call external services inside a long-running transaction

Imagine:

```text
START TRANSACTION
    ↓
MongoDB update
    ↓
HTTP call to payment service
    ↓
HTTP call to email service
    ↓
wait
    ↓
commit
```

You've now tied your database transaction lifetime to network dependencies.

Bad design.

Better:

```text
database transaction
    ↓
commit local state
    ↓
event/message/outbox
    ↓
external processing
```

We'll eventually discuss patterns such as the **outbox pattern** when we reach more advanced backend/system-design material.

---

# 42. Transactions on sharded clusters

MongoDB supports transactions across shards.

But distributed transactions that span multiple shards incur additional cost, and MongoDB documents specific constraints and considerations for them. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-sharded-clusters/?utm_source=chatgpt.com "Production Considerations (Sharded Clusters) - Database Manual - MongoDB Docs"))

This will matter much more when we study:

```text
replication
sharding
distributed systems
```

For now:

```text
single replica-set transaction
        ↓
already enough complexity

multi-shard transaction
        ↓
even more complexity/cost
```

Don't design your beginner projects around multi-shard transactions.

---

# 43. Important Node.js transaction rule: don't use `Promise.all()`

The official Node.js driver documentation explicitly says that parallel operations within a single transaction are not supported. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

So don't do:

```javascript
await Promise.all([
    collectionA.updateOne(..., { session }),
    collectionB.updateOne(..., { session })
]);
```

inside one transaction.

Use sequential:

```javascript
await collectionA.updateOne(..., { session });

await collectionB.updateOne(..., { session });
```

This is an important practical point for your Node.js backend.

---

# 44. Another transaction rule: await everything

Inside:

```javascript
withTransaction()
```

you want:

```javascript
await operation1()
await operation2()
await operation3()
```

not:

```javascript
operation1()
operation2()
operation3()
```

because you need the transaction callback to represent the actual sequence of database operations and errors.

The Node.js driver documentation shows session operations being awaited and warns about parallelism inside transactions. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 45. What happens on abort?

Suppose:

```text
Transaction:

1. update A
2. update B
3. insert C
4. operation fails
```

Then:

```text
abort
 ↓
1 rolled back
2 rolled back
3 rolled back
```

The uncommitted transaction changes don't become visible as committed outside the transaction. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/session.committransaction/?utm_source=chatgpt.com "Session - Database Manual - MongoDB Docs"))

This is the core power of a transaction.

---

# 46. What happens on commit?

Suppose:

```text
1. update A
2. update B
3. insert C
4. commit
```

Conceptually:

```text
commit
   ↓
all transaction changes become committed
```

MongoDB guarantees transaction atomicity: it doesn't commit only some of the transaction's changes and roll back the others. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/session.committransaction/?utm_source=chatgpt.com "Session - Database Manual - MongoDB Docs"))

---

# 47. But visibility has nuance

For a normal replica-set scenario, you can mentally think:

```text
before commit
    → outside readers don't see transaction's uncommitted changes

after commit
    → committed changes become visible
```

On sharded deployments and with different read concerns, visibility can have more nuance during commit propagation; MongoDB documents those details explicitly. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/session.committransaction/?utm_source=chatgpt.com "Session - Database Manual - MongoDB Docs"))

That's one reason read concern exists.

---

# 48. Transactions and unique indexes

Remember Lesson 7?

Suppose:

```text
email unique
```

The unique index still enforces the uniqueness constraint.

Transactions don't replace indexes.

A transaction could contain:

```text
create user
create profile
```

while:

```text
email unique
```

is still enforced by the unique index.

Again:

```text
transaction
    → atomicity

unique index
    → uniqueness
```

Different responsibilities.

---

# 49. A better ChatApp architecture

Suppose you want:

```text
send message
```

A possible architecture is:

```text
POST /messages
       ↓
validate request
       ↓
authorize sender
       ↓
insert message
       ↓
emit Socket.IO event
```

Do you automatically need a transaction?

No.

If the message itself is the primary source of truth and other state is derived, a single atomic message insert may be perfectly sufficient.

You shouldn't turn every Socket.IO message send into a transaction just because there are multiple application steps.

---

# 50. A case where ChatApp might need one

Suppose your requirements say:

```text
Every sent message must:
    1. create message
    2. increment billing usage
    3. decrement monthly quota
```

and the billing/quota state is stored in separate documents.

Now you may need:

```text
message exists
+
usage incremented
+
quota decremented
```

as one atomic unit.

That's a legitimate transaction candidate.

---

# 51. Transaction vs event-driven architecture

There's another important architectural idea.

You might instead do:

```text
create message
      ↓
commit
      ↓
publish event
      ↓
billing service updates usage
```

Now the system becomes:

```text
strongly consistent message
+
eventually consistent billing
```

Whether that's acceptable depends on the requirements.

This is why database transactions are ultimately about **business consistency requirements**, not just database syntax.

---

# 52. A practical decision tree

When you are about to write multiple MongoDB operations, think:

```text
Multiple operations?
        │
        ↓
Do they need to succeed/fail together?
        │
       NO ─────────────→ Separate operations
        │
       YES
        ↓
Can I model them in one document?
        │
       YES ────────────→ Prefer single-document atomicity
        │
       NO
        ↓
Use multi-document transaction
        ↓
Keep it short
        ↓
Handle transaction errors/retries
```

This is an excellent mental model.

---

# 53. Exercises

## Exercise 1 — Race condition

You have:

```javascript
{
    _id: 1,
    stock: 1
}
```

Two users execute:

```javascript
find({ _id: 1 })
```

and both see:

```text
stock = 1
```

Then both execute:

```javascript
$set: {
    stock: 0
}
```

What is the problem?

### Answer

Both requests made a decision from stale state.

Both observed:

```text
stock = 1
```

and both believed they could complete the sale.

The writes don't express:

> “Only do this if stock is still positive.”

A better approach is to put the expected condition directly in the atomic update:

```javascript
db.products.updateOne(
    {
        _id: 1,
        stock: { $gt: 0 }
    },
    {
        $inc: {
            stock: -1
        }
    }
)
```

MongoDB explicitly recommends using the update filter to encode the expected current value when protecting against concurrent modifications. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 54. Exercise 2 — Does this need a transaction?

You have:

```text
users
```

and:

```text
profiles
```

When a user registers, you:

```text
1. create user
2. create profile
```

If the profile creation fails, you don't want a user without a profile.

Transaction or no transaction?

### Answer

A **multi-document transaction is a reasonable solution** if both records must be created atomically.

But there's another design question:

> Could profile information simply be embedded inside the user?

If yes, you may eliminate the transaction entirely through schema design.

That's exactly the connection between Lessons 3 and 9.

MongoDB specifically recommends schema design that minimizes the need for distributed transactions. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 55. Exercise 3 — Transaction or atomic update?

You need:

```text
likes = likes + 1
```

Do you need a transaction?

### Answer

No.

A single-document atomic update:

```javascript
$inc: {
    likes: 1
}
```

is enough.

Don't use a transaction when the single-document operation already gives you the required atomicity.

---

# 56. Exercise 4 — Real-world

A money transfer does:

```text
Account A → -₹1000
Account B → +₹1000
```

Could this be safely implemented as two independent updates?

### Answer

Not if you require the transfer to behave as one all-or-nothing operation.

You need a mechanism that guarantees:

```text
both happen
OR
neither happens
```

A multi-document transaction is the natural MongoDB mechanism when the two account documents must change atomically. ([MongoDB](https://www.mongodb.com/docs/v7.0/core/transactions/?utm_source=chatgpt.com "Transactions - Database Manual v7.0 - MongoDB Docs"))

---

# 57. Difficult conceptual question

Which is the best statement?

**A.** MongoDB transactions should be used whenever a request contains more than one database operation.

**B.** Single-document writes are not atomic, so transactions are required for every update.

**C.** Good schema design can often eliminate the need for multi-document transactions; transactions are appropriate when multiple documents/collections genuinely need all-or-nothing semantics.

**D.** Transactions make every operation faster.

### Answer

**C**

That's one of the most important MongoDB architecture principles. MongoDB explicitly warns that transactions generally cost more than single-document writes and should not substitute for effective schema design. ([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

---

# 58. Difficult Node.js question

Inside a Node.js transaction, which is preferable?

### A

```javascript
await Promise.all([
    collectionA.updateOne(..., { session }),
    collectionB.updateOne(..., { session })
]);
```

### B

```javascript
await collectionA.updateOne(..., { session });
await collectionB.updateOne(..., { session });
```

### Answer

**B**

The official Node.js driver documentation says parallel operations within a single transaction aren't supported. Use sequential awaited operations. ([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

---

# 59. The most important distinction in this lesson

There are three different techniques that people often confuse.

### Technique 1 — Atomic single-document update

```javascript
$inc
$set
$push
```

Use when:

```text
all required state fits into one document
```

---

### Technique 2 — Optimistic concurrency

```javascript
filter:
{
    _id: ...,
    version: expectedVersion
}
```

Use when:

```text
you need to detect that someone changed the document
```

---

### Technique 3 — Multi-document transaction

```text
start
 ↓
multiple operations
 ↓
commit / abort
```

Use when:

```text
multiple documents must change as one logical unit
```

Those are three different tools.

---

# 60. The complete mental model

```text
                    CONCURRENCY
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
      One document    Detect stale    Multiple docs
          │             changes           │
          ↓                ↓              ↓
     Atomic update    Version/filter   Transaction
          │                │              │
          ↓                ↓              ↓
       $inc/$set       optimistic       ACID
                         control
```

And around transactions:

```text
Transaction
    │
    ├── Session
    │
    ├── Read concern
    │
    ├── Write concern
    │
    ├── Operations
    │
    ├── Commit
    │
    └── Abort
```

---

# 61. What I want you to remember permanently

### 1.

> **A single MongoDB document is an atomic unit.**

([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

### 2.

> **Don't solve concurrency problems with “read first, then write” when a single atomic conditional update can express the operation.**

([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

### 3.

> **Use transactions when multiple documents/collections genuinely need all-or-nothing behavior.**

([MongoDB](https://www.mongodb.com/docs/v7.0/core/transactions/?utm_source=chatgpt.com "Transactions - Database Manual v7.0 - MongoDB Docs"))

### 4.

> **Good schema design can reduce the need for transactions.**

([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

### 5.

> **Transactions cost more than simple single-document writes, so don't use them casually.**

([MongoDB](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/?utm_source=chatgpt.com "Atomicity and Transactions - Database Manual - MongoDB Docs"))

### 6.

> **In Node.js transactions, operations need the session and should be awaited sequentially; don't use parallel operations such as `Promise.all()` within one transaction.**

([MongoDB](https://www.mongodb.com/docs/drivers/node/v6.x/crud/transactions/?utm_source=chatgpt.com "Transactions - Node.js Driver v6.x - MongoDB Docs"))

### 7.

> **Transaction retries are part of real-world transaction handling.**

The Node.js callback API provides handling for transient transaction errors and unknown commit results. ([MongoDB](https://www.mongodb.com/docs/manual/core/transactions-in-applications/?utm_source=chatgpt.com "Handle Transactions in Applications with the Drivers API - Database Manual - MongoDB Docs"))

---

# 62. Where this puts you in the MongoDB course

You've now covered:

```text
1. Fundamentals & CRUD
2. Querying
3. Data Modeling
4. Updates & Arrays
5. Aggregation Fundamentals
6. Advanced Aggregation
7. Indexes & Query Performance
8. Schema Validation
9. Transactions & Concurrency ✅
```

At this point, you know most of the **application-facing MongoDB fundamentals**.

The next phase becomes much more **database-engineering oriented**:

```text
10. MongoDB Architecture & Storage Internals
        ↓
11. Replication & High Availability
        ↓
12. Sharding & Horizontal Scaling
        ↓
13. Change Streams & Advanced Features
        ↓
14. Node.js + Mongoose Deep Dive
        ↓
15. Production Security / Monitoring / Backup / Performance
        ↓
16. Real Project Database Design
```

And Lesson 10 is going to be different from the previous lessons: we'll open the hood and study **what MongoDB actually looks like internally**—`mongod`, storage engine, WiredTiger, memory/cache, BSON on disk, data files, journal, oplog, indexes, checkpoints, and what happens from the moment your Node.js application sends a query until MongoDB returns the result.