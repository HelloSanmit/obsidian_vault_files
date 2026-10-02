
# Lesson 50 — How Advanced React Pieces Fit Together

This is the lesson where React stops looking like a collection of APIs.

So far you've learned:

```text
useState
useReducer
useEffect
useRef
useContext
custom hooks
React Router
TanStack Query
mutations
Actions
Suspense
Transitions
SSR
Server Components
Server Functions
external stores
WebSockets
testing
security
deployment
observability
```

The natural question is:

> **When I'm building a real application, which one do I use, where, and how do they interact?**

Let's build one.

---

# 1. Our application

Imagine a project-management application called:

```text
Orbit
```

Users can:

```text
sign in
view projects
open a project
create/edit/delete tasks
filter tasks
comment
receive real-time updates
```

We want:

```text
SSR
Server Components
React Router
TanStack Query
React Actions
Server Functions
optimistic updates
WebSockets
authentication
authorization
testing
```

Not because every app needs all of these.

But because seeing them together will make the boundaries obvious.

---

# 2. Start from the URL

Suppose a user visits:

```text
/projects/42?status=open
```

The first thing to understand is that the URL itself represents application state.

```text
/projects/42
```

means:

```text
resource = project 42
```

and:

```text
?status=open
```

means:

```text
filter = open
```

So our application begins with:

```text
URL
 ↓
Router
```

---

# 3. Router's responsibility

React Router answers questions like:

```text
What route are we on?
What parameters are present?
What nested UI should render?
What route-level data/action is involved?
Should navigation occur?
```

It does **not** own every piece of application state.

For example:

```text
/project/42
```

belongs naturally to routing.

But:

```text
draft comment = "hello"
```

doesn't belong in the router.

And:

```text
cached tasks from the server
```

don't belong in ordinary component state.

That's the first architecture principle:

> **A piece of state should live where its meaning naturally belongs.**

---

# 4. Our route tree

Conceptually:

```text
App
└── /
    ├── login
    └── projects
        ├── index
        └── :projectId
            ├── overview
            ├── tasks
            └── settings
```

A URL:

```text
/projects/42/tasks
```

therefore matches:

```text
projects
  ↓
projectId = 42
  ↓
tasks
```

The nested layout can preserve shared UI:

```text
┌───────────────────────────────────────┐
│ Orbit                 Profile   Logout│
├───────────────┬───────────────────────┤
│ Overview      │                       │
│ Tasks         │    Task content       │
│ Settings      │                       │
└───────────────┴───────────────────────┘
```

Router structure gives us the skeleton.

---

# 5. Now ask: where does project data come from?

There are several possibilities.

We could fetch in:

```text
useEffect
```

We could use:

```text
TanStack Query
```

We could use:

```text
Router loader
```

We could fetch inside:

```text
Server Component
```

Which one is correct?

There is no universal winner.

The answer depends on **what owns the data and how it is used**.

---

# 6. Route-owned data

Suppose we say:

> "To render `/projects/:projectId`, the route must know which project this is."

That makes route-level data a natural fit for the router.

Conceptually:

```text
navigation
   ↓
route match
   ↓
loader
   ↓
project data
   ↓
route render
```

This is especially useful when route rendering itself depends on the data.

---

# 7. Server-state-owned data

Now imagine the project contains:

```text
10,000 tasks
```

with:

```text
filters
pagination
sorting
stale data
background refetching
mutations
cache sharing
```

That's a natural TanStack Query problem.

```text
component
  ↓
useQuery
  ↓
query cache
  ↓
server
```

Now the distinction becomes clearer.

Router answers:

> "Which page/resource are we visiting?"

TanStack Query answers:

> "What server data does the client currently know about?"

---

# 8. They can coexist

This is important.

You don't have to choose:

```text
Router OR TanStack Query
```

You can have:

```text
Router
  ↓
projectId from URL
       ↓
TanStack Query
  ↓
['tasks', projectId, filters]
```

For example:

```js
const { projectId } = useParams();

const { data: tasks } = useQuery({
  queryKey: [
    'tasks',
    projectId,
    filters
  ],
  queryFn: () =>
    fetchTasks(projectId, filters)
});
```

The flow is:

```text
URL
 ↓
projectId
 ↓
query key
 ↓
server data cache
```

That is a very common and powerful architecture.

---

# 9. URL state and server state are different

Suppose the user selects:

```text
status = open
```

You may put it in the URL:

```text
/projects/42/tasks?status=open
```

Then:

```text
URL state
    ↓
query key
    ↓
server state
```

For example:

```js
const [searchParams] = useSearchParams();

const status =
  searchParams.get('status') ?? 'all';

const query = useQuery({
  queryKey: ['tasks', projectId, status],
  queryFn: () =>
    fetchTasks(projectId, { status })
});
```

This is a beautiful chain:

```text
shareable URL
      ↓
filter state
      ↓
cache identity
      ↓
server request
```

---

# 10. Now introduce Server Components

Imagine the project overview contains:

```text
Project name
Owner
Description
Recent activity
Task statistics
```

Some of this can be rendered server-side.

Conceptually:

```text
Server Component
      ↓
database / server API
      ↓
HTML + React server output
      ↓
browser
```

This lets certain data-heavy UI be prepared on the server.

But not everything should become a Server Component.

---

# 11. What stays client-side?

Suppose our task editor needs:

```text
useState
useRef
useOptimistic
event handlers
browser APIs
```

Then:

```text
TaskEditor
```

belongs in the client module graph.

The architecture might therefore be:

```text
Server Component
└── ProjectPage
    ├── ProjectHeader
    ├── ProjectStats
    └── TaskEditor        ← Client Component
```

The server prepares much of the page.

The client handles interaction.

---

# 12. `"use client"` as a boundary

Think of:

```text
"use client"
```

not as:

> "Render this in the browser."

Instead:

> **"This module and the dependencies it pulls into the client graph need to support client execution."**

That distinction matters enormously.

You don't want:

```text
Server Component
 ↓
accidentally import massive client-only tree
 ↓
huge client bundle
```

So component boundaries should be deliberate.

---

# 13. Server Component → Client Component

Suppose:

```jsx
<ProjectHeader
  project={project}
/>
```

where `ProjectHeader` is a Client Component.

The server can provide serializable data:

```js
{
  id: 42,
  name: "Orbit"
}
```

The conceptual flow is:

```text
server data
 ↓
serializable boundary
 ↓
client component
```

That means the client receives what it needs without moving the entire server-side data source into the browser.

---

# 14. But what if the Client Component needs fresh data?

Suppose:

```text
TaskBoard
```

changes frequently.

You might decide:

```text
Server Component
     ↓
initial shell/data
     ↓
Client TaskBoard
     ↓
TanStack Query
```

So the server gives the initial view and the client takes over dynamic server-state synchronization.

This is an important pattern:

> **Server rendering and client caching are complementary, not mutually exclusive.**

---

# 15. Now we need a mutation

User clicks:

```text
Create Task
```

The request crosses from the browser to the server.

There are several ways to implement it:

```text
regular fetch POST
React Action
Router action
Server Function
```

Again, what matters is the ownership boundary.

---

# 16. React Action vs Server Function

These are easy to confuse.

An Action-oriented flow might be:

```text
form
 ↓
useActionState
 ↓
action function
 ↓
async operation
 ↓
state result
```

A Server Function adds a network/server execution boundary:

```text
client
 ↓
Server Function call
 ↓
server
 ↓
database
```

So:

```text
React Action
= UI/action lifecycle abstraction

Server Function
= callable server-side execution boundary
```

They can work together.

---

# 17. A task form

Imagine:

```jsx
function CreateTaskForm() {
  const [state, submit, isPending] =
    useActionState(createTask, initialState);

  return (
    <form action={submit}>
      <input name="title" />

      <button disabled={isPending}>
        {isPending
          ? 'Creating...'
          : 'Create task'}
      </button>

      {state.error && (
        <p>{state.error}</p>
      )}
    </form>
  );
}
```

The browser is responsible for:

```text
input
submit
pending UI
error UI
```

The server is responsible for:

```text
validation
authorization
database mutation
```

That's a clean trust boundary.

---

# 18. Server function

Conceptually:

```js
'use server';

export async function createTask(
  previousState,
  formData
) {
  const title =
    formData.get('title');

  validate(title);

  const user =
    await requireAuthenticatedUser();

  await authorizeCreateTask(user);

  const task =
    await db.task.create({
      data: {
        title
      }
    });

  return {
    success: true,
    task
  };
}
```

The client should never be trusted to decide:

```text
who is acting
whether they have permission
whether input is valid
```

