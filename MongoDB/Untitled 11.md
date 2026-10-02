
# MongoDB — Lesson 12: Sharding & Horizontal Scaling

This is the first MongoDB lesson where we're dealing with a problem bigger than a single server:

> **What do you do when one MongoDB server—or even one replica set—isn't enough?**

You've already learned replication:

```text
                 Replica Set
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Primary    Secondary   Secondary
```

Replication gives you:

```text
high availability
redundancy
failover
```

But there's a fundamental limitation:

> **Every member of that replica set still maintains the same dataset.**

Suppose your database grows to:

```text
2 TB
10 TB
50 TB
100 TB
```

and your workload becomes enormous.

Adding more secondaries doesn't partition the data. They still maintain copies of the same data.

That's where **sharding** comes in.

MongoDB describes a sharded cluster as a collection of **shards**, each containing a subset of the sharded data, with `mongos` routing requests and config servers maintaining cluster metadata. Each shard itself must be a replica set. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 1. Replication vs sharding

This distinction needs to become automatic.

## Replication

```text
                 SAME DATA
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        Node A     Node B     Node C
```

Purpose:

```text
availability
redundancy
failover
```

---

## Sharding

```text
                 TOTAL DATA
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
       Shard 1     Shard 2     Shard 3
       data A-C    data D-H    data I-Z
```

Purpose:

```text
horizontal scaling
data distribution
larger storage capacity
distributed read/write workload
```

MongoDB states that sharding distributes collection data across shards and allows both storage and read/write workloads to scale horizontally. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 2. Sharding is not “adding replicas”

Suppose you have:

```text
10 TB database
```

with a replica set:

```text
Primary   → 10 TB
Secondary → 10 TB
Secondary → 10 TB
```

You have roughly:

```text
30 TB of physical copies
```

but your logical dataset is still:

```text
10 TB
```

Now suppose instead you shard across three shards:

```text
Shard 1 → ~3.3 TB
Shard 2 → ~3.3 TB
Shard 3 → ~3.3 TB
```

and each shard is itself a replica set:

```text
Shard 1
 ├── Primary
 ├── Secondary
 └── Secondary

Shard 2
 ├── Primary
 ├── Secondary
 └── Secondary

Shard 3
 ├── Primary
 ├── Secondary
 └── Secondary
```

Now you're getting both:

```text
horizontal distribution
+
replication
```

This is the architecture MongoDB uses for large sharded deployments. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 3. The complete sharded-cluster architecture

The architecture you should memorize is:

```text
                         APPLICATION
                              │
                              ▼
                           mongos
                              │
                    ┌─────────┼─────────┐
                    │         │         │
                    ▼         ▼         ▼
                 Shard 1   Shard 2   Shard 3
                    │         │         │
                 Replica    Replica    Replica
                   Set        Set        Set
```

And separately:

```text
                     Config Servers
                           │
                           │ metadata
                           ▼
                         mongos
```

MongoDB's current architecture requires:

```text
shards
mongos
config servers
```

with each shard deployed as a replica set and config servers deployed as a replica set (**CSRS**) in the standard architecture; MongoDB 8.0 also supports a **config shard**, which lets a shard serve the config-server role. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 4. The three components

## Shard

Stores:

> **a subset of the application's data.**

Each shard is a replica set. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

## `mongos`

Acts as:

> **query router.**

Applications communicate with `mongos`, not directly with shard members in a normal sharded deployment. `mongos` uses cluster metadata to determine where operations should go. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

## Config servers

Store:

> **cluster metadata.**

This includes information such as:

```text
which ranges/chunks exist
which shard owns them
sharding configuration
cluster metadata
```

`mongos` caches this metadata and uses it to route operations. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-config-servers/?utm_source=chatgpt.com "Config Servers - Database Manual v8.0 - MongoDB Docs"))

---

# 5. Why do config servers exist?

Imagine:

```text
users collection
```

is distributed as:

```text
Shard 1 → users with key range A-B
Shard 2 → users with key range C-H
Shard 3 → users with key range I-Z
```

When a request arrives:

```javascript
db.users.find({
    username: "riya24"
})
```

`mongos` needs to know:

> Which shard contains documents relevant to this shard-key range?

The config servers maintain that metadata.

So:

```text
Application
    ↓
mongos
    ↓
"What shard owns this range?"
    ↓
cluster metadata
    ↓
route request
```

MongoDB explicitly describes config servers as storing the cluster metadata that maps chunks/ranges to shards, while `mongos` caches and uses that metadata for routing. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-config-servers/?utm_source=chatgpt.com "Config Servers - Database Manual v8.0 - MongoDB Docs"))

---

# 6. `mongos` has no persistent state

This is an important detail.

A `mongos` router maintains cached metadata but does not itself hold the database's persistent application data.

MongoDB describes `mongos` as having no persistent state and requiring relatively few resources. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

That means you can have:

```text
Application
   │
┌──┴────┐
▼       ▼
mongos A   mongos B
```

for routing redundancy.

If one `mongos` disappears, another can serve the application.

---

# 7. Why applications shouldn't connect directly to shards

In a sharded deployment, the normal architecture is:

```text
Application
    ↓
mongos
    ↓
shards
```

not:

```text
Application
    ↓
Shard 1 directly
```

Why?

Because the application shouldn't have to understand:

```text
which chunk lives where
when chunks move
which shard owns which range
how metadata changes
```

`mongos` abstracts that away.

MongoDB explicitly states that applications communicate with `mongos`, which routes operations to the correct shard(s). ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 8. Now the most important term: shard key

Everything in MongoDB sharding revolves around:

> **the shard key.**

A shard key is one field or a compound set of fields used to determine how documents are distributed across shards. MongoDB requires an appropriate index beginning with the shard key and uses the key's value/range to route data. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-shard-key/?utm_source=chatgpt.com "Shard Keys - Database Manual - MongoDB Docs"))

For example:

```javascript
{
    userId: 12345
}
```

you could potentially shard on:

```javascript
{ userId: 1 }
```

Or compound:

```javascript
{ tenantId: 1, userId: 1 }
```

The shard key is one of the most important decisions in a sharded database.

---

# 9. Why shard-key selection is so important

A bad shard key can cause:

```text
uneven storage
uneven workload
hot shards
scatter/gather queries
poor scaling
large/jumbo chunks
```

MongoDB's documentation explicitly identifies cardinality, frequency, monotonicity, and query patterns as major shard-key considerations. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

That's why you'll hear:

> **The hardest part of MongoDB sharding is often choosing the shard key.**

