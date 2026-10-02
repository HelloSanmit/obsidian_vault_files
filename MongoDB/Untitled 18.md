
# Lesson 15 — Part 2: Production Failure Lab

Now we're going to do the most valuable part of Lesson 15:

> **A production problem happens. What do you investigate, in what order, and why?**

The goal is not memorizing commands.

The goal is developing the instinct:

```text
Symptom
  ↓
Measure
  ↓
Narrow the cause
  ↓
Verify with evidence
  ↓
Fix the root cause
  ↓
Measure again
```

---

# Incident 1 — “MongoDB suddenly became slow”

Imagine JobForge normally has:

```text
GET /jobs
≈ 40 ms
```

Suddenly:

```text
GET /jobs
≈ 2,000 ms
```

Developers say:

> “MongoDB is slow.”

That's not enough information.

Start with the request itself.

```text
Request
 ↓
Controller
 ↓
Service
 ↓
Mongo query
```

Measure each section.

For example:

```js
console.time("request-db");

const jobs = await Job
  .find({
    status: "OPEN"
  })
  .lean()
  .exec();

console.timeEnd("request-db");
```

Suppose you discover:

```text
MongoDB query = 1,900 ms
```

Now the database is actually implicated.

---

# 2. Inspect the query plan

The next question:

> What is MongoDB actually doing?

You investigate the query plan with `explain()`.

You might discover something conceptually like:

```text
COLLSCAN
```

instead of:

```text
IXSCAN
```

### Simple meaning

```text
COLLSCAN
=
scan collection

IXSCAN
=
use an index
```

If you have 20 million jobs and MongoDB is scanning a huge portion of them to return 20 results, that's a strong clue.

---

# 3. Look at the numbers

Suppose the result says roughly:

```text
nReturned = 20
totalDocsExamined = 4,000,000
```

That's suspicious.

You're asking for:

```text
20 documents
```

but examining:

```text
4 million
```

Now suppose another plan gives:

```text
nReturned = 20
totalDocsExamined = 22
```

That's dramatically different.

The important lesson:

> **Don't judge a query from how short the Mongoose code looks.**

---

# 4. Fixing the incident

Suppose the actual workload is:

```js
Job.find({
  companyId,
  status: "OPEN"
})
.sort({
  createdAt: -1
})
.limit(20);
```

Your investigation may lead to an index designed around that exact query shape.

Then:

```text
before
→ collection scan / excessive document examination

after
→ selective index access
```

But don't stop there.

Measure again.

```text
Before:
1,900 ms

After:
35 ms
```

Now you have evidence that the change worked.

---

# 5. Don't blindly add an index

Suppose someone says:

> “Just index `status`.”

Maybe.

But if:

```text
95% of jobs = OPEN
```

then `status` alone may not be very selective.

Your actual query also uses:

```text
companyId
createdAt
```

Therefore the proper index should be driven by the workload.

This is your Lesson 7 knowledge becoming production knowledge.

---

# Incident 2 — “The database isn't slow, but the API is slow”

Suppose:

```text
GET /jobs = 1.2 seconds
```

You measure the DB query:

```text
MongoDB = 30 ms
```

Now don't touch MongoDB.

Investigate the rest:

```text
30 ms
+
processing
+
serialization
+
network
```

Maybe your Node code is doing:

```js
const jobs = await Job.find(...).lean();

for (const job of jobs) {
  // expensive computation
}
```

and that consumes:

```text
950 ms
```

Then:

```text
MongoDB = healthy
Node processing = bottleneck
```

This is why **measurement beats assumptions**.

---

# Incident 3 — “The API works, but duplicate applications appeared”

This is a very realistic JobForge incident.

The code is:

```js
const existing = await Application.findOne({
  candidateId,
  jobId
});

if (!existing) {
  await Application.create({
    candidateId,
    jobId
  });
}
```

Looks correct.

But one candidate has:

```text
3 applications
```

for the same job.

---

# 6. Reproduce the race

Two requests arrive:

```text
Request A
Request B
```

Both execute:

```text
findOne()
```

Both get:

```text
null
```

Then:

```text
A → insert
B → insert
```

You have a race condition.

---

# 7. Proper fix

Add:

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

Now:

```text
A → success
B → duplicate-key error
```

Your service can then translate that into:

```text
409 Conflict
```

This is a beautiful example of:

```text
application logic
+
database constraint
```

