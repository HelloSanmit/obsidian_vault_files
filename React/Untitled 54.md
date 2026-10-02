
# Lesson 51 — Capstone: Build a Production-Grade React Application

This is where we stop learning concepts one by one and start making architectural decisions.

We're going to design a realistic application from scratch:

# **Orbit — Project Management Platform**

Think:

```text
Linear + Trello + Slack-lite
```

Users can create projects, manage tasks, collaborate with teammates, comment, filter work, and see live updates.

We are not going to pretend every feature needs every React API.

Instead, we'll deliberately decide:

> **What belongs where, and why?**

---

# 1. Requirements first

Before writing React, define the product.

Orbit needs:

```text
Authentication
 ├── Sign in
 ├── Sign out
 └── session persistence

Projects
 ├── project list
 ├── project details
 ├── create project
 └── archive project

Tasks
 ├── list
 ├── create
 ├── edit
 ├── delete
 ├── status
 ├── priority
 ├── filtering
 └── pagination

Collaboration
 ├── comments
 └── live task updates

UX
 ├── loading states
 ├── optimistic updates
 ├── errors
 ├── keyboard accessibility
 └── responsive layout

Production
 ├── SSR
 ├── code splitting
 ├── security
 ├── testing
 ├── observability
 └── deployment
```

Notice something important:

We're defining **behavior**, not components.

Don't start with:

```text
Let's make 43 components.
```

Start with:

```text
What must the product do?
```

---

# 2. Domain model

Before React, define the business entities.

We might have:

```text
User
 ├── id
 ├── name
 └── email

Organization
 ├── id
 └── name

Project
 ├── id
 ├── organizationId
 ├── name
 ├── description
 └── archived

Task
 ├── id
 ├── projectId
 ├── title
 ├── description
 ├── status
 ├── priority
 ├── assigneeId
 └── updatedAt

Comment
 ├── id
 ├── taskId
 ├── authorId
 ├── body
 └── createdAt
```

This gives us a domain vocabulary.

---

# 3. Why domain modeling comes before React

Suppose you start with:

```jsx
function TaskCard() {
  ...
}
```

You can build beautiful UI without understanding:

```text
Who owns the task?
Who can edit it?
What does "done" mean?
Who can delete it?
What happens across organizations?
```

Architecture should start with the **business rules**.

Then React represents those rules.

---

# 4. Authorization rules

Let's define a few.

```text
Organization member
 → can view organization projects

Project member
 → can view project

Task editor
 → can create/update tasks

Project admin
 → can archive project

Organization admin
 → can manage membership
```

Now we have an authorization graph.

This matters because the UI must reflect it, but the server must enforce it.

---

# 5. The first architectural map

Our application has four major layers:

```text
                    Orbit
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
      UI            Server        External systems
       │              │              │
   React            business       WebSocket
   Router           logic          email/payment/etc.
   forms            auth
   components       DB
```

Now we can assign responsibilities.

---

# 6. UI state

Examples:

```text
modal open?
currently focused input?
draft title?
selected tab?
hover state?
dragging?
```

These are local UI concerns.

Use:

```text
useState
```

or:

```text
useReducer
```

when transitions get complicated.

---

# 7. URL state

Suppose the user is viewing:

```text
/projects/42/tasks?status=open&priority=high&page=2
```

That information belongs in the URL because it is:

```text
shareable
bookmarkable
navigation-related
meaningful to browser history
```

So:

```text
projectId → route param
status    → search param
priority  → search param
page      → search param
```

Don't hide all of that in local state.

---

# 8. Server state

Tasks come from the backend.

They are not owned by React.

Therefore:

```text
Task database
      ↓
TanStack Query
      ↓
React UI
```

The query cache is a client-side representation of server state.

---

# 9. Authentication state

The user's authenticated identity should fundamentally come from the server-side session model.

Conceptually:

```text
session cookie
     ↓
server
     ↓
authenticated user
```

The browser may display:

```text
Alice
```

but that displayed value isn't what makes Alice authenticated.

The server's session is authoritative.

---

# 10. External real-time state

WebSocket:

```text
WebSocket
   ↓
connection manager
   ↓
external store
   ↓
React
```

This avoids each component opening its own connection.

---