---

# 10. Shard key is about two separate problems

A good shard key needs to consider:

```text
1. DATA DISTRIBUTION
2. QUERY ROUTING
```

You need data spread reasonably across the cluster.

And you want important queries to be able to find the right shard without broadcasting everywhere.

MongoDB explicitly describes the ideal shard key as one that distributes documents evenly while supporting common query patterns. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 11. First criterion: cardinality

**Cardinality** roughly means:

> How many distinct values can this field have?

Example:

```text
country
```

might have:

```text
~200
```

distinct values.

That's relatively low cardinality.

Whereas:

```text
userId
```

could have:

```text
millions of distinct values.
```

That's high cardinality.

MongoDB recommends high cardinality for ranged sharding because low-cardinality shard keys make it harder to create sufficiently fine-grained distribution. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/ranged-sharding/?utm_source=chatgpt.com "Ranged Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 12. Why low cardinality can be bad

Suppose you shard:

```text
users
```

by:

```javascript
{ country: 1 }
```

Possible values:

```text
India
USA
UK
Canada
...
```

Imagine:

```text
India → 600 million users
USA   → 300 million
others → smaller
```

Now chunks corresponding to common values can become huge or heavily loaded.

MongoDB explicitly notes that high-frequency shard-key values can become a bottleneck and that low cardinality can prevent effective distribution. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 13. Cardinality is NOT enough

This is a very important subtlety.

Suppose:

```text
userId
```

has:

```text
100 million distinct values
```

Excellent cardinality.

But suppose your writes arrive as:

```text
100000001
100000002
100000003
100000004
...
```

That's monotonically increasing.

You can still have a problem under range-based sharding.

MongoDB explicitly says that high cardinality by itself does not guarantee even distribution; frequency and monotonicity also matter. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 14. Frequency

Shard-key **frequency** asks:

> How often does each value occur?

Imagine:

```text
status
```

with:

```text
open
closed
```

Only two values.

Very high frequency.

If you shard on:

```javascript
{ status: 1 }
```

you have very little ability to distribute the data finely.

MongoDB explicitly warns that high-frequency values can create hot chunks and bottlenecks. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 15. Monotonicity

Suppose your shard key is:

```javascript
{
    createdAt: 1
}
```

and new records always arrive:

```text
10:00
10:01
10:02
10:03
10:04
...
```

That is monotonically increasing.

In range sharding, new values tend to fall toward the current upper range.

MongoDB explains that a monotonically changing shard key can concentrate inserts into a single chunk and lead to a hot shard. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

This is one of the most famous sharding pitfalls.

---

# 16. The “hot shard” problem

Imagine:

```text
Shard 1
Shard 2
Shard 3
```

and all new writes land here:

```text
Shard 3
████████████████████
```

while:

```text
Shard 1
██

Shard 2
██
```

You technically have three shards.

But your workload behaves like:

```text
one overloaded shard
+
two mostly idle shards
```

You have not achieved useful horizontal scaling.

This is a **hot shard**.

A poor shard key can create exactly this situation. MongoDB's shard-key documentation explicitly warns about uneven load distribution and monotonically changing keys. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 17. Ranged sharding

MongoDB supports two major shard-key distribution strategies:

```text
Ranged sharding
Hashed sharding
```

Ranged sharding divides shard-key space into contiguous ranges. Documents whose keys are close together are likely to be stored together. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/ranged-sharding/?utm_source=chatgpt.com "Ranged Sharding - Database Manual v8.0 - MongoDB Docs"))

For example:

```text
Shard key = age

Range A:
0 ─────── 20

Range B:
20 ────── 40

Range C:
40 ────── 60
```

Those ranges can be distributed across shards.

---

# 18. Why ranged sharding is powerful

Suppose you query:

```javascript
{
    age: {
        $gte: 20,
        $lt: 30
    }
}
```

MongoDB knows that the relevant data falls within a particular key range.

`mongos` may therefore target only the shard(s) owning that range.

That is a **targeted query**.

MongoDB explicitly says range sharding allows efficient range queries when the shard key is appropriately chosen. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/ranged-sharding/?utm_source=chatgpt.com "Ranged Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 19. Hashed sharding

Hashed sharding computes a hash of the shard-key value and distributes based on the hashed values.

Conceptually:

```text
userId
  ↓
hash(userId)
  ↓
hashed key space
  ↓
chunks
  ↓
shards
```

MongoDB automatically computes hashes when resolving queries; applications do not have to hash the shard key themselves. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 20. Why hashed sharding exists

Take:

```text
createdAt
```

Suppose values are:

```text
10:00
10:01
10:02
10:03
...
```

Range-based sharding may concentrate new writes at one end.

Hashed distribution scrambles the values:

```text
10:00 → hash A
10:01 → hash F
10:02 → hash B
10:03 → hash Z
...
```

This can distribute writes more evenly.

MongoDB explicitly notes that hashed sharding helps distribute monotonically changing values more evenly. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 21. But hashed sharding has a major trade-off

Suppose you query:

```javascript
{
    createdAt: {
        $gte: start,
        $lt: end
    }
}
```

With range-based sharding, those nearby dates may be physically nearby.

With hashed sharding:

```text
date range
  ↓
hashes all over keyspace
  ↓
potentially many shards
```

So range queries can become inefficient.

MongoDB explicitly states that range-based queries on a hashed shard key are less likely to target a single shard and may require cluster-wide operations. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

This is the central trade-off:

```text
Ranged
    ↓
better range locality
potentially weaker write distribution for monotonic keys

Hashed
    ↓
better distribution
weaker range locality
```

---

# 22. Ranged vs hashed

|Property|Ranged|Hashed|
|---|---|---|
|Nearby key values|Stay relatively close|Spread out|
|Range queries|Often excellent|Often less targeted|
|Monotonic inserts|Can create hot ranges|Better distribution|
|Locality|Stronger|Weaker|
|Write distribution|Depends heavily on key|Usually more even|

MongoDB documents these trade-offs directly. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 23. Chunks

Now you need another term:

> **chunk**

MongoDB divides the shard-key space of a sharded collection into chunks/ranges.

A chunk corresponds to an inclusive lower bound and exclusive upper bound of shard-key values and is assigned to a shard. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

Imagine:

```text
Shard key space
──────────────────────────────────────────

Chunk 1
[MinKey, 100)

Chunk 2
[100, 200)

Chunk 3
[200, 300)

Chunk 4
[300, 400)
```

The chunks can be distributed:

```text
Shard 1 → Chunk 1, Chunk 4
Shard 2 → Chunk 2
Shard 3 → Chunk 3
```

The collection is therefore distributed without every shard storing every document.

---

# 24. Why chunks exist

You might ask:

> “Why not permanently assign a range directly to a shard?”

Because MongoDB needs to **rebalance** data.

Suppose:

```text
Shard 1
100 chunks

Shard 2
100 chunks

Shard 3
100 chunks
```

Balanced.

Then you add:

```text
Shard 4
0 chunks
```

You want data to move to Shard 4.

Moving **chunks/ranges** is the mechanism.

MongoDB's balancer migrates ranges between shards to improve distribution. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 25. The balancer

MongoDB runs a background **balancer** for sharded collections.

Its job is essentially:

```text
observe distribution
      ↓
find imbalance
      ↓
migrate ranges/chunks
      ↓
improve distribution
```

MongoDB explicitly describes the balancer as the background process responsible for migrating ranges between shards to achieve more even distribution. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 26. Adding a shard

Suppose:

```text
Shard 1 → 500 GB
Shard 2 → 500 GB
```

Add:

```text
Shard 3 → 0 GB
```

You might expect:

```text
500
500
0
```

to become instantly:

```text
333
333
334
```

It doesn't happen instantaneously.

MongoDB begins migrating ranges, and balancing takes time. The balancer operates in the background. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-balancer-administration/?utm_source=chatgpt.com "Sharded Cluster Balancer - Database Manual v8.0 - MongoDB Docs"))

---

# 27. Why balancing is not free

Moving a chunk means:

```text
read range
   ↓
copy data
   ↓
copy indexes/related state as needed
   ↓
coordinate metadata
   ↓
commit ownership change
   ↓
delete old range data
```

So balancing consumes:

```text
network
CPU
disk I/O
storage bandwidth
metadata work
```

MongoDB explicitly describes range migration as resource-intensive. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-balancer-administration/?utm_source=chatgpt.com "Sharded Cluster Balancer - Database Manual v8.0 - MongoDB Docs"))

Therefore:

> **Don't think of sharding as adding a shard and magically getting instant performance gains.**

---

# 28. Chunk splitting

Suppose a range gets large.

MongoDB can split a range into smaller chunks.

Conceptually:

```text
Before:

[0 ───────────────── 1000]

After:

[0 ─── 500]
[500 ── 1000]
```

This allows ranges to be distributed more finely.

However, modern MongoDB's balancing/defragmentation behavior has evolved, and current documentation notes that chunks may exceed configured `chunkSize`, because splitting and migration behavior is tied to the balancing process. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-balancer-administration/?utm_source=chatgpt.com "Sharded Cluster Balancer - Database Manual v8.0 - MongoDB Docs"))

So don't carry older tutorial assumptions such as:

> “Every chunk must always be exactly 128 MB.”

That's not the current model.

---

# 29. `chunkSize` is not a strict chunk-size guarantee

This is a particularly useful modern detail.

Current MongoDB documentation says `chunkSize` influences:

```text
maximum amount of data the balancer attempts to migrate in one chunk-migration operation
```

and defragmentation behavior, while actual chunk sizes can exceed the configured value. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-balancer-administration/?utm_source=chatgpt.com "Sharded Cluster Balancer - Database Manual v8.0 - MongoDB Docs"))

So:

```text
configured chunkSize
    ≠
hard physical maximum for every chunk
```

That's an important correction to older MongoDB mental models.

---

# 30. Jumbo chunks

Suppose a chunk becomes too large to split or migrate effectively because of the distribution of shard-key values.

You can get a **jumbo chunk**.

This is often associated with:

```text
low-cardinality shard key
high-frequency shard-key value
```

MongoDB's shard-key documentation lists jumbo chunks among the common symptoms of a suboptimal shard key. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-shard-key/?utm_source=chatgpt.com "Shard Keys - Database Manual - MongoDB Docs"))

This is one of the reasons:

> **Shard-key cardinality matters.**

---

# 31. How a jumbo chunk happens

Imagine:

```text
shard key = country
```

and:

```text
country = India
```

accounts for hundreds of gigabytes.

If all those documents have the exact same shard-key value:

```text
India
```

MongoDB cannot simply split the value:

```text
India
```

into:

```text
India A
India B
India C
```

because the shard-key value itself doesn't provide a finer ordering boundary.

So you can end up with a very large indivisible range.

That's the conceptual reason low-frequency/high-cardinality keys matter.

---

# 32. Compound shard keys

You don't have to shard on one field.

You can use:

```javascript
{
    tenantId: 1,
    userId: 1
}
```

This is a **compound shard key**.

This can solve some problems.

Suppose:

```text
tenantId
```

has relatively few values.

You could increase distribution granularity by adding:

```text
userId
```

MongoDB's documentation explicitly recommends considering compound keys when a single-field key has low cardinality or poor distribution. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 33. But compound keys have order

Suppose:

```javascript
{
    tenantId: 1,
    userId: 1
}
```

The order matters.

Conceptually:

```text
tenantId
    ↓
userId
```

means values are distributed/routed using that compound key ordering.

And query routing benefits strongly from the shard key or a **prefix** of a compound shard key. MongoDB's routing documentation explicitly describes this. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 34. Targeted queries

Suppose shard key is:

```javascript
{
    tenantId: 1,
    userId: 1
}
```

Query:

```javascript
{
    tenantId: "T42",
    userId: "U100"
}
```

`mongos` can determine where the relevant data is and route the query to the appropriate shard(s).

MongoDB calls this a **targeted operation**. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

Targeting is one of the biggest practical reasons to choose a good shard key.

---

# 35. Partial targeting

With the same shard key:

```javascript
{
    tenantId: 1,
    userId: 1
}
```

query:

```javascript
{
    tenantId: "T42"
}
```

uses the prefix:

```text
tenantId
```

So `mongos` may target the subset of shards whose chunks could contain that tenant's documents.

MongoDB explicitly documents targeting using the full shard key **or a prefix of a compound shard key**. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 36. Scatter-gather

Now the bad case.

Suppose your shard key is:

```javascript
{
    tenantId: 1,
    userId: 1
}
```

and you query:

```javascript
{
    city: "Pune"
}
```

`city` isn't part of the shard key.

`mongos` may have no way to know which shard contains the result.

So:

```text
mongos
 ├──→ Shard 1
 ├──→ Shard 2
 └──→ Shard 3
```

All respond.

`mongos` gathers the responses.