working together.

---

# Incident 4 — “Unread count is wrong”

Suppose the application does:

```js
const user = await User.findById(userId);

user.unreadCount++;

await user.save();
```

Now 50 messages arrive almost simultaneously.

Multiple processes may do:

```text
read count = 10
read count = 10
read count = 10
...
```

and then write:

```text
11
11
11
```

instead of:

```text
60
```

This is a lost-update style problem.

---

# 8. Use atomic update

Instead:

```js
await User.updateOne(
  {
    _id: userId
  },
  {
    $inc: {
      unreadCount: 1
    }
  }
);
```

MongoDB performs the increment atomically.

Now concurrent increments are much safer.

This is why I keep stressing:

> **When the database can perform an atomic operation, don't unnecessarily turn it into read → modify → write application logic.**

---

# Incident 5 — “Primary database suddenly goes down”

Your replica set:

```text
Primary
Secondary
Secondary
```

Then:

```text
Primary
  X
```

What happens?

The remaining eligible members communicate and an election can choose a new primary.

Conceptually:

```text
Primary
   X
   ↓
Election
   ↓
New primary
```

Your application needs to be able to reconnect and discover the appropriate primary through the MongoDB driver's replica-set-aware connection behavior.

---

# 9. What should you investigate?

During the incident, you want to know:

```text
Which member failed?
Are the remaining members healthy?
Was an election completed?
Is there replication lag?
Did clients reconnect?
Are writes succeeding?
```

The goal is not merely:

> “A server is down.”

The real question is:

> **“Did the system maintain the availability guarantees we designed it for?”**

---

# 10. What replication does NOT solve

Suppose an admin runs:

```text
delete all applications
```

The primary executes it.

Replicas receive the deletion.

Now:

```text
Primary     → deleted
Secondary 1 → deleted
Secondary 2 → deleted
```

Replication worked perfectly.

The data is still gone.

That's why:

```text
replication
≠
backup
```

This distinction should now be automatic in your head.

---

# Incident 6 — “Oops, millions of documents were deleted”

This is where your backup strategy matters.

Suppose:

```text
10:00 → good
10:15 → good
10:30 → accidental deletion
10:40 → discovered
```

You don't necessarily want:

> “Restore last night's backup.”

You want the appropriate recovery mechanism that gets you as close as possible to before the incident.

That's where:

```text
backup
+
change history / point-in-time recovery capability
```

become important.

---

# 11. RPO becomes real now

Suppose the organization's RPO is:

```text
15 minutes
```

and the recovery mechanism can restore to a recent point.

Then the technical architecture is built around the business statement:

> “We can tolerate approximately 15 minutes of data loss.”

RPO isn't a database command.

It's a **business/recovery requirement** that influences the database architecture.

---

# Incident 7 — “We have backups, but restore failed”

This is one of the most dangerous situations.

Company says:

> “Our backups are fine.”

Then disaster occurs.

Restore attempt:

```text
FAILED
```

Now you discover:

```text
backup job succeeded
but
restore process was never tested
```

This is why recovery testing matters.

A healthy backup strategy includes:

```text
backup
 ↓
restore test
 ↓
data verification
 ↓
recovery-time measurement
```

---

# 12. Restore verification

After a restore, don't just check:

```text
MongoDB started
```

Check:

```text
collections exist
expected document counts
important records exist
indexes exist
application can connect
queries work
relationships are intact
```

And depending on your system:

```text
authentication
authorization
application functionality
```

should also be tested.

---

# Incident 8 — “MongoDB connections exploded”

Suppose the backend is deployed with:

```text
50 Node instances
```

and each instance has a large connection pool.

You may suddenly see:

```text
MongoDB connections
↑↑↑↑↑
```

Developers might say:

> “MongoDB can't handle our traffic.”

But perhaps the main issue is:

```text
too many application connections
```

rather than query execution.

---

# 13. The math matters

Conceptually:

```text
total possible connection pressure
≈
application instances
×
per-instance pool capacity
```

For example:

```text
20 instances
×
50 connections
=
1000 potential pooled connections
```

This isn't a universal exact formula for every driver/deployment behavior, but it's the right architectural mental model.

When scaling application servers, always reconsider database connection pressure.

---

# 14. What do you do?

Investigate:

```text
number of app instances
pool configuration
active connections
query duration
connection wait time
```

