
# Lesson 19 — Advanced Server Data: Parallel Queries, Dependent Queries, Pagination, and Changing Query Keys

In the previous lesson, we established the basic TanStack Query model:

```text
component
   ↓
useQuery
   ↓
queryKey + queryFn
   ↓
cache
   ↓
server
```

Today we go one level deeper.

The important shift is this:

> **Real applications rarely make one request at a time.**

A dashboard might need users, notifications, statistics, and permissions.

A product page might need the product, reviews, related products, and seller information.

A search page might need products for page 4 while the user is simultaneously changing filters.

So we need to understand how multiple queries interact.

---

# 1. There are two fundamentally different kinds of multiple requests

Suppose we need:

```text
users
teams
projects
```

Maybe none depends on another.

That's **parallel**.

But suppose we need:

```text
email
  ↓
user
  ↓
user.id
  ↓
projects belonging to user
```

That's **dependent/sequential**.

Understanding this distinction is extremely useful because it tells you whether the browser _can_ perform requests concurrently.

---

# 2. Parallel queries

Here's a simple dashboard:

```jsx
function Dashboard() {
  const usersQuery = useQuery({
    queryKey: ["users"],
    queryFn: fetchUsers,
  });

  const projectsQuery = useQuery({
    queryKey: ["projects"],
    queryFn: fetchProjects,
  });

  const notificationsQuery = useQuery({
    queryKey: ["notifications"],
    queryFn: fetchNotifications,
  });

  // ...
}
```

These three queries can run independently.

Conceptually:

```text
                  Dashboard
                      │
          ┌───────────┼───────────┐
          ↓           ↓           ↓
       users       projects   notifications
          ↓           ↓           ↓
       request      request      request
```

They don't have to wait for one another.

TanStack Query supports this directly: multiple `useQuery` calls can execute in parallel. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/parallel-queries?utm_source=chatgpt.com "Parallel Queries | TanStack Query React Docs"))

This is actually an important connection to React's rules of hooks.

You might wonder:

> "Why can't I just do this?"

```jsx
users.map(user => {
  const query = useQuery({
    queryKey: ["user", user.id],
    queryFn: () => fetchUser(user.id),
  });

  return query;
});
```

Because the number and order of hook calls would depend on the data.

That's not allowed.

Hooks need to be called consistently across renders.

So this:

```jsx
useQuery(...)
useQuery(...)
useQuery(...)
```

is fine when those queries are statically known.

But dynamically having an arbitrary number of queries requires a different mechanism.

---

# 3. Dynamic parallel queries: `useQueries`

Suppose we have:

```jsx
const users = [
  { id: 1, name: "Ava" },
  { id: 2, name: "Noah" },
  { id: 3, name: "Mia" },
];
```

And we want one server request per user:

```text
/user/1
/user/2
/user/3
```

Use:

```jsx
const userQueries = useQueries({
  queries: users.map(user => ({
    queryKey: ["user", user.id],
    queryFn: () => fetchUser(user.id),
  })),
});
```

Notice what happened.

We are absolutely using `map()`:

```jsx
users.map(user => ({
  queryKey: ["user", user.id],
  queryFn: () => fetchUser(user.id),
}))
```

But we're not calling a hook inside `map()`.

We're producing **configuration objects**.

Conceptually:

```text
users
 ↓
map
 ↓
[
  {
    queryKey: ["user", 1],
    queryFn: ...
  },
  {
    queryKey: ["user", 2],
    queryFn: ...
  },
  {
    queryKey: ["user", 3],
    queryFn: ...
  }
]
 ↓
useQueries
 ↓
multiple queries
```

That distinction is worth remembering:

> `map()` can construct data describing queries; it must not conditionally invoke hooks itself.

Current TanStack Query exposes `useQueries` specifically for this dynamic parallel-query pattern. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/parallel-queries?utm_source=chatgpt.com "Parallel Queries | TanStack Query React Docs"))

---

# 4. Now the opposite: dependent queries

Here's a common application flow.

The user types:

