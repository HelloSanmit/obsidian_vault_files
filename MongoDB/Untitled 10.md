
# MongoDB — Lesson 11: Replication & High Availability

This is one of the most important MongoDB architecture lessons.

Lesson 10 explained:

```text
mongod
  ↓
WiredTiger
  ↓
cache
  ↓
journal
  ↓
checkpoint
  ↓
disk
```

Lesson 11 adds **multiple MongoDB servers**:

```text
                 Application
                      │
                      ▼
                  MongoDB
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        Primary    Secondary    Secondary
```

Now the questions become:

> What happens when the primary dies?

> How does a secondary know what changed?

> How quickly does it catch up?

> What does “majority” actually mean?

> Can a read return stale data?

> What happens during a network partition?

> What is the oplog?

> Why can a write acknowledged with `w: "majority"` still be absent from a secondary immediately afterward in MongoDB 8.0?

These are real distributed-systems questions, and they matter for backend engineering.

MongoDB calls the group of servers maintaining the same data set a **replica set**. Replica sets provide redundancy and high availability. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/glossary/?utm_source=chatgpt.com))

---

# 1. Why replication exists

Imagine your application connects to one MongoDB server:

```text
Application
    ↓
MongoDB
```

Now that server suffers:

```text
power failure
hardware failure
disk failure
network failure
software crash
```

Your application loses access to the database.

Replication changes the architecture to:

```text
                 ┌───────────────┐
                 │   MongoDB     │
                 │  Replica Set  │
                 └───────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Primary       Secondary       Secondary
```

The secondaries maintain copies of the data.

If the primary becomes unavailable, an eligible secondary can become the new primary through an election. MongoDB's documentation describes replica sets as providing redundancy and automatic failover through elections. ([mongodb.com](https://www.mongodb.com/docs/manual/replication/?utm_source=chatgpt.com))

---

# 2. Replica set terminology

There are several member types.

The most important:

```text
Primary
Secondary
```

and less commonly:

```text
Arbiter
Hidden member
Delayed member
Priority-0 member
Non-voting member
```

We'll start with primary and secondary.

---

# 3. Primary

A replica set can have at most one primary at a time.

The primary:

```text
receives writes
records changes in the oplog
```

Secondaries then replicate those changes.

MongoDB's official documentation states that the primary is the only member that receives write operations; it applies them and records them in its oplog, while secondaries copy and apply those operations. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-primary/?utm_source=chatgpt.com))

Conceptually:

```text
Client
  ↓
PRIMARY
  │
  ├── data change
  │
  └── oplog entry
           ↓
       SECONDARIES
```

---

# 4. Secondary

A secondary:

```text
maintains a copy of the data set
```

and continuously replicates changes from a source member.

The replication is asynchronous.

That one word is extremely important:

> **Asynchronous.**

MongoDB's secondaries copy the oplog and apply entries asynchronously. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

So don't imagine:

```text
PRIMARY writes
     ↓
all secondaries instantly become identical
```

Instead:

```text
PRIMARY
  ↓
oplog
  ↓
network
  ↓
SECONDARY receives
  ↓
SECONDARY applies
```

There can be a delay.

---

# 5. Replication is not “copy the database every time”

This is a major misconception.

Suppose the database contains:

```text
500 GB
```

and you insert:

```text
one 2 KB document
```

The secondary doesn't normally copy all 500 GB again.

Instead, it receives the change through the **oplog**.

That's the central mechanism of MongoDB replication.

---

# 6. The oplog

You encountered the word in Lesson 10.

Now understand it properly.

**Oplog = operations log.**

MongoDB describes it as a special capped collection containing a rolling record of operations that modify data. It lives in:

```text
local.oplog.rs
```

on replica-set members. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

Conceptually:

```text
PRIMARY

data changes
    ↓
oplog
    ↓
SECONDARY
```

Example:

```javascript
{
    op: "i",
    ns: "chatapp.messages",
    ...
}
```

The exact oplog representation can be more involved depending on the operation/version, so don't memorize this example as a literal universal format.

The important idea is:

> **The oplog is the stream of data changes used for replication.**

---

# 7. Oplog vs journal

This distinction is absolutely essential.

You learned about the journal in Lesson 10.

They are **not the same thing**.

### Journal

Primarily:

```text
storage-engine crash recovery / durability
```

### Oplog

Primarily:

```text
replication of data changes between replica-set members
```

MongoDB's documentation separately describes WiredTiger journaling and replica-set oplog behavior. ([mongodb.com](https://www.mongodb.com/docs/manual/core/journaling/?utm_source=chatgpt.com))

Think:

```text
             WRITE
               │
       ┌───────┴────────┐
       ▼                ▼
    Journal           Oplog
       │                │
       ▼                ▼
 crash recovery     replication
```

One change can therefore participate in multiple mechanisms serving different purposes.

---

# 8. Where is the oplog stored?

In the special:

```text
local
```

database:

```text
local.oplog.rs
```

MongoDB documents the local database as containing internal replica-set information, including the oplog and replica-set configuration. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/local-database/?utm_source=chatgpt.com))

This is special infrastructure.

You should not treat:

```text
local.oplog.rs
```

like a normal application collection.

---

# 9. The oplog is capped, but not simply “fixed forever”

The oplog is a **rolling record**.

Older entries are removed as space is needed.

Current MongoDB documentation says that with WiredTiger the default oplog size on Unix/Windows systems is generally 5% of free disk space, subject to documented minimum and maximum constraints, unless you configure it differently. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

The exact defaults vary by platform, so don't memorize “5%” as universal.

The more important concept is:

```text
oplog = finite history window
```

---

# 10. The oplog window