Then determine whether you have:

```text
too many connections
too much query concurrency
slow queries holding connections
```

These are different problems.

---

# Incident 9 — “Queries are fast individually, but production still feels slow”

This is an interesting one.

Imagine:

```text
single query = 10 ms
```

Sounds excellent.

But:

```text
1000 queries
```

may still overwhelm the system.

Especially if your application produces:

```text
N+1 queries
```

Example:

```text
1 query → jobs
1000 queries → companies
```

Even if each company query is fast, you've created unnecessary database traffic.

This is why:

> **Query count matters, not only query latency.**

---

# 15. Detecting N+1

Suppose one request returns:

```text
100 jobs
```

and logs show:

```text
101 MongoDB queries
```

That's suspicious.

You then investigate whether:

```js
for (const job of jobs) {
  await Company.findById(job.companyId);
}
```

is responsible.

Possible improvements:

```text
populate
$lookup
batching
denormalization
query redesign
```

depending on the actual use case.

---

# Incident 10 — “Database memory pressure is high”

Suppose your working set is:

```text
huge
```

while available RAM/cache capacity is:

```text
much smaller
```

You might see more expensive storage access.

Don't immediately conclude:

> “We need more indexes.”

Possible causes include:

```text
huge working set
large documents
too many indexes
inefficient queries
cache churn
```

Sometimes adding indexes can even increase memory pressure because indexes themselves consume resources.

---

# 16. Working set mental model

Think:

```text
Frequently accessed
       ↓
     cache
       ↓
      fast
```

versus:

```text
cold data
   ↓
disk
   ↓
slower
```

If your application's frequently accessed data grows beyond what the system can efficiently keep in memory, latency characteristics may change significantly.

---

# Incident 11 — “Disk is almost full”

Imagine:

```text
disk = 90%
```

Your instinct should be:

> “This is now an operational risk.”

Don't wait for:

```text
100%
```

Investigate:

```text
database data size
index size
logs
backup storage
growth rate
unused files
```

Then estimate:

```text
days until capacity exhaustion
```

based on actual growth.

---

# 17. Why growth rate matters

Suppose:

```text
current free space = 200 GB
daily growth = 20 GB
```

Very roughly:

```text
~10 days
```

before you run out, ignoring changes in growth rate and safety margin.

Now suppose daily growth doubles:

```text
40 GB/day
```

Your available time is dramatically shorter.

That is why **capacity planning** isn't a one-time task.

---

# Incident 12 — “Someone leaked the MongoDB password”

Suppose:

```text
MONGODB_URI
```

was accidentally committed to GitHub.

The response should not be:

```text
delete the GitHub file
```

Treat the credential as compromised.

Conceptually:

```text
leak detected
    ↓
rotate credential
    ↓
revoke old credential
    ↓
deploy new secret
    ↓
review access/logs
    ↓
investigate exposure
```

This is where secret rotation matters.

---

# 18. Least privilege becomes important here

Suppose the leaked credential can:

```text
drop databases
create users
modify cluster configuration
```

The blast radius is much larger.

If the application user only has:

```text
required application DB privileges
```

the damage may be more contained.

Again:

> **Least privilege reduces blast radius.**

---

# Incident 13 — “A user is seeing another company's jobs”

This is a major security incident.

Suppose a multi-tenant application uses:

```text
tenantId
```

A developer writes:

```js
Job.find({
  status: "OPEN"
});
```

instead of:

```js
Job.find({
  tenantId,
  status: "OPEN"
});
```

Now data isolation is broken.

---

# 19. Why this is dangerous

Your MongoDB database may be perfectly healthy.

Your indexes may be excellent.

Your application can still leak data.

The problem is:

```text
authorization / tenant isolation
```

not:

```text
database availability
```

This is why database engineering cannot be separated completely from application security.

---

# 20. Defense in depth for multi-tenancy

A reasonable conceptual model:

```text
Authenticated user
        ↓
Determine tenant
        ↓
Authorization
        ↓
Construct tenant-scoped query
        ↓
MongoDB
```

And ideally the architecture makes it difficult to accidentally omit tenant scope.

This is one place where careful service/repository design can help.

---

# Incident 14 — “A query lets users execute arbitrary filters”

Suppose your API does:

```js
Job.find(req.query);
```

Now the client effectively controls the query object.

That's dangerous.

The client might try to pass unexpected operators or expensive query parameters.