```text
alice@example.com
```

We first ask:

```http
GET /users?email=alice@example.com
```

Suppose the response is:

```js
{
  id: 42,
  name: "Alice"
}
```

Then we need:

```http
GET /users/42/projects
```

We can't make the second request until we know:

```text
user.id === 42
```

That's a dependent query.

---

# 5. The crucial idea: `enabled`

```jsx
function UserProjects({ email }) {
  const userQuery = useQuery({
    queryKey: ["user", email],
    queryFn: () => fetchUserByEmail(email),
  });

  const userId = userQuery.data?.id;

  const projectsQuery = useQuery({
    queryKey: ["projects", userId],
    queryFn: () => fetchProjects(userId),
    enabled: !!userId,
  });

  // ...
}
```

Look carefully.

The second query exists during rendering.

But:

```js
enabled: !!userId
```

controls whether it is allowed to execute.

Initially:

```text
userId = undefined
```

so:

```js
!!userId
```

is:

```js
false
```

Therefore:

```text
user query
   ↓
running
   ↓
user data arrives
   ↓
userId = 42
   ↓
enabled becomes true
   ↓
projects query runs
```

That's much cleaner than manually writing an effect that watches the first request and starts the second request.

Current TanStack Query supports dependent queries using `enabled`. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/dependent-queries?from=reactQueryV3&utm_source=chatgpt.com "Dependent Queries | TanStack Query React Docs"))

---

# 6. Notice something subtle about the query key

We wrote:

```jsx
queryKey: ["projects", userId]
```

not:

```jsx
queryKey: ["projects"]
```

Why?

Because these are different pieces of server data:

```text
["projects", 42]
["projects", 81]
["projects", 125]
```

represent:

```text
Alice's projects
Bob's projects
Someone else's projects
```

The user ID is part of the identity of the requested data.

This gives us a very powerful rule:

> **Anything that changes which server data you're asking for should generally be represented in the query key.**

For example:

```jsx
queryKey: [
  "products",
  {
    category,
    search,
    page,
    sort,
  }
]
```

means:

```text
different search
    → different query

different category
    → different query

different page
    → different query

different sort
    → different query
```

And that naturally leads us to pagination.

---

# 7. Pagination is really just changing the query key

Imagine:

```jsx
const [page, setPage] = useState(1);
```

Then:

```jsx
const productsQuery = useQuery({
  queryKey: ["products", page],
  queryFn: () => fetchProducts(page),
});
```

When:

```text
page = 1
```

the query is:

```text
["products", 1]
```

When the user clicks Next:

```jsx
setPage(page + 1);
```

React renders again.

Now:

```text
["products", 2]
```

That's a **different query key**.

TanStack Query therefore treats it as a different cached query.

Conceptually:

```text
Query cache

["products", 1]
       ↓
 Page 1 data


["products", 2]
       ↓
 Page 2 data


["products", 3]
       ↓
 Page 3 data
```

This is one of the most beautiful things about query keys.

Pagination doesn't require you to manually maintain:

```js
page1Data
page2Data
page3Data
```

The cache identity handles it.

TanStack Query's current pagination approach is to include page information in the query key. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/paginated-queries?utm_source=chatgpt.com "Paginated / Lagged Queries | TanStack Query React Docs"))

---

# 8. But there's a UX problem

Suppose page 1 is visible.

The user clicks:

```text
Next
```

The key changes:

```text
["products", 1]
       ↓
["products", 2]
```

Page 2 hasn't arrived yet.

A naive UI might do:

```jsx
if (isPending) {
  return <Spinner />;
}
```

So the experience becomes:

```text
Page 1
 ↓
click Next
 ↓
Page disappears
 ↓
spinner
 ↓
Page 2
```

That's not very pleasant.

A better experience is:

```text
Page 1
 ↓
click Next
 ↓
Page 1 remains visible
 + small "Loading..." indicator
 ↓
Page 2 arrives
 ↓
Page 2 replaces Page 1
```