Suppose the oplog contains:

```text
oldest entry:  10:00
newest entry:  18:00
```

Then the oplog window is roughly:

```text
8 hours
```

MongoDB defines the oplog window as the time difference between the newest and oldest oplog timestamps. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

Now imagine a secondary goes offline.

It comes back:

```text
2 hours later
```

The oplog still contains the operations it missed.

It can catch up.

But suppose it comes back:

```text
20 hours later
```

and the oplog only contains the last 8 hours.

Some required entries are gone.

The secondary can no longer simply continue from where it left off.

It needs **initial sync**.

MongoDB explicitly states that if a secondary's replication gap exceeds the oplog window, it must perform an initial sync. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

---

# 11. This makes oplog sizing important

Suppose:

```text
Replica set
    ↓
Primary writes heavily
    ↓
oplog fills quickly
```

Your oplog window might be:

```text
2 hours
```

Then a secondary that is offline for 3 hours can't catch up from the oplog.

A healthier workload might give:

```text
24-hour oplog window
```

or more, depending on requirements.

So oplog size isn't primarily about:

> “How many gigabytes of logs should I have?”

It's often more useful to think:

> **“How much time of write history can my oplog retain?”**

MongoDB explicitly recommends ensuring the oplog window is sufficient for secondary recovery and initial-sync-related operations. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

---

# 12. Initial sync

Suppose we add a new secondary:

```text
Primary
Secondary
    ↓
brand new
```

It doesn't contain your 2 TB database.

It needs an **initial sync**.

MongoDB's current synchronization process has two broad forms:

```text
initial sync
+
ongoing replication
```

([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

---

# 13. Logical initial sync

In a logical initial sync, MongoDB roughly:

```text
1. clones databases/collections
2. builds indexes
3. catches up using buffered oplog entries
4. applies remaining oplog changes
5. becomes SECONDARY
```

MongoDB's current documentation describes this process in these terms. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

Conceptually:

```text
PRIMARY
  │
  ├───────────────┐
  │               │
  │   copy data   │
  │──────────────>│
  │               │
  │   oplog       │
  │──────────────>│
  │               │
  └── catch up ───┘
                  ↓
               SECONDARY
```

---

# 14. Why does initial sync need the oplog?

Imagine:

```text
10:00
secondary starts copying database
```

But while it's copying:

```text
10:01 → write A
10:02 → write B
10:03 → write C
```

The copied snapshot from the beginning doesn't contain those later changes.

So the secondary needs:

```text
initial data
+
changes that happened during copying
```

That's exactly where oplog entries come in.

MongoDB buffers and later applies oplog entries generated during the copy phase. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

---

# 15. Why oplog window affects initial sync

Suppose:

```text
initial sync takes 8 hours
```

but:

```text
oplog window = 4 hours
```

You could finish copying the data only to discover that some of the operations from early in the sync have already fallen out of the oplog.

Then the secondary may need to restart initial sync.

MongoDB explicitly warns that the oplog must be large enough to maintain the needed history during logical initial sync. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

This creates a very important relationship:

```text
large dataset
     +
slow initial sync
     ↓
needs sufficiently large oplog window
```

---

# 16. Streaming replication

Modern MongoDB uses streaming replication.

Instead of secondaries periodically asking for large batches in the older style, a source can send a continuous stream of oplog entries.

MongoDB documents streaming replication as reducing staleness, replication lag, and some write latency associated with write concerns that wait for replication. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

Conceptually:

```text
PRIMARY
    │
    │  entry 1
    │  entry 2
    │  entry 3
    │  entry 4
    │  entry 5
    ▼
SECONDARY
```

rather than:

```text
wait
 ↓
batch
 ↓
wait
 ↓
batch
```

---

# 17. Replication lag

Suppose primary has processed:

```text
operation at 10:00:20
```

and secondary has only applied through:

```text
10:00:17
```

Then the secondary is behind.

This is **replication lag**.

MongoDB defines replication lag as the delay between operations on the primary and their corresponding application on a secondary. ([mongodb.com](https://www.mongodb.com/docs/manual/replication/?utm_source=chatgpt.com))

---

# 18. Why lag happens

Common causes include:

```text
network problems
secondary CPU pressure
slow disk
secondary overloaded by reads
primary generating writes faster than secondary can apply
resource contention
```

MongoDB's performance guidance lists networking, insufficient secondary capacity, and slow operations as common causes. ([mongodb.com](https://www.mongodb.com/docs/manual/administration/performance-tuning/?utm_source=chatgpt.com))

---

# 19. Replication lag is not automatically a disaster

Suppose:

```text
lag = 5 ms
```

That's different from:

```text
lag = 45 minutes
```

Some small replication delay is expected because replication is asynchronous.

MongoDB's monitoring documentation explicitly says some delay may be acceptable, while significant lag creates increasingly serious problems. ([mongodb.com](https://www.mongodb.com/docs/v8.0/administration/monitoring/?utm_source=chatgpt.com))

The critical question is:

> **Does the application tolerate the amount of staleness currently observed?**

---

# 20. What happens if lag gets very large?

Suppose:

```text
oplog window = 6 hours
```

and:

```text
secondary lag = 7 hours
```

The secondary can no longer use the existing oplog history to fully catch up.

It needs initial synchronization.

MongoDB explicitly identifies this as one of the consequences of lag exceeding the oplog window. ([mongodb.com](https://www.mongodb.com/docs/v8.0/administration/monitoring/?utm_source=chatgpt.com))

So:

```text
replication lag
      ↓
oplog window
      ↓
critical relationship
```

---

# 21. How do members know each other's state?

Replica set members send **heartbeats**.

MongoDB documents heartbeat communication as part of replica-set operation and uses it to monitor member availability and state. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

Conceptually:

```text
Primary ───── heartbeat ───→ Secondary
Primary ←──── heartbeat ──── Secondary
Secondary ←── heartbeat ───→ Secondary
```

The members use this information to determine things like:

```text
Who is reachable?
What state is each member in?
Is the primary still available?
```

---

# 22. Election

Now the exciting part.

Suppose you have:

```text
             PRIMARY
             /     \
            /       \
      Secondary   Secondary
```

and the primary crashes.

```text
              X
           PRIMARY
```

The remaining eligible members detect that it is unavailable.

An election occurs.

One secondary becomes:

```text
PRIMARY
```

MongoDB's replication documentation says that when the primary does not communicate with the other members for longer than the configured `electionTimeoutMillis`—10 seconds by default in the documented configuration—an eligible secondary calls for an election. ([mongodb.com](https://www.mongodb.com/docs/manual/replication/?utm_source=chatgpt.com))

---

# 23. Is failover instantaneous?

No.

There is a period during which:

```text
old primary unavailable
        ↓
members detect failure
        ↓
election
        ↓
new primary
        ↓
clients reconnect / retry
```

During this window, some writes may fail or be temporarily unavailable.

This is why a production application needs a driver that understands replica-set server selection and failover behavior.

MongoDB drivers implement server-selection logic and can re-evaluate suitable members for operations. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference-mechanics/?utm_source=chatgpt.com))

---

# 24. Why do we care about an odd number of members?

Suppose:

```text
3 voting members
```

Majority:

```text
2
```

You can tolerate:

```text
1 failure
```

and still have a majority.

MongoDB documents:

```text
3 members → majority 2 → fault tolerance 1
5 members → majority 3 → fault tolerance 2
```

for typical voting configurations. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-architectures/?utm_source=chatgpt.com))

This is why three voting data-bearing nodes are a very common architectural baseline.

---

# 25. The formula

For voting members:

```text
majority = floor(votes / 2) + 1
```

For 3:

```text
floor(3/2) + 1
= 1 + 1
= 2
```

For 5:

```text
floor(5/2) + 1
= 2 + 1
= 3
```

This is the majority needed for election decisions.

---

# 26. Why two nodes are awkward

Suppose:

```text
Node A
Node B
```

Majority:

```text
2
```

If A dies:

```text
B = only remaining vote
```

B doesn't have a majority.

So it cannot safely elect itself primary under the normal majority rules.

You can end up with:

```text
database alive
but no writable primary
```

This illustrates why simply saying:

> “Let's use two copies for redundancy.”

doesn't solve the election/quorum problem.

---

# 27. Arbiter

An arbiter is a special member that:

```text
votes in elections
but
does not store a copy of the data
```

MongoDB documents this distinction explicitly. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-members/?utm_source=chatgpt.com))

