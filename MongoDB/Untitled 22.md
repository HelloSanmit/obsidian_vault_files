
# Gap Module 5 — Lessons 10–12: Operational + Replica Set + Sharding Administration

This module closes the remaining “I understand the architecture, but could I actually operate/debug it?” gaps.

You do **not** need to memorize every command here. The goal is to recognize what tool you reach for when something goes wrong.

---

# Lesson 10 Gap — Operational Toolbox

We previously covered:

- `serverStatus()`
    
- `dbStats()`
    
- `currentOp()`
    
- profiler / slow-query analysis
    
- `$indexStats`
    
- `$queryStats`
    

Now add four practical tools:

1. `db.collection.stats()`
    
2. `mongostat`
    
3. `mongotop`
    
4. `mongodump` / `mongorestore`
    

---

## 1. `db.collection.stats()`

### What is it?

It gives statistics about **one collection**.

Think:

> “Tell me how big this collection is, how many documents it has, and how large its indexes are.”

MongoDB documents `db.collection.stats()` as a wrapper around collection statistics, with information such as document count, collection size, index count, index sizes, and storage size. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/db.collection.stats/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

### Example

```javascript
db.applications.stats()
```

You might see things conceptually like:

```javascript
{
  ns: "jobforge.applications",
  count: 1500000,
  avgObjSize: ...,
  storageSize: ...,
  nindexes: 5,
  totalIndexSize: ...,
  indexSizes: {
    "_id_": ...,
    "candidateId_1_createdAt_-1": ...
  }
}
```

### Important fields

|Field|Meaning|
|---|---|
|`count`|Number of documents|
|`avgObjSize`|Average document size|
|`storageSize`|Storage allocated for collection data|
|`nindexes`|Number of indexes|
|`totalIndexSize`|Total index storage|
|`indexSizes`|Size of each index|

---

## `dataSize` vs `storageSize`

This is a good interview concept.

Suppose:

```text
dataSize      = logical/uncompressed data
storageSize   = storage allocated on disk
```

With WiredTiger compression, `storageSize` can be smaller than `dataSize`.

Also, `storageSize` is about allocated collection storage and does not include index space; index storage is reported separately. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/dbstats/?utm_source=chatgpt.com "dbStats (database command) - Database Manual - MongoDB Docs"))

So don't look at:

```text
dataSize
```

and conclude:

> “My disk must contain exactly this many bytes.”

It isn't that simple.

---

## Useful example

```javascript
db.applications.stats({
  scale: 1024
})
```

### What is `scale`?

It changes the displayed size units.

```text
1      → bytes
1024   → roughly KB
1048576 → roughly MB
```

---

# 2. `mongostat`

### What is it?

`mongostat` is a **command-line monitoring tool** that gives a quick overview of a running `mongod` or `mongos`.

Think:

> “What is MongoDB doing right now?”

