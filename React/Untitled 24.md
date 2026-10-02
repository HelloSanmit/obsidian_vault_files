
# Lesson 20 — Mutations: Changing Server State

So far we've mostly talked about **reading** server state:

```text
GET
 ↓
useQuery
 ↓
cache
 ↓
UI
```

But applications are not read-only.

Users create things, edit things, delete things, like things, complete tasks, upload files, change settings, and so on.

That gives us a second major concept:

```text
Queries     → read server state
Mutations   → change server state
```

TanStack Query has a separate `useMutation` abstraction for this. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/mutations?utm_source=chatgpt.com "Mutations | TanStack Query React Docs"))

And this distinction is much deeper than simply:

> "`useQuery` for GET, `useMutation` for POST."

The interesting part is what happens **after the server changes**.

---

# 1. Start with an ordinary mutation

Suppose our backend has:

```http
POST /api/todos
```

We want to create a todo.

```jsx
import { useMutation } from "@tanstack/react-query";

function AddTodo() {
  const mutation = useMutation({
    mutationFn: createTodo,
  });

  function handleSubmit() {
    mutation.mutate({
      title: "Learn React",
    });
  }

  // ...
}
```

And:

```jsx
async function createTodo(todo) {
  const response = await fetch("/api/todos", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(todo),
  });

  if (!response.ok) {
    throw new Error("Could not create todo");
  }

  return response.json();
}
```

Notice an important difference from `useQuery`.

With a query:

```jsx
useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
});
```

we are describing:

> "Here is a piece of server data that this component wants to observe."

With a mutation:

```jsx
useMutation({
  mutationFn: createTodo,
});
```

we are describing:

> "Here is an operation that changes something on the server."

The mutation doesn't automatically mean:

> "Run this request now."

You explicitly trigger it:

```jsx
mutation.mutate({
  title: "Learn React",
});
```

---

# 2. The mutation has its own lifecycle

Just like queries have state, mutations have state.

Conceptually:

```text
idle
 ↓
pending
 ↓
success
```

or:

```text
idle
 ↓
pending
 ↓
error
```

So:

```jsx
mutation.isPending
mutation.isSuccess
mutation.isError
mutation.error
mutation.data
```

can describe what's happening.

A form might therefore do:

```jsx
<button disabled={mutation.isPending}>
  {mutation.isPending ? "Saving..." : "Save"}
</button>
```

The flow is:

```text
User clicks Save
       ↓
mutation.mutate(...)
       ↓
pending
       ↓
request
       ↓
server
       ↓
success/error
       ↓
UI reacts
```

This part is straightforward.

The interesting problem comes next.

---

# 3. The server changed, but your cache didn't

Imagine your application currently has:

```text
GET /api/todos
```

and the cache contains:

```js
[
  { id: 1, title: "Learn React" },
  { id: 2, title: "Build app" }
]
```

Then the user creates:

```text
"Study TypeScript"
```

The server now has:

```js
[
  { id: 1, title: "Learn React" },
  { id: 2, title: "Build app" },
  { id: 3, title: "Study TypeScript" }
]
```

But your React Query cache might still contain:

```js
[
  { id: 1, title: "Learn React" },
  { id: 2, title: "Build app" }
]
```

Why?

Because the mutation changed the **server**.

It didn't magically rewrite every cached query that might be affected.

So now we have:

```text
SERVER
   ✅ new data

CACHE
   ❌ old data
```

We need a synchronization strategy.

---

# 4. The simplest strategy: invalidate the related query

This is one of the most important patterns in TanStack Query.

```jsx
import {
  useMutation,
  useQueryClient,
} from "@tanstack/react-query";

function AddTodo() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: createTodo,

    onSuccess: async () => {
      await queryClient.invalidateQueries({
        queryKey: ["todos"],
      });
    },
  });

  // ...
}
```

What does this mean?

After successfully creating a todo:

```text
mutation succeeds
       ↓
todos cache is now potentially outdated
       ↓
mark todos query stale
       ↓
active observers can refetch
       ↓
fresh server data arrives
       ↓
UI updates
```

