
# Lesson 18 — Server State and TanStack Query

This lesson is where React starts to feel much more like **real application engineering**.

Until now, we've treated API data like ordinary React state:

```jsx
const [products, setProducts] = useState([]);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
```

and then:

```jsx
useEffect(() => {
  fetch(...)
    .then(...)
}, []);
```

That works.

But once an application has dozens of API requests, you'll notice that you're repeatedly rebuilding the same machinery:

```text
fetch
loading
error
retry
refetch
caching
stale data
request cancellation
deduplication
pagination
mutations
```

This is where the idea of **server state** becomes important.

---

# 1. What exactly is server state?

Let's first separate two things.

Suppose your app has:

```text
isModalOpen
```

and:

```text
products
```

They are both JavaScript values, but conceptually they're different.

### `isModalOpen`

Your application owns it.

```text
React
 ↓
isModalOpen
```

It's client/UI state.

### `products`

The backend owns the authoritative version.

```text
Backend
 ↓
products
 ↓
React has a copy
```

Your React app is temporarily holding a representation of server data.

That's **server state**.

---

# 2. Why server state is different

Imagine:

```text
Server database
      ↓
   products
      ↓
React application
```

Your app receives:

```js
[
  { id: 1, name: "Laptop" }
]
```

But another user might change the product.

Or an administrator might update the price.

Now:

```text
Server:
Laptop = ₹80,000

Your app:
Laptop = ₹75,000
```

Your local copy is **stale**.

So server state has characteristics that normal UI state doesn't necessarily have:

```text
remote
asynchronous
shared
cacheable
potentially stale
refetchable
```

That difference drives the entire design of server-state libraries.

---

# 3. The key question

With ordinary state, you often ask:

> "What is the value right now?"

With server state, you also need to ask:

> "Where did this value come from, how old is it, is it still valid, and when should I fetch it again?"

For example:

```text
products
```

isn't just:

```js
products = [...]
```

It also has surrounding information:

```text
Do we have data?
Are we fetching?
When was it fetched?
Did it fail?
Is there a newer request?
Should we refetch?
Does another component need the same data?
Can we reuse cached data?
```

That's why server state deserves its own mental model.

---

# 4. Why our `useEffect` solution starts getting messy

Suppose you have:

```text
Products page
Users page
Orders page
Comments page
Profile page
```

Each component might have:

```jsx
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
```

Then:

```jsx
useEffect(() => {
  ...
}, []);
```

Now imagine you revisit the Products page.

Should you:

```text
fetch again?
```

Probably.

But could you reuse the products you already fetched five seconds ago?

Maybe.

What if two components both request:

```text
GET /api/products
```

at nearly the same time?

Could they share one result?

Ideally, yes.

What if a request fails?

Should you retry?

Maybe.

What if the user switches tabs and comes back?

Should you refetch?

Maybe.

You can build all of this yourself.

But you're now building a **server-state management system**.

That is exactly the problem libraries like TanStack Query solve.

---

# 5. What TanStack Query actually is

TanStack Query is **not a replacement for React**.

It's not an alternative rendering system.

It's not primarily a global UI-state system.

Its job is roughly:

```text
React
  ↓
TanStack Query
  ↓
manage asynchronous/server data
```

The current React Query API uses `QueryClient`, `QueryClientProvider`, `useQuery`, `useMutation`, and `useQueryClient`; the current package is `@tanstack/react-query`. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/functions/QueryClientProvider?utm_source=chatgpt.com "QueryClientProvider | TanStack Query React Docs"))

Think of it as:

> **A cache + synchronization system for server data.**

That's a much better mental model than:

> "It's a fancy replacement for `useEffect`."

---

# 6. The first new concept: Query

A **query** represents a request for some server data.

For example:

```text
"Give me all products."
```

or:

```text
"Give me product 42."
```

or:

```text
"Give me page 3 of orders."
```

A query has two essential pieces:

```text
queryKey
+
queryFn
```

For example:

```jsx
const query = useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts
});
```

That's the basic shape of modern TanStack Query. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/queries?utm_source=chatgpt.com "Queries | TanStack Query React Docs"))

---

# 7. `queryFn`

This is the function that actually gets the data.

For example:

```jsx
async function fetchProducts() {
  const response =
    await fetch("/api/products");

  if (!response.ok) {
    throw new Error(
      "Failed to fetch products"
    );
  }

  return response.json();
}
```

Then:

```jsx
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts
});
```

The roles are:

```text
queryKey
→ identity of this data

queryFn
→ how to retrieve it
```

That's a very important distinction.

---

# 8. Why does the query need a key?

Imagine your app asks for:

```text
products
```

and somewhere else it asks for:

```text
product 42
```

These are obviously different data.

TanStack Query needs a way to identify them.

So:

```jsx
queryKey: ["products"]
```

and:

```jsx
queryKey: ["product", 42]
```

represent different queries.

The query key is used to identify, cache, share, and refetch query data. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/queries?utm_source=chatgpt.com "Queries | TanStack Query React Docs"))

Think of it like a cache key:

```text
["products"]
      ↓
cached result for all products

["product", 42]
      ↓
cached result for product 42
```

---

# 9. Query keys should represent the inputs

Suppose:

```jsx
function Product({ productId }) {
  const query = useQuery({
    queryKey: ["product", productId],
    queryFn: () => fetchProduct(productId)
  });
}
```

Why include `productId` in the key?

Because:

```text
/product/42
```

and:

```text
/product/73
```

are different pieces of server data.

Therefore:

```text
["product", 42]
```

must be different from:

```text
["product", 73]
```

This is a very useful rule:

> **If changing a value means you're asking the server for different data, that value usually belongs in the query key.**

---

# 10. Compare this with `useEffect`

Earlier we had:

```jsx
useEffect(() => {
  fetch(`/api/products/${productId}`);
}, [productId]);
```

Notice the dependency:

```text
productId
```

Now with TanStack Query:

```jsx
useQuery({
  queryKey: ["product", productId],
  queryFn: () =>
    fetchProduct(productId)
});
```

The same conceptual relationship exists:

```text
productId
   ↓
different data request
```

but now we're declaring that relationship as part of the query itself.

---

# 11. Set up the Query Client

At the root of the application:

```jsx
import {
  QueryClient,
  QueryClientProvider
} from "@tanstack/react-query";

const queryClient =
  new QueryClient();

function AppRoot() {
  return (
    <QueryClientProvider
      client={queryClient}
    >
      <App />
    </QueryClientProvider>
  );
}
```

The `QueryClient` owns the query/mutation caches and configuration, while `QueryClientProvider` makes that client available to components underneath it. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/functions/QueryClientProvider?utm_source=chatgpt.com "QueryClientProvider | TanStack Query React Docs"))

Conceptually:

```text
QueryClient
 ├── query cache
 ├── mutation cache
 └── configuration

        ↓

QueryClientProvider

        ↓

   React application
```

---

# 12. Why do we need a QueryClient?

Because we want a central place for server-data management.

Imagine:

```text
Component A
   ↓
products

Component B
   ↓
products

Component C
   ↓
products
```

Ideally, these components shouldn't necessarily make three independent network requests.

They can subscribe to the same query:

```text
["products"]
```

and share the cached result.

That's one of the major benefits of a query cache.

---

# 13. Your first `useQuery`

Let's build a complete example.

```jsx
import { useQuery } from "@tanstack/react-query";

async function fetchProducts() {
  const response =
    await fetch("/api/products");

  if (!response.ok) {
    throw new Error(
      "Failed to fetch products"
    );
  }

  return response.json();
}

function Products() {
  const query = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts
  });

  if (query.isPending) {
    return <p>Loading...</p>;
  }

  if (query.isError) {
    return (
      <p>
        Error: {query.error.message}
      </p>
    );
  }

  return (
    <ul>
      {query.data.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

Already notice what disappeared:

```text
useState for data
useState for loading
useState for error
useEffect
manual fetch lifecycle
```

The query abstraction handles those pieces.

---

# 14. Understand the returned query object

This:

```jsx
const query = useQuery(...);
```

gives you an object containing information about the request/data.

The important pieces include:

```text
query.data
query.error
query.isPending
query.isError
query.isSuccess
query.isFetching
```

The current API distinguishes the primary states `pending`, `error`, and `success`, while `isFetching` can also be true during background fetching even when cached data already exists. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/queries?utm_source=chatgpt.com "Queries | TanStack Query React Docs"))

So:

```text
isPending
→ no data yet