# 11. Folder structure

A scalable project might look like:

```text
src/
│
├── app/
│   ├── router/
│   ├── providers/
│   ├── styles/
│   └── config/
│
├── features/
│   ├── auth/
│   ├── projects/
│   ├── tasks/
│   ├── comments/
│   └── notifications/
│
├── shared/
│   ├── ui/
│   ├── hooks/
│   ├── utils/
│   └── types/
│
├── realtime/
│   ├── socket.js
│   └── store.js
│
└── server/
    ├── auth/
    ├── db/
    ├── services/
    └── functions/
```

The exact naming doesn't matter.

The principle does:

> **Organize around business capabilities, while keeping infrastructure boundaries obvious.**

---

# 12. Why feature-oriented structure scales

Imagine:

```text
components/
hooks/
services/
utils/
```

with 400 files.

Where is the code for task creation?

You might need to search:

```text
components/
hooks/
services/
types/
utils/
```

Feature organization gives:

```text
features/tasks/
```

and the related behavior lives closer together.

---

# 13. But don't create a mini-framework

Avoid:

```text
features/
  tasks/
    components/
      common/
        shared/
          reusable/
```

until you've earned the complexity.

Start simple.

Architecture should emerge from actual cohesion.

---

# 14. Application providers

At the application root we might conceptually have:

```jsx
<QueryClientProvider client={queryClient}>
  <AuthProvider>
    <RouterProvider router={router} />
  </AuthProvider>
</QueryClientProvider>
```

Each provider has a different responsibility.

```text
QueryClientProvider
 → server-state cache

AuthProvider
 → client-readable authentication context

RouterProvider
 → navigation
```

Don't throw arbitrary state into these providers.

---

# 15. Route structure

Our application:

```text
/login

/app
  /projects
  /projects/:projectId
  /projects/:projectId/tasks
  /projects/:projectId/settings
```

Perhaps:

```text
/app
 ├── sidebar
 ├── top bar
 └── Outlet
      ├── projects
      ├── project/:id
      ├── tasks
      └── settings
```

The route hierarchy becomes the application's navigation skeleton.

---

# 16. Protected route vs actual authorization

At the UI level:

```text
/app
```

may redirect unauthenticated users to:

```text
/login
```

Good.

But:

```text
GET /api/projects/42
```

still needs server-side authorization.

Remember:

```text
route guard
 ≠
security boundary
```

---

# 17. Project page

Our project page might look like:

```text
┌──────────────────────────────────────────────┐
│ Orbit                                Profile │
├───────────┬──────────────────────────────────┤
│ Overview  │ Project: Apollo                  │
│ Tasks     │                                  │
│ Activity  │ Open: 23   Done: 102             │
│ Settings  │                                  │
│           │ [New Task] [Filter] [Search]     │
│           │                                  │
│           │ Task list...                     │
└───────────┴──────────────────────────────────┘
```

Now decompose by responsibility.

---

# 18. Component tree

Conceptually:

```text
ProjectLayout
├── ProjectSidebar
├── ProjectHeader
└── Outlet
    ├── ProjectOverview
    ├── TaskBoard
    │   ├── TaskToolbar
    │   ├── TaskColumn
    │   │   └── TaskCard
    │   └── TaskEditor
    └── ProjectSettings
```

Notice:

`ProjectLayout` owns layout.

`TaskBoard` owns board behavior.

`TaskCard` represents one task.

We don't create one giant:

```text
ProjectPage.jsx = 2000 lines
```

---

# 19. TaskBoard state

What does TaskBoard need?

```text
tasks
filter
sort
page
selectedTask
editorOpen
```

But these don't all belong in one state bucket.

Correct classification:

```text
tasks
 → TanStack Query

filter
sort
page
 → URL

selectedTask
 → local state

editorOpen
 → local state
```

That's architecture through classification.

---

# 20. Query key design

Suppose:

```js
const queryKey = [
  'tasks',
  projectId,
  {
    status,
    priority,
    page
  }
];
```

That identifies a particular representation of task data.

Conceptually:

```text
project 42
+
status=open
+
priority=high
+
page=2
```

becomes cache identity.

Now two components requesting the same logical data can share cache state.

---

# 21. Query function