Current TanStack Query's `invalidateQueries` marks matching queries stale, overriding their usual freshness configuration, and active queries can refetch in the background. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation?utm_source=chatgpt.com "Query Invalidation | TanStack Query React Docs"))

This pattern is specifically recommended for queries affected by successful mutations. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/invalidations-from-mutations?utm_source=chatgpt.com "Invalidations from Mutations | TanStack Query React Docs"))

---

# 5. What "invalidate" actually means

The word can be misleading.

It does **not** mean:

```text
delete cache
```

And it does not simply mean:

```text
immediately fetch everything
```

Think:

> "I no longer trust this cached result to be fresh."

For example:

```text
before mutation

["todos"]
   ↓
fresh
```

Then:

```text
mutation succeeds
       ↓
invalidate ["todos"]
       ↓
stale
```

If a component is currently observing that query, TanStack Query can refetch it.

So:

```text
invalidate
```

is fundamentally a statement about **freshness**.

---

# 6. Why invalidation is often better than manually updating everything

Suppose you have these queries:

```text
["todos"]
["todos", { status: "open" }]
["todos", { status: "completed" }]
["todo", 42]
["todo", 81]
```

You create a new todo.

Which cache entries are affected?

Potentially several.

For example:

```text
all todos
open todos
```

but not necessarily:

```text
completed todos
```

and certainly not unrelated:

```text
["users"]
["notifications"]
```

TanStack Query lets you target a family of queries through their key structure:

```jsx
queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

A prefix such as `["todos"]` can match queries beginning with that key. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation?utm_source=chatgpt.com "Query Invalidation | TanStack Query React Docs"))

So query-key design isn't just about fetching.

It also becomes your **cache organization system**.

---

# 7. Think of query keys as folders

This analogy is useful.

Imagine:

```text
Cache
│
├── todos
│   ├── all
│   ├── open
│   ├── completed
│   └── todo/42
│
├── products
│   ├── electronics/page1
│   ├── electronics/page2
│   └── books/page1
│
└── users
    ├── user/1
    └── user/2
```

Then:

```jsx
invalidateQueries({
  queryKey: ["todos"],
});
```

is conceptually saying:

> "Something happened that may have affected this whole family."

Whereas:

```jsx
invalidateQueries({
  queryKey: ["todos", { status: "open" }],
});
```

targets something more specific. Current query matching supports this hierarchical/prefix-style matching. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation?utm_source=chatgpt.com "Query Invalidation | TanStack Query React Docs"))

This is one reason consistent query-key design matters so much.

---

# 8. Invalidation is often enough

Let's say:

```text
POST /todos
```

returns:

```json
{
  "id": 3,
  "title": "Study TypeScript",
  "completed": false
}
```

You have two broad options.

### Option A — invalidate

```jsx
onSuccess: async () => {
  await queryClient.invalidateQueries({
    queryKey: ["todos"],
  });
}
```

Then let the server send the canonical list.

### Option B — directly update the cache

Use the response:

```jsx
onSuccess: (newTodo) => {
  queryClient.setQueryData(
    ["todo", newTodo.id],
    newTodo
  );
}
```

TanStack Query explicitly supports updating an existing cached query from a mutation response with `setQueryData`. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/updates-from-mutation-responses?utm_source=chatgpt.com "Updates from Mutation Responses | TanStack Query React Docs"))

So which should you use?

The important conceptual difference is:

```text
invalidate
    =
"the server has authoritative data; go get it again"

setQueryData
    =
"I already know exactly what the new data should be"
```

---

# 9. `setQueryData` is a direct cache write

Suppose the cache contains:

```js
{
  id: 42,
  title: "Learn React",
  completed: false
}
```

Mutation returns:

```js
{
  id: 42,
  title: "Learn React",
  completed: true
}
```

We can write:

```jsx
queryClient.setQueryData(
  ["todo", 42],
  updatedTodo
);
```

Now the cached query immediately becomes:

```js
{
  id: 42,
  title: "Learn React",
  completed: true
}
```

No additional GET request is required just to discover information we already received.

Current TanStack Query supports this mutation-response → cache update pattern. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/updates-from-mutation-responses?utm_source=chatgpt.com "Updates from Mutation Responses | TanStack Query React Docs"))

---

# 10. But cache updates must remain immutable

This is important because it connects directly to our earlier React state lessons.

Don't do:

```jsx
queryClient.setQueryData(
  ["todo", 42],
  oldTodo => {
    oldTodo.completed = true;
    return oldTodo;
  }
);
```

You're mutating the existing cached object.

Instead:

```jsx
queryClient.setQueryData(
  ["todo", 42],
  oldTodo =>
    oldTodo
      ? {
          ...oldTodo,
          completed: true,
        }
      : oldTodo
);
```

You're creating a new object.

The same mental model from React state applies:

```text
old object
    ↓