isSuccess
→ data available

isError
→ request failed

isFetching
→ a fetch is happening right now
```

That last distinction is very important.

---

# 15. `isPending` vs `isFetching`

Suppose the first request is happening:

```text
no data
+
request running
```

Then:

```text
isPending = true
isFetching = true
```

Now suppose we already have:

```text
products = [...]
```

and TanStack Query performs a background refetch.

You can have:

```text
data = existing products
isPending = false
isFetching = true
```

This gives you a much richer UI model:

```text
existing data
+
small "refreshing..." indicator
```

instead of:

```text
erase everything
+
show giant spinner
```

That is one of the biggest UX benefits of separating **having data** from **currently fetching**.

---

# 16. Think about loading as two different questions

Instead of:

> "Is it loading?"

ask:

### Do I have data yet?

```text
isPending
```

### Is a network request currently happening?

```text
isFetching
```

Those aren't always the same.

This is a very important server-state mental model.

---

# 17. Caching

Suppose the user visits:

```text
/products
```

TanStack Query requests:

```text
GET /api/products
```

and gets:

```text
[product1, product2, product3]
```

That result is stored in the query cache under:

```text
["products"]
```

Now the user navigates away.

Then comes back.

The cache may still contain the data.

So instead of necessarily starting from:

```text
nothing
```

the application can immediately access the cached result and then decide whether a refetch is appropriate.

That's fundamentally different from a component's local state disappearing when the component unmounts.

---

# 18. This is why server state is not just `useState`

Suppose:

```jsx
const [products, setProducts] =
  useState([]);
```

When the component goes away:

```text
component unmounts
 ↓
local state disappears
```

With a query cache:

```text
component unmounts
 ↓
query cache can retain the data
```

The cache belongs to the query-management layer, not to one specific component.

That distinction is extremely important.

---

# 19. Multiple components can share the same query

Suppose:

```jsx
function ProductList() {
  const query = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts
  });

  ...
}
```

and:

```jsx
function ProductCount() {
  const query = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts
  });

  return (
    <p>
      {query.data?.length} products
    </p>
  );
}
```

Both use:

```text
["products"]
```

They are referring to the same query identity.

So the query cache can share the server result between them rather than treating the requests as unrelated pieces of local state. Query keys are specifically used for caching and sharing. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/guides/queries?utm_source=chatgpt.com "Queries | TanStack Query React Docs"))

---

# 20. This is a major difference from custom Hooks

Remember:

```jsx
function useProducts() {
  const [products, setProducts] =
    useState(...);

  ...
}
```

If two components call that custom Hook, each gets its own state instance.

```text
Component A → useProducts() → state A
Component B → useProducts() → state B
```

But with:

```jsx
useQuery({
  queryKey: ["products"],
  ...
})
```

both components are subscribing to the same cache entry:

```text
["products"]
      ↓
shared server data
   ↙       ↘
Comp A    Comp B
```

That's a critical distinction.

---

# 21. Stale data

Now we need another new concept:

# Fresh vs stale

Imagine you fetched products at:

```text
12:00:00
```

At:

```text
12:00:01
```

are they definitely outdated?

No.

At:

```text
12:00:02
```

still probably fine.

At:

```text
12:00:10
```

maybe.

At:

```text
12:30:00
```

perhaps they should be refreshed.

Your application needs a policy.

TanStack Query uses the concept of **stale time** to determine how long fetched data should be considered fresh. The current default behavior treats cached queries as stale immediately unless you configure a `staleTime`. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/interfaces/UseQueryOptions?utm_source=chatgpt.com "UseQueryOptions | TanStack Query React Docs"))

That default surprises beginners.

---

# 22. `staleTime`

You can configure:

```jsx
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  staleTime: 60 * 1000
});
```

This says conceptually:

```text
After successful fetch:
for 60 seconds → consider data fresh
after that     → consider it stale
```

Fresh doesn't necessarily mean:

> "The server data is objectively correct."

It means:

> "According to my application's cache policy, I don't need to consider this query stale yet."

That's a much better understanding.

---

# 23. Fresh vs fetching vs stale

These are different dimensions.

A query can conceptually be:

```text
Data:
available

Freshness:
stale