You might see:

```text
Primary
Secondary
Arbiter
```

The arbiter gives you:

```text
3 votes
```

but only:

```text
2 data-bearing nodes
```

---

# 28. Why arbiters are not a magic solution

An arbiter doesn't store data.

So:

```text
A → data
B → data
C → vote only
```

If B fails:

```text
A + C
```

can maintain voting majority.

But you still only have:

```text
one surviving data-bearing copy
```

MongoDB's current architecture guidance generally favors an odd number of data-bearing members over relying on arbiters where possible; it specifically warns about certain PSA configurations in sharded clusters. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-members/?utm_source=chatgpt.com))

For a production architecture, don't automatically reach for an arbiter to “make three.”

---

# 29. Election priority

Replica-set members can have different:

```text
priority
```

A higher priority makes a member more likely to become primary.

MongoDB documents `priority` as a relative likelihood for becoming primary. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/replica-configuration/?utm_source=chatgpt.com))

For example:

```text
Node A → priority 10
Node B → priority 5
Node C → priority 1
```

This can influence future elections.

You might use that to prefer:

```text
stronger machine
same-region machine
preferred data center
```

as primary.

---

# 30. Priority 0

A:

```text
priority: 0
```

member cannot become primary.

MongoDB documents priority-0 members as normal secondaries for data replication, but they cannot become primary or trigger elections. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-priority-0-member/?utm_source=chatgpt.com))

This can be useful for:

```text
disaster-recovery node
remote-region node
special reporting member
```

where you don't want it taking over primary responsibilities.

---

# 31. Hidden members

A hidden member doesn't appear as a normal read target through read preference.

MongoDB documents hidden members as members hidden from `hello`, preventing normal read routing to them via secondary read preference. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/replica-configuration/?utm_source=chatgpt.com))

Why use one?

For things like:

```text
backup
analytics
special reporting
```

while keeping it out of normal application routing.

---

# 32. Delayed members

A delayed member intentionally stays behind the primary by a configured delay.

For example:

```text
Primary
   ↓
normal secondary
   ↓
delayed secondary
```

The delayed member might represent data from:

```text
one hour ago
```

MongoDB documents delayed members as maintaining a copy that reflects the data set at some time in the past. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/replica-configuration/?utm_source=chatgpt.com))

This can be useful for certain recovery scenarios.

But don't treat it like a normal high-availability replacement.

---

# 33. Now: write concern

You've already learned this in Lesson 9.

Replication makes it much easier to understand.

Suppose:

```text
Primary
Secondary
Secondary
```

and you write:

```text
insert message
```

How long should MongoDB wait before telling your application:

```text
"success"
```

That's where:

```text
write concern
```

comes in.

MongoDB's write concern controls the level of acknowledgment requested for writes. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

---

# 34. `w: 1`