This is a **scatter-gather** operation. MongoDB explicitly describes queries without shard-key targeting as broadcast/scatter-gather operations and warns that they can become inefficient. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/distributed-queries/?utm_source=chatgpt.com "Distributed Queries - Database Manual v8.0 - MongoDB Docs"))

---

# 37. Why scatter-gather is expensive

Suppose:

```text
1 query
3 shards
```

You may do:

```text
3 database operations
```

Now:

```text
1 query
20 shards
```

could mean:

```text
20 shard operations
```

And:

```text
1 query
100 shards
```

could become:

```text
100 shard operations
```

The application still sees one logical query.

But internally:

```text
fan-out
+
parallel work
+
network
+
merge
```

is happening.

MongoDB warns that scatter-gather queries can be long-running and become problematic as clusters grow. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/distributed-queries/?utm_source=chatgpt.com "Distributed Queries - Database Manual v8.0 - MongoDB Docs"))

---

# 38. Scatter-gather is not always evil

You should not think:

> “Any query touching multiple shards is automatically bad.”

Sometimes it's necessary.

For example:

```text
analytics
global reporting
administrative operations
queries without shard-key predicates
```

may legitimately need multiple shards.

The point is:

> **A routine high-volume API request should ideally be targetable whenever practical.**

MongoDB explicitly recommends favoring targeted operations when possible. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 39. Sorting in a sharded cluster

Suppose:

```javascript
db.jobs.find({
    location: "Pune"
}).sort({
    salary: -1
})
```

and the query touches multiple shards.

Each shard may produce sorted results.

Then `mongos` may need to merge those results.

Conceptually:

```text
Shard 1 → sorted results
Shard 2 → sorted results
Shard 3 → sorted results
          ↓
      merge results
          ↓
       application
```

So even if every individual shard can sort efficiently, the cluster-level operation may require merging.

This is one reason targeted queries are generally easier to scale.

---

# 40. Aggregation on a sharded cluster

You learned `$group` and `$sort` in Lesson 5.

Now imagine:

```text
1 billion orders
```

sharded across:

```text
Shard 1
Shard 2
Shard 3
```

An aggregation might execute partially on each shard:

```text
Shard 1
  ↓
partial aggregation

Shard 2
  ↓
partial aggregation

Shard 3
  ↓
partial aggregation
```

then merge:

```text
     ↓
   mongos
     ↓
final result
```

MongoDB's aggregation documentation and `mongos` routing behavior describe split/merge pipelines for aggregations on sharded clusters. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

This is a big idea:

> **Distributed aggregation often means local partial work + global merge.**

---

# 41. Example: total sales

Suppose:

```text
Shard 1 → 1000 + 2000
Shard 2 → 3000 + 500
Shard 3 → 1000
```

Each shard can calculate:

```text
Shard 1 → 3000
Shard 2 → 3500
Shard 3 → 1000
```

Then merge:

```text
3000 + 3500 + 1000
= 7500
```

That's much better conceptually than sending every raw order back to the router.

---

# 42. But not every aggregation is this simple

Some stages require:

```text
global sorting
global grouping
joins
window operations
```

and can therefore require significant cross-shard coordination.

MongoDB's explain output for aggregation can show:

```text
mergeType
splitPipeline
shards
```

which lets you inspect where aggregation work occurred. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

This becomes a valuable debugging tool once you work with real sharded analytics.

---

# 43. `updateMany()` and `deleteMany()`

Here's a surprising rule.

MongoDB's current routing documentation says:

> `updateMany()` and `deleteMany()` are broadcast operations unless the query specifies the shard key in full. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

So:

```javascript
db.users.updateMany(
    {
        city: "Pune"
    },
    {
        $set: {
            ...
        }
    }
)
```

may have to go to:

```text
Shard 1
Shard 2
Shard 3
...
```

because `city` isn't sufficient for routing.

This is an important sharded-cluster operational consideration.

---

# 44. `updateOne()` and `deleteOne()`

MongoDB's current routing behavior is stricter here.

`updateOne()`, `replaceOne()`, and `deleteOne()` must include the shard key or `_id` in the query document in a sharded collection; MongoDB returns an error if these are issued without sufficient targeting information under the documented rules. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

That's a major difference from thinking about a standalone collection.

---

# 45. Insert routing

With:

```javascript
insertOne()
```

MongoDB knows the new document's shard-key value and can route it to the appropriate shard.

MongoDB documents `insertOne()` as targeting one shard per document, although `insertMany()` may distribute different documents to different shards. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

So:

```text
insertOne
   ↓
one document
   ↓
one target shard
```

while:

```text
insertMany([
    doc1,
    doc2,
    doc3
])
```

can involve:

```text
doc1 → shard 1
doc2 → shard 3
doc3 → shard 2
```

---

# 46. Changing a shard-key value

This gets more interesting.

Suppose a document has:

```javascript
{
    tenantId: "A",
    userId: "U1"
}
```

and your shard key places it on:

```text
Shard 1
```

Now you change the shard-key value:

```text
tenantId = "B"
```

and `"B"` maps to:

```text
Shard 3
```

The document may need to move between shards.

MongoDB allows updating shard-key values in supported configurations, except when the shard-key field is immutable `_id`; MongoDB warns that moving a document across shards can affect cluster performance. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

This gives another shard-key consideration:

> **How often does the shard-key value change?**

---

# 47. Why changing shard keys can be expensive

Normally:

```text
document
  ↓
update
```

is straightforward.

But if the update changes the shard-key value enough to move the document:

```text
Shard 1
   ↓
migration
   ↓
Shard 3
```

Now more infrastructure work is involved.

MongoDB explicitly warns about frequently changing shard-key values that cause documents to migrate between shards. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

Therefore:

> **A good shard key is usually stable.**

---

# 48. `refineCollectionShardKey`

Suppose you started with:

```javascript
{
    customerId: 1
}
```

but later realize that it's too coarse.

MongoDB can **refine** the key by appending fields.

For example:

```text
Old:
{ customerId: 1 }

New:
{ customerId: 1, orderId: 1 }
```

MongoDB documents `refineCollectionShardKey` as adding suffix fields to an existing shard key for finer-grained distribution. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-refine-a-shard-key/?utm_source=chatgpt.com "Refine a Shard Key - Database Manual - MongoDB Docs"))

This is useful because you don't always have to abandon the existing key completely.

---

# 49. But refinement does not mean “change anything”

MongoDB explicitly warns:

> Do not change the range/hashed type of an existing shard-key field during refinement.