The query function might conceptually be:

```js
async function fetchTasks({
  projectId,
  status,
  priority,
  page
}) {
  const params = new URLSearchParams({
    status,
    priority,
    page
  });

  const response = await fetch(
    `/api/projects/${projectId}/tasks?${params}`
  );

  if (!response.ok) {
    throw new Error('Failed to load tasks');
  }

  return response.json();
}
```

Notice:

```text
URL state
 ↓
query parameters
 ↓
server request
```

One state source naturally drives the next.

---

# 22. Task rendering

Then:

```jsx
const { data, isPending, isError } =
  useQuery({
    queryKey,
    queryFn: ...
  });
```

The UI becomes a projection:

```text
query state
 ↓
React render
 ↓
task list
```

React isn't storing the canonical task collection.

It's rendering the query's current cache state.

---

# 23. Pagination

Suppose page 1 has:

```text
tasks 1–50
```

page 2:

```text
tasks 51–100
```

Their query identities differ:

```text
['tasks', 42, { page: 1 }]
['tasks', 42, { page: 2 }]
```

The cache can therefore hold both.

With placeholder/previous-data techniques, switching pages can avoid making the interface unnecessarily jump back into an empty state while the next page loads.

---

# 24. Search and filter

Suppose the user types:

```text
bug
```

Should every keystroke trigger a network request?

Maybe.

Maybe not.

For expensive search:

```text
input
 ↓
debounced value
 ↓
URL
 ↓
query key
 ↓
request
```

You need to distinguish:

```text
user input
```

from:

```text
server query state
```

Again, different concepts.

---

# 25. Creating a task

The user opens:

```text
New Task
```

This is local UI state:

```js
const [open, setOpen] = useState(false);
```

The form draft:

```js
const [title, setTitle] = useState('');
```

When submitted:

```text
form
 ↓
Action
 ↓
server function
 ↓
database
```

Different lifetimes.

---

# 26. Task creation with an Action

Conceptually:

```jsx
function NewTaskForm() {
  const [
    state,
    submit,
    isPending
  ] = useActionState(
    createTask,
    initialState
  );

  return (
    <form action={submit}>
      <label>
        Title
        <input name="title" />
      </label>

      <button disabled={isPending}>
        {isPending ? 'Creating...' : 'Create'}
      </button>

      {state.error && (
        <p>{state.error}</p>
      )}
    </form>
  );
}
```

The Action owns:

```text
submission lifecycle
result
pending state
```

---

# 27. Server Function

Server code conceptually:

```js
'use server';

export async function createTask(
  previousState,
  formData
) {
  const user =
    await requireUser();

  const title =
    validateTaskTitle(
      formData.get('title')
    );

  await authorizeTaskCreation(user);

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

The order matters:

```text
authenticate
 ↓
validate
 ↓
authorize
 ↓
mutate
```

---

# 28. Why authenticate before authorization?

You cannot meaningfully ask:

> "Can this user perform the action?"

until you know:

> "Which user is this?"

So:

```text
authentication
 ↓
identity
 ↓
authorization
```

These are separate steps.

---

# 29. Why validate before mutation?

Suppose:

```text
title = 10MB string
```

or:

```text
title = null
```

The server should reject invalid input before touching the database.

So:

```text
request
 ↓
validation
 ↓
authorization
 ↓
database
```

Some systems choose a slightly different order depending on information disclosure concerns, but the general principle is explicit server-side validation and authorization before sensitive mutation.

---

# 30. Optimistic creation

Now let's make the UX faster.

User submits:

```text
Fix dashboard
```

Immediately:

```text
temporary task
```

appears.

Perhaps:

```text
Fix dashboard
Creating...
```

Then:

### Success

```text
temporary
 ↓
server confirms
 ↓
canonical task
```

### Failure

```text
temporary
 ↓
server rejects
 ↓
remove/mark failed
 ↓
show retry
```

The optimistic state should never be mistaken for database truth.

---

# 31. Updating a task

Suppose the user clicks:

```text
Done
```

The event flow:

```text
click
 ↓
optimistic status=done
 ↓
mutation
 ↓
server authorization
 ↓
database UPDATE
 ↓
canonical result
 ↓