Conceptually:

```text
write to primary
     ↓
primary acknowledges
```

```javascript
{ w: 1 }
```

This doesn't require the write to be acknowledged by a majority of voting/data-bearing nodes.

So:

```text
fast acknowledgment
+
less replication assurance
```

The exact durability semantics also depend on journaling/write-concern configuration.

---

# 35. `w: "majority"`

This is more interesting.

Suppose:

```text
3 voting data-bearing members
```

Majority:

```text
2
```

In current MongoDB 8.0 behavior, a `{ w: "majority" }` write is acknowledged after a majority of data-bearing members have **durably written the oplog entry** for the change. The secondaries then asynchronously apply the operation to their data sets. This changed from earlier behavior, where acknowledgment waited for majority application. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

That distinction is extremely important.

---

# 36. This MongoDB 8.0 change is subtle

Suppose:

```text
Primary
Secondary A
Secondary B
```

You issue:

```text
write with w:"majority"
```

MongoDB 8.0 can acknowledge after:

```text
Primary oplog → durable
Secondary A oplog → durable
```

even though:

```text
Secondary A data set
```

hasn't necessarily applied the operation yet.

So:

```text
majority write acknowledged
        ≠
every secondary's collection has immediately applied the write
```

MongoDB explicitly documents this in the 8.0 release notes and write-concern documentation. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

This is a very important modern MongoDB detail.

---

# 37. So what happens if you immediately read from a secondary?

Imagine:

```text
write
  ↓
w:"majority"
  ↓
ACK
  ↓
immediately read from secondary
```

The secondary could still return data that doesn't contain the newly written document because it hasn't applied the oplog entry yet.

MongoDB explicitly warns about this behavior in 8.0. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

This is where read preference and causal consistency become important.

---

# 38. Read preference

Read preference answers:

> **Which replica-set member should handle my read?**

MongoDB's default is:

```text
primary
```

The available modes are:

```text
primary
primaryPreferred
secondary
secondaryPreferred
nearest
```

MongoDB documents these read-preference modes and their routing behavior. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 39. `primary`

Default:

```text
read → primary
```

So:

```text
Application
     ↓
Primary
```

This gives you the simplest consistency model because you're normally reading from the node that receives writes.

If the primary is unavailable:

```text
read fails / waits for server selection
```

rather than automatically reading from a secondary.

MongoDB documents this as the default read-preference mode. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 40. `primaryPreferred`

Means:

```text
prefer primary
but
if primary unavailable → secondary
```

This increases availability at the cost of potentially stale data.

MongoDB explicitly documents that `primaryPreferred` may return stale secondary data during periods when reads are routed there. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 41. `secondary`

Means:

```text
read only from secondaries
```

Why would you want this?

Possibly:

```text
analytics
reporting
ETL
special read workloads
```

But there is a cost:

```text
secondary data may be stale
```

MongoDB explicitly warns that non-primary read preferences can return stale data due to asynchronous replication. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 42. `secondaryPreferred`

Means:

```text
prefer secondary
but
fall back to primary
```

Useful when you want to offload reads in normal circumstances but preserve availability if no secondary is suitable.

Again:

```text
stale data possible
```

MongoDB documents this behavior directly. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 43. `nearest`

This one means roughly:

> Choose an eligible member based primarily on network latency, whether it is primary or secondary.

MongoDB's documentation says `nearest` can choose among eligible primaries and secondaries based on the configured latency window. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

This can be useful geographically.

Example:

```text
Application in Mumbai
   ↓
nearest MongoDB member
```

But:

> nearest does not mean freshest.

MongoDB explicitly warns that `nearest` does not guarantee consistency and doesn't account for CPU/I/O load when choosing the nearest member. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference-use-cases/?utm_source=chatgpt.com))

---

# 44. Read preference vs read concern

Another critical distinction.

### Read preference

```text
WHERE should I read?
```

### Read concern

```text
WHAT consistency level should the read require?
```

For example:

```text
secondary + local
```

means:

```text
read from secondary
+
use local visibility
```

whereas:

```text
primary + majority
```

means:

```text
read from primary
+
require majority-committed data
```

These solve different problems.

MongoDB documents read preference and read concern as separate mechanisms. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 45. `readConcern: "majority"`