For example, changing:

```text
{ customerId: 1 }
```

to:

```text
{ customerId: "hashed", orderId: 1 }
```

is not a valid refinement. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-refine-a-shard-key/?utm_source=chatgpt.com "Refine a Shard Key - Database Manual - MongoDB Docs"))

Refinement means:

```text
existing key
+
additional suffix fields
```

not arbitrary redesign.

---

# 50. Resharding

What if you chose the wrong shard key?

MongoDB has **resharding**.

Starting in MongoDB 5.0, MongoDB supports changing a collection's shard key with `reshardCollection`. MongoDB 8.0 also introduced the ability to reshard to the **same shard key** to redistribute data across new shards or zones more efficiently. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

This is incredibly useful.

It means:

```text
bad shard key
   ↓
not necessarily
"database is doomed forever"
```

But don't interpret resharding as:

> “Shard keys don't matter because I can fix them later.”

Resharding large datasets is an operationally significant process.

MongoDB's current documentation explicitly describes resharding as resource-intensive and recommends careful shard-key selection up front. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-reshard-a-collection/?utm_source=chatgpt.com "Reshard a Collection - Database Manual v8.0 - MongoDB Docs"))

---

# 51. MongoDB 8.0: resharding to the same key

This is a nice modern feature.

Suppose:

```text
Shard 1
Shard 2
```

and you add:

```text
Shard 3
```

but the distribution isn't where you'd like it.

MongoDB 8.0 allows resharding to the same key using:

```text
forceRedistribution: true
```

to redistribute the collection onto the available shards without changing the shard key. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-reshard-a-collection/?utm_source=chatgpt.com "Reshard a Collection - Database Manual v8.0 - MongoDB Docs"))

This is useful when the key is fine but the physical distribution needs improvement.

---

# 52. `shardAndDistributeCollection()`

MongoDB 8.0 introduced:

```javascript
sh.shardAndDistributeCollection()
```

for deployments meeting the documented requirements.

MongoDB recommends this in appropriate 8.0 deployments because it combines sharding and redistribution so data can be distributed across shards without waiting for normal balancer behavior to do the initial rebalance. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

This is another reason older sharding tutorials can be misleading.

---

# 53. Zones

Now suppose you have:

```text
Europe application servers
India application servers
US application servers
```

and you'd like:

```text
European data → Europe shards
Indian data   → India shards
US data       → US shards
```

MongoDB supports **zones**.

Zones associate shard-key ranges with one or more shards.