This is where current TanStack Query's `placeholderData` + `keepPreviousData` helper becomes useful. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/functions/keepPreviousData?utm_source=chatgpt.com "keepPreviousData | TanStack Query React Docs"))

---

# 9. Keeping the previous page visible

Current TanStack Query uses:

```jsx
import {
  useQuery,
  keepPreviousData,
} from "@tanstack/react-query";
```

Then:

```jsx
const productsQuery = useQuery({
  queryKey: ["products", page],
  queryFn: () => fetchProducts(page),
  placeholderData: keepPreviousData,
});
```

Now when the key changes:

```text
["products", 1]
        ↓
["products", 2]
```

the application can continue displaying the previous successful data while the new query is fetching.

Conceptually:

```text
              query key changes
                     ↓
         ┌───────────┴───────────┐
         ↓                       ↓
old data remains          page 2 request
visible                       starts
                                 ↓
                              response
                                 ↓
                        old data replaced
```

And:

```jsx
isPlaceholderData
```

can tell you that the currently displayed data is being used as placeholder data for the new query. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/paginated-queries?utm_source=chatgpt.com "Paginated / Lagged Queries | TanStack Query React Docs"))

---

# 10. `isPending` vs `isFetching` becomes extremely important here

This connects directly to the previous lesson.

Imagine page 1 has already loaded.

Then page 2 starts fetching while page 1 is still displayed.

You might have:

```text
data           → page 1
isPending      → false
isFetching     → true
```

That combination is completely legitimate.

Think of the variables as answering different questions:

```text
isPending
    ↓
"Do I currently have no data yet?"

isFetching
    ↓
"Is a network fetch happening right now?"
```

So a good paginated UI might do:

```jsx
{productsQuery.isPending ? (
  <FullPageSpinner />
) : (
  <>
    <ProductList products={productsQuery.data.products} />

    {productsQuery.isFetching && (
      <SmallLoadingIndicator />
    )}
  </>
)}
```

Now the UI distinguishes:

```text
nothing available yet
```

from:

```text
we already have useful data, but it's being refreshed
```

That distinction is fundamental to good server-state UX.

---

# 11. Why this is different from local state

You could technically write:

```jsx
const [products, setProducts] = useState([]);
const [page, setPage] = useState(1);
const [loading, setLoading] = useState(false);
```

and manually fetch every page.

But now you're responsible for answering:

```text
Where is page 1?
Where is page 2?
Is page 2 already cached?
Should I reuse page 2?
What happens if I go back?
What if two components request page 2?
What if the request is still running?
What if page 2 becomes stale?
When should it refetch?
What happens if the user leaves?
```

This is exactly why server-state libraries exist.

The problem isn't:

> "How do I fetch?"

The problem is:

> **"How do I manage asynchronous, remote, cached, potentially stale data?"**

---

# 12. Query keys give you a mental model for the cache

Suppose we have:

```jsx
useQuery({
  queryKey: ["products", "electronics", 1],
  queryFn: ...
});
```

Then later:

```jsx
useQuery({
  queryKey: ["products", "electronics", 2],
  queryFn: ...
});
```

These are two pieces of cached data.

Later the user goes back:

```text
page 1
```

The query key becomes:

```text
["products", "electronics", 1]
```

If the cache still contains that data, TanStack Query can use it according to its freshness rules.

So the URL → query-key connection becomes extremely powerful.

---

# 13. URL state + query state

Imagine our URL is:

```text
/products?category=electronics&search=phone&page=3
```

We already learned that these values belong naturally to URL state.

Now connect that to server state:

```text
URL
 │
 ├── category = electronics
 ├── search = phone
 └── page = 3
       │
       ↓
queryKey
       │
       ↓
["products", {
   category: "electronics",
   search: "phone",
   page: 3
}]
       │
       ↓
server request
       │
       ↓
cache
       │
       ↓
UI
```

This is a real application architecture pattern.

The URL determines **what view the user is looking at**.

The query key translates that view into **server-data identity**.

The query cache remembers **what we've already fetched**.

---