copy
    ↓
change desired property
    ↓
new object
```

TanStack Query's current guidance also requires immutable cache updates. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/updates-from-mutation-responses?utm_source=chatgpt.com "Updates from Mutation Responses | TanStack Query React Docs"))

---

# 11. Lists make direct updates more interesting

Suppose:

```text
["todos"]
```

contains:

```js
[
  { id: 1, title: "Learn React", completed: false },
  { id: 2, title: "Build app", completed: false }
]
```

We toggle todo 1.

The mutation response is:

```js
{
  id: 1,
  title: "Learn React",
  completed: true
}
```

We can update the list:

```jsx
queryClient.setQueryData(
  ["todos"],
  oldTodos => {
    if (!oldTodos) {
      return oldTodos;
    }

    return oldTodos.map(todo =>
      todo.id === updatedTodo.id
        ? updatedTodo
        : todo
    );
  }
);
```

Notice how many concepts from our earlier JavaScript/React lessons appear here:

```text
setQueryData
   ↓
callback
   ↓
old cache
   ↓
map
   ↓
object identity
   ↓
new array
```

This is why your earlier work with `map`, immutability, callbacks, and state updates wasn't separate trivia.

They're all coming back now.

---

# 12. What about multiple representations of the same entity?

This is where things get genuinely interesting.

Suppose we have:

```text
["todo", 42]
```

and:

```text
["todos"]
```

The same todo may exist in both.

And maybe:

```text
["todos", { status: "open" }]
```

contains it too.

Now imagine:

```text
todo 42 changes
```

Updating only:

```text
["todo", 42]
```

doesn't automatically mean every list containing that todo has been rewritten.

This is one of the fundamental complexities of cached server state.

You can choose to:

```text
invalidate related queries
```

or:

```text
manually update the relevant cache entries
```

or combine both.

This is a deliberate architectural choice.

---

# 13. Why TanStack Query doesn't behave like a normalized database cache

Some systems try to maintain one canonical entity:

```text
Todo #42
```

and automatically update every place where that entity appears.

TanStack Query doesn't require you to build that kind of normalized entity graph.

Instead, its tools are roughly:

```text
query keys
+
invalidation
+
direct cache updates
+
refetching
```

That approach is often simpler because you don't need to teach the cache how every possible relationship in your application works. TanStack Query itself describes targeted invalidation and atomic cache updates as an alternative to maintaining a normalized cache. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/query-invalidation?utm_source=chatgpt.com "Query Invalidation | TanStack Query React Docs"))

---

# 14. Now we reach optimistic UI

This is one of the most useful ideas in modern applications.

Imagine a todo:

```text
☐ Learn React
```

User clicks it.

A conventional application might do:

```text
click
 ↓
send request
 ↓
wait
 ↓
success
 ↓
checkbox becomes checked
```

That can feel sluggish.

Instead, we can optimistically say:

```text
click
 ↓
immediately show checked
 ↓
send request in background
```

The assumption is:

> "The request will probably succeed."

That's **optimistic UI**.

---

# 15. The optimistic update timeline

Suppose initial cache is:

```js
{
  id: 42,
  completed: false
}
```

User clicks.

We immediately change the cache:

```js
{
  id: 42,
  completed: true
}
```

Then:

```text
request succeeds
    ↓
keep it

request fails
    ↓
restore false
```

So:

```text
             User action
                  ↓
          optimistic update
                  ↓
             UI changes
                  ↓
             server request
              ↙        ↘
           success      error
             ↓            ↓
          keep it      rollback