---

# 21. Safer query construction

Instead:

```js
const filter = {};

if (req.query.location) {
  filter.location = req.query.location;
}

if (req.query.status) {
  filter.status = req.query.status;
}
```

Now the server says:

> “These are the only fields I'm willing to turn into database filters.”

This provides:

```text
security
+
predictability
+
performance control
```

---

# Incident 15 — “One endpoint crashes under a huge request”

Suppose:

```http
GET /jobs?limit=10000000
```

Your code does:

```js
.limit(Number(req.query.limit))
```

Now MongoDB is asked for an enormous result.

Or perhaps:

```http
POST /message
```

with a gigantic body.

The database may not even be the first thing that breaks.

You need limits at the API boundary.

---

# 22. Defensive API design

Think:

```text
request size limit
+
maximum page size
+
allowed filters
+
allowed sort fields
+
rate limiting
+
timeout
```

This prevents clients from turning your database into an unrestricted query engine.

---

# Incident 16 — “The database isn't failing, but requests hang”

A request may hang because of:

```text
server selection
connection acquisition
network
database execution
application processing
```

That's why timeouts matter.

Without sensible timeout behavior, the system can accumulate stuck requests.

Imagine:

```text
100 requests waiting
+
100 more
+
100 more
```

Soon your application itself becomes overloaded.

---

# 23. Timeout mental model

Think in layers:

```text
HTTP layer
   ↓
application layer
   ↓
database connection layer
   ↓
database operation
```

Each layer should have behavior appropriate to its role.

You don't want:

```text
request
→ wait forever
```

---

# Incident 17 — “Retries doubled something”

Suppose a client submits:

```text
Create something
```

The network fails after the server may have processed it.

The client retries.

Now:

```text
first request
→ maybe succeeded

retry
→ succeeds again
```

You get:

```text
duplicate side effect
```

That's why retries need idempotency thinking.

---

# 24. Example: idempotent application creation

Suppose:

```text
candidateId = X
jobId = Y
```

and your database has:

```text
unique(candidateId, jobId)
```

Then a retry won't create a second application.

Instead:

```text
first attempt → insert
retry → duplicate-key
```

Your application can interpret the duplicate appropriately.

This is a very strong design pattern.

---

# Incident 18 — “Change stream consumer processed something twice”

Suppose your change-stream worker receives:

```text
Application created
```

It sends:

```text
email
```

Then the worker crashes before recording that the event was processed.

After restart, it may see the event again.

Now:

```text
email sent twice
```

That's why change-stream consumers should be designed with retry/idempotency in mind.

---

# 25. Event processing pattern

A useful conceptual design is:

```text
Change event
     ↓
derive stable event identity
     ↓
check whether already processed
     ↓
perform side effect
     ↓
record processing state
```

The exact implementation depends on your architecture.

But the key principle is:

> **Assume an event may be delivered again.**

---

# Incident 19 — “Transaction took too long”

Suppose:

```text
transaction starts
 ↓
database operation
 ↓
HTTP API call
 ↓
email service
 ↓
another HTTP API
 ↓
commit
```

This is a poor pattern.

Why?

The transaction stays open while external systems are being contacted.

That increases the duration for which transactional resources are held.

---

# 26. Better design

Try to separate:

```text
database transaction
```

from:

```text
external side effects
```

A common conceptual approach:

```text
MongoDB transaction
   ↓
persist state / event
   ↓
commit
   ↓
worker processes external side effects
```

This connects to the event-driven architecture we discussed in Lesson 13.

---

# Incident 20 — “Backup succeeded, but backup storage is full”

A backup system can itself have operational problems.

Suppose:

```text
daily backup = 100 GB
retention = 90 days
```

That can become a significant storage footprint.

So you monitor:

```text
backup success
backup size
retention
storage capacity
restore test results
```

Backups need operations too.

---

# 27. Production incident workflow

Now let's generalize everything.

When an incident occurs:

## Step 1 — Establish the symptom

Example:

```text
API latency increased from 40 ms to 2 seconds
```

Don't just say:

```text
MongoDB is broken
```

---

## Step 2 — Measure the layers

```text
HTTP
Node
Mongoose
MongoDB
network
```

Find where time is being spent.

---

## Step 3 — Determine the failure class

Is it:

```text
performance
availability
correctness
security
capacity
data loss
```

?

These need different responses.