Network:
not currently fetching
```

or:

```text
Data:
available

Freshness:
fresh

Network:
not fetching
```

or:

```text
Data:
available

Freshness:
stale

Network:
currently fetching
```

This is why `isPending`, `isFetching`, and freshness concepts are not interchangeable.

---

# 24. Background refetching

Suppose cached product data exists.

User comes back to the page.

TanStack Query may decide:

```text
cached data exists
+
data is stale
```

so it can:

```text
show cached data immediately
+
fetch newer data in background
```

Conceptually:

```text
cached data
   ↓
render immediately
   ↓
background request
   ↓
new data
   ↓
update UI
```

This is sometimes described as **stale-while-revalidate** behavior.

The important UX idea is:

> Don't force the user to stare at a blank page when you already have usable data.

---

# 25. Refetching on window focus

A common situation:

```text
You open dashboard.
You switch to another browser tab.
Someone changes data.
You come back.
```

A server-state library can use browser focus as a signal to refetch stale queries.

The current TanStack Query defaults include refetch-on-window-focus behavior when data is stale. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/interfaces/UseQueryOptions?utm_source=chatgpt.com "UseQueryOptions | TanStack Query React Docs"))

This is something you'd otherwise need to implement manually.

---

# 26. Retries

Network requests can fail temporarily.

TanStack Query provides retry behavior for queries.

For example, the current client-side default is three retries for failed queries unless configured otherwise. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/interfaces/UseQueryOptions?utm_source=chatgpt.com "UseQueryOptions | TanStack Query React Docs"))

Conceptually:

```text
request
 ↓
fails
 ↓
retry
 ↓
fails
 ↓
retry
 ↓
fails
 ↓
show error
```

You can configure this policy based on the application.

For example:

```jsx
useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
  retry: 1
});
```

or:

```jsx
retry: false
```

depending on the situation.

---

# 27. Why retrying isn't always good

Imagine:

```text
POST /api/create-order
```

You generally don't want to automatically repeat a mutation blindly.

Why?

Because:

```text
create order
```

may actually have succeeded even if the response was lost.

Repeating it might create a duplicate order.

This is one reason **queries and mutations are treated differently**.

---

# 28. Queries vs mutations

This distinction is extremely important.

### Query

Usually:

```text
give me server data
```

Examples:

```text
GET products
GET user
GET orders
```

### Mutation

Usually:

```text
change something on the server
```

Examples:

```text
create product
update profile
delete order
```

TanStack Query gives you different abstractions for these.

Query:

```jsx
useQuery(...)
```

Mutation:

```jsx
useMutation(...)
```

The current TanStack Query docs explicitly separate queries from mutations and use query invalidation to keep related cached data synchronized after mutations. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/quick-start?utm_source=chatgpt.com "Quick Start | TanStack Query React Docs"))

---

# 29. First mutation example

Suppose:

```js
async function createTodo(todo) {
  const response =
    await fetch("/api/todos", {
      method: "POST",
      headers: {
        "Content-Type":
          "application/json"
      },
      body: JSON.stringify(todo)
    });

  if (!response.ok) {
    throw new Error("Failed");
  }

  return response.json();
}
```

Then:

```jsx
const mutation = useMutation({
  mutationFn: createTodo
});
```

To execute:

```jsx
mutation.mutate({
  title: "Learn React"
});
```

The idea is:

```text
user action
 ↓
mutation.mutate(...)
 ↓
server changes
```

---

# 30. Why isn't this just `useQuery`?

Because we are not asking:

> "What is the current value?"

We're asking:

> "Perform an action that changes server data."

That's a fundamentally different operation.

Think:

```text
Query:
READ

Mutation:
CHANGE
```

This distinction is foundational.

---

# 31. The cache problem after a mutation

Suppose we have:

```text
Query cache:
["todos"] → 5 todos
```

Then:

```text
POST /api/todos
```

creates another todo.

Now the server says:

```text
6 todos
```

but our cache may still say:

```text
5 todos
```

So after a successful mutation, we need to tell the query system:

> The cached `todos` data may no longer be valid.

That's called **invalidation**.

---

# 32. `useQueryClient`

You can get the query client:

```jsx
import {
  useQueryClient
} from "@tanstack/react-query";