cache reconciliation
```

This should happen without rewriting the entire application state.

---

# 32. Cache update or invalidation?

After the mutation, you have choices.

### Direct cache update

```text
setQueryData(...)
```

Useful when you know the canonical new value.

### Invalidation

```text
invalidateQueries(...)
```

Useful when many representations may have become stale.

For example:

```text
task list
task detail
filtered list
project statistics
activity feed
```

A single mutation can affect many views.

---

# 33. Example

Task 91 changes:

```text
Open → Done
```

Potentially stale:

```text
['tasks', 42]
['task', 91]
['tasks', 42, { status: 'open' }]
['tasks', 42, { status: 'done' }]
['projectStats', 42]
```

This is why mutation correctness isn't merely:

```text
"database UPDATE succeeded"
```

It's also:

```text
"all relevant client representations converge toward server truth"
```

---

# 34. Real-time collaboration

Now another user changes task 91.

Your server emits:

```json
{
  "type": "task.updated",
  "task": {
    "id": 91,
    "status": "done"
  }
}
```

The event arrives:

```text
WebSocket
 ↓
event router
 ↓
query cache / external store
 ↓
React
 ↓
TaskCard
```

The current user sees the update without refreshing.

---

# 35. What if the current user is also editing?

Interesting problem.

Suppose user Alice is editing:

```text
title = "Fix dashboard quickly"
```

Bob changes the task title remotely.

Now the server sends:

```text
title = "Fix dashboard urgently"
```

Should Alice's draft immediately overwrite?

Probably not.

This is a conflict-resolution problem.

Possible behavior:

```text
server update arrives
 ↓
editing draft exists
 ↓
don't overwrite draft
 ↓
show "Task updated by Bob"
```

This is why real-time systems aren't just:

```text
socket.onmessage = setState
```

Real collaboration requires product semantics.

---

# 36. Comments

Comments are server state:

```text
useQuery(['comments', taskId])
```

Creating comments is a mutation:

```text
useMutation(...)
```

New comments from other users arrive through:

```text
WebSocket event
```

The comment editor draft is:

```text
local UI state
```

Again:

```text
draft
≠
server comment collection
```

---

# 37. Notifications

Notifications are a nice example of external + server state.

The server may maintain:

```text
unread notifications
```

but live delivery is:

```text
WebSocket
```

So:

```text
initial unread count
 ↓
TanStack Query
```

and:

```text
new notification event
 ↓
query cache update/invalidation
```

This combines server state with real-time transport.

---

# 38. SSR strategy

When a user opens:

```text
/projects/42
```

we want useful content quickly.

The server can render:

```text
project name
description
summary
navigation
```

The browser receives HTML.

Then:

```text
hydrate
 ↓
interactive task board
```

Some slower/dynamic areas can be coordinated with Suspense.

---

# 39. Server vs client component boundary

For example:

```text
ProjectOverview
```

could be server-oriented because it mostly displays server-known information.

While:

```text
TaskBoard
```

is highly interactive and client-oriented.

So:

```text
ProjectOverview
├── ProjectHeader
├── ProjectStats
└── TaskBoard ← Client Component
```

This keeps interactive logic where it belongs.

---

# 40. Why not make the whole project page client-side?

You could.

But then you might ship:

```text
all task interaction code
all editor code
all client state
all query logic
```

just to render a page whose top half is mostly static information.

Breaking boundaries thoughtfully can reduce client JavaScript and make server capabilities more useful.

---

# 41. Suspense boundary design

Imagine:

```text
Project page
├── Header             ← immediate
├── Statistics         ← fast
├── Task board         ← slower
└── Activity feed      ← slower
```

You might structure:

```jsx
<ProjectHeader />

<ProjectStats />

<Suspense fallback={<TaskBoardSkeleton />}>
  <TaskBoard />
</Suspense>

<Suspense fallback={<ActivitySkeleton />}>
  <ActivityFeed />
</Suspense>
```

Now one slow region doesn't necessarily block every other region.

---

# 42. Error boundaries

We can further isolate failures:

```text
Project page
├── Header
├── Task board
│   └── Error Boundary
└── Activity
    └── Error Boundary