---

## Step 4 — Verify with evidence

Use:

```text
logs
metrics
traces
query plans
database statistics
replication status
connection information
```

---

## Step 5 — Contain the damage

For example:

```text
stop abusive traffic
disable broken feature
rotate compromised credential
pause destructive job
```

---

## Step 6 — Recover

Depending on the incident:

```text
failover
rollback
restore
scale
fix index
restart unhealthy component
rotate credentials
```

---

## Step 7 — Verify

Don't simply say:

> “It's back.”

Check:

```text
data
latency
errors
replication
application behavior
```

---

## Step 8 — Prevent recurrence

Ask:

> Why was the system vulnerable to this incident?

Then improve:

```text
code
indexes
alerts
backups
tests
permissions
architecture
runbooks
```

---

# 28. Incident classification

A useful mental model:

```text
                 INCIDENT
                     |
      ┌──────────────┼──────────────┐
      ↓              ↓              ↓
  Performance   Availability    Correctness
      |              |              |
 slow query      primary down     wrong data
      |
      └──────────────┬──────────────┘
                     ↓
                 Security
                     |
               unauthorized access
```

And:

```text
Data loss
Capacity
Recovery
```

can overlap multiple categories.

---

# 29. Root cause vs symptom

Suppose:

```text
Symptom:
API latency = 2 seconds
```

Possible causes:

```text
missing index
```

But what if the deeper reason is:

```text
new feature introduced a query pattern
```

and nobody added the corresponding index?

Then:

```text
Immediate cause
=
bad query plan

Root cause
=
engineering process failed to account for new workload
```

Strong incident analysis goes beyond the immediate symptom.

---

# 30. One very important production principle

> **Never make a database change during an incident without knowing what problem you're trying to solve.**

For example:

```text
“It's slow, let's add indexes.”
```

is not a diagnosis.

Better:

```text
Observed:
COLLSCAN

Observed:
4 million documents examined for 20 results

Hypothesis:
missing compound index

Action:
add/test candidate index

Measurement:
latency reduced from 1.9s → 40ms
```

Now your change is evidence-driven.

---

# 31. Your MongoDB production toolbox

At this point, you should recognize the purpose of these categories:

### Query diagnosis

```text
explain()
```

### Current activity

```text
currentOp
```

### Server metrics

```text
serverStatus
```

### Database statistics

```text
dbStats
```

### Index information

```text
$indexStats
```

### Replica health

```text
replica-set status tools
```

### Application timing

```js
console.time()
console.timeEnd()
```

### Application logs

```text
request ID
operation
latency
error
```

### Recovery

```text
backup
restore
point-in-time recovery strategy
```

The commands are less important than knowing **which category of tool answers which question**.

---

# 32. What question does each tool answer?

|Tool/concept|Question|
|---|---|
|`explain()`|How does MongoDB execute this query?|
|`serverStatus`|What is the server doing overall?|
|`dbStats`|What is happening at database level?|
|`currentOp`|What is running right now?|
|`$indexStats`|Which indexes are being used?|
|Replica-set status|Are members/elections/health okay?|
|Application logs|What happened in my code?|
|Metrics|How much/how often?|
|Traces|Where did this request spend time?|
|Backup|Can I recover historical data?|
|Restore test|Does recovery actually work?|

That is the important knowledge.

---

# 33. Let's simulate one complete incident

Imagine this alert:

```text
ERROR RATE: 18%
API LATENCY: 3.4 seconds
```

You investigate.

### Observation 1

Node:

```text
CPU = normal
```

### Observation 2

MongoDB:

```text
CPU = high
```

### Observation 3

Slow query analysis:

```text
Job search query suddenly expensive
```

### Observation 4

`explain()`:

```text
large document examination
```

### Observation 5

You discover:

```text
new search filter
```

was deployed today.

### Observation 6

The new filter isn't supported by a suitable index.

Now you have:

```text
application deployment
      ↓
new query shape
      ↓
inefficient query plan
      ↓
MongoDB load
      ↓
API latency
      ↓
error rate
```

That's the chain.

---

# 34. Fix

You don't simply say:

> “Database fixed.”

You:

```text
design candidate index
 ↓
test query plan
 ↓
deploy safely
 ↓
measure latency
 ↓
monitor MongoDB load
```

Suppose:

```text
API latency
3.4s → 70ms

MongoDB CPU
92% → 48%
```