The balancer then keeps those ranges on shards belonging to the relevant zone. ([MongoDB](https://www.mongodb.com/docs/manual/core/zone-sharding/?utm_source=chatgpt.com "Zones - Database Manual - MongoDB Docs"))

---

# 54. Why zones matter

Zones can be used for:

```text
geographic locality
hardware classes
tenant isolation
regulatory/data-placement requirements
```

MongoDB explicitly lists geographical locality and hardware-based routing among zone use cases. ([MongoDB](https://www.mongodb.com/docs/manual/core/zone-sharding/?utm_source=chatgpt.com "Zones - Database Manual - MongoDB Docs"))

For example:

```text
Zone India
    ↓
Indian shard-key ranges
    ↓
Mumbai/Hyderabad shard group
```

---

# 55. Zones require planning the shard key around them

Suppose you want geographic zones based on:

```text
country
```

Then the shard key needs to contain the relevant geographical field.

MongoDB explicitly states that zone ranges must use fields contained in the shard key, and for compound keys the range must use the key prefix. ([MongoDB](https://www.mongodb.com/docs/manual/core/zone-sharding/?utm_source=chatgpt.com "Zones - Database Manual - MongoDB Docs"))

This is a very important architectural implication:

> **Future zone requirements should influence shard-key selection.**

---

# 56. Zone example

Suppose shard key:

```javascript
{
    country: 1,
    userId: 1
}
```

You might define:

```text
Zone India
    country = "India"

Zone US
    country = "USA"
```

and associate:

```text
India zone → Shard A + Shard B
USA zone   → Shard C + Shard D
```

Then the balancer respects those placement constraints. ([MongoDB](https://www.mongodb.com/docs/manual/core/zone-sharding/?utm_source=chatgpt.com "Zones - Database Manual - MongoDB Docs"))

---

# 57. Important: zones don't eliminate balancing

The balancer still works.

It's just constrained:

```text
normal balancing
     +
zone restrictions
```

So:

```text
Zone India
   ↓
only shards assigned to India zone
```

can receive those ranges.

MongoDB's documentation explicitly says that once zones are configured, the balancer respects them during future balancing. ([MongoDB](https://www.mongodb.com/docs/manual/core/zone-sharding/?utm_source=chatgpt.com "Zones - Database Manual - MongoDB Docs"))

---

# 58. Config-shard architecture in MongoDB 8.0

Here's a modern detail worth knowing.

Traditionally:

```text
Config Server Replica Set
```

is separate from shard replica sets.

Starting with MongoDB 8.0, MongoDB supports a **config shard**:

```text
one replica set
    ↓
acts as
    config servers
    +
    shard
```

This can reduce node count and simplify some deployments. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/config-shard/?utm_source=chatgpt.com "Config Shard - Database Manual v8.0 - MongoDB Docs"))

The conceptual architecture can therefore be:

```text
       Config Shard
       ├── config metadata
       └── application data
```

instead of:

```text
Config RS
+
Shard 1 RS
+
Shard 2 RS
...
```

This doesn't eliminate the need for configuration metadata or routing; it changes where the config-server role is hosted.

---

# 59. Is sharding automatically necessary for a large database?

No.

This is an important architectural principle.

Suppose:

```text
database = 2 TB
```

You might still operate comfortably with a properly sized replica set depending on:

```text
workload
storage
RAM
CPU
query patterns
latency requirements
growth
```

Sharding adds significant operational complexity. MongoDB explicitly says that sharding's trade-off is increased infrastructure and maintenance complexity. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

So:

> **Don't shard because “sharding is advanced.”**

Shard because the workload actually needs horizontal scale.

---

# 60. Vertical scaling vs horizontal scaling

Two broad approaches:

### Vertical

```text
bigger machine
 ↓
more CPU
more RAM
faster storage
```

### Horizontal

```text
more machines
 ↓
split workload/data
```

Sharding is horizontal scaling.

Replication doesn't solve the same problem because replicas generally maintain copies of the same data.

---

# 61. The CAP-theorem connection

Sharding itself isn't the same thing as CAP theorem.

But distributed MongoDB clusters inevitably encounter distributed-systems concerns:

```text
network partitions
latency
availability
consistency
quorum
```

You already saw some of this in Lesson 11.

Once you have:

```text
mongos
+
multiple shards
+
multiple replica sets
+
config servers
```

you have a genuinely distributed system.

Network failure is therefore a first-class engineering concern.

---

# 62. Sharding and replication together

This is the key architecture:

```text
                  SHARDED CLUSTER
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       SHARD 1       SHARD 2       SHARD 3
          │             │             │
      Replica Set    Replica Set    Replica Set
       / | \          / | \          / | \
      P  S  S        P  S  S        P  S  S
```

So each shard gives you:

```text
subset of data
+
high availability
```

This is how MongoDB combines:

```text
sharding
+
replication
```

for large production clusters. MongoDB requires each shard to be a replica set in the standard sharded architecture. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 63. A JobForge example

Now let's make this practical.

Suppose JobForge grows to:

```text
5 billion job documents
```

and you have users all around the world.

You might consider:

```text
jobs
```

with a possible compound shard key such as:

```text
tenantId + jobId
```

for a multi-tenant system.

Then:

```text
tenant A → distributed across shards by jobId
tenant B → distributed across shards by jobId
...
```

This could offer:

```text
high cardinality
good distribution
tenant-aware query targeting
```

But we should not declare this the “correct” key without looking at actual workload.

That is the point.

---

# 64. Suppose the main query is:

```text
"Give me jobs for tenant A"
```

Then:

```javascript
{
    tenantId: "A"
}
```

can use the shard-key prefix:

```text
tenantId
```

and potentially target only relevant shards.

MongoDB explicitly supports routing based on a compound shard-key prefix. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 65. But suppose your main query is:

```text
"Give me all Java jobs in Pune"
```

and your shard key is:

```text
tenantId + jobId
```

The query has no shard-key information.

Now:

```text
mongos
   ↓
broadcast
   ↓
all relevant shards
```

That's scatter-gather.

This is why shard-key selection starts from:

> **What does the application query?**

not:

> “Which field sounds unique?”

MongoDB explicitly recommends considering the most common query patterns when selecting a shard key. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 66. A ChatApp example

Suppose:

```text
messages
```

has billions of records.

Your dominant query is:

```text
messages between user A and user B
```

A possible shard-key candidate might involve:

```text
conversationId
```

if the data model has a conversation entity.

Why could this be useful?

```text
conversationId
      ↓
all messages for a conversation
      ↓
same shard/range locality
```

Then:

```text
GET conversation
```

may be targetable.

But there are trade-offs.

If one conversation becomes extremely hot:

```text
massive group
millions of messages
```

then a single shard can still become hot.

Again:

> **There is no universally perfect shard key.**

The workload determines what is good.

---

# 67. Why a random UUID isn't automatically the answer

You may hear:

> “Use UUID because it's random.”

Random/high-cardinality values can provide good distribution characteristics, but that's only one part of the problem.

Suppose:

```text
uuid
```

is your shard key.

You might get excellent distribution.

But your primary query might be:

```text
find all messages for conversationId
```

and conversationId isn't the shard key.

Then you may produce lots of scatter-gather queries.

So:

```text
good distribution
+
bad routing
=
bad overall shard-key choice
```

MongoDB explicitly emphasizes balancing distribution with common query patterns. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 68. Why query patterns matter more than people realize

Imagine two candidates:

```text
Key A
    perfect distribution
    terrible routing

Key B
    slightly less perfect distribution
    excellent routing
```

The right choice isn't determined from cardinality alone.

You have to ask:

```text
How frequently are key-based queries executed?
How evenly do writes distribute?
How hot can values become?
How large can ranges grow?
```

That's why MongoDB calls shard-key selection a workload-driven decision. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 69. The shard-key checklist

When evaluating a candidate shard key, ask:

```text
1. Cardinality?
2. Frequency of repeated values?
3. Monotonic?
4. Stable over time?
5. Can it target common queries?
6. Can it avoid scatter-gather?
7. Will writes distribute?
8. Will reads distribute?
9. Could one value create a hot chunk?
10. Could it create jumbo chunks?
11. Might we need zones later?
12. Could the field change?
```

MongoDB's official shard-key guidance covers many of these considerations explicitly. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 70. A subtle concept: hot read vs hot write

A shard key can fail in different ways.

### Hot write

```text
most writes
   ↓
one shard
```

Example:

```text
monotonic timestamp
```

under range sharding.

### Hot read

```text
most queries
   ↓
one shard
```

Example:

```text
one extremely popular tenant
```

Even if data is well distributed overall, one workload hotspot can dominate.

So ask both:

```text
Where does data live?
Where does the workload go?
```

Those aren't always the same thing.

---

# 71. A second subtle issue: data distribution vs query distribution

Imagine:

```text
Shard 1 → 33% data
Shard 2 → 33% data
Shard 3 → 34% data
```

Looks perfect.

But your API traffic is:

```text
Shard 1 → 10%
Shard 2 → 10%
Shard 3 → 80%
```

Why?

Maybe 80% of users query a particular range/tenant.

So:

```text
balanced data
    ≠
balanced workload
```

A good sharding architecture has to consider both.

---

# 72. Shard key and indexes

You already learned in Lesson 7:

```text
indexes
compound indexes
prefixes
```

Those concepts carry directly into sharding.

MongoDB requires the supporting index for the shard key, and for a populated collection the index must begin with the shard key. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-shard-key/?utm_source=chatgpt.com "Shard Keys - Database Manual - MongoDB Docs"))

So a shard key isn't just:

```text
"some field"
```

It has an indexing implication.

---

# 73. Example

Shard key:

```javascript
{
    tenantId: 1,
    jobId: 1
}
```

Supporting index:

```javascript
{
    tenantId: 1,
    jobId: 1
}
```

And additional indexes might exist for:

```text
status
location
skills
salary
createdAt
```

depending on query workload.

Each shard uses its indexes to execute the operations routed to it. MongoDB's routing documentation notes that once a shard receives a query, it chooses the most efficient available index. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 74. A very important misconception

> **“Sharding means every query becomes faster.”**

False.

Sharding is about scaling the system horizontally.

A badly targeted query can become:

```text
mongos
 ↓
10 shards
 ↓
10 queries
 ↓
10 responses
 ↓
merge
```

which can be worse than a well-indexed query on one server.

MongoDB explicitly warns that scatter-gather queries can be inefficient. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/distributed-queries/?utm_source=chatgpt.com "Distributed Queries - Database Manual v8.0 - MongoDB Docs"))

---

# 75. Another misconception

> **“More shards always mean more performance.”**

Not necessarily.

Suppose you have:

```text
10 shards
```

but 90% of your workload hits:

```text
Shard 7
```

You effectively have:

```text
one hot server
+
nine underutilized servers
```

This is why shard-key design is more important than simply increasing shard count.

---

# 76. Another misconception

> **“The balancer will fix any bad shard key.”**

No.

The balancer can move chunks.

But it can't magically turn:

```text
country = India
```

into many independently splittable values if millions of documents share the same shard-key value.

That is one source of jumbo/hot chunks.

MongoDB explicitly identifies poor shard keys as a cause of uneven distribution and jumbo chunks. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-shard-key/?utm_source=chatgpt.com "Shard Keys - Database Manual - MongoDB Docs"))