MongoDB's `"majority"` read concern guarantees that a non-transactional read returns data acknowledged by a majority of replica-set members, and that data is durable and won't roll back under the documented conditions. ([mongodb.com](https://www.mongodb.com/docs/manual/reference/read-concern-majority/?utm_source=chatgpt.com))

Mental model:

```text
local
    ↓
"What this node currently sees"

majority
    ↓
"What is majority-committed"
```

The actual semantics depend on deployment/context, but this is the correct intuition.

---

# 46. The weird-sounding “majority commit point”

You may hear:

```text
majority commit point
```

This means roughly:

> the point in the replication history known to be durably replicated to the required majority.

It's important for:

```text
majority reads
rollbacks
transaction visibility
replication state
```

MongoDB uses this concept throughout replica-set consistency behavior. Its documentation describes majority-commit behavior in the context of write concern and read concern. ([mongodb.com](https://www.mongodb.com/docs/manual/reference/read-concern-majority/?utm_source=chatgpt.com))

---

# 47. Why majority matters during failure

Imagine:

```text
Primary     → has write W
Secondary A → has write W
Secondary B → doesn't
```

With 3 voting members:

```text
2/3
```

The write has reached majority.

Now the primary crashes.

Secondary A can potentially become the new primary while retaining W.

That's fundamentally different from:

```text
Primary     → has W
Secondary A → doesn't
Secondary B → doesn't
```

If the primary dies before the write is replicated elsewhere, that write may be lost/rolled back during failover under weaker write concerns.

MongoDB documents that rollbacks occur for writes accepted by an old primary that weren't successfully replicated before the failover. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-rollbacks/?utm_source=chatgpt.com))

---

# 48. Rollback

This is a frightening word, but the mechanism is logical.

Suppose:

```text
Primary
  ↓
write W
```

but:

```text
secondaries haven't replicated W
```

Then:

```text
PRIMARY CRASH
```

and another member becomes:

```text
NEW PRIMARY
```

The old primary eventually rejoins as a secondary.

It discovers:

```text
"I have data that doesn't belong to the new primary's history."
```

Those writes can be rolled back.

MongoDB describes rollback as reverting writes performed on the former primary that weren't successfully replicated before failover. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-rollbacks/?utm_source=chatgpt.com))

MongoDB attempts to keep rollbacks rare.

---

# 49. Example rollback

Initial:

```text
Primary A
Secondary B
Secondary C
```

Primary gets:

```text
Order #500
```

but only A has it.

Then:

```text
A crashes
```

B becomes primary.

B's state:

```text
Order #500 doesn't exist
```

Eventually A returns.

A has:

```text
Order #500
```

but the replica set's accepted history doesn't.

So A must discard/rollback that divergent write before becoming a normal secondary.

That's the essence of rollback.

---

# 50. Why `w: "majority"` helps

If:

```text
write W
```

is acknowledged with:

```text
w: "majority"
```

then W is considered majority-acknowledged according to the documented MongoDB 8.0 write-concern semantics.

This greatly reduces the possibility that failover causes that acknowledged write to be lost.

MongoDB's documentation notes that writes replicated to another member and accessible to a majority do not require rollback during the corresponding failover scenario. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-rollbacks/?utm_source=chatgpt.com))

---

# 51. But majority doesn't mean “all nodes”

Very important.

With:

```text
5 voting members
```

majority:

```text
3
```

So:

```text
3 → enough for majority
2 → may still lag
```

A majority write does not require every replica to be up-to-date.

It requires the configured majority condition.

MongoDB's write-concern documentation explicitly describes majority in terms of the majority of voting/data-bearing members. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

---

# 52. What if a secondary is far away?

Imagine:

```text
Primary → Mumbai
Secondary → Singapore
Secondary → London
```

Network latency differs.

You might configure:

```text
tags
readPreference
write concern
```

to shape where reads occur and which members contribute to certain operations.

MongoDB supports tags and latency-based server selection for replica sets. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference-tags/?utm_source=chatgpt.com))

This becomes important for geographically distributed systems.

---

# 53. Why you shouldn't automatically read from secondaries

A common beginner architecture is:

```text
Primary → writes
Secondaries → all reads
```

because it sounds like free read scaling.

But MongoDB's documentation explicitly cautions against using secondary read preferences simply as a general read-scaling strategy. The write workload is also replicated to the secondaries, secondaries can be stale, and distributing reads changes availability characteristics. MongoDB recommends considering sharding when genuine read/write capacity scaling is needed. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference-use-cases/?utm_source=chatgpt.com))

So replication's primary roles are:

```text
redundancy
high availability
failover
```

not:

```text
infinite read scaling
```

---

# 54. Replication + your ChatApp

Let's apply everything.

Suppose:

```text
ChatApp
   ↓
MongoDB replica set
```

You send:

```text
Message #9001
```

Flow:

```text
Node.js
  ↓
Primary
  ↓
insert message
  ↓
oplog entry
  ↓
Secondary A
  ↓
Secondary B
```

If you're using:

```text
readPreference = primary
```

your next message history request goes to:

```text
Primary
```

so you don't normally have the secondary-staleness issue.

That is why the default primary read preference is often perfectly sensible for an ordinary transactional application. MongoDB documents `primary` as the default mode. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 55. ChatApp failure scenario

Imagine:

```text
Primary
   ↓
Message #9001
```

then primary dies before the write has replicated sufficiently.

If the client used:

```text
w: 1
```

the application may already have received success.

During failover, that write could potentially be rolled back.

This is a fundamental distributed-systems trade-off:

```text
faster acknowledgment
       ↕
stronger replication guarantee
```

MongoDB's rollback and write-concern documentation explains this relationship. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-rollbacks/?utm_source=chatgpt.com))

---

# 56. JobForge example

Suppose a candidate submits an application.

Your backend writes:

```text
Application A
```

with:

```text
w: "majority"
```

This gives stronger durability/replication assurance than simply waiting for the primary, though in MongoDB 8.0 the acknowledgement is tied specifically to durable oplog replication to a majority—not necessarily application to the secondary's collection yet. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

If you immediately read from the primary:

```text
primary
   ↓
application exists
```

If you instead route the read to a secondary:

```text
secondary
   ↓
application may not yet be visible there
```

unless you use appropriate causal consistency/read semantics.

This is one of those subtle bugs that can make distributed systems feel “random” when they're actually following precise rules.

---

# 57. Causal consistency

Suppose your application does:

```text
WRITE A
then
READ A
```

and you need the read to see the write.

This is a **causal relationship**:

```text
write happened before read
```

MongoDB supports causally consistent sessions.

Its documentation explains that causally consistent sessions can provide guarantees such as read-your-own-writes and monotonic reads when used with appropriate read/write concerns. ([mongodb.com](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/?utm_source=chatgpt.com))

---

# 58. Why sessions matter

Suppose:

```text
Session S1
    ↓
write A

Session S2
    ↓
read
```

The system does not have the same causal context.

But:

```text
Session S1
    ↓
write A
    ↓
read A
```

can carry causal ordering information.

MongoDB explicitly states that causal consistency guarantees apply within the same session. ([mongodb.com](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/?utm_source=chatgpt.com))

This connects directly back to Lesson 9.

---

# 59. “Read your own writes”

This phrase is worth remembering.

Suppose user submits:

```text
Job application
```

and immediately refreshes.

They expect:

```text
"My application exists."
```

If the write goes to the primary, and the read goes to a lagging secondary, you can get:

```text
POST /applications → success
GET /applications  → doesn't show it yet
```

That can look like a bug.

But it can simply be:

```text
asynchronous replication
+
secondary read
```

MongoDB's read-preference documentation explicitly warns that secondary reads can return stale data. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

---

# 60. How to avoid this

For simple applications:

```text
use primary reads
```

For more advanced distributed behavior:

```text
causally consistent session
+
appropriate read/write concerns
```

The correct solution depends on the application's consistency requirements.

Don't automatically make every read:

```text
readPreference = primary
```

forever without understanding why.

But don't route everything to secondaries simply because they exist.

---

# 61. Monitoring a replica set

MongoDB provides:

```javascript
rs.status()
```

This is one of the commands you should remember.

MongoDB's monitoring documentation specifically recommends `rs.status()` / `replSetGetStatus` for viewing replica-set state. ([mongodb.com](https://www.mongodb.com/docs/v8.0/administration/monitoring/?utm_source=chatgpt.com))

You can inspect things such as:

```text
primary/secondary states
optime
health
replication progress
election information
```

---

# 62. `rs.printReplicationInfo()`

For oplog information:

```javascript
rs.printReplicationInfo()
```

MongoDB documents this helper for viewing oplog size and time-range information. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

This helps answer:

```text
How large is my oplog window?
```

That's an important operational question.

---

# 63. `rs.printSecondaryReplicationInfo()`

You can also inspect secondaries' replication state using:

```javascript
rs.printSecondaryReplicationInfo()
```

This can help you reason about:

```text
secondary lag
```

The exact helper output depends on your MongoDB shell/version, so use it as an operational convenience rather than memorizing its output format.

---

# 64. What should you monitor?

At minimum:

```text
primary availability
replication lag
oplog window
election frequency
member health
network
disk/CPU/memory
```

MongoDB's production monitoring guidance explicitly emphasizes replication lag and member health. ([mongodb.com](https://www.mongodb.com/docs/v8.0/administration/monitoring/?utm_source=chatgpt.com))

---

# 65. Flow control

You saw this briefly in Lesson 10's related material.

MongoDB can use **flow control** to limit how quickly the primary applies writes when replica members are lagging, with the goal of keeping majority-committed lag near a target.

MongoDB's current replication documentation states that flow control is enabled by default and uses tickets to limit primary write throughput as replication lag approaches the configured target. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

Conceptually:

```text
Secondary falling behind
       ↓
lag increases
       ↓
primary is slowed somewhat
       ↓
secondary catches up
```

This is a fascinating example of a database deliberately reducing its own throughput to maintain a healthier replication state.

---

# 66. Why would slowing the primary help?

Suppose:

```text
Primary generates:
100,000 writes/sec

Secondary can apply:
70,000 writes/sec
```

Every second:

```text
lag grows by ~30,000 operations
```

Eventually:

```text
oplog window threatened
```

Flow control attempts to avoid letting replication fall unboundedly behind.

This doesn't magically solve an undersized system, but it can prevent replication lag from spiraling.

MongoDB documents flow control as a mechanism for maintaining majority-committed lag within a configurable target. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

---

# 67. A complete failure scenario

Let's walk through one carefully.

Initial:

```text
                 PRIMARY
                /       \
               /         \
         SECONDARY A   SECONDARY B
```

Suppose:

```text
A = fully caught up
B = slightly behind
```

Client sends:

```text
write W
```

Primary:

```text
W applied
```

Oplog:

```text
W
```

A receives:

```text
W
```

B hasn't yet.

Then:

```text
PRIMARY CRASHES
```

An election happens.

A becomes:

```text
NEW PRIMARY
```

Because A had W, the write can remain part of the new primary's history.

B later catches up.

This is the normal high-availability path.

---

# 68. Now the dangerous variant

Initial:

```text
PRIMARY → W
A        → no W
B        → no W
```

Primary crashes before W reaches another suitable member.

A becomes primary.

Its history:

```text
no W
```

When old primary returns:

```text
old primary has W
new primary doesn't
```

That divergent W must be rolled back.

That's why a write acknowledged only by the old primary has weaker failover durability than a majority-acknowledged write. MongoDB documents this rollback scenario. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-rollbacks/?utm_source=chatgpt.com))

---

# 69. The distributed-systems trade-off

Now you should recognize this triangle:

```text
             WRITE
               │
      ┌────────┼─────────┐
      ↓        ↓         ↓
   latency   durability consistency
```

Roughly:

```text
w:1
  ↓
faster acknowledgment
weaker replication guarantee

w:"majority"
  ↓
stronger durability/failover semantics
potentially more latency
```

The exact latency and guarantees depend on your deployment and write/read concerns.

There isn't one universally correct setting.

---

# 70. Replica set is not backup

This is another extremely important lesson.

Suppose you have:

```text
Primary
Secondary
Secondary
```

and someone executes:

```javascript
db.jobs.deleteMany({})
```

That destructive operation is replicated.

So you get:

```text
Primary     → delete
Secondary A → delete
Secondary B → delete
```

Replication protects against:

```text
server failure
```

It does **not automatically protect against**:

```text
logical mistakes
accidental deletion
malicious deletion
application bugs
```

That's why you still need backups.

We'll cover backup/recovery in Lesson 15.

---

# 71. Replica set is also not disaster recovery by itself

Imagine all three replicas are in:

```text
same data center
```

and the entire data center disappears.

You've lost all three.

High availability and disaster recovery are related but different goals.

Replication provides:

```text
redundancy
automatic failover
```

but your geographic architecture and backup strategy determine broader disaster resilience.

---

# 72. One subtle point: secondary reads aren't automatically “eventually consistent” in one simple fixed sense

People sometimes say:

> MongoDB secondaries are eventually consistent.

That's a useful beginner shorthand, but incomplete.

Actual behavior depends on:

```text
replication lag
read preference
read concern
causal consistency
session
deployment
```

MongoDB's docs explicitly distinguish stale-read possibilities from stronger causal/majority guarantees. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

So you should think:

```text
secondary reads
    ↓
potential staleness
```

and then ask:

```text
what consistency guarantees did I configure?
```

---

# 73. Replication source isn't necessarily always primary

Another interesting detail.

Secondaries can replicate from another member, not necessarily directly from the primary.

MongoDB supports **chaining** by default.

The secondary can select an appropriate sync source based on member state, lag, connectivity, and other criteria. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

Conceptually:

```text
Primary
   ↓
Secondary A
   ↓
Secondary B
```

instead of:

```text
Primary ─→ A
Primary ─→ B
```

This can reduce network pressure on the primary in some topologies.

---

# 74. But chaining has trade-offs

Suppose:

```text
Primary
  ↓
A
  ↓
B
```

If A is slow or isolated:

```text
B can become farther behind
```

That's why sync-source selection considers factors such as lag, reachability, and performance. MongoDB documents source selection rules for replica-set members. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-sync/?utm_source=chatgpt.com))

Again:

> Distributed systems are about trade-offs, not magic.

---

# 75. Replica-set topology and application connection

Your Node.js application should generally connect using a replica-set-aware connection string / driver configuration rather than hardcoding one MongoDB server as the only possible server.

The driver performs server selection and can discover the current primary and appropriate members. MongoDB documents server selection as occurring per operation according to read preference and topology information. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference-mechanics/?utm_source=chatgpt.com))