# 14. Dependent query + dynamic URL state

Let's make this slightly more realistic.

Suppose the route is:

```text
/users/:userId
```

and inside the page we want:

```text
user
projects
activity
```

We can do:

```jsx
function UserPage() {
  const { userId } = useParams();

  const userQuery = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });

  const projectsQuery = useQuery({
    queryKey: ["projects", userId],
    queryFn: () => fetchProjects(userId),
  });

  const activityQuery = useQuery({
    queryKey: ["activity", userId],
    queryFn: () => fetchActivity(userId),
  });

  // ...
}
```

Notice something interesting:

The three requests are **not dependent on one another**.

They all depend on:

```text
userId
```

but they don't depend on each other.

Therefore they can run in parallel:

```text
                userId
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
      user      projects   activity
        ↓         ↓         ↓
     request   request    request
```

Don't accidentally interpret:

```text
projects depends on userId
```

as:

```text
projects must wait for user query
```

It only needs the **ID**, which we already have from the route.

That's an architectural distinction worth developing.

---

# 15. When dependencies genuinely exist

Now imagine instead:

```text
/users/:email
```

The URL gives us an email, but not the user's numeric ID.

So:

```text
email
 ↓
fetch user
 ↓
user.id
 ↓
fetch projects
```

Now there really is a dependency.

```jsx
const userQuery = useQuery({
  queryKey: ["user", email],
  queryFn: () => fetchUserByEmail(email),
});

const projectsQuery = useQuery({
  queryKey: ["projects", userQuery.data?.id],
  queryFn: () => fetchProjects(userQuery.data.id),
  enabled: !!userQuery.data?.id,
});
```

The dependency is represented explicitly by:

```jsx
enabled
```

rather than by manually chaining asynchronous effects.

---

# 16. A subtle but important distinction: "query exists" vs "query is fetching"

This code:

```jsx
const projectsQuery = useQuery({
  queryKey: ["projects", userId],
  queryFn: () => fetchProjects(userId),
  enabled: !!userId,
});
```

does **not** mean:

> "Don't create this query until there is a user ID."

The component still calls `useQuery`.

The query observer still exists.

It's the execution that is gated.

When `userId` doesn't exist yet, the query is not ready to fetch.

Current TanStack Query exposes `enabled` specifically for this type of dependent execution. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/dependent-queries?from=reactQueryV3&utm_source=chatgpt.com "Dependent Queries | TanStack Query React Docs"))

That's conceptually important because React's rendering and the query's fetching lifecycle are separate things.

---

# 17. Query cancellation

Now let's return to something we touched in our plain `fetch` lesson.

Suppose:

```text
User searches "lap"
       ↓
request A

User quickly types "laptop"
       ↓
request B
```

We don't really care about request A anymore.

TanStack Query passes an `AbortSignal` into the query function, allowing your request to participate in cancellation. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/query-cancellation?utm_source=chatgpt.com "Query Cancellation | TanStack Query React Docs"))

For example:

```jsx
async function fetchProducts({ queryKey, signal }) {
  const [, search] = queryKey;

  const response = await fetch(
    `/api/products?search=${encodeURIComponent(search)}`,
    { signal }
  );

  if (!response.ok) {
    throw new Error("Failed to fetch products");
  }

  return response.json();
}
```

Then:

```jsx
const query = useQuery({
  queryKey: ["products", search],
  queryFn: fetchProducts,
});
```

TanStack Query can provide the signal to your function.

The key idea isn't actually the API syntax.

It's:

```text
query becomes obsolete
       ↓
cancellation signal
       ↓
request can abort
```

So the query system has more knowledge of request lifecycle than a plain `useEffect` implementation typically does.

---

# 18. Why query keys and cancellation work together

This is a beautiful sequence to understand.

Suppose:

```text
search = "iphone"
```

so:

```text
queryKey = ["products", "iphone"]
```

Request begins.

Then user changes search:

```text
"iphone 15"
```

Now:

```text
queryKey = ["products", "iphone 15"]
```

The old query and new query are conceptually different.