```

That rollback is the key part.

Without rollback, optimistic UI can lie permanently.

---

# 16. `onMutate`: the optimistic phase

A typical optimistic mutation has three conceptual jobs:

### 1. Stop conflicting refetches

```text
"Don't let some old response overwrite my optimistic change."
```

### 2. Save the old value

```text
"Remember what things looked like before."
```

### 3. Apply the optimistic value

```text
"Change the cache immediately."
```

Current TanStack Query's mutation lifecycle supports this pattern through `onMutate`, with rollback information returned from it. ([TanStack](https://old.tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates?utm_source=chatgpt.com "Optimistic Updates | TanStack Query React Docs"))

A simplified example:

```jsx
const mutation = useMutation({
  mutationFn: updateTodo,

  onMutate: async updatedTodo => {
    await queryClient.cancelQueries({
      queryKey: ["todo", updatedTodo.id],
    });

    const previousTodo =
      queryClient.getQueryData([
        "todo",
        updatedTodo.id,
      ]);

    queryClient.setQueryData(
      ["todo", updatedTodo.id],
      updatedTodo
    );

    return { previousTodo };
  },
});
```

Read this slowly.

---

# 17. What `cancelQueries` is protecting us from

Imagine this:

```text
GET /todo/42
```

is currently fetching.

At the same time:

```text
User clicks complete
```

We optimistically change:

```js
completed: false
```

to:

```js
completed: true
```

But the old GET request might finish afterward with:

```js
completed: false
```

Then:

```text
optimistic update
      ↓
true
      ↓
old GET finishes
      ↓
false
```

Our optimistic result just got overwritten.

That's why optimistic updates may cancel relevant in-flight queries first. Current TanStack Query's optimistic-update pattern explicitly does this. ([TanStack](https://old.tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates?utm_source=chatgpt.com "Optimistic Updates | TanStack Query React Docs"))

---

# 18. Save the previous state

We need rollback information:

```jsx
const previousTodo =
  queryClient.getQueryData([
    "todo",
    updatedTodo.id,
  ]);
```

Imagine:

```js
previousTodo = {
  id: 42,
  completed: false
}
```

Then optimistic update:

```js
{
  id: 42,
  completed: true
}
```

If request fails:

```jsx
onError: (error, variables, context) => {
  queryClient.setQueryData(
    ["todo", variables.id],
    context.previousTodo
  );
}
```

We restore:

```js
false
```

---

# 19. `onSettled`: reconcile with reality

After success **or** failure, we often invalidate:

```jsx
onSettled: async () => {
  await queryClient.invalidateQueries({
    queryKey: ["todo", updatedTodo.id],
  });
}
```

Why do this if we already updated the cache?

Because the server is still the final authority.

Perhaps:

```text
you thought result = completed: true
```

but the server actually applies some business rule and returns:

```js
{
  completed: false,
  reason: "Task is locked"
}
```

Or perhaps another user changed the record simultaneously.

So a useful pattern is:

```text
optimistic update
        ↓
fast UI
        ↓
server request
        ↓
rollback on failure
        ↓
refetch/reconcile with server
```

This gives both responsiveness and correctness.

TanStack Query's current optimistic-update examples use invalidation after the mutation settles to reconcile the cache with server state. ([TanStack](https://old.tanstack.com/query/v5/docs/framework/react/guides/optimistic-updates?utm_source=chatgpt.com "Optimistic Updates | TanStack Query React Docs"))

---

# 20. But optimistic updates aren't always appropriate

Suppose you're deleting a bank transaction.

Or submitting an important legal form.

Or performing a complex server-side calculation.

You may not want:

```text
pretend success
```

until the server confirms it.

Optimistic updates are easiest when:

```text
operation is predictable
+
rollback is straightforward
+
failure is relatively uncommon
```

They're harder when:

```text
server logic is complicated
+
many things can fail
+
rollback affects many pieces of state
```

So optimistic UI is a tool, not a default requirement.

---

# 21. A much simpler optimistic technique

There's actually a simpler approach when only the **current component** needs to show the optimistic state.

Suppose you add a todo.

You don't necessarily need to modify the global cache immediately.

The mutation itself knows:

```jsx
mutation.variables
mutation.isPending
```

You can render the submitted item temporarily:

```jsx
{todos.map(todo => (
  <Todo key={todo.id} todo={todo} />
))}