The server decides those.

---

# 19. Now we have a successful mutation

The server returns:

```js
{
  success: true,
  task: {
    id: 91,
    title: "Fix dashboard"
  }
}
```

But our UI has another problem.

Our TanStack Query cache might currently contain:

```text
['tasks', 42, 'open']
```

with 17 tasks.

Creating task 91 means that cache may now be stale.

So the next step is:

```text
mutation
 ↓
cache consistency
```

---

# 20. Action and query cache have different jobs

This is worth memorizing.

React Action:

```text
"What is happening with this submitted operation?"
```

TanStack Query:

```text
"What server data does the client currently have?"
```

Therefore:

```text
submit form
 ↓
Action lifecycle
 ↓
server mutation
 ↓
invalidate/update query cache
```

They complement each other.

---

# 21. Invalidation flow

After creation:

```text
createTask succeeds
       ↓
tasks are potentially stale
       ↓
invalidate ['tasks', 42, ...]
       ↓
query refetches when appropriate
       ↓
UI gets canonical server state
```

Now:

```text
server
  ↓
canonical task list
  ↓
query cache
  ↓
React UI
```

This is a consistency cycle.

---

# 22. Optimistic UI

But waiting for the request can feel slow.

We can make the task appear immediately:

```text
user submits
 ↓
temporary task appears
 ↓
server confirms
 ↓
temporary → canonical
```

That's optimistic state.

So:

```text
React UI
 ↓
optimistic overlay
 ↓
TanStack Query canonical state
```

This is an important distinction:

```text
optimistic state
≠
server truth
```

---

# 23. What if the request fails?

Then:

```text
optimistic task
      ↓
server rejects
      ↓
remove temporary task
      ↓
show error
```

Or perhaps:

```text
temporary task
      ↓
failed
      ↓
mark task as failed
      ↓
"Retry"
```

The exact UX depends on the operation.

Reliability means deciding this explicitly.

---

# 24. Now add real-time updates

Suppose another user creates a task.

Your user is currently looking at:

```text
/project/42/tasks
```

They didn't click anything.

But the server emits:

```text
task.created
```

over WebSocket.

Now the flow is:

```text
WebSocket
   ↓
real-time event
   ↓
client state/cache update
   ↓
React render
```

This is where the external-store lesson becomes valuable.

---

# 25. Don't create ten WebSockets

Imagine:

```text
TaskList → socket
ActivityFeed → socket
NotificationBell → socket
Header → socket
ProjectStats → socket
```

You now have:

```text
5 components
5 socket connections
```

That's usually unnecessary.

Instead:

```text
                         Server
                           │
                       WebSocket
                           │
                           ▼
                   shared connection
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
         TaskList      ActivityFeed   Notifications
```

One connection can feed shared application state.

---

# 26. External store for real-time state

Suppose the socket manager owns:

```js
{
  connected: true,
  events: [...]
}
```

React can subscribe using:

```text
useSyncExternalStore
```

The flow becomes:

```text
WebSocket
   ↓
external store
   ↓
useSyncExternalStore
   ↓
React components
```

This is especially useful when the real-time source is not inherently React state.

---

# 27. Real-time + TanStack Query

This is an advanced but extremely useful combination.

Suppose server sends:

```json
{
  "type": "task.updated",
  "task": {
    "id": 91,
    "status": "done"
  }
}
```

Instead of immediately refetching the entire task list every time, you might update the query cache directly:

```text
WebSocket event
 ↓
setQueryData(...)
 ↓
cached task updated
 ↓
UI rerenders
```

Now you get:

```text
low-latency UI
+
server-state cache
```

---

# 28. But events can be missed

A WebSocket isn't your database.

Connection:

```text
connected
```

doesn't guarantee:

```text
client has every event
```

Suppose:

```text
event 100
event 101
event 102
```

connection drops after 101.

You reconnect.

Now you need a resynchronization strategy.

For example:

```text
reconnect
 ↓
send lastSeen=101
 ↓
server sends missed events
```

or:

```text
reconnect
 ↓
invalidate queries
 ↓
full refetch
```

The second is simpler.

The first may be more efficient at larger scale.

---

# 29. Suspense enters the picture

Suppose the route contains:

```text
Project overview
Task board
Activity feed
```

Not everything needs to be ready at once.

You can structure the UI:

```text
Project page
 ├── Project header
 ├── Suspense → task board
 └── Suspense → activity feed
```

Conceptually:

```text
HTML shell
 ↓
important content appears
 ↓
slower sections resolve
 ↓
their fallbacks are replaced
```

This helps the user perceive progress.

---

# 30. Suspense is not an HTTP strategy

This is important.

Suspense says:

> "This React subtree isn't ready."

It does not itself decide:

```text
which API to call
how to cache data
when to retry
how to authenticate
```

Those are separate concerns.

Suspense is a rendering coordination mechanism.

TanStack Query is a server-state management mechanism.

Router is a navigation/data-loading mechanism.

Server Functions are server execution boundaries.

Do not blur them together.

---

# 31. Transitions now make sense

Suppose switching from:

```text
Open
```

to:

```text
Completed
```

causes a large list to recompute.

We can mark the update as non-urgent:

```text
user click
 ↓
urgent interaction response
 ↓
transition
 ↓
new list rendering
```

The browser can keep the interface responsive while the lower-priority UI work is scheduled.

So:

```text
Transition
≠
data fetching system
≠
debounce
≠
cache
```

It is about rendering priority.

---

# 32. `useDeferredValue`

Suppose the user types:

```text
rea
```

into a search box.

Input should remain immediately responsive.

But the enormous search results section can lag slightly:

```text
typed value
   ↓
urgent
   ↓
input updates

deferred value
   ↓
non-urgent
   ↓
expensive results update
```

This is especially useful when:

```text
input is fast
rendering is expensive
```

---

# 33. Router + Suspense + transition

Navigation can involve all three.

Imagine:

```text
user clicks Tasks
```

The flow may conceptually become:

```text
click
 ↓
start navigation
 ↓
route changes
 ↓
data/code not ready
 ↓
Suspense fallback
 ↓
task UI appears
```

Transitions control update priority.

Suspense controls rendering while something is not ready.

Router controls navigation.

Three different jobs.

---

# 34. SSR enters at the beginning

Now rewind.

User first requests:

```text
/projects/42
```

Server might produce initial HTML.

```text
request
 ↓
server render
 ↓
HTML response
 ↓
browser paints
```

So the user can see meaningful UI before the full client-side application is hydrated.

Then:

```text
HTML
 ↓
hydrateRoot
 ↓
interactive React
```

This is why SSR is important for initial experience.

---

# 35. SSR doesn't mean everything becomes server-only

Our page might still contain:

```text
Server Components
+
Client Components
```

SSR is about:

> **how initial UI is produced and delivered**

Server Components are about:

> **where component code/data logic executes**

These concepts overlap in some architectures but are not identical.

---

# 36. Hydration

After SSR, the browser has:

```html
<button>Save</button>
```

but it needs client React to attach behavior.

So:

```text
server HTML
 ↓
browser
 ↓
hydrate
 ↓
event handling becomes active
```

That's why:

> visible ≠ interactive

during the earliest phase.

---

# 37. Hydration mismatch

Suppose server renders:

```text
"Welcome"
```

but client immediately renders:

```text
"Welcome, Alice"
```

Now the trees differ.

That's a hydration mismatch.

This can happen with:

```text
Date.now()
Math.random()
window
localStorage
timezone
browser-only state
```

used incorrectly during initial rendering.

So one production architecture rule is:

> **The server and the initial client render need compatible output.**

---

# 38. Server state during SSR

Suppose:

```text
project 42
```

is fetched server-side.

You have to decide:

```text
How does the client continue from that initial state?
```

Possible architecture:

```text
server-rendered data
 ↓
serialized/streamed result
 ↓
client hydration/cache initialization
```

or the client may simply make its own request afterward.

The best architecture depends on the framework and cache strategy.

The key is:

> Avoid accidentally doing the same expensive work twice.

---

# 39. A bad SSR architecture

Imagine:

```text
Server
 ↓
fetch project
 ↓
render HTML
```

Then browser hydrates:

```text
Client
 ↓
useEffect
 ↓
fetch project AGAIN
```

Now:

```text
same data
two requests
```

Sometimes that's acceptable.

But often it's wasteful.

You should deliberately decide where initial data comes from and how the client takes over.

---

# 40. Now combine everything

Let's trace:

```text
GET /projects/42
```

from beginning to end.

---

## Step 1 — Browser requests URL

```text
Browser
 ↓
GET /projects/42
```