const queryClient =
  useQueryClient();
```

Then:

```jsx
queryClient.invalidateQueries({
  queryKey: ["todos"]
});
```

Conceptually:

```text
mutation succeeds
       ↓
invalidate ["todos"]
       ↓
todos query is stale
       ↓
query can refetch
       ↓
cache updated
       ↓
UI updates
```

The current API exposes `useQueryClient` specifically to access the active `QueryClient`. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/functions/useQueryClient?utm_source=chatgpt.com "useQueryClient | TanStack Query React Docs"))

---

# 33. Complete mutation flow

```jsx
const queryClient =
  useQueryClient();

const mutation =
  useMutation({
    mutationFn: createTodo,

    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: ["todos"]
      });
    }
  });
```

Then:

```jsx
<button
  onClick={() =>
    mutation.mutate({
      title: "Learn React"
    })
  }
>
  Add Todo
</button>
```

The full flow:

```text
click
 ↓
mutate(data)
 ↓
POST request
 ↓
server creates todo
 ↓
mutation succeeds
 ↓
invalidate ["todos"]
 ↓
todos query refetches
 ↓
cache gets 6 todos
 ↓
components using ["todos"] update
```

That's an incredibly useful pattern.

---

# 34. Notice how this is different from manually calling `setTodos`

With plain React, you might do:

```jsx
setTodos([
  ...todos,
  newTodo
]);
```

But now there might be several components using the server's todo list.

Instead of manually synchronizing all of them, we can treat the server as the source of truth:

```text
mutation
 ↓
server changes
 ↓
invalidate query
 ↓
fresh server data
 ↓
all consumers update
```

That is the server-state philosophy.

---

# 35. Does invalidation immediately fetch?

Think of invalidation as:

> "This cached data should no longer be considered trustworthy/fresh."

Exactly what happens afterward depends on whether/how the query is being observed and configured.

For an active query, invalidation commonly leads to a refetch.

The important conceptual distinction is:

```text
invalidate
≠
manually replace cache with arbitrary data
```

It's more like:

```text
"This cached result needs reconsideration."
```

---

# 36. Optimistic updates

Sometimes you don't want to wait for the server.

Imagine:

```text
Like button ❤️
```

User clicks.

A poor experience might be:

```text
click
 ↓
wait 800 ms
 ↓
heart finally changes
```

An optimistic UI does:

```text
click
 ↓
immediately show liked
 ↓
send request
 ↓
if success → keep it
if failure → roll back
```

Conceptually:

```text
user intent
 ↓
optimistically update UI
 ↓
server request
 ├── success → keep
 └── failure → revert
```

TanStack Query supports patterns for optimistic updates, but there are several approaches and tradeoffs. We'll study this properly in the mutation lesson rather than mixing it into today's fundamentals.

---

# 37. Query caching vs Context

Now another architectural distinction.

You might think:

```text
Context
 ↓
store products
```

But Context doesn't inherently solve:

```text
cache
stale time
refetch
retry
deduplication
background updates
```

TanStack Query does.

So:

```text
Context
→ distribute a value through a tree

TanStack Query
→ manage server data and its lifecycle
```

These are different problems.

They can even work together.

---

# 38. Query cache vs Redux/Zustand

Similarly:

```text
Redux / Zustand
→ client/application state

TanStack Query
→ server state
```

You can store server data in a general-purpose client store, but then you're likely responsible for designing the server-data lifecycle yourself.

With a specialized server-state system, things like:

```text
caching
stale data
refetching
query identity
invalidation
request state
```

are part of the model.

This is why we're not treating every kind of state as the same thing.

---

# 39. Query keys are extremely important

Suppose:

```jsx
useQuery({
  queryKey: ["products", {
    category,
    sort,
    page
  }],
  queryFn: fetchProducts
});
```

Now your query identity includes:

```text
category
sort
page
```

That means:

```text
category=electronics
```

and:

```text
category=accessories
```

represent different cached queries.

Conceptually:

```text
["products", { category: "electronics", page: 1 }]
```

is one cache entry, while:

```text
["products", { category: "accessories", page: 1 }]
```

is another.

The query key should describe the data being requested. The current query API hashes query keys into a stable internal identity and automatically updates a query when its key changes. ([TanStack](https://tanstack.com/query/latest/docs/framework/react/reference/interfaces/UseQueryOptions?utm_source=chatgpt.com "UseQueryOptions | TanStack Query React Docs"))

---

# 40. This connects directly to our URL lesson

Remember:

```text
/products?category=electronics&page=2
```

We can now combine:

```text
URL state
+
server state
```

For example:

```jsx
const category =
  searchParams.get("category") ?? "all";