```

Suppose Activity fails.

User still has:

```text
Header
Task board
```

The failure blast radius is smaller.

---

# 43. Accessibility

Our TaskBoard needs:

```text
keyboard navigation
buttons with labels
proper headings
form labels
aria-invalid
error descriptions
focus management
```

For example:

```jsx
<label htmlFor="task-title">
  Task title
</label>

<input
  id="task-title"
  aria-invalid={hasError}
  aria-describedby={
    hasError ? 'task-title-error' : undefined
  }
/>
```

Accessibility belongs directly in the feature implementation.

Not in a final "make it accessible" pass.

---

# 44. Modal architecture

When the user opens the task editor:

```text
TaskBoard
 ↓
TaskEditor
 ↓
portal
 ↓
document.body
```

The modal may render through a portal.

But it still belongs to the same React tree.

Therefore:

```text
context still works
events still follow React tree
ownership remains with TaskBoard
```

Only the DOM placement changes.

---

# 45. Focus management

Opening the modal:

```text
click Edit
 ↓
modal mounts
 ↓
focus moves to title input
```

Closing:

```text
close
 ↓
modal unmounts
 ↓
focus returns to trigger
```

That is not a visual detail.

It's part of the component's interaction contract.

---

# 46. Security of the task editor

Suppose task descriptions support rich text.

Now we have:

```text
user content
 ↓
parser
 ↓
sanitization
 ↓
controlled HTML rendering
```

We must not do:

```js
dangerouslySetInnerHTML={{
  __html: rawUserInput
}}
```

without an appropriate safety boundary.

---

# 47. Authentication flow

Let's trace sign-in:

```text
Login form
 ↓
POST /login
 ↓
server validates credentials
 ↓
server creates session
 ↓
HttpOnly cookie
 ↓
browser stores cookie
 ↓
subsequent requests
 ↓
server authenticates session
```

React may then refresh:

```text
current user
```

so UI knows what to display.

---

# 48. Logout

Logout:

```text
click Logout
 ↓
server invalidates session
 ↓
cookie cleared/invalidated
 ↓
client auth state updates
 ↓
navigate /login
```

Don't think:

```js
setUser(null)
```

is logout by itself.

That's only a client UI update.

---

# 49. Multi-tenant authorization

Orbit might have:

```text
Organization A
 ├── Project 1
 └── Project 2

Organization B
 ├── Project 3
 └── Project 4
```

A request:

```http
GET /api/projects/3
```

must verify:

```text
currentUser belongs to organization owning project 3
```

You cannot rely on:

```text
React route
projectId
hidden UI
```

to enforce tenant isolation.

This is one of the most important real-world authorization patterns.

---

# 50. API/service architecture

We don't want:

```text
React component
 ↓
SQL query
```

Instead:

```text
React
 ↓
server function / API
 ↓
application service
 ↓
repository
 ↓
database
```

Why?

Because business logic can then be reused and tested without coupling it directly to React.

---

# 51. Example

Bad conceptual structure:

```text
TaskCard
 → database.updateTask(...)
```

Better:

```text
TaskCard
 ↓
updateTask(...)
 ↓
Task service
 ↓
repository
 ↓
database
```

The React layer asks:

> "Perform this business operation."

It doesn't need to know SQL details.

---

# 52. Domain logic

Suppose task status transitions have rules:

```text
Open → In Progress
In Progress → Done
```

but:

```text
Archived → In Progress
```

is illegal.

Put this rule in domain/application logic:

```js
canTransitionTaskStatus(
  currentStatus,
  nextStatus
)
```

Then test it directly.

React should not be the only place enforcing business rules.

---

# 53. Testing architecture

Now our tests have natural homes.

### Unit

```text
status transition
validation
formatters
reducers
pure functions
```

### Component/integration

```text
task form
task board
query states
optimistic behavior
router interaction
```

### Server integration

```text
authorization
database
Server Functions
API contracts
```

### E2E

```text
login
open project
create task
edit task
collaboration flow
logout
```

---

# 54. Test one critical journey

Imagine:

```text
Sign in
 ↓
Projects
 ↓
Open Apollo
 ↓
Tasks
 ↓
New Task
 ↓
"Fix dashboard"
 ↓
Create
 ↓