So conceptually:

```text
Node.js
   ↓
MongoDB driver
   ↓
discover topology
   ↓
current primary / eligible secondary
```

rather than:

```text
Node.js
   ↓
192.168.x.x
   ↓
hope it's still primary
```

---

# 76. What happens during primary failover from the driver's perspective?

Roughly:

```text
Application
   ↓
driver thinks:
"Primary = A"

A dies
   ↓
driver topology changes
   ↓
election
   ↓
B becomes primary
   ↓
driver discovers B
   ↓
writes route to B
```

Some in-flight operations can fail during the transition.

Your application's retry behavior and the driver's retryable-write support matter.

We'll revisit this during the production lesson.

---

# 77. One very important distinction

Don't confuse:

```text
replication
```

with:

```text
sharding
```

### Replication

```text
same data
    ↓
multiple members
```

Purpose:

```text
redundancy
availability
failover
```

### Sharding

```text
different data partitions
    ↓
multiple shards
```

Purpose:

```text
horizontal scale
data distribution
```

We'll study sharding in Lesson 12.

---

# 78. The full architecture now

You should now be able to picture:

```text
                         APPLICATION
                              │
                         MongoDB Driver
                              │
                              ▼
                    ┌──────────────────┐
                    │   Replica Set    │
                    └────────┬─────────┘
                             │
                    ┌────────┼────────┐
                    ▼        ▼        ▼
                 PRIMARY      S        S
                    │
                    │ write
                    ▼
                  oplog
                    │
              ┌─────┴─────┐
              ▼           ▼
           Secondary   Secondary
              │           │
              ▼           ▼
             apply       apply
```