const page =
  Number(
    searchParams.get("page") ?? 1
  );

const productsQuery = useQuery({
  queryKey: [
    "products",
    {
      category,
      page
    }
  ],
  queryFn: () =>
    fetchProducts({
      category,
      page
    })
});
```

Now the architecture is:

```text
URL
 ↓
search params
 ↓
query key
 ↓
server request
 ↓
cached server data
 ↓
UI
```

This is a very powerful real-world pattern.

---

# 41. Why query keys shouldn't be treated casually

Suppose your request depends on:

```text
category
page
sort
```

but your key is:

```jsx
queryKey: ["products"]
```

Now your code may be asking for:

```text
electronics page 1
```

and then:

```text
accessories page 2
```

while using the same query identity.

That's conceptually wrong.

The key should reflect the inputs that determine the requested data.

So:

```jsx
queryKey: [
  "products",
  {
    category,
    page,
    sort
  }
]
```

is much more meaningful.

---

# 42. One query key, many consumers

Now imagine:

```text
ProductList
ProductCount
ProductSummary
```

all use:

```text
["products", { category, page }]
```

They can all consume the same cached query.

So:

```text
                     Query Cache
                         │
                  ["products", ...]
                  /       |       \
                 ↓        ↓        ↓
            ProductList Count   Summary
```

That's very different from manually copying server data into multiple pieces of local state.

---

# 43. A query is declarative

This is one of the nicest ways to think about `useQuery`.

Instead of:

```text
"Please fetch this now."
```

you're saying:

```text
"My component depends on this server data."
```

For example:

```jsx
useQuery({
  queryKey: ["product", productId],
  queryFn: () => fetchProduct(productId)
});
```

Conceptually:

```text
Component depends on:
    product #42
```

If the dependency changes:

```text
product #42
 ↓
product #73
```

the query identity changes:

```text
["product", 42]
 ↓
["product", 73]
```

and TanStack Query handles the new query relationship.

That's very close to the declarative React philosophy.

---

# 44. A complete example with URL filters

Let's combine almost everything from the last few lessons.

```jsx
import {
  useSearchParams
} from "react-router";

import {
  useQuery
} from "@tanstack/react-query";

async function fetchProducts({
  category,
  page
}) {
  const params = new URLSearchParams({
    category,
    page: String(page)
  });

  const response =
    await fetch(
      `/api/products?${params}`
    );

  if (!response.ok) {
    throw new Error(
      "Failed to load products"
    );
  }

  return response.json();
}

function Products() {
  const [searchParams, setSearchParams] =
    useSearchParams();

  const category =
    searchParams.get("category") ?? "all";

  const page =
    Number(
      searchParams.get("page") ?? 1
    );

  const productsQuery = useQuery({
    queryKey: [
      "products",
      {
        category,
        page
      }
    ],

    queryFn: () =>
      fetchProducts({
        category,
        page
      }),

    staleTime: 30_000
  });

  if (productsQuery.isPending) {
    return <p>Loading...</p>;
  }

  if (productsQuery.isError) {
    return (
      <p>
        {productsQuery.error.message}
      </p>
    );
  }

  return (
    <main>
      <select
        value={category}
        onChange={e => {
          setSearchParams({
            category: e.target.value,
            page: "1"
          });
        }}
      >
        <option value="all">
          All
        </option>

        <option value="electronics">
          Electronics
        </option>

        <option value="accessories">
          Accessories
        </option>
      </select>

      {productsQuery.isFetching && (
        <p>Refreshing...</p>
      )}

      <ul>
        {productsQuery.data.map(
          product => (
            <li key={product.id}>
              {product.name}
            </li>
          )
        )}
      </ul>
    </main>
  );
}
```

Look at the architecture:

```text
URL
 ↓
category/page
 ↓
queryKey
 ↓
useQuery
 ↓
query cache
 ↓
server
 ↓
