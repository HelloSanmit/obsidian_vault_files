
# Lesson 21 — Keeping Server Data Consistent Across Lists, Details, Filters, and Pagination

The previous lesson taught us:

```text
mutation
   ↓
server changes
   ↓
cache may now be stale
   ↓
invalidate / update / optimistic update
```

Now let's tackle a problem that appears in real applications almost immediately:

> **The same piece of data can appear in several different queries at the same time.**

This is where server-state management becomes genuinely interesting.

---

## 1. One todo can exist in many places

Suppose our application has:

```text
Todo #42
```

It might appear in:

```text
Todo detail page
Todo list
"Open" todos
Search results
User's recent todos
Dashboard
```

That could mean our cache contains:

```text
["todo", 42]

["todos"]

["todos", { status: "open" }]

["todos", { search: "react" }]

["recentTodos", 7]

["dashboard"]
```

All of these may contain information about the same underlying server entity.

Now the user edits Todo #42.

The server changes.

Which cached queries are now wrong?

Potentially many.

---

# 2. This is the central consistency problem

Imagine the old server state is:

```js
{
  id: 42,
  title: "Learn React",
  completed: false
}
```

The user changes the title to:

```text
Learn React deeply
```

The server accepts it.

Now:

```text
SERVER
{
  id: 42,
  title: "Learn React deeply",
  completed: false
}
```

But suppose the cache currently contains:

```text
["todo", 42]
→ "Learn React"

["todos"]
→ "Learn React"

["todos", { status: "open" }]
→ "Learn React"

["search", "react"]
→ "Learn React"
```

Now we have a consistency problem.

---

# 3. The easiest answer is often invalidation

We can say:

```jsx
await queryClient.invalidateQueries({
  queryKey: ["todo", 42],
});

await queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

Conceptually:

```text
mutation
   ↓
known affected query families
   ↓
mark stale
   ↓
refetch active queries
   ↓
server becomes source of truth
```

This is often much simpler than trying to manually find and rewrite every cached copy.

---

# 4. But broad invalidation has a cost

Suppose you have 100 different todo queries.

You do:

```jsx
invalidateQueries({
  queryKey: ["todos"],
});
```

That may make many related queries stale.

That's often fine.

But imagine an enormous application with:

```text
hundreds of filters
many paginated views
many dashboards
many users
```

Now you start thinking about precision.

Which queries actually need updating?

This is where **cache design** becomes an architectural decision rather than simply an API call.

---

# 5. Direct cache update for the detail query

Suppose our mutation returns the complete updated todo:

```js
{
  id: 42,
  title: "Learn React deeply",
  completed: false
}
```

Then:

```jsx
onSuccess: updatedTodo => {
  queryClient.setQueryData(
    ["todo", updatedTodo.id],
    updatedTodo
  );
}
```

That's perfect for:

```text
["todo", 42]
```

because we know exactly what the new value is.

No GET request is needed just to discover the result we already received.

---

# 6. But the list is trickier

Suppose:

```text
["todos"]
```

contains:

```js
[
  {
    id: 42,
    title: "Learn React",
    completed: false
  },
  {
    id: 43,
    title: "Build app",
    completed: false
  }
]
```

We could update it:

```jsx
queryClient.setQueryData(
  ["todos"],
  oldTodos => {
    if (!oldTodos) return oldTodos;

    return oldTodos.map(todo =>
      todo.id === updatedTodo.id
        ? updatedTodo
        : todo
    );
  }
);
```

Now the list immediately contains:

```text
Learn React deeply
```

No additional request.

---

# 7. But what if the item no longer belongs in the list?

This is where things get more interesting.

Suppose our list is:

```text
["todos", { status: "open" }]
```

and Todo #42 was:

```js
{
  id: 42,
  completed: false
}
```

Now the user marks it completed.

The server returns:

```js
{
  id: 42,
  completed: true
}
```

What should happen to the **open-todos list**?

We cannot simply replace the item with the updated object.

The item may no longer belong in this query at all.

So this:

```jsx
.map(...)
```

isn't enough.

We may need:

```text
remove item
```

instead.

For example:

```jsx
queryClient.setQueryData(
  ["todos", { status: "open" }],
  oldTodos => {
    if (!oldTodos) return oldTodos;

    return oldTodos.filter(
      todo =>
        todo.id !== updatedTodo.id
    );
  }
);
```

Now the item disappears from the open list.

---

# 8. This is why invalidation can be safer

Consider how much logic we'd need to maintain manually:

```text
all todos
open todos
completed todos
search results
page 1
page 2
page 3
```

An update might affect all of them differently.

Manual cache synchronization becomes increasingly complicated.

So a common strategy is:

```text
update detail cache directly
+
invalidate list queries
```

Meaning:

```text
"I know exactly what the individual object is now."