And under every `mongod`:

```text
mongod
  ↓
WiredTiger
  ├── cache
  ├── indexes
  ├── journal
  ├── checkpoint
  └── disk
```

Now the relationship between Lessons 10 and 11 is much clearer.

---

# 79. The most important distinctions

I want these to become automatic.

### Journal

```text
storage-engine durability/recovery
```

### Oplog

```text
replica-set replication history
```

### Primary

```text
normal write target
```

### Secondary

```text
replicated copy
```

### Replication lag

```text
secondary behind primary
```

### Election

```text
choose a new primary after failure
```

### Write concern

```text
how much write acknowledgment/replication assurance?
```

### Read preference

```text
which member should serve the read?
```

### Read concern

```text
what consistency/visibility level?
```

### Backup

```text
protection against data loss/logical disasters
```

That's the conceptual backbone of MongoDB replication.

---

# 80. Exercises

## Exercise 1 — Basic

Which mechanism carries changes from a MongoDB primary to secondaries?

**A.** WiredTiger cache  
**B.** Oplog  
**C.** MongoDB log  
**D.** `_id` index

### Answer

**B — Oplog.**

Secondaries copy and apply operations from the oplog asynchronously. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

---

# 81. Exercise 2

Which statement is correct?

**A.** A secondary always has exactly the same data as the primary at every instant.

**B.** Secondaries replicate asynchronously, so they can temporarily lag behind.

**C.** Secondaries receive writes directly from clients by default.

**D.** The oplog exists only on the primary.

### Answer

**B.**

MongoDB's secondaries asynchronously replicate oplog operations and can therefore be behind. Each replica-set member has its own oplog. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

---

# 82. Exercise 3 — Important

You have:

```text
3 voting members
```

How many votes constitute a majority?

### Answer

```text
2
```

Because:

```text
floor(3/2) + 1 = 2
```