data
 ↓
UI
```

That is much closer to the architecture you'll encounter in serious React applications.

---

# 45. What just happened to `useEffect`?

Notice that the component doesn't contain:

```jsx
useEffect(() => {
  fetch(...)
}, [category, page]);
```

The query abstraction owns that relationship.

This doesn't mean:

> "`useEffect` is obsolete."

It means:

> **When your problem is specifically server-state synchronization, use a tool designed around that problem rather than hand-building everything from Effects.**

You still need Effects for plenty of other external systems.

---

# 46. The big state taxonomy

At this point, I want your mental model to become:

```text
                    APPLICATION DATA
                           │
        ┌──────────────────┼───────────────────┐
        ↓                  ↓                   ↓
    UI/client            URL                server
      state              state               state
        │                  │                   │
    useState /          router            useQuery /
    useReducer          APIs               mutations
```

And then:

```text
derived values
```

sit on top of these:

```text
state/data
   ↓
calculation
   ↓
derived value
```

And Context is another concern:

```text
"How should a value be made available?"
```

rather than:

```text
"What kind of data is this?"
```

This is the state architecture we're building toward.

---

# 47. A very useful comparison

|Problem|Typical tool|
|---|---|
|Modal is open|`useState`|
|Complex local workflow|`useReducer`|
|Search/filter should survive URL sharing|URL/search params|
|Theme available throughout subtree|Context|
|DOM node reference|`useRef`|
|WebSocket synchronization|`useEffect`|
|Products from backend|`useQuery`|
|Create/update/delete on backend|`useMutation`|
|Filtered products|derived calculation|

This table is more valuable than memorizing individual hooks.

The question is always:

> **What problem am I solving?**

---

# 48. One more thing: query data isn't automatically "global state"

Suppose:

```jsx
ProductPage
```

uses:

```text
["products"]
```

and:

```jsx
Sidebar
```

uses:

```text
["products"]
```

They can share the query cache.

But you're not necessarily saying:

```text
"products is a global variable."
```

You're saying:

> These components depend on the same piece of server data.

That's a subtle but important distinction.

---

# 49. Server state belongs to the server

This is the philosophy to keep in your head:

```text
Browser
   ↓
temporary representation

Server
   ↓
source of truth
```

Therefore your React application should think in terms of:

```text
fetch
cache
stale
refetch
invalidate
mutate
synchronize
```

rather than:

```text
"I have an array in useState."
```

The array is just the current representation.

---

# 50. Where we go next

We've only covered the **query fundamentals**.

There is much more to server-state management:

```text
query keys
caching
staleTime
gcTime
refetching
dependent queries
parallel queries
pagination
infinite queries
placeholder data
prefetching
mutations
optimistic updates
invalidation
query cancellation
error retries
authentication
server rendering
```

We'll learn the important parts progressively rather than dumping the entire library API on you.

---

# The core mental model for today's lesson

You started with:

```text
useEffect
+
useState
+
fetch
```

and now you should see:

```text
                 SERVER
                   │
                   │ data
                   ↓
              Query Cache
                   │
             queryKey
                   │
                   ↓
              useQuery()
                   │
                   ↓
               Component
                   │
                   ↓
                   UI
```

For changes:

```text
UI event
   ↓
useMutation()
   ↓
server changes
   ↓
invalidate related query
   ↓
refetch / refresh cached data
   ↓
UI updates
```

That is the essential server-state model.

---

## Practice

Take the product application we've been building and change it so:

```text
/products
```

reads:

```text
search
category
page
```

from the URL.

Then create a query whose key looks conceptually like:

```jsx
[
  "products",
  {
    search,
    category,
    page
  }
]
```

The result should behave like:

```text
URL changes
    ↓
query key changes
    ↓
different query
    ↓
different server request
    ↓
cached result
    ↓
UI
```

The important part is understanding **why the URL values belong in the query key**, not merely getting the code to run.

---

## Next lesson — Advanced Data Fetching

Next we'll stay with server state and go deeper into the problems that appear in real applications:

**dependent queries, parallel queries, pagination, keeping previous results visible while changing pages, request cancellation, retry behavior, stale data, and how mutations interact with the cache.**

That lesson will turn today's basic `useQuery()` understanding into a practical data-fetching system.