Task appears
```

That is a high-value E2E test.

It verifies many layers simultaneously.

---

# 55. Test authorization separately

Another important server test:

```text
Alice
 ↓
DELETE /projects/42
 ↓
Alice lacks permission
 ↓
403
 ↓
database unchanged
```

The frontend test:

```text
Delete button hidden
```

is useful.

But the server test proves the security rule.

---

# 56. Test optimistic rollback

Another integration test:

```text
Task is visible
 ↓
click Done
 ↓
UI immediately shows Done
 ↓
server returns 500
 ↓
UI returns previous state
 ↓
error appears
```

That verifies a difficult behavior that users will absolutely notice when broken.

---

# 57. Observability

Suppose task creation fails.

We want:

```text
frontend event
requestId=AB123

server function
requestId=AB123

database error
requestId=AB123
```

And perhaps:

```text
release=2026.09.21
```

Now production debugging becomes tractable.

---

# 58. React error monitoring

At application startup:

```jsx
createRoot(container, {
  onCaughtError(error, info) {
    reportError(error, info);
  },

  onUncaughtError(error, info) {
    reportError(error, info);
  },

  onRecoverableError(error, info) {
    reportError(error, info);
  }
});
```

Then:

```text
React error
 ↓
telemetry
 ↓
monitoring dashboard
```

The user receives an appropriate fallback.

The engineering team gets diagnostics.

---

# 59. Performance budget

Suppose Orbit's project page has:

```text
huge task lists
rich text editor
charts
real-time connection
```

We should establish performance goals.

For example conceptually:

```text
initial page
 ↓
small amount of client JS
 ↓
interactive quickly
```

Then:

```text
large TaskBoard
 ↓
virtualization
```

and:

```text
rarely used editor
 ↓
lazy load
```

Architecture creates performance opportunities.

---

# 60. Code splitting

Don't load every feature immediately.

For example:

```text
/projects
   ↓
normal bundle

/project/42/settings
   ↓
settings chunk

/project/42/activity
   ↓
activity chunk
```

Then:

```text
lazy loading
 +
Suspense
 +
route boundaries
```

can reduce initial work.

---

# 61. Production deployment architecture

Orbit might deploy as:

```text
                     INTERNET
                        │
                        ▼
                       CDN
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        static assets          server runtime
                                    │
                         ┌──────────┼──────────┐
                         ▼          ▼          ▼
                       API      Server Fn   SSR/RSC
                         │
                         ▼
                       DB
                         │
                         ▼
                    WebSocket
```

The exact physical infrastructure depends on the framework/platform.

But the conceptual architecture remains.

---

# 62. Environment variables

Client:

```text
VITE_PUBLIC_API_URL
```

may be public.

Server:

```text
DATABASE_URL
PAYMENT_SECRET
SESSION_SECRET
```

must remain server-side.

Never:

```text
VITE_DATABASE_PASSWORD
```

because anything exposed to the client isn't secret.

---

# 63. Deployment-safe database migrations

Suppose version 1 has:

```text
name
```

and version 2 wants:

```text
displayName
```

A dangerous deployment:

```text
drop name
add displayName
deploy new code
```

Old application instances may still expect:

```text
name
```

A safer expand/contract process is:

```text
1. add displayName
2. deploy code that understands both
3. backfill
4. switch reads
5. switch writes
6. remove old field later
```

This is a general production system lesson.

---

# 64. Real-time scaling

Suppose Orbit has:

```text
4 server instances
```

and WebSocket connections distributed across them.

Bob connects to server A.

Alice connects to server C.

Bob updates a task.

How does Alice receive it?

You may need:

```text
server A
   ↓
pub/sub
   ↓
server C
   ↓
Alice's socket
```

So horizontal scaling changes the real-time architecture.

One-process WebSocket design doesn't automatically scale to many servers.

---

# 65. Full end-to-end data flow

Let's trace the entire application now.

User opens:

```text
/projects/42/tasks?status=open
```

### Browser

```text
URL
 ↓
Router
```

### Server

```text
route
 ↓
SSR / Server Components
 ↓
initial HTML
```

### Browser

```text
HTML
 ↓
hydrate
 ↓
interactive UI
```

### TaskBoard

```text
route params + search params
 ↓
query key
 ↓
TanStack Query
 ↓