MongoDB's documented architecture example gives 3 members → majority 2 → fault tolerance 1. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-architectures/?utm_source=chatgpt.com))

---

# 83. Exercise 4 — Difficult

Suppose:

```text
Primary    → write W
Secondary A → write W
Secondary B → no W
```

Then the primary crashes.

Why is the situation safer for W than if only the primary had W?

### Answer

Because W has reached another voting/data-bearing member and can be part of the majority-confirmed history, depending on the write concern and member state.

An acknowledged majority write is specifically designed to provide stronger protection against losing the write during primary failure. MongoDB's rollback documentation notes that writes replicated to another member and available to the majority don't need to be rolled back in the corresponding failover scenario. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-rollbacks/?utm_source=chatgpt.com))

---

# 84. Exercise 5 — MongoDB 8.0 subtlety

Suppose a 3-member replica set receives:

```text
w: "majority"
```

and returns success.

Can you conclude that **every secondary has already applied the write to its collection**?

### Answer

**No.**

In MongoDB 8.0, majority write acknowledgment occurs after a majority of data-bearing members have durably written the oplog entry. Those members then asynchronously apply the operation to their data sets.

Therefore an immediate read from a secondary may not yet show the write. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

This is one of the most important modern MongoDB details from today's lesson.

---

# 85. Exercise 6 — Real backend scenario

Your API does:

```text
POST /applications
```

The write succeeds.

Then your frontend immediately does:

```text
GET /applications
```

But the second request uses:

```text
readPreference = secondary
```

and doesn't see the newly created application.

Is MongoDB necessarily broken?

### Answer

No.

A likely explanation is:

```text
POST
 ↓
primary accepts write
 ↓
replication asynchronous
 ↓
GET routed to lagging secondary
 ↓
new application isn't applied there yet
```

MongoDB explicitly warns that secondary reads can return stale data. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/read-preference/?utm_source=chatgpt.com))

Possible solutions depend on your requirements:

```text
read from primary
```

or use an appropriate causally consistent session/read-write concern configuration when secondary reads are necessary. ([mongodb.com](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/?utm_source=chatgpt.com))

---

# 86. Exercise 7 — Very important

True or false:

> A replica set is a replacement for backups.

### Answer

**False.**

Replication copies logical changes.

If an application accidentally executes:

```javascript
db.users.deleteMany({})
```

that deletion can replicate to the other members.

Backups solve a different problem: recovering from logical mistakes, corruption, ransomware, catastrophic infrastructure loss, and other scenarios where all live replicas may contain the bad state.

---

# 87. Exercise 8 — Advanced

Your replica set has:

```text
oplog window = 4 hours
```

A secondary is offline for:

```text
7 hours
```

Can it necessarily catch up simply by replaying its missing oplog entries?

### Answer

No.

Some of the operations it needs may already have fallen out of the oplog.

MongoDB requires the secondary to be able to access the relevant oplog history; if its replication gap exceeds the oplog window, it must perform initial sync. ([mongodb.com](https://www.mongodb.com/docs/v8.0/core/replica-set-oplog/?utm_source=chatgpt.com))

---

# 88. The final mental model

Here is the entire lesson condensed:

```text
                     REPLICA SET
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       PRIMARY       SECONDARY       SECONDARY
          │              ▲              ▲
          │              │              │
          └── oplog ─────┴──────────────┘
                asynchronous
                  replication
                         │
                         ▼
                 replication lag
                         │
                 ┌───────┴───────┐
                 ▼               ▼
             healthy          too much lag
                 │               │
                 │          oplog window
                 │               │
                 │          exceeded
                 │               ▼
                 │          initial sync
                 │
                 ▼
              failure?
                 │
                 ▼
              election
                 │
                 ▼
            new primary
```

And the consistency side:

```text
                 APPLICATION
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
     WRITE CONCERN           READ PREFERENCE
          │                       │
          ▼                       ▼
  How much acknowledgment?   Which member?
                                  │
                                  ▼
                            READ CONCERN
                                  │
                                  ▼
                         What consistency?
```

---

# 89. If you remember only 10 things

```text
1. Replica set = multiple MongoDB members maintaining the same data.

2. Primary normally receives writes.

3. Secondaries replicate asynchronously.

4. Oplog is the replication history.

5. Journal and oplog are different things.

6. Secondaries can lag.

7. If lag exceeds the oplog window, initial sync may be required.

8. Primary failure triggers an election for a new primary.

9. Read preference decides where reads go; non-primary reads can be stale.

10. Replication is not backup.
```

The two details I especially want you to remember from the modern MongoDB 8.0 architecture are:

```text
w: "majority"
      ≠
all secondaries have already applied the operation
```

and:

```text
oplog
      ≠
journal
```

MongoDB 8.0 changed the majority-write acknowledgment behavior specifically in this area, so older tutorials can give you an outdated mental model. ([mongodb.com](https://www.mongodb.com/docs/v8.0/reference/write-concern/?utm_source=chatgpt.com))

---

## Where we are now

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
11. Replication & High Availability ✅
```

### Next: Lesson 12 — Sharding & Horizontal Scaling

This will answer a fundamentally different problem:

> **What happens when one MongoDB server—or even one replica set—is no longer enough?**

We'll build the architecture from scratch:

```text
                    Application
                         │
                       mongos
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
       Shard 1        Shard 2        Shard 3
       replica set    replica set    replica set
```

Then we'll go deep into **shard keys, range vs hashed sharding, chunks, balancing, `mongos`, config servers, targeted vs scatter-gather queries, jumbo chunks, hot shards, resharding, and how bad shard-key design can destroy an otherwise well-designed MongoDB system**.