Now you have evidence.

---

# 35. Post-incident learning

Afterward, you might improve the engineering process:

```text
new query introduced
      ↓
required query-plan review
      ↓
performance test
      ↓
index review
      ↓
deployment
```

This is how production systems become more reliable over time.

---

# 36. Your biggest Lesson 15 takeaway

You should now understand that MongoDB production engineering is not:

```text
“Know MongoDB commands.”
```

It is:

```text
          PRODUCTION MONGODB
                 |
      ┌──────────┼──────────┐
      ↓          ↓          ↓
   SECURITY   RELIABILITY PERFORMANCE
      |          |          |
   auth        replica     indexes
   roles       backup      queries
   TLS         recovery    cache
   network     failover    memory
   secrets     RPO/RTO     storage
      \          |          /
       \         |         /
        └── OBSERVABILITY ─┘
             logs
            metrics
            traces
```

And on top:

```text
incident response
capacity planning
disaster recovery
```

---

# 37. New methods/functions from this section

A few additional ones you should now recognize:

|Function / feature|Simple meaning|
|---|---|
|`Model.find()`|Search for matching documents|
|`Model.findById()`|Search by `_id`|
|`Model.findOne()`|Return one matching document|
|`Model.updateOne()`|Update one matching document|
|`Model.updateMany()`|Update many matching documents|
|`Model.deleteOne()`|Delete one matching document|
|`Model.bulkWrite()`|Execute many write operations as a batch|
|`.sort()`|Control result ordering|
|`.limit()`|Restrict maximum number of returned documents|
|`.skip()`|Skip a number of matching results|
|`.select()`|Choose returned fields|
|`.lean()`|Return plain JS objects instead of full Mongoose Documents|
|`.populate()`|Resolve referenced documents through Mongoose|
|`.orFail()`|Throw when nothing is found|
|`.exec()`|Explicitly execute a Mongoose query|
|`ObjectId.isValid()`|Check ObjectId-form validity|
|`console.time()`|Start timing code|
|`console.timeEnd()`|Stop timing code|

Notice that some are **Mongoose methods**, some are **JavaScript built-ins**, and some are **MongoDB administrative concepts**.

I want you to keep those categories separate.

---

# 38. The production skill I want you to leave with

When someone tells you:

> “MongoDB is slow.”

Your response in your own head should be:

```text
What endpoint?
What exact query?
How long?
How many DB operations?
What does explain() say?
What index is being used?
How many docs are examined?
How many results returned?
Is the pool under pressure?
Is memory/cache under pressure?
Is storage slow?
Is this a new query shape?
```

When someone tells you:

> “We lost data.”

Think:

```text
Was it accidental deletion?
Replication issue?
Application bug?
Backup availability?
Point-in-time recovery?
RPO?
Restore verification?
```

When someone tells you:

> “The database is exposed.”

Think:

```text
credentials
roles
network access
TLS
secret rotation
audit/log review
least privilege
```

That is the mindset of someone who can actually operate a database.

---

# Lesson 15 status

You now have the complete conceptual chain:

```text
Security
✓ Authentication
✓ Authorization
✓ Least privilege
✓ TLS
✓ Network restrictions
✓ Secret management
✓ Credential rotation

Reliability
✓ Replication
✓ Failover
✓ Backup
✓ Restore
✓ Point-in-time recovery concepts
✓ RPO
✓ RTO
✓ Disaster recovery

Performance
✓ Explain
✓ Query plans
✓ Indexes
✓ Working set
✓ Memory/cache
✓ Disk
✓ Connection pools
✓ N+1
✓ Projection
✓ Pagination
✓ Capacity planning

Observability
✓ Logs
✓ Metrics
✓ Traces
✓ Profiling
✓ Current operations
✓ Index statistics
✓ Server statistics

Production engineering
✓ Incident diagnosis
✓ Root cause analysis
✓ Idempotency
✓ Concurrency
✓ Data isolation
✓ Rate/query limits
✓ Safe query construction
✓ Recovery testing
```

And now there is only **one lesson left in the original MongoDB curriculum: Lesson 16 — Real Project Database Design.**

That is where we stop learning MongoDB as isolated concepts and design an actual database from scratch for a realistic application, including **entities, relationships, embedding vs referencing, indexes, validation, transactions, pagination, high-growth collections, query patterns, and production trade-offs**.