"But I don't want to manually reason about
every list/filter/pagination representation."
```

That's a very practical compromise.

---

# 9. Pagination makes the problem even more interesting

Suppose:

```text
["todos", { page: 1 }]
["todos", { page: 2 }]
["todos", { page: 3 }]
```

Todo #42 currently lives on page 2.

You update it.

Do you know whether it should remain on page 2?

Maybe the list is sorted by:

```text
lastUpdated
```

Then editing the todo might move it from page 2 to page 1.

Or perhaps filtering depends on some field that changed.

So a direct cache update like:

```jsx
map(...)
```

could actually leave your pagination inconsistent.

This is one of the strongest reasons to understand **what each query represents**, rather than thinking of the cache as one giant object.

---

# 10. Query data is not just "some data"

A useful way to think about a query is:

```text
query key
    ↓
definition of a particular server view
```

For example:

```text
["todos"]
```

might mean:

> all todos

while:

```text
["todos", { status: "open" }]
```

means:

> todos satisfying the open filter

and:

```text
["todos", { status: "open", page: 2 }]
```

means:

> page 2 of the open-todo result set

These aren't simply different arrays.

They're different **server-derived views**.

That's why mutations can affect them differently.

---

# 11. Filtering creates a particularly important consequence

Suppose:

```text
query:
["todos", { search: "react" }]
```

and the list contains:

```text
Learn React
React Router
Build a web app
```

Now the user edits:

```text
"Learn React"
```

to:

```text
"Learn Vue"
```

The item may no longer match the query.

So again, direct replacement isn't necessarily correct.

The correct resulting list might be:

```text
React Router
Build a web app
```

The mutation changed the **membership of the result set**.

This is a major idea:

> A mutation can change not only an object's contents, but whether that object belongs to a particular query result.

---

# 12. Deleting is conceptually easier

Suppose:

```text
DELETE /todos/42
```

succeeds.

For a detail query:

```text
["todo", 42]
```

we probably don't want to keep pretending the object exists.

We might remove the cached query:

```jsx
queryClient.removeQueries({
  queryKey: ["todo", 42],
});
```

Then invalidate the lists:

```jsx
queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

Conceptually:

```text
DELETE succeeds
       ↓
individual entity no longer exists
       ↓
remove detail representation
       ↓
list representations may contain deleted item
       ↓
invalidate relevant lists
```

Notice the division of responsibilities.

---

# 13. Create has the opposite problem

Suppose:

```text
POST /todos
```

creates:

```js
{
  id: 99,
  title: "Study React Query",
  completed: false
}
```

Which lists should contain it?

Maybe:

```text
["todos"]
```

yes.

```text
["todos", { status: "open" }]
```

yes.

```text
["todos", { status: "completed" }]
```

no.

```text
["todos", { search: "react" }]
```

yes.

```text
["todos", { search: "database" }]
```

no.

```text
["todos", { page: 1 }]
```

maybe.

It depends on sorting, pagination, and filtering.

So you can see why simply saying:

> "Whenever mutation succeeds, append the item to the array."

is not generally correct.

---

# 14. This is where server authority becomes extremely useful

Instead of pretending we can predict every consequence perfectly:

```text
mutation
 ↓
invalidate affected query family
 ↓
server recomputes the correct result
 ↓
client gets authoritative representation
```

That is often the safest model.

The cache becomes:

> a temporary performance optimization and UI representation

rather than:

> the ultimate source of truth.

This is a very healthy mental model for server state.

---

# 15. Optimistic updates make these problems harder

Suppose you optimistically delete Todo #42.

The user clicks:

```text
Delete
```

and the UI immediately removes it.

Great.

But then the server says:

```text
403 Forbidden
```

because the user isn't allowed to delete it.

Now we need to restore it.

But where?

```text
["todo", 42]
["todos"]
["todos", { status: "open" }]
["todos", { page: 2 }]
```

Potentially several places.

This is why optimistic updates become much more complicated when the same entity appears in many cached views.

---

# 16. A useful optimistic-update boundary

One good strategy is to keep the optimistic operation as narrow as possible.

For example:

```text
detail page
```

changes immediately:

```text
Completed ✅
```

Then:

```text
server request
   ↓
success?
```

After success, invalidate lists.

That can be much easier than trying to optimistically rewrite every cached representation.

The broader principle is:

> **Optimistic updates are easiest when you know exactly which UI you are making optimistic.**

---

# 17. Concurrent mutations

Now let's make things harder.

Suppose the user rapidly performs:

```text
toggle Todo #42
toggle Todo #42
toggle Todo #42
```

Or perhaps two different UI components update the same todo.

Now there may be:

```text
mutation A
mutation B
mutation C
```

all in flight.

The server might process them in an order different from what you intuitively expect.