```text
["products", "iphone"]
          ↓
        old

["products", "iphone 15"]
          ↓
        current
```

Now we have a system that can reason about:

```text
identity
freshness
fetching
caching
cancellation
```

That's much more powerful than simply:

```text
useEffect(() => fetch(...))
```

---

# 19. Retry behavior needs thought too

Network requests fail.

For example:

```text
GET /products
        ↓
500 Server Error
```

TanStack Query has retry behavior for queries, but retrying isn't something you should blindly treat as universally good.

Consider:

```text
temporary network failure
```

versus:

```text
401 Unauthorized
```

versus:

```text
400 Bad Request
```

Retrying a temporary network failure can potentially make sense.

Retrying an invalid request repeatedly may accomplish nothing.

So server-state logic needs to distinguish:

```text
transient failure
```

from:

```text
permanent/application failure
```

And because retry behavior can affect UX and server load, it's something you should configure thoughtfully rather than thinking:

> "Queries failed, so retry forever."

---

# 20. Cursor pagination

Page numbers aren't the only way APIs paginate.

A backend might return:

```json
{
  "items": [...],
  "nextCursor": "abc123"
}
```

Then the next request is:

```text
GET /products?cursor=abc123
```

instead of:

```text
GET /products?page=2
```

The conceptual model is still the same:

```text
cursor
   ↓
query identity
   ↓
request
   ↓
next cursor
```

But there is an important difference.

With numbered pages:

```text
1
2
3
4
```

With cursors:

```text
null
 ↓
abc123
 ↓
def456
 ↓
ghi789
```

The cursor usually represents the backend's continuation position.

This matters especially for feeds, activity streams, social timelines, and large datasets.

Later, this naturally leads to `useInfiniteQuery`.

---

# 21. A realistic product browser

Let's combine everything we've learned.

Imagine:

```text
/products
```

URL:

```text
/products?search=phone&category=electronics&page=2
```

Component:

```jsx
function ProductPage() {
  const [searchParams, setSearchParams] = useSearchParams();

  const search = searchParams.get("search") ?? "";
  const category = searchParams.get("category") ?? "all";
  const page = Number(searchParams.get("page") ?? "1");

  const productsQuery = useQuery({
    queryKey: [
      "products",
      {
        search,
        category,
        page,
      },
    ],
    queryFn: () =>
      fetchProducts({
        search,
        category,
        page,
      }),
    placeholderData: keepPreviousData,
  });

  // ...
}
```

Now watch the complete flow.

### User changes category

```text
category:
"electronics"
      ↓
"books"
```

React renders again.

The query key changes:

```text
["products", {
  search: "phone",
  category: "electronics",
  page: 2
}]
```

to:

```text
["products", {
  search: "phone",
  category: "books",
  page: 2
}]
```

That means:

```text
different server-data identity
```

So TanStack Query looks for the new query in its cache and fetches as needed.

Meanwhile `placeholderData: keepPreviousData` can allow the previous results to remain visible while the new results arrive. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/paginated-queries?utm_source=chatgpt.com "Paginated / Lagged Queries | TanStack Query React Docs"))

That's an entire application architecture emerging from one simple principle:

> **State that determines server data belongs in the query key.**

---

# 22. There's a trap here: putting too little into the query key

Suppose your request uses:

```jsx
search
category
page
sort
```

but your key is:

```jsx
queryKey: ["products", page]
```

That's wrong conceptually.

Because these requests are different:

```text
/products?page=1&category=books
/products?page=1&category=phones
```

yet your cache identity says:

```text
["products", 1]
```

for both.

You've told the cache:

> "These represent the same data."

But they don't.

So the cache can no longer correctly distinguish them.

The safer design is:

```jsx
queryKey: [
  "products",
  {
    search,
    category,
    page,
    sort,
  },
]
```

---

# 23. The opposite mistake: putting irrelevant things into the key

Suppose:

```jsx
const isModalOpen = true;
```

and you write:

```jsx
queryKey: [
  "products",
  {
    page,
    isModalOpen,
  }
]
```

That makes no sense.

Opening a modal does not change which products you're requesting.

So:

```text
query key
```

should contain the values that define **server-data identity**, not every piece of state in the component.

That's an important architectural distinction.

---

# 24. A useful rule for deciding query-key contents

Ask:

> **"If this value changes, should the server request potentially return different data?"**

If yes:

```text
probably part of query key
```

If no:

```text
probably not part of query key
```

Examples:

```text
productId       → yes
page            → yes
search          → yes
category        → yes
sort            → yes

isModalOpen     → no
sidebarOpen     → no
darkMode        → no
selectedTab     → usually no
```

The word **usually** matters because architecture depends on what the server request actually depends on.

Don't memorize the list. Reason from the request.

---

# 25. Putting the entire lesson together

Here's the mental picture I want you to have now:

```text
                    React UI
                       │
                       ↓
              local / URL state
                       │
                       ↓
                 query inputs
                       │
                       ↓
                  queryKey
                       │
              ┌────────┴────────┐
              ↓                 ↓
          cache lookup      queryFn
              │                 │
              │                 ↓
              │             network
              │                 │
              │                 ↓
              └────────────→ server data
                       │
                       ↓
                     cache
                       │
                       ↓
                       UI
```

And when multiple requests exist:

```text
                      inputs
                        │
          ┌─────────────┼─────────────┐
          ↓             ↓             ↓
       query A       query B       query C
          │             │             │
          ↓             ↓             ↓
       server        server        server
```

unless there's a dependency:

```text
query A
  ↓
data from A
  ↓
input for B
  ↓
query B
```

And when a query key changes:

```text
old key
  ↓
new key
  ↓
new query identity
  ↓
cache lookup / fetch
```

And with pagination:

```text
page 1
  ↓
["products", 1]

page 2
  ↓
["products", 2]

page 3
  ↓
["products", 3]
```

Each can have its own cached result.

---

# 26. The deeper connection to React

There's something worth noticing here.

React itself doesn't know that:

```text
["products", 1]
```

means page 1.

React doesn't know what a query is.

React knows:

```text
component renders
hooks are called
state/props change
component renders again
```

TanStack Query builds another system on top:

```text
query identity
cache
freshness
fetching
retries
cancellation
synchronization
```

So we're now dealing with **two different state machines**:

### React's rendering state

```text
render
  ↓
UI
  ↓
state changes
  ↓
render again
```

### Server-query state

```text
idle/pending
  ↓
fetching
  ↓
success
  ↓
stale
  ↓
background fetching
  ↓
fresh data
```

Those two systems interact, but they are not the same thing.

That's why a component can simultaneously be:

```text
React:
rendering normally

Query:
fetching in background

UI:
showing old data
```

And that's perfectly valid.

---

# 27. What I want you to be able to read now

When you see this:

```jsx
const query = useQuery({
  queryKey: ["products", category, page],
  queryFn: () => fetchProducts(category, page),
  placeholderData: keepPreviousData,
  enabled: Boolean(category),
});
```

Don't read it as:

> "Some TanStack Query syntax."

Read it as:

```text
What server data is this?
        ↓
products

What determines which data?
        ↓
category + page

Where does data come from?
        ↓
fetchProducts(category, page)

Should it run yet?
        ↓
only if category exists

What should happen while changing pages?
        ↓
keep previous successful data visible
```

That is the level of understanding we're aiming for.

---

## One final distinction

Don't fall into this mental model:

```text
useQuery = advanced fetch()
```

It's much bigger than that.

Think:

```text
fetch()
    =
"perform this HTTP request"

useQuery()
    =
"manage this piece of server state"
```

That difference becomes increasingly important as applications grow.

And we've now crossed into the point where **server-state architecture** matters more than simply knowing how to call an API.

Next, the natural step is to flip the direction: we've mostly been **reading** server state. The next deep concept is **changing** server state—create/update/delete mutations, invalidation, direct cache updates, optimistic UI, and rollback.