MongoDB describes it as a quick overview useful for identifying system bottlenecks. ([MongoDB](https://www.mongodb.com/docs/database-tools/mongostat/?utm_source=chatgpt.com "mongostat - Database Tools - MongoDB Docs"))

You run it from the **system terminal**, not inside `mongosh`.

```bash
mongostat
```

Or against a deployment:

```bash
mongostat "<connection-string>"
```

---

## Mental model

Suppose your API suddenly becomes slow.

You could look at:

```text
mongostat
```

and get a quick operational picture of activity.

It is more like:

```text
MongoDB live dashboard in terminal
```

than a deep query profiler.

---

# 3. `mongotop`

### What is it?

`mongotop` tells you how much time MongoDB is spending **reading and writing by namespace/collection**.

Think:

> “Which collection is getting hammered?”

MongoDB describes it as tracking the amount of time `mongod` spends reading and writing data, with statistics at the collection level. ([MongoDB](https://www.mongodb.com/docs/database-tools/mongotop/?utm_source=chatgpt.com "mongotop - Database Tools - MongoDB Docs"))

Run:

```bash
mongotop
```

Or:

```bash
mongotop 30
```

Here:

```text
30
```

means approximately:

> report every 30 seconds.

---

## `mongostat` vs `mongotop`

This distinction is worth remembering:

```text
mongostat
    ↓
overall MongoDB activity

mongotop
    ↓
which namespaces/collections are consuming read/write time
```

So:

> “MongoDB looks busy.”

→ `mongostat`

> “Which collection is causing that activity?”

→ `mongotop`

---

# 4. `mongodump`

### What is it?

`mongodump` creates a **logical backup** of MongoDB data.

Example:

```bash
mongodump --uri="<connection-string>" --out=./backup
```

It produces BSON dump files plus metadata.

Think:

```text
MongoDB
   ↓
mongodump
   ↓
backup files
```

---

# 5. `mongorestore`

### What is it?

`mongorestore` loads a dump created by `mongodump` back into MongoDB. It is a separate Database Tools utility and is run from the system command line, not `mongosh`. ([MongoDB](https://www.mongodb.com/docs/database-tools/mongorestore/?utm_source=chatgpt.com "mongorestore - Database Tools - MongoDB Docs"))

Example:

```bash
mongorestore --uri="<connection-string>" ./backup
```

Mental model:

```text
mongodump
    = export/backup

mongorestore
    = import/restore
```

---

## Very important distinction

You already learned:

```text
replication ≠ backup
```

Now add:

```text
replica set
    → high availability

mongodump
    → logical backup

backup/PITR system
    → disaster recovery
```

A replica set does **not** protect you from every bad application action.

Example:

```javascript
db.users.deleteMany({})
```

If a bad application operation deletes millions of documents and successfully replicates, the secondaries also receive the deletion.

Replication faithfully copies the mistake.

That is why backups remain necessary.

---

# Lesson 11 Gap — Replica Set Administration

Now let's move from:

> “I understand replica sets”

to:

> “I know the basic commands operators use.”

---

# 1. `rs.add()`

### What is it?

Adds a new member to an existing replica set.

Example:

```javascript
rs.add("mongodb4:27017")
```

Conceptually:

```text
Primary
   ├── Secondary 1
   ├── Secondary 2
   └── New member
```

The command is normally run against the replica-set primary. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/?utm_source=chatgpt.com "mongosh Methods - Database Manual - MongoDB Docs"))

---

# 2. `rs.remove()`

### What is it?

Removes a member from the replica set.

```javascript
rs.remove("mongodb4:27017")
```

MongoDB notes that removing a member can cause the shell to briefly disconnect while the replica set renegotiates the primary. ([MongoDB](https://www.mongodb.com/docs/v7.0/reference/method/rs.remove/?utm_source=chatgpt.com "Database Manual v7.0 - MongoDB Docs"))

### Operational idea

You might do this when:

```text
old server
    ↓
remove from replica set
    ↓
retire server
```

---

# 3. `rs.stepDown()`

This one is **very important**.

### What does “step down” mean?

It means:

> Tell the current primary to stop being primary.

Example:

```javascript
rs.stepDown()
```

MongoDB then triggers an election and an eligible secondary can become the new primary. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/?utm_source=chatgpt.com "mongosh Methods - Database Manual - MongoDB Docs"))

---

## Why would you intentionally do that?

Imagine:

```text
Primary: db1
Secondary: db2
Secondary: db3
```

You want to restart `db1`.

You don't want to suddenly kill your primary and discover what happens.

Instead:

```text
rs.stepDown()
       ↓
db2 becomes primary
       ↓
restart db1
```

That's a controlled failover test / maintenance workflow.

---

# 4. `rs.freeze()`

### What is it?

It temporarily prevents the current replica-set member from seeking election as primary.

Example:

```javascript
rs.freeze(60)
```

Meaning conceptually:

```text
For 60 seconds:
don't try to become primary
```

MongoDB still documents `rs.freeze()` for self-managed deployments; it is not supported in Atlas clusters. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/replSetFreeze/?utm_source=chatgpt.com "replSetFreeze (database command) - Database Manual - MongoDB Docs"))

To remove the freeze:

```javascript
rs.freeze(0)
```

---

## Why is this useful?

Suppose you are doing controlled replica-set maintenance and don't want a particular secondary to become primary.

You can temporarily make it ineligible.

### But remember

For modern production design, **member priority/configuration** is usually a more fundamental mechanism for determining which members are intended to be electable.

`rs.freeze()` is an operational tool, not your everyday topology design.

---

# 5. `rs.reconfig()`

### What is it?

Changes the replica-set configuration.

Start by inspecting configuration:

```javascript
rs.conf()
```

Then modify it:

```javascript
let cfg = rs.conf()

cfg.members[1].priority = 0

rs.reconfig(cfg)
```

The replica-set config contains things such as:

```text
members
priority
votes
hidden
secondaryDelaySecs
...
```

`rs.reconfig()` can add/remove members and modify member configuration. MongoDB warns that a reconfiguration can cause the current primary to step down, resulting in an election and client disconnections. ([MongoDB](https://www.mongodb.com/docs/manual/reference/command/replsetreconfig/?utm_source=chatgpt.com "replSetReconfig (database command) - Database Manual - MongoDB Docs"))

---

## Very important mental distinction

```text
rs.add()
    simple member addition

rs.remove()
    member removal

rs.reconfig()
    modify the actual replica-set configuration

rs.stepDown()
    intentionally give up primary role

rs.freeze()
    temporarily prevent a member from seeking primary
```

---

# 6. `hello`

This is a particularly useful one because it explains how drivers understand the topology.

### What is `hello`?

It asks a MongoDB server:

> “Who are you, and what role do you currently have?”

Example:

```javascript
db.runCommand({ hello: 1 })
```

or in `mongosh`:

```javascript
db.hello()
```

For replica sets, its response can include things such as:

```text
am I primary?
am I secondary?
what is the replica-set name?
who is the current primary?
who are the other members?
```

MongoDB drivers use `hello` for topology discovery and replica-set member discovery. ([MongoDB](https://www.mongodb.com/docs/v8.0/reference/command/hello/?utm_source=chatgpt.com "hello (database command) - Database Manual v8.0 - MongoDB Docs"))

---

## Why should YOU care?

You previously learned:

> MongoDB drivers automatically reconnect/fail over.

Now you can understand one part of how they know what is happening.

Conceptually:

```text
Application
     ↓
MongoDB Driver
     ↓
hello
     ↓
"What is the topology?"
     ↓
Primary / Secondary / Hosts / Set name
```

So `hello` is not just some random admin command.

It is closely connected to **driver topology discovery**.

---

# Lesson 11 — Failure Scenario

Imagine:

```text
db1 = primary
db2 = secondary
db3 = secondary
```

Your monitoring system says:

```text
db1 needs maintenance
```

A controlled workflow could conceptually be:

```javascript
rs.status()
```

Check health.

Then:

```javascript
rs.stepDown()
```

Primary gives up the role.

Then:

```javascript
rs.status()
```

Verify a new primary exists.

Then maintenance on old `db1`.

This is the operational version of what we studied conceptually in Lesson 11.

---

# Lesson 12 Gap — Sharding Administration

Now let's do the equivalent for sharding.

Remember our architecture:

```text
Application
     ↓
   mongos
     ↓
 ┌───────┬───────┬───────┐
Shard A  Shard B  Shard C
```

The commands below help you inspect and control that cluster.

---

# 1. `sh.status()`

### What is it?

A high-level report about a sharded cluster.

```javascript
sh.status()
```

It can show things such as:

```text
shards
databases
shard keys
chunk distribution
balancer state
active migrations
```

MongoDB specifically documents the balancer section as useful when troubleshooting an unbalanced sharded cluster. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/sh.status/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

## Think of it as:

```text
rs.status()
       ↓
replica-set overview

sh.status()
       ↓
sharded-cluster overview
```

That comparison is useful in interviews.

---

# 2. `sh.enableSharding()`

This one requires a **modern-version correction**.

Historically, you often saw:

```javascript
sh.enableSharding("jobforge")
```

followed by:

```javascript
sh.shardCollection(...)
```

But since MongoDB 6.0, you do **not need** `sh.enableSharding()` first in order to shard a collection. `sh.shardCollection()` can handle that directly. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/sh.enableSharding?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

The current helper still exists and is used to explicitly create a database:

```javascript
sh.enableSharding("jobforge")
```

So learn:

```text
old/common tutorial flow:
enableSharding()
    ↓
shardCollection()

modern MongoDB:
shardCollection() is not dependent on enableSharding()
```

This is exactly the sort of version-awareness that prevents blindly memorizing outdated tutorials.

---

# 3. `sh.shardCollection()`

### What is it?

It turns a collection into a **sharded collection** and specifies its shard key.

Example:

```javascript
sh.shardCollection(
    "jobforge.applications",
    { candidateId: 1 }
)
```

Meaning roughly:

```text
jobforge.applications
        ↓
distributed across shards
        ↓
candidateId determines placement/routing
```

MongoDB defines `sh.shardCollection()` as sharding a collection using the supplied shard key. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/sh.shardcollection/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

## Important

This command does **not** mean:

> “Put half the documents on shard A and half on shard B.”

The shard key determines how MongoDB partitions/routs the data.

That connects directly to the Lesson 12 concepts:

```text
cardinality
frequency
monotonicity
query targeting
hot shards
range vs hashed
```

---

# 4. Balancer

Now we reach one of the most important operational concepts.

Remember:

```text
Chunks / ranges
       ↓
may become unevenly distributed
       ↓
balancer moves ranges between shards
```

The balancer helps redistribute data across shards.

---

## Current `mongosh` helper

The modern `mongosh` helper is:

```javascript
sh.startBalancer()
```

to enable/start balancing.

And:

```javascript
sh.stopBalancer()
```

to stop it.

MongoDB's current `mongosh` method reference uses `startBalancer()` / `stopBalancer()`. The underlying database commands are `balancerStart` and `balancerStop`. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/?utm_source=chatgpt.com "mongosh Methods - Database Manual - MongoDB Docs"))

So don't memorize:

```javascript
sh.balancerStart()
```

as your modern `mongosh` method.

---

# 5. `sh.stopBalancer()`

Example:

```javascript
sh.stopBalancer()
```

Conceptually:

```text
Stop automatic balancing/migration activity
```

You may do this temporarily during certain administrative procedures.

MongoDB's current operational procedures use `sh.stopBalancer()` and then check its state before making changes that could interact with migrations. ([MongoDB](https://www.mongodb.com/docs/manual/release-notes/8.3-upgrade-sharded-cluster/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

---

# 6. `sh.startBalancer()`

Re-enable balancing:

```javascript
sh.startBalancer()
```

Conceptually:

```text
balancer OFF
     ↓
maintenance/configuration
     ↓
balancer ON
     ↓
cluster can rebalance
```

---

# 7. `sh.isBalancerRunning()`

### What is it?

Checks whether balancing is currently running.

```javascript
sh.isBalancerRunning()
```

This is useful after:

```javascript
sh.stopBalancer()
```

because you don't just want to assume it stopped immediately.

Likewise after starting the balancer, it gives you a way to inspect its current running state. MongoDB documents the helper for reporting balancer status. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/?utm_source=chatgpt.com "mongosh Methods - Database Manual - MongoDB Docs"))

---

# Sharding Operational Scenario

Suppose:

```text
Shard A → 70% of data
Shard B → 20%
Shard C → 10%
```

You investigate:

```javascript
sh.status()
```

and determine that migrations/balancing need attention.

You might inspect:

```javascript
sh.isBalancerRunning()
```

Then, depending on the maintenance task:

```javascript
sh.stopBalancer()
```

Perform the administrative work.

Then:

```javascript
sh.startBalancer()
```

And inspect:

```javascript
sh.status()
```

again.

---

# Very Important: Balancer ≠ Query Router

This is a common interview confusion.

### `mongos`

Routes queries:

```text
Client
  ↓
mongos
  ↓
correct shard(s)
```

### Balancer

Moves distributed data/ranges between shards when necessary:

```text
Shard A ←→ Shard B
        ↑
    balancer
```

So:

```text
mongos
    = request routing

balancer
    = data distribution management
```

Completely different responsibilities.

---

# The Most Important Operational Mental Model

You now have three different levels of MongoDB operations.

## Level 1 — Collection/data investigation

```javascript
db.applications.stats()
db.stats()
db.currentOp()
db.applications.getIndexes()
```

Question:

> “What is happening inside this database/collection?”

---

## Level 2 — Replica-set operations

```javascript
rs.status()
rs.conf()

rs.add()
rs.remove()
rs.stepDown()
rs.freeze()
rs.reconfig()

db.hello()
```

Question:

> “What is happening with my high-availability topology?”

---

## Level 3 — Sharded-cluster operations

```javascript
sh.status()

sh.shardCollection()

sh.startBalancer()
sh.stopBalancer()
sh.isBalancerRunning()
```

Question:

> “What is happening with my horizontally distributed topology?”

---

# Interview Drill

You should be able to answer these without looking anything up.

### Q1. A collection is suddenly using much more disk space. What would you inspect?

Start with:

```javascript
db.collection.stats()
```

Then inspect:

```text
storageSize
indexSizes
totalIndexSize
```

---

### Q2. MongoDB is very busy, but you don't know which collection is responsible.

Use:

```bash
mongotop
```

---

### Q3. You want a quick overall view of MongoDB activity.

Use:

```bash
mongostat
```

---

### Q4. You need to intentionally move the replica-set primary role to another member.

Use:

```javascript
rs.stepDown()
```

Then verify:

```javascript
rs.status()
```

---

### Q5. You need to temporarily prevent a replica-set member from becoming primary.

```javascript
rs.freeze(...)
```

---

### Q6. Your application needs to know whether the connected node is primary.

```javascript
db.hello()
```

---

### Q7. You need a high-level view of a sharded cluster.

```javascript
sh.status()
```

---

### Q8. You want to make `applications` a sharded collection using `candidateId`.

Conceptually:

```javascript
sh.shardCollection(
    "jobforge.applications",
    { candidateId: 1 }
)
```

---

### Q9. You want to stop shard balancing temporarily.

```javascript
sh.stopBalancer()
```

---

### Q10. You want to check whether balancing is currently active.

```javascript
sh.isBalancerRunning()
```

---

# One More Important Version-Awareness Point

MongoDB has changed several administrative interfaces over time.

For example:

```text
sh.enableSharding()
```

is no longer a prerequisite for `sh.shardCollection()` from MongoDB 6.0 onward. ([MongoDB](https://www.mongodb.com/docs/manual/reference/method/sh.shardcollection/?utm_source=chatgpt.com "Database Manual - MongoDB Docs"))

Likewise, don't confuse the old/underlying command naming:

```text
balancerStart
balancerStop
```

with the current `mongosh` helpers:

```text
sh.startBalancer()
sh.stopBalancer()
```

This matters because many YouTube courses and older tutorials mix MongoDB versions.

---

# Your MongoDB Knowledge After Module 5

At this point your coverage is roughly:

```text
L1   CRUD + BSON                         ✅
L2   Querying                            ✅
L3   Data Modeling                       ✅
L4   Updates + Arrays                    ✅
L5   Aggregation                         ✅
L6   Advanced Aggregation                ✅
L7   Indexing + Performance              ✅
L8   Schema Validation                   ✅
L9   Transactions + Concurrency           ✅
L10  Architecture + Storage              ✅ + operational gaps
L11  Replication + HA                    ✅ + admin gaps
L12  Sharding                            ✅ + admin gaps
L13  Change Streams                      ✅
L14  Node.js + Mongoose                  ✅
L15  Production MongoDB                  ✅
L16  Real Project Database Design        ✅
```

You are now past the point where MongoDB should feel like “CRUD plus aggregation.” You have the important application-development, performance, consistency, availability, scaling, and production mental models.

**Next gap module:** Lesson 13–14 edge cases: advanced Change Streams behavior plus the remaining Mongoose/Node-driver details (`discriminators`, plugins, query middleware, deeper session/transaction behavior, and a few driver-level patterns).