This is where optimistic state gets difficult.

You can't simply assume:

```text
latest UI click
=
latest server result
```

Those are different timelines.

---

# 18. Client timeline vs server timeline

Think about this:

```text
Client timeline

A starts
   ↓
B starts
   ↓
C starts
```

But the server might complete:

```text
B
 ↓
A
 ↓
C
```

or:

```text
C
 ↓
A
 ↓
B
```

depending on the system.

This is why mutation design sometimes requires thinking about:

```text
ordering
idempotency
conflicts
server-side validation
```

We're reaching beyond just React now and into distributed-system behavior.

That's an important transition.

---

# 19. React isn't the source of these problems

Notice something.

The difficulty here isn't caused by JSX.

It's caused by:

```text
network
+
concurrency
+
caching
+
server authority
+
multiple representations
```

React gives us the UI machinery.

TanStack Query gives us server-state machinery.

But the underlying problem is fundamentally:

> **How do two systems—the browser and the server—stay reasonably consistent while time is passing and requests are happening asynchronously?**

That's why server-state management is such a deep topic.

---

# 20. A practical strategy hierarchy

For many real applications, a useful hierarchy is:

### First choice: invalidate

When the server can easily provide authoritative data:

```text
mutation
 ↓
invalidate affected queries
 ↓
refetch
```

### Second: directly update a precise cached result

When the mutation response gives you exactly the canonical data:

```text
mutation response
 ↓
setQueryData(["todo", id], response)
```

### Third: optimistic update

When immediate feedback is valuable and rollback is manageable:

```text
optimistic
 ↓
request
 ↓
success / rollback
 ↓
reconcile
```

This isn't a strict universal hierarchy, but it's a useful way to think.

---

# 21. Let's connect this to our state taxonomy

We now have:

```text
LOCAL STATE
   ↓
button open/closed

URL STATE
   ↓
page=2
search=react

SERVER STATE
   ↓
todos

DERIVED DATA
   ↓
visibleTodos

MUTATION STATE
   ↓
saving/deleting

CONTEXT
   ↓
theme/auth/config
```

And now notice:

```text
URL state
   ↓
query key
   ↓
server state
   ↓
mutation
   ↓
cache invalidation
   ↓
derived UI
```

A modern React application is really a network of these different forms of information.

The skill is knowing **which kind of state you're dealing with**.

---

# 22. One of the most useful debugging questions

Suppose your UI looks wrong.

Don't immediately ask:

> "Why didn't React rerender?"

Ask:

```text
Where is the wrong value coming from?
```

Then trace:

```text
UI
 ↓
component data
 ↓
query result
 ↓
cache
 ↓
query key
 ↓
request
 ↓
server
```

Maybe the server is correct but the cache is stale.

Maybe the query key is missing a filter.

Maybe the mutation updated one cache entry but not another.

Maybe the request returned unexpected data.

Maybe your UI is displaying derived data incorrectly.

This debugging model is incredibly powerful.

---

# 23. A realistic architecture

A product application might eventually look something like:

```text
                       ROUTER
                         │
                         ↓
                    URL STATE
                         │
               ┌─────────┴─────────┐
               ↓                   ↓
          local UI state       query inputs
                                   │
                                   ↓
                              QUERY CACHE
                                   │
                      ┌────────────┼────────────┐
                      ↓            ↓            ↓
                    list         detail       related
                      │            │            │
                      └────────────┼────────────┘
                                   ↓
                                MUTATION
                                   │
                           ┌───────┼───────┐
                           ↓       ↓       ↓
                       optimistic update
                           │
                           ↓
                       server
                           │
                    ┌──────┴──────┐
                    ↓             ↓
                 success        failure
                    ↓             ↓
                reconcile      rollback
```

That's much closer to how serious frontend applications actually behave.

---

# 24. What you should take away from today

The important lesson isn't:

```text
useQueryClient()
setQueryData()
invalidateQueries()
removeQueries()
```

Those are tools.

The underlying concepts are:

```text
1. The same server entity can appear in many queries.

2. A mutation can affect multiple cached representations.

3. A query represents a particular server-derived view.

4. Direct cache updates are precise but require you to understand the view.

5. Invalidation is less precise but often safer.

6. Optimistic updates improve responsiveness but introduce rollback and consistency problems.

7. The server remains the authority.

8. Query-key design determines how effectively you can reason about and invalidate cache families.
```

And this brings us to a particularly important topic we haven't really unpacked yet:

**what actually travels over HTTP when React talks to the backend**—GET vs POST vs PUT vs PATCH vs DELETE, request methods, headers, JSON bodies, status codes, authentication, and how all of that maps cleanly onto `fetch`, queries, and mutations.

That's where we'll properly cover the HTTP/API topic you were expecting, rather than sprinkling pieces of it through examples.