GET /api/projects/42/tasks?status=open
```

### User clicks New Task

```text
local state
 ↓
modal opens
```

### User submits

```text
form
 ↓
Action
 ↓
Server Function
 ↓
authenticate
 ↓
validate
 ↓
authorize
 ↓
database
```

### Result

```text
success
 ↓
optimistic state settles
 ↓
query cache updated/invalidated
 ↓
TaskBoard rerenders
```

### Another user changes task

```text
database mutation
 ↓
event
 ↓
WebSocket
 ↓
client cache/store
 ↓
React
```

### Something crashes

```text
error
 ↓
Error Boundary / query error
 ↓
fallback UI
 ↓
observability
```

That is a production React system.

---

# 66. Notice how little `useEffect` we needed

This is one of the biggest architectural lessons.

When React developers discover `useEffect`, they often start doing:

```text
fetch data
sync everything
listen to everything
set state
initialize everything
```

Our capstone architecture instead uses more specialized mechanisms:

```text
Router
→ navigation/data

TanStack Query
→ server state

Actions
→ form submission lifecycle

Server Functions
→ server mutations

WebSocket/external store
→ real-time external state

useState
→ local UI

Context
→ subtree-wide shared state

useEffect
→ genuine external synchronization
```

Effects are still useful.

They simply aren't the universal solution.

---

# 67. The state ownership map for Orbit

This is worth keeping.

|State|Owner|
|---|---|
|Current route|Router|
|Project ID|Router param|
|Filter|URL|
|Page number|URL|
|Task collection|TanStack Query|
|Current task draft|local state|
|Modal open|local state|
|Complex editor workflow|reducer/Action|
|Theme|Context|
|WebSocket connection|external store|
|Optimistic task|optimistic state|
|Form result|Action|
|Auth session|server|
|Permissions|server|
|Database records|database|

If you remember only one table from this course, remember this one.

---

# 68. What happens when architecture is wrong?

Suppose we instead put everything into:

```text
AppContext
```

Then:

```text
tasks
filters
modals
user
notifications
drafts
theme
socket
permissions
```

all flow through one context.

Soon:

```text
everything changes
 ↓
everything may rerender
```

and nobody knows which state is authoritative.

Architecture problems often appear first as:

```text
performance bugs
stale data
complicated hooks
hard tests
```

not obvious "architecture errors."

---

# 69. What if everything uses server fetching?

Another extreme:

```text
every interaction
 ↓
route navigation
 ↓
server reload
```

You may get:

```text
too many round trips
poor interaction responsiveness
```

For highly interactive areas, client-side server-state caching can give a better user experience.

Again, balance.

---

# 70. What if everything uses WebSockets?

Also wrong.

Most data does not need:

```text
persistent bidirectional connection
```

For ordinary project data:

```text
HTTP query
```

is perfectly appropriate.

Use real-time transport where real-time behavior actually matters.

---

# 71. What if everything is optimistic?

You can create extremely responsive UI.

But now you have to handle:

```text
rollback
conflicts
duplicate operations
server rejection
ordering
canonical state
```

Optimistic UI is a strategic optimization, not the default for every mutation.

---

# 72. What if everything becomes a Server Component?

Then highly interactive behavior becomes awkward:

```text
dragging
local drafts
browser APIs
event handlers
```

still need client execution.

Server Components are not "better components."

They are components with a different execution environment.

---

# 73. What if everything becomes a Client Component?

Then you lose opportunities for:

```text
server-only dependencies
server-side data access
reduced client JS
server rendering architecture
```

Again:

```text
choose boundary based on responsibility
```

---

# 74. Architecture is controlled trade-offs

There isn't one magic stack.

For the same feature, a different application might reasonably choose:

```text
Router loader
instead of
TanStack Query
```

or:

```text
regular HTTP mutation
instead of
Server Function
```

or:

```text
polling
instead of
WebSocket
```

The skill you've gained is knowing what problem each solves.

---

# 75. Capstone decision tree

When adding a feature, ask:

```text
Is this URL-visible?
      ↓ yes → URL

Is this server-owned data?
      ↓ yes → query/server data mechanism

Is this local UI state?
      ↓ yes → useState/useReducer

Does a subtree share it?
      ↓ yes → Context