{mutation.isPending && (
  <Todo
    todo={{
      id: "temporary",
      title: mutation.variables.title,
      completed: false,
    }}
  />
)}
```

Current TanStack Query v5 also supports this UI-oriented optimistic approach using mutation variables, which can be simpler when only one part of the UI needs the optimistic representation. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/migrating-to-v5?utm_source=chatgpt.com "Migrating to TanStack Query v5 | TanStack Query React Docs"))

The conceptual distinction is:

```text
UI-only optimistic state
        vs
cache-level optimistic state
```

Don't modify the entire cache when you only need a temporary visual effect in one component.

---

# 22. Mutation callbacks have useful roles

You've seen:

```jsx
onSuccess
onError
onSettled
onMutate
```

Don't memorize them as arbitrary API names.

Think in terms of the timeline:

```text
before request
     ↓
   onMutate
     ↓
 mutation running
     ↓
   success ─────────→ onSuccess
     │
     └─ error ───────→ onError
                \
                 → onSettled
```

So:

```text
onMutate
   =
prepare / optimistic work

onSuccess
   =
server accepted it

onError
   =
server/request failed

onSettled
   =
the operation is over, regardless of outcome
```

That mental model is more useful than memorizing method signatures.

---

# 23. A realistic update flow

Let's build a proper "complete todo" operation.

```jsx
function TodoItem({ todo }) {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: updateTodo,

    onMutate: async updatedTodo => {
      await queryClient.cancelQueries({
        queryKey: ["todo", updatedTodo.id],
      });

      const previousTodo =
        queryClient.getQueryData([
          "todo",
          updatedTodo.id,
        ]);

      queryClient.setQueryData(
        ["todo", updatedTodo.id],
        updatedTodo
      );

      return {
        previousTodo,
      };
    },

    onError: (error, variables, context) => {
      queryClient.setQueryData(
        ["todo", variables.id],
        context.previousTodo
      );
    },

    onSettled: async (data, error, variables) => {
      await queryClient.invalidateQueries({
        queryKey: ["todo", variables.id],
      });
    },
  });

  function handleToggle() {
    mutation.mutate({
      ...todo,
      completed: !todo.completed,
    });
  }

  return (
    <button onClick={handleToggle}>
      {todo.completed ? "✅" : "⬜"} {todo.title}
    </button>
  );
}
```

Now don't just read the syntax.

Trace the flow.

---

# 24. Execution trace

Initial:

```text
todo.completed = false
```

User clicks:

```text
handleToggle()
```

creates:

```js
{
  id: 42,
  completed: true
}
```

Then:

```text
mutation.mutate(...)
```

fires.

### `onMutate`

We cancel an existing query:

```text
GET todo 42
```

Then save:

```text
previousTodo = false
```

Then cache becomes:

```text
true
```

React sees the changed cached data.

UI immediately becomes:

```text
✅ Learn React
```

No server response was needed to produce that visual change.

---

### Network request

Server receives:

```http
PUT /api/todos/42
```

Now two possibilities.

### Success

```text
server → success
        ↓
onSuccess, if defined
        ↓
onSettled
        ↓
invalidate ["todo", 42]
        ↓
fresh server data
```

### Failure

```text
server → error
        ↓
onError
        ↓
restore previousTodo
        ↓
false
        ↓
onSettled
        ↓
invalidate
```

The UI therefore behaves like:

```text
click
 ↓
instant change
 ↓
request
 ↓
server decides reality
```

That's optimistic UI.

---

# 25. What happens to the list query?

Here's a subtle problem.

We updated:

```text
["todo", 42]
```

But perhaps our page also has:

```text
["todos"]
```

The list could still contain:

```js
{
  id: 42,
  completed: false
}
```

while the individual query says:

```js
{
  id: 42,
  completed: true
}
```

Now our cache has inconsistent representations.

This is where invalidation becomes useful.

We might do:

```jsx
await queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

as well.

Then:

```text
["todo", 42]
      ↓
updated

["todos"]
      ↓
marked stale
      ↓
refetch
```

This is why mutations often affect **multiple query keys**, not just one.