---

# 77. What happens when you query through `mongos`?

Let's make the complete flow.

Suppose:

```javascript
db.jobs.find({
    tenantId: "T100",
    jobId: "J400"
})
```

and shard key:

```javascript
{
    tenantId: 1,
    jobId: 1
}
```

Flow:

```text
Application
    ↓
mongos
    ↓
inspect shard-key predicate
    ↓
consult cached cluster metadata
    ↓
identify relevant chunk
    ↓
identify shard
    ↓
send query
    ↓
Shard
    ↓
local query planner
    ↓
index
    ↓
result
    ↓
mongos
    ↓
Application
```

MongoDB's routing documentation describes this metadata-driven targeting behavior. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 78. What if the query doesn't contain the shard key?

Example:

```javascript
db.jobs.find({
    salary: {
        $gte: 1000000
    }
})
```

Flow becomes:

```text
Application
    ↓
mongos
    ↓
"I cannot determine one target shard"
    ↓
Shard 1
Shard 2
Shard 3
...
    ↓
responses
    ↓
mongos merges
    ↓
application
```

That's scatter-gather.

([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 79. How do you diagnose this?

You can use:

```javascript
explain()
```

just like in Lesson 7.

For sharded queries, explain output can show work performed by individual shards.

For aggregation, `explain` can additionally show:

```text
mergeType
splitPipeline
shards
```

MongoDB specifically documents these fields for understanding distributed aggregation execution. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

So the workflow becomes:

```text
query
 ↓
explain()
 ↓
targeted?
broadcast?
which shards?
how much work?
```

---

# 80. MongoDB sharding in one picture

Here's the architecture I want you to be able to draw from memory:

```text
                         APPLICATION
                              │
                              ▼
                           mongos
                              │
                   cluster metadata
                              ▲
                              │
                        Config Servers
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
              Shard 1      Shard 2      Shard 3
                 │            │            │
              Replica       Replica      Replica
                Set            Set          Set
              / | \          / | \        / | \
             P  S  S        P  S  S      P  S  S
```

Inside each shard:

```text
             subset of collection data
                       ↓
                    chunks
                       ↓
                  local indexes
                       ↓
                  WiredTiger
```

The balancer can move those chunks between shards.

---

# 81. The entire lifecycle of sharding

Think:

```text
Design
   ↓
Choose shard key
   ↓
Shard collection
   ↓
Create/maintain chunks
   ↓
Route through mongos
   ↓
Monitor distribution
   ↓
Balancer moves ranges
   ↓
Add/remove shards as needed
   ↓
Refine or reshard if requirements change
```

That is the actual lifecycle.

---

# 82. A practical design exercise: ChatApp

Suppose your ChatApp has:

```text
10 billion messages
```

and the dominant query is:

```text
messages for conversation C
sorted by createdAt
```

What would you investigate?

Not:

```text
"Which field is random?"
```

Instead:

```text
What is the cardinality of conversationId?
How large can one conversation become?
How many messages per conversation?
How many active conversations?
Can one conversation become extremely hot?
Does the query always specify conversationId?
Do we need geographic zones?
How often does conversationId change?
```

Those questions determine the shard-key strategy.

---

# 83. A practical design exercise: JobForge

Suppose your system has:

```text
1 billion jobs
```

Queries:

```text
jobs for tenant
jobs by location
jobs by skill
jobs by salary
jobs sorted by recency
```

You can't pick a shard key from only one field without considering:

```text
tenant distribution
location distribution
skill distribution
query frequency
sort patterns
write distribution
```

This is why actual production shard-key design can require workload analysis.

MongoDB's current documentation explicitly recommends analyzing query patterns as part of shard-key selection. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 84. The biggest lesson of today's lesson

This is the statement I really want you to remember:

> **Sharding is not primarily about splitting a large database. It is about designing a distributed data placement and query-routing strategy.**

The physical splitting is the easy part.

The hard part is:

```text
Where should data live?
        +
Which server should receive this query?
        +
Will writes remain balanced?
        +
Will reads remain balanced?
        +
What happens as the cluster grows?
```

That is why shard-key selection is so important.

---

# 85. Exercises

## Exercise 1 — Fundamental

Which component is primarily responsible for routing application operations in a sharded cluster?

**A.** WiredTiger  
**B.** `mongos`  
**C.** `mongosh`  
**D.** Secondary

### Answer

**B — `mongos`.**

Applications normally connect to `mongos`, which uses cluster metadata to route operations to the appropriate shard(s). ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 86. Exercise 2

Which statement best describes replication vs sharding?

**A.** Replication splits data; sharding copies it.

**B.** Replication primarily provides redundancy/availability; sharding distributes data and workload horizontally.

**C.** They are exactly the same.

**D.** Sharding is only for backups.

### Answer

**B.** ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 87. Exercise 3 — Important

Suppose your shard key is:

```javascript
{
    tenantId: 1,
    userId: 1
}
```

Which query has the strongest opportunity for targeted routing?

### A

```javascript
{
    city: "Pune"
}
```

### B

```javascript
{
    tenantId: "T42"
}
```

### C

```javascript
{
    status: "active"
}
```

### D

```javascript
{
    salary: { $gte: 1000000 }
}
```

### Answer

**B.**

`tenantId` is the prefix of the compound shard key, so `mongos` can use it to restrict routing to the relevant shard subset. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharded-cluster-query-router/?utm_source=chatgpt.com "Routing with mongos - Database Manual v8.0 - MongoDB Docs"))

---

# 88. Exercise 4 — Hard

Why might this be a poor ranged-sharding key?

```javascript
{
    createdAt: 1
}
```