Does an external system own it?
      ↓ yes → useSyncExternalStore

Is this form submission lifecycle?
      ↓ yes → Action

Does code need server execution?
      ↓ yes → Server Function/API

Needs live updates?
      ↓ yes → WebSocket/SSE

Needs delayed/async rendering?
      ↓ yes → Suspense

Needs lower-priority UI work?
      ↓ yes → transition/deferred value
```

That's a reusable architecture tool.

---

# 76. The capstone doesn't end at code

A production feature isn't finished when:

```text
"it works locally"
```

It needs:

```text
correctness
accessibility
security
testing
observability
performance
deployment
rollback strategy
```

For a feature like task creation:

```text
Build
 ↓
Test
 ↓
Secure
 ↓
Observe
 ↓
Deploy
 ↓
Measure
 ↓
Improve
```

That's the actual engineering loop.

---

# 77. Your complete React journey

You began with:

```jsx
function App() {
  return <h1>Hello</h1>;
}
```

Then gradually learned:

```text
components
 ↓
JSX
 ↓
state
 ↓
events
 ↓
forms
 ↓
effects
 ↓
refs
 ↓
context
 ↓
reducers
 ↓
custom hooks
 ↓
rendering/reconciliation
 ↓
composition
 ↓
routing
 ↓
URL state
 ↓
data fetching
 ↓
server state
 ↓
mutations
 ↓
cache consistency
 ↓
memoization/performance
 ↓
portals
 ↓
error boundaries
 ↓
Suspense
 ↓
transitions
 ↓
accessibility
 ↓
testing
 ↓
TypeScript
 ↓
external stores
 ↓
Actions
 ↓
SSR
 ↓
Server Components
 ↓
Server Functions
 ↓
production architecture
 ↓
deployment
 ↓
security
 ↓
observability
 ↓
full-stack architecture
```

That's a very different level of understanding from merely knowing React hooks.

---

# 78. The final architecture principle

When a new requirement arrives tomorrow:

> "We need collaborative comments."

Don't immediately think:

```text
"Which hook should I use?"
```

Think:

```text
Who owns comments?
Where is canonical truth?
How are comments loaded?
How are they mutated?
Who is authorized?
Do updates need to be live?
What is temporary?
What must be in the URL?
What happens offline?
How do we test it?
How do we observe failures?
```

Then choose the React mechanisms.

That is the difference between:

```text
using React
```

and:

```text
engineering with React
```

---

# 79. Capstone architecture in one screen

```text
                           ORBIT
                             │
             ┌───────────────┼────────────────┐
             │               │                │
           BROWSER          SERVER         EXTERNAL
             │               │                │
          React UI        Auth/Rules       WebSocket
             │               │                │
          Router           Services          │
             │               │                │
         URL state          DB               │
             │               │                │
      ┌──────┴───────┐      │          External Store
      │              │      │                │
   Client UI     Query Cache │                │
      │              │      │                │
 useState        TanStack   │                │
 useReducer       Query     │                │
 Context                     │                │
      │                       │               │
      └──────────┬────────────┘               │
                 │                            │
              Actions                         │
                 │                            │
           Server Functions                   │
                 │                            │
        authenticate/validate                 │
             /authorize                       │
                 │                            │
              Database                        │
                 │                            │
                 └──────────────┬─────────────┘
                                │
                              React
                                │
                          render / commit
                                │
                                ▼
                                UI

      + SSR + Suspense + Transitions
      + Testing + Security
      + Performance + Observability
      + Deployment
```

That is the capstone.

---

# 80. One final rule to keep forever

When React gets complicated, **don't add more technology first**.

First identify:

```text
state
ownership
lifetime
trust
data flow
failure mode
```

Then choose the smallest mechanism that naturally fits.

That principle will remain useful even when the React ecosystem changes.

---

## Next lesson — the final review

The next lesson is the final one: **React Master Review**.

We'll compress the entire course into a set of powerful mental models:

```text
rendering
state
effects
identity
routing
server state
async work
concurrency
server/client boundaries
security
testing
performance
architecture
```

and then go through the mistakes that experienced React developers still make, plus the questions that tell you whether you truly understand a React system rather than merely remembering its APIs.