---

# 26. The mutation itself does not know what should be refreshed

This is an architectural point worth understanding.

TanStack Query knows:

```text
mutation function succeeded
```

It does **not automatically know the complete business meaning** of that mutation.

For example:

```text
updateUser(42)
```

might affect:

```text
["user", 42]
["users"]
["team-members", 7]
["search-users", "alice"]
["dashboard"]
```

The library cannot magically know every relationship.

**You** define which cached views are now potentially stale.

That's why query-key design and invalidation strategy are part of application architecture.

---

# 27. Mutation variables are different from query keys

Another distinction:

For a query:

```jsx
useQuery({
  queryKey: ["todo", id],
  queryFn: () => fetchTodo(id),
});
```

`id` determines **which server data we're observing**.

For a mutation:

```jsx
mutation.mutate({
  id,
  completed: true,
});
```

the variables describe **what operation we want performed**.

So:

```text
query
  key → identity of data

mutation
  variables → input to an operation
```

These are related, but they're not interchangeable concepts.

---

# 28. Mutation does not mean "refresh everything"

This is an important anti-pattern.

Imagine:

```jsx
onSuccess: () => {
  queryClient.invalidateQueries();
}
```

Technically, that's possible.

But conceptually you're saying:

> "Something changed somewhere, therefore every cached piece of server state should now be considered stale."

That's often unnecessarily broad.

Better:

```jsx
queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

or:

```jsx
queryClient.invalidateQueries({
  queryKey: ["todo", variables.id],
});
```

Use the knowledge you have about what the mutation actually affected.

---

# 29. A mutation architecture we can now recognize

A realistic application might have:

```text
                 SERVER
                   │
        ┌──────────┴───────────┐
        ↓                      ↓
      queries              mutations
        ↓                      ↓
      cache                operation
        │                      │
        │                ┌─────┴─────┐
        │                ↓           ↓
        │             success      failure
        │                ↓           ↓
        │             update/      rollback
        │             invalidate
        │                │
        └────────────────┘
                 ↓
                UI
```

The server is still the authority.

The cache is a fast, temporary representation.

The mutation is the mechanism for requesting a change.

Invalidation/cache updates keep the representation aligned with reality.

---

# 30. The most important mental model from this lesson

Don't think:

```text
mutation
↓
request
↓
done
```

Think:

```text
USER ACTION
    ↓
MUTATION
    ↓
SERVER CHANGES
    ↓
WHICH CACHED DATA IS NOW SUSPECT?
    ↓
┌───────────────┬──────────────────┐
↓               ↓                  ↓
invalidate   update cache      optimistic UI
↓               ↓                  ↓
refetch       immediate        rollback if needed
               representation
        └──────────┬───────────────┘
                   ↓
                  UI
```

That is the real problem mutations solve.

---

# 31. One more connection to everything we've learned

Look at how many earlier React concepts are hiding inside this one lesson:

```text
useMutation
   ↓
callbacks
   ↓
closures
   ↓
query client
   ↓
immutable updates
   ↓
map
   ↓
object identity
   ↓
render
   ↓
derived UI
```

And our earlier state taxonomy appears too:

```text
Form input
    → local state

page/filter
    → URL state

todos
    → server state

theme
    → context

isSaving
    → mutation state

filteredTodos
    → derived data
```

This is why I'm not going to teach React as a collection of APIs.

We're gradually building a model of **where information lives, who owns it, how it changes, and how the UI stays synchronized with it**.

---

## A good exercise

Try mentally implementing a task app with:

```text
GET    /todos
POST   /todos
PATCH  /todos/:id
DELETE /todos/:id
```

For each operation, determine:

```text
What is the mutation input?

Which query keys become stale?

Can the mutation response directly update a cached query?

Should the UI be optimistic?

What happens if the request fails?

Which data can safely remain visible while the request is running?
```

Once you can answer those without thinking primarily about syntax, you've started understanding server-state architecture rather than just TanStack Query.

The next interesting step is to go one level deeper into **query/mutation consistency**: editing and deleting items inside paginated/filterable lists, cache updates across multiple query keys, and the tricky cases where optimistic updates collide with refetches and concurrent mutations.