if:

```text
99% of writes are new records
```

### Answer

Because `createdAt` changes monotonically.

New values are concentrated toward the current upper end of the key range, which can concentrate writes into the newest chunk/shard and create a hot shard.

MongoDB explicitly identifies monotonically changing shard keys as a potential cause of uneven write distribution. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 89. Exercise 5 — Very important

Why might hashed sharding help with a monotonically increasing key?

### Answer

Hashing transforms the original ordered values into a distributed hashed key space, making consecutive original values much less likely to cluster in the same range.

MongoDB specifically notes that hashed sharding can provide more even distribution for monotonically changing values. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

The trade-off is that range queries on that hashed key become less naturally targetable. ([MongoDB](https://www.mongodb.com/docs/v8.0/sharding/?utm_source=chatgpt.com "Sharding - Database Manual v8.0 - MongoDB Docs"))

---

# 90. Exercise 6 — Architecture

You have:

```text
Shard 1 → 1 TB
Shard 2 → 1 TB
Shard 3 → 1 TB
```

You add:

```text
Shard 4 → 0 TB
```

What mechanism moves data toward the new shard?

### Answer

The **balancer** migrates ranges/chunks between shards over time. The redistribution is not instantaneous and consumes system resources. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/sharding-balancer-administration/?utm_source=chatgpt.com "Sharded Cluster Balancer - Database Manual v8.0 - MongoDB Docs"))

---

# 91. Exercise 7 — Difficult

Suppose:

```javascript
{
    country: 1
}
```

is your shard key.

And 80% of all documents have:

```text
country = "India"
```

What problems might you encounter?

### Answer

Potentially:

```text
high-frequency shard-key value
↓
uneven distribution
↓
hot chunk/shard
↓
limited ability to split that value
↓
possible jumbo chunk
```

MongoDB explicitly identifies high-frequency values and low-cardinality shard keys as causes of poor distribution and potential jumbo chunks. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 92. Exercise 8 — Real engineering question

A team says:

> “We have 12 shards, so every query will automatically be faster.”

What's wrong with that statement?

### Answer

Sharding does not automatically make every query fast.

Queries that contain shard-key information can often be targeted.

Queries without it may become scatter-gather operations:

```text
mongos
 ↓
all shards
 ↓
responses
 ↓
merge
```

MongoDB explicitly warns that scatter-gather queries can be long-running and inefficient, especially as the cluster grows. ([MongoDB](https://www.mongodb.com/docs/v8.0/core/distributed-queries/?utm_source=chatgpt.com "Distributed Queries - Database Manual v8.0 - MongoDB Docs"))

---

# 93. One final challenge

Imagine you're designing a sharded `messages` collection.

Requirements:

```text
10 billion messages

Main query:
"Give me messages for conversation X,
newest first."

Some conversations:
few hundred messages

Largest conversations:
100 million messages
```

You are considering:

```text
A. createdAt
B. conversationId
C. hashed conversationId
D. conversationId + createdAt
```

Don't automatically pick a winner.

The right engineering analysis would ask:

```text
Does conversationId have high enough cardinality?
Can one conversation become a hot shard?
Can the largest conversation overwhelm one shard?
Do we need exact conversation targeting?
Do we need range locality by time?
Will writes be concentrated?
Would a hashed key destroy useful time-range locality?
Would a compound key improve distribution?
What does the actual read/write workload look like?
```

That is **exactly** the kind of thinking I want you to develop.

MongoDB's documentation itself emphasizes that shard-key design depends on cardinality, frequency, monotonicity, and actual query patterns rather than a single universal recipe. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

# 94. Lesson 12 — permanent mental model

```text
                    SHARDING
                       │
                       ▼
                horizontal scale
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
           Shard 1  Shard 2  Shard 3
              │        │        │
           Replica   Replica   Replica
             Set       Set       Set
              ▲
              │
        routed by mongos
              ▲
              │
          Application
```

The routing decision depends on:

```text
                    SHARD KEY
                        │
        ┌───────────────┼────────────────┐
        ▼               ▼                ▼
    Cardinality      Frequency       Monotonicity
        │               │                │
        └───────────────┼────────────────┘
                        ▼
                  distribution
                        +
                  query patterns
                        ↓
               targeted operations
                        vs
                 scatter-gather
```

And underneath all of it:

```text
Chunks
   ↓
Balancer
   ↓
Range migration
   ↓
Data redistribution
```

---

# 95. What you should know after Lesson 12

You should now understand these terms without memorizing a textbook definition:

```text
Sharding
   → horizontal distribution

Shard
   → subset of collection data

Shard key
   → determines data distribution/routing

mongos
   → query router

Config servers
   → cluster metadata

Chunk
   → shard-key range assigned to a shard

Balancer
   → migrates ranges to improve distribution

Ranged sharding
   → contiguous key ranges

Hashed sharding
   → hashed key distribution

Targeted query
   → routed to relevant shard(s)

Scatter-gather
   → broadcast to multiple/all shards

Hot shard
   → disproportionate workload on one shard

Jumbo chunk
   → range too large/problematic to split/migrate

Zone
   → constrained placement of shard-key ranges

Refine shard key
   → add suffix fields

Reshard
   → change/reorganize shard-key distribution
```

And the biggest principle:

> **A shard key should be chosen from the application's real workload, not just from the data model.**

MongoDB's current documentation emphasizes exactly this: effective shard keys need appropriate distribution characteristics **and** support common query patterns. ([MongoDB](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/?utm_source=chatgpt.com "Choose a Shard Key - Database Manual - MongoDB Docs"))

---

## Where you are now

You've completed:

```text
1. Fundamentals & CRUD
2. Querying
3. Data Modeling
4. Updates & Arrays
5. Aggregation Fundamentals
6. Advanced Aggregation
7. Indexes & Query Performance
8. Schema Validation
9. Transactions & Concurrency
10. Architecture & Storage Internals
11. Replication & High Availability
12. Sharding & Horizontal Scaling ✅
```

The next lesson is **Lesson 13 — Change Streams & Advanced MongoDB Features**.

We'll cover the mechanisms that let an application react to database changes in real time:

```text
insert
update
delete
        ↓
Change Stream
        ↓
application event
        ↓
WebSocket / Socket.IO
Kafka
notifications
search indexing
analytics
audit systems
```

We'll also get into **resume tokens, watch(), filtering change events, `fullDocument`, deployment requirements, change-stream behavior with replica sets/sharded clusters, and how this could be used to improve your ChatApp architecture**.