---

## Step 2 — Server/router resolves route

```text
request
 ↓
Router
 ↓
projectId = 42
```

---

## Step 3 — Server-side rendering

```text
Server Component / route data
 ↓
project data
 ↓
initial UI
```

---

## Step 4 — HTML reaches browser

```text
server
 ↓
HTML/stream
 ↓
browser paint
```

User sees the application.

---

## Step 5 — Client hydrates

```text
HTML
 ↓
hydrate
 ↓
interactive components
```

---

## Step 6 — Dynamic task state becomes active

```text
TaskBoard
 ↓
TanStack Query
 ↓
tasks cache
```

---

## Step 7 — User changes filter

```text
click "Open"
 ↓
URL → ?status=open
 ↓
query key changes
 ↓
query cache lookup/request
 ↓
TaskBoard updates
```

---

## Step 8 — User creates task

```text
submit form
 ↓
React Action
 ↓
Server Function
 ↓
authentication
 ↓
authorization
 ↓
validation
 ↓
database INSERT
```

---

## Step 9 — UI updates

Possibility A:

```text
server result
 ↓
invalidate query
 ↓
refetch
 ↓
canonical list
```

Possibility B:

```text
optimistic update
 ↓
server confirms
 ↓
cache reconciles
```

---

## Step 10 — Another user changes a task

```text
server
 ↓
WebSocket event
 ↓
shared external store / query cache
 ↓
React
 ↓
TaskBoard rerender
```

---

# 41. That's the entire architecture

Notice how each technology has a separate job.

```text
URL
 ↓
Router

route/page data
 ↓
Router / Server Components

server state
 ↓
TanStack Query

local interaction state
 ↓
useState / useReducer

shared UI state
 ↓
Context

external real-time state
 ↓
useSyncExternalStore

form submission lifecycle
 ↓
Actions

server mutation boundary
 ↓
Server Functions / API

loading coordination
 ↓
Suspense

render priority
 ↓
Transitions

initial page generation
 ↓
SSR

authentication/authorization
 ↓
server
```

That's the architecture map you should have in your head.

---

# 42. The biggest mistake: using one tool for everything

Imagine someone says:

> "I'll just put everything in Context."

Then:

```text
user
projects
tasks
filters
server data
notifications
theme
modal
drafts
permissions
```

all become context state.

Problems appear:

```text
large provider
wide rerenders
unclear ownership
stale server data
hard testing
```

Or:

> "I'll use TanStack Query for everything."

Then:

```text
modalOpen
inputDraft
selectedTab
hoveredRow
```

become "server state."

Wrong category.

Or:

> "I'll use Redux/external store for everything."

Same issue.

Architecture is fundamentally about classification.

---

# 43. State ownership map

Let's classify Orbit.

|State|Owner|
|---|---|
|Current route|Router|
|`?status=open`|URL|
|Draft task title|local component state|
|Complex task-editor workflow|reducer / Action|
|Theme|Context|
|Cached tasks|TanStack Query|
|Current WebSocket connection|external store|
|Temporary optimistic task|optimistic state|
|Submitted form state|React Action|
|Session identity|server-authenticated session|
|Authorization decision|server|
|Database records|database|

This is one of the most useful tables in the entire course.

---

# 44. Why state ownership matters

Suppose:

```text
task list
```

is stored locally in:

```js
useState([])
```

But the same task list also exists in:

```text
server database
TanStack Query cache
WebSocket event state
```

Now you have multiple sources of truth.

Changes can drift.

Instead:

```text
Database
   ↓
canonical server truth

TanStack Query
   ↓
client cache of server truth

Optimistic state
   ↓
temporary predicted state
```

That is much clearer.

---

# 45. Server truth vs client truth

A useful distinction:

### Server truth

```text
database
permissions
canonical task state
```

### Client state

```text
input draft
open modal
selected tab
optimistic overlay
```

### Cached server state

```text
TanStack Query
```

### URL state

```text
route
filters
pagination
```

Each category has different consistency requirements.

---

# 46. Why Server Functions should not replace TanStack Query

Suppose someone says:

> "We have Server Functions, so we don't need a query library."

Not necessarily.

Server Functions answer:

```text
How does the browser invoke this server-side operation?
```

TanStack Query answers:

```text
How does the client cache, synchronize and reason about server data?
```

So:

```text
Server Function
= command / mutation boundary

TanStack Query
= server-state cache/synchronization
```

They solve different problems.

---

# 47. Why Router loaders don't replace everything either

A route loader is excellent for:

```text
navigation-dependent route data
```

But suppose:

```text
notification count
live task board
background refetching
shared cache
```

These may have lifetimes beyond one route navigation.

That's where a server-state library or external store can become useful.

Again:

> **Choose based on ownership and lifetime.**

---

# 48. Lifetime is an underrated architectural concept

Ask:

> "How long should this state exist?"

### One render

```text
derived value
```

### One component instance

```text
useState
```

### Subtree lifetime

```text
Context
```

### URL lifetime

```text
search params / route
```

### Server-cache lifetime

```text
TanStack Query
```

### Application/external-system lifetime

```text
external store
socket
```

### Database lifetime

```text
server persistence
```

State lifetime often tells you where it belongs.

---

# 49. Scope is equally important

Another question:

> "Who needs this state?"

```text
one component
 ↓
useState

sibling components
 ↓
lift state / shared parent

large subtree
 ↓
Context

entire client app
 ↓
external store, when justified

server data
 ↓
query cache
```

This is why we learned state architecture before advanced integrations.

---

# 50. Data flow should be visible

Good architecture makes the data journey understandable.

For a task:

```text
DB
 ↓
Server Component / API
 ↓
TanStack Query
 ↓
TaskBoard
 ↓
TaskRow
```

For a filter:

```text
click
 ↓
URL
 ↓
query key
 ↓
server request
 ↓
TaskBoard
```

For creation:

```text
form
 ↓
Action
 ↓
Server Function
 ↓
database
 ↓
cache invalidation
 ↓
TaskBoard
```

When debugging, you can literally trace the arrow.

---

# 51. Now add authorization

Suppose a task deletion request occurs.

Client:

```text
Delete button
 ↓
mutation
 ↓
Server Function
```

Server:

```text
authenticate
 ↓
identify user
 ↓
load task
 ↓
check project membership
 ↓
check delete permission
 ↓
delete
```

This is the actual security boundary.

React may hide the button.

The server still must enforce the rule.

---

# 52. Error handling across boundaries

Now imagine database deletion fails.

The server returns:

```text
error
```

What happens?

```text
Server Function
 ↓
error result
 ↓
Action state
 ↓
UI error
```

If the error is a render failure instead:

```text
component throws
 ↓
Error Boundary
 ↓
fallback UI
 ↓
observability
```

If it's a network problem:

```text
query/mutation failure
 ↓
error state
 ↓
retry / message
```

Different failure types have different owners.

---

# 53. This leads to a failure taxonomy

Think:

```text
render failure
→ Error Boundary

server request failure
→ query/mutation/error state

validation failure
→ Action/form state

authorization failure
→ server 401/403 + UI response

network failure
→ retry/offline behavior

unexpected system failure
→ telemetry + fallback
```

This is much cleaner than one giant:

```text
try/catch everywhere
```

architecture.

---

# 54. Performance boundaries appear naturally too

Suppose the task board is huge.

Now:

```text
TaskBoard
```

might need:

```text
virtualization
memoization
deferred updates
transitions
```

But don't immediately add all of them.

Start with:

```text
measure
 ↓
identify bottleneck
 ↓
optimize specific boundary
```

For example:

```text
task rendering
 ↓
10,000 DOM nodes
 ↓
virtualization
```

That's more meaningful than:

```text
add useMemo everywhere
```

---

# 55. Code splitting fits here too

Suppose:

```text
/project/42
```

contains a heavy analytics editor that's rarely used.

Don't ship it to every user immediately.

Instead:

```text
route
 ↓
lazy chunk
 ↓
load editor when needed
```

Then:

```text
Suspense
 ↓
loading fallback
 ↓
editor
```

So code splitting, Suspense, and routing can combine naturally.

---

# 56. Accessibility belongs at the component boundary

For example:

```text
TaskEditor
```

should define:

```text
keyboard behavior
labels
focus management
error announcement
disabled state
pending state
```

That means accessibility isn't a final audit step.

It's part of the component's contract.

For example:

```text
Action pending
 ↓
button disabled
 ↓
accessible status
 ↓
focus behavior
```

---

# 57. Testing now becomes much easier to organize

For Orbit:

```text
pure validation
 → unit tests

TaskEditor
 → component tests

Query + API behavior
 → integration tests

Server Function + authorization
 → server integration tests

login → create task → realtime update
 → E2E
```

Notice how testing follows architecture.

That isn't accidental.

Good architecture gives clear testing boundaries.

---

# 58. Observability follows the same boundaries

For a failed task creation:

```text
browser
 ↓
requestId
 ↓
Server Function
 ↓
authorization
 ↓
database
```

You can attach telemetry at each meaningful boundary.

So architecture isn't just about writing code.

It determines:

```text
where to test
where to monitor
where to authorize
where to cache
where to recover
```

---

# 59. Deployment follows the same architecture

Our application may require:

```text
browser assets
server runtime
database
WebSocket infrastructure
environment variables
CDN
```

So deployment is not just:

```text
npm run build
```

You need to ensure:

```text
Client bundle
     ↓
CDN/static host

Server functions
     ↓
server runtime

WebSocket
     ↓
connection-capable infrastructure

Database
     ↓
managed persistence
```

The architecture dictates deployment.

---

# 60. Security follows the same boundaries

Look at the task creation flow:

```text
Browser
  ↓
untrusted FormData
  ↓
Server Function
  ↓
validation
  ↓
authentication
  ↓
authorization
  ↓
database
```

Every boundary has a security rule.

This is why security is easier when architecture is explicit.

---

# 61. The most important architecture diagram

Here is the whole Orbit application:

```text
                             BROWSER
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
             React UI                     Browser APIs
                 │                             │
                 ▼                             ▼
              Router                    External Store
                 │                             │
        ┌────────┴────────┐              WebSocket
        │                 │
     URL state       route structure
        │
        ▼
   Server / Client UI
        │
   ┌────┴─────────────────────────┐
   │                              │
Server Components             Client Components
   │                              │
server data                  local state
   │                         reducer/context
   │                              │
   │                       TanStack Query
   │                              │
   └──────────────┬───────────────┘
                  │
             Actions / Forms
                  │
             Server Functions
                  │
       ┌──────────┴───────────┐
       │                      │
 business logic          authorization
       │                      │
       └──────────┬───────────┘
                  ▼
               DATABASE
```

Around the whole thing:

```text
SSR
Suspense
Transitions
Testing
Security
Observability
Deployment
```

---

# 62. Here's the deeper pattern

Every advanced React technology we've learned can be understood through four questions.

## Where does the state live?

```text
React?
URL?
server?
external system?
```

## Who owns the operation?

```text
component?
router?
server action?
server function?
```

## Who is allowed to mutate it?

```text
client?
server?
authorized server actor?
```

## What happens when it isn't ready or fails?

```text
Suspense?
pending?
error state?
rollback?
retry?
Error Boundary?
fallback?
```

This framework is more valuable than memorizing APIs.

---

# 63. A common architectural mistake: mixing commands and queries

This is a subtle distinction.

A **query** asks:

```text
"What is the current state?"
```

Examples:

```text
GET project
GET tasks
GET notifications
```

A **command/mutation** says:

```text
"Make something happen."
```

Examples:

```text
create task
delete task
archive project
change password
```

This is why:

```text
TanStack Query
```

is naturally query-oriented while:

```text
Actions / Server Functions / mutations
```

are naturally command-oriented.

Keeping this distinction clear makes data flow much easier.

---

# 64. Another subtle distinction: canonical vs derived state

Suppose:

```js
const completedCount =
  tasks.filter(task =>
    task.status === 'done'
  ).length;
```

Don't store:

```js
completedCount
```

unless necessary.

Because:

```text
tasks
 ↓
derived count
```

is deterministic.

Canonical data:

```text
tasks
```

Derived data:

```text
completedCount
```

This reduces synchronization problems.

---

# 65. Another subtle distinction: server cache vs UI cache

TanStack Query cache:

```text
"what does the server currently say?"
```

UI state:

```text
"what is the user currently doing?"
```

For example:

```text
tasks = server state
isEditorOpen = UI state
```

Don't put:

```text
isEditorOpen
```

into a server query cache.

And don't make every task fetch a local component state problem.

---

# 66. Another subtle distinction: temporary vs durable state

Suppose the user is typing:

```text
"Fix dashboard..."
```

That's:

```text
temporary draft
```

Once saved:

```text
database
```

becomes:

```text
durable state
```

During submission:

```text
optimistic state
```

may temporarily predict the durable state.

So one user action can involve:

```text
draft
 ↓
optimistic prediction
 ↓
server command
 ↓
canonical durable state
```

That is an advanced but very useful mental model.

---

# 67. The architecture of a single user action

Let's trace:

> User changes a task from Open → Done.

```text
1. click checkbox
        ↓
2. client event
        ↓
3. optimistic state
        ↓
4. mutation / Action
        ↓
5. Server Function
        ↓
6. authenticate
        ↓
7. authorize
        ↓
8. DB update
        ↓
9. server response
        ↓
10. query cache reconciliation
        ↓
11. optimistic state settles
        ↓
12. React rerender
        ↓
13. WebSocket event may notify other users
```

That's a modern full-stack React interaction.

---

# 68. And now imagine failure

```text
click
 ↓
optimistic Done
 ↓
server request
 ↓
database timeout
 ↓
failure
 ↓
rollback
 ↓
error UI
 ↓
telemetry
```

A truly production-ready application doesn't just define:

```text
happy path
```

It defines:

```text
happy path
+
failure path
+
recovery path
```

---

# 69. This is what "production-ready React" really means

It does **not** mean:

```text
"I know 50 hooks."
```

It means you can reason about:

```text
state ownership
data flow
network boundaries
server/client boundaries
authorization
consistency
failure
performance
accessibility
testing
observability
deployment
```

React APIs are just mechanisms for implementing that reasoning.

---

# 70. The final decision framework

When you encounter a new piece of state, ask:

```text
Is it purely derived?
→ don't store it

Does one component own it?
→ useState

Complex local transitions?
→ useReducer

Does a subtree need it?
→ Context

Should it live in the URL?
→ Router/search params

Is it server-owned?
→ server-state/cache mechanism

Is it from an external mutable system?
→ useSyncExternalStore

Is it temporary optimistic UI?
→ optimistic state

Is it a form submission lifecycle?
→ Action

Is it a server mutation boundary?
→ Server Function / API
```

Then ask:

```text
Where is the trust boundary?
Where can this fail?
How does it recover?
How do I test it?
How will I observe it in production?
```

That's the real skill.

---

# 71. One complete mental picture

You started this course thinking about:

```jsx
function Counter() {
  const [count, setCount] =
    useState(0);
}
```

Now the mental model has become:

```text
                         USER
                           │
                           ▼
                        BROWSER
                           │
                  ┌────────┴─────────┐
                  ▼                  ▼
                ROUTER           UI STATE
                  │                  │
                  ▼                  ▼
           server/client UI     Context/reducer
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
 Server Components     Client Components
        │                   │
        │              TanStack Query
        │                   │
        │             optimistic state
        │                   │
        └─────────┬─────────┘
                  ▼
                ACTION
                  │
                  ▼
            SERVER FUNCTION
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
   AUTHORIZATION          VALIDATION
        │                    │
        └─────────┬──────────┘
                  ▼
              DATABASE
                  │
                  ▼
          canonical server state
                  │
        ┌─────────┴───────────┐
        ▼                     ▼
    query cache          WebSocket events
        │                     │
        └─────────┬───────────┘
                  ▼
                REACT
                  │
           render / commit
                  │
                  ▼
                 UI
```

And surrounding it:

```text
SSR
Suspense
Transitions
Accessibility
Security
Testing
Observability
Performance
Deployment
```

That's the complete system.

---

# 72. The single most valuable lesson

Don't ask:

> "Should I use `useState`, Context, Redux, TanStack Query, a loader, a Server Function, or an Action?"

That question is too API-oriented.

Ask:

> **"What is this data, who owns it, how long should it live, where does it cross a trust boundary, and what should happen when it changes or fails?"**

Once you answer those questions, the technology choice usually becomes much easier.

---

## Where we are now

The remaining course is essentially the finishing phase:

```text
50  ✅ Advanced architecture / integration

51  Capstone:
    build a realistic production React app
    from requirements all the way to deployment

52  Final review:
    React mental models
    common mistakes
    architecture decisions
    interview-level reasoning
    production readiness
```

The next lesson is the **capstone**: we'll design a realistic application from scratch and make every decision we've learned—state ownership, routing, server state, Server Components, Server Functions, Actions, optimistic UI, real-time updates, auth, security, testing, observability, and deployment—fit together into one project.