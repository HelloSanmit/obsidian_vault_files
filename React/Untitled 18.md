
# Lesson 17 — Data Fetching in React

We've reached the point where our applications stop using:

```js
const products = [
  ...
];
```

and start getting data from a server:

```text
React application
      ↓
HTTP request
      ↓
backend/API
      ↓
response
      ↓
React
      ↓
UI
```

This is one of the most important parts of real React development.

And there's a crucial distinction we'll build throughout this lesson:

> **The server is the source of truth for server data. React is displaying and interacting with that data.**

---

# 1. What is data fetching?

Suppose your backend exposes:

```text
GET /api/products
```

The response might be:

```json
[
  {
    "id": 1,
    "name": "Laptop",
    "price": 80000
  },
  {
    "id": 2,
    "name": "Phone",
    "price": 50000
  }
]
```

Your React component needs to:

```text
request the data
        ↓
wait
        ↓
receive response
        ↓
store/display data
```

That's data fetching.

---

# 2. The simplest possible example

```jsx
import { useEffect, useState } from "react";

function Products() {
  const [products, setProducts] = useState([]);

  useEffect(() => {
    fetch("/api/products")
      .then(response => response.json())
      .then(data => {
        setProducts(data);
      });
  }, []);

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

Let's not rush through this.

There are several layers here.

---

# 3. Why do we need state?

The server gives us data asynchronously.

When the component first renders:

```text
products = []
```

We don't have the server response yet.

Later:

```text
server response arrives
       ↓
setProducts(data)
       ↓
products changes
       ↓
render again
```

So `products` needs to be state because its value changes over time and the UI depends on it.

---

# 4. Why is the fetch inside `useEffect`?

Remember what we learned about Effects.

Rendering should mainly answer:

> "What should the UI look like given the current data?"

Fetching from a server is interaction with an external system.

So conceptually:

```text
render
  ↓
Effect
  ↓
network request
```

That's why a basic client-side fetch often lives inside an Effect.

But keep in mind: this is a **basic client-side pattern**, not the only or always-best way modern React applications load server data. We'll get to route loaders and server-state libraries later.

---

# 5. The first problem: loading

Our first version has a hidden problem.

Initially:

```text
products = []
```

The UI might show:

```text
Products

[nothing]
```

But the user has no idea whether:

```text
there are no products
```

or:

```text
the request is still running
```

So we need loading state.

```jsx
function Products() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch("/api/products")
      .then(response => response.json())
      .then(data => {
        setProducts(data);
        setLoading(false);
      });
  }, []);

  if (loading) {
    return <p>Loading...</p>;
  }

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

Now the flow is:

```text
initial render
    ↓
loading = true
    ↓
"Loading..."
    ↓
request
    ↓
response
    ↓
setProducts(...)
setLoading(false)
    ↓
render
    ↓
products appear
```

---

# 6. The second problem: errors

What if the server is down?

What if the network fails?

What if the request fails?

We need error state:

```jsx
function Products() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch("/api/products")
      .then(response => {
        if (!response.ok) {
          throw new Error("Failed to fetch products");
        }

        return response.json();
      })
      .then(data => {
        setProducts(data);
      })
      .catch(error => {
        setError(error.message);
      })
      .finally(() => {
        setLoading(false);
      });
  }, []);

  if (loading) {
    return <p>Loading...</p>;
  }

  if (error) {
    return <p>Error: {error}</p>;
  }

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

Now we've modeled:

```text
loading
success
error
```

This is the beginning of proper server-data handling.

---

# 7. Why check `response.ok`?

This is a subtle JavaScript/browser detail.

A failed HTTP response such as:

```text
404
500
```

doesn't automatically cause `fetch()` to reject its Promise.

So this:

```js
fetch("/api/products")
  .then(...)
```

can receive a response object even when the HTTP status indicates failure.

That's why you commonly write:

```js
if (!response.ok) {
  throw new Error("Request failed");
}
```

Then your `.catch()` or `try/catch` can handle the failure.

---

# 8. `async/await` makes it easier to read

Let's rewrite it.

```jsx
function Products() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    async function loadProducts() {
      try {
        const response =
          await fetch("/api/products");

        if (!response.ok) {
          throw new Error(
            "Failed to fetch products"
          );
        }

        const data =
          await response.json();

        setProducts(data);
      } catch (error) {
        setError(error.message);
      } finally {
        setLoading(false);
      }
    }

    loadProducts();
  }, []);

  if (loading) {
    return <p>Loading...</p>;
  }

  if (error) {
    return <p>Error: {error}</p>;
  }

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

This is often easier to understand.

---

# 9. Why not make the Effect callback itself `async`?

You might try:

```jsx
useEffect(async () => {
  ...
}, []);
```

Don't.

The function you pass to `useEffect` is supposed to optionally return a cleanup function.

An `async` function returns a Promise.

Those are different contracts.

Instead:

```jsx
useEffect(() => {
  async function loadProducts() {
    ...
  }

  loadProducts();
}, []);
```

The Effect callback itself remains synchronous while the nested function handles the asynchronous work.

---

# 10. Now let's introduce the real state model

Our data-fetching component has:

```text
products
loading
error
```

These aren't three random variables.

They represent three pieces of information about one process.

Conceptually:

```text
Request state
├── loading
├── data
└── error
```

This can be modeled more explicitly as:

```js
{
  status: "loading",
  data: null,
  error: null
}
```

or:

```js
{
  status: "success",
  data: products,
  error: null
}
```

or:

```js
{
  status: "error",
  data: null,
  error: "..."
}
```

This is one reason `useReducer` can eventually become useful for data fetching.

---

# 11. The hidden fourth state: empty

Suppose the request succeeds:

```js
products = [];
```

That's not the same as:

```text
loading
```

and not the same as:

```text
error
```

It's:

```text
success + empty data
```

So your UI often has:

```text
loading
error
empty
success with data
```

This is important.

A real data-driven interface needs to distinguish those cases.

---

# 12. Build that explicitly

```jsx
if (loading) {
  return <p>Loading...</p>;
}

if (error) {
  return <p>Something went wrong.</p>;
}

if (products.length === 0) {
  return <p>No products found.</p>;
}

return (
  <ProductList products={products} />
);
```

The state machine is roughly:

```text
             request
                ↓
             loading
            /       \
        success     error
          ↓
       products
       /      \
    empty     non-empty
```

This way of thinking becomes extremely useful.

---

# 13. The fetch lifecycle

Let's trace a successful request:

```text
Component renders
     ↓
products = []
loading = true
error = null
     ↓
Effect starts
     ↓
fetch()
     ↓
waiting...
     ↓
server responds
     ↓
response.json()
     ↓
setProducts(data)
setLoading(false)
     ↓
render again
     ↓
loading = false
products = [...]
     ↓
UI displays products
```

This is the basic client-side fetch lifecycle.

---

# 14. Now add a route parameter

Suppose we have:

```text
/products/:productId
```

and:

```jsx
function ProductDetails() {
  const { productId } = useParams();

  const [product, setProduct] =
    useState(null);

  useEffect(() => {
    async function loadProduct() {
      const response =
        await fetch(
          `/api/products/${productId}`
        );

      const data =
        await response.json();

      setProduct(data);
    }

    loadProduct();
  }, [productId]);

  ...
}
```

Now the request depends on:

```text
productId
```

So:

```jsx
[productId]
```

belongs in the dependency list.

---

# 15. Why does the request happen again when `productId` changes?

Suppose:

```text
/product/1
```

Then:

```text
productId = "1"
```

Effect runs:

```text
fetch("/api/products/1")
```

Then user navigates to:

```text
/product/2
```

The same component may remain in place, but:

```text
productId = "2"
```

changes.

Therefore:

```text
Effect dependency changed
       ↓
cleanup if necessary
       ↓
new Effect execution
       ↓
fetch product 2
```

That's a good example of why dependencies aren't just about "running once."

---

# 16. Add loading/error states to that

```jsx
function ProductDetails() {
  const { productId } = useParams();

  const [product, setProduct] =
    useState(null);

  const [loading, setLoading] =
    useState(true);

  const [error, setError] =
    useState(null);

  useEffect(() => {
    async function loadProduct() {
      try {
        setLoading(true);
        setError(null);

        const response =
          await fetch(
            `/api/products/${productId}`
          );

        if (!response.ok) {
          throw new Error(
            "Failed to load product"
          );
        }

        const data =
          await response.json();

        setProduct(data);
      } catch (error) {
        setError(error.message);
      } finally {
        setLoading(false);
      }
    }

    loadProduct();
  }, [productId]);

  if (loading) {
    return <p>Loading...</p>;
  }

  if (error) {
    return <p>{error}</p>;
  }

  if (!product) {
    return <p>Product not found.</p>;
  }

  return (
    <div>
      <h1>{product.name}</h1>
      <p>₹{product.price}</p>
    </div>
  );
}
```

Now you've connected:

```text
routing
+
useEffect
+
state
+
API
+
conditional rendering
```

This is real application behavior.

---

# 17. But now we encounter a serious problem

Suppose the user moves quickly:

```text
/product/1
```

then:

```text
/product/2
```

and then:

```text
/product/3
```

You could potentially have:

```text
Request 1 → product 1
Request 2 → product 2
Request 3 → product 3
```

in flight at the same time.

Now imagine:

```text
request 3 finishes
request 1 finishes later
```

The old response could overwrite the newer result.

You could end up displaying:

```text
URL: /product/3

UI: Product 1
```

That's a **race condition**.

This is one of the reasons data fetching gets much more complicated than `fetch(...).then(...)`.

---

# 18. Request cancellation

The browser provides `AbortController`.

You can do:

```jsx
useEffect(() => {
  const controller =
    new AbortController();

  async function loadProduct() {
    try {
      const response =
        await fetch(
          `/api/products/${productId}`,
          {
            signal: controller.signal
          }
        );

      const data =
        await response.json();

      setProduct(data);
    } catch (error) {
      if (error.name === "AbortError") {
        return;
      }

      setError(error.message);
    }
  }

  loadProduct();

  return () => {
    controller.abort();
  };
}, [productId]);
```

Now:

```text
product 1 request starts
      ↓
productId changes
      ↓
cleanup
      ↓
abort product 1 request
      ↓
product 2 request starts
```

This is much safer.

---

# 19. Why does cleanup happen here?

Remember our Effect lifecycle:

```text
setup
 ↓
synchronization
 ↓
dependency changes
 ↓
cleanup old synchronization
 ↓
setup new synchronization
```

Here:

```jsx
return () => {
  controller.abort();
};
```

is cleanup.

It says:

> "If this Effect is no longer responsible for this request, cancel it."

That's a very natural use of the Effect lifecycle.

---

# 20. Should you always use `AbortController`?

Not every network operation needs identical handling.

But when requests can overlap or become irrelevant due to changing inputs, cancellation and stale-response handling become important.

This is especially relevant for:

```text
search
route changes
autocomplete
filters
pagination
```

We'll explore these scenarios later.

---

# 21. Search-as-you-type is a perfect example

Suppose:

```text
Search:
[ laptop          ]
```

Every time the user types:

```text
l
la
lap
lapt
lapto
laptop
```

you might trigger:

```text
request 1
request 2
request 3
request 4
request 5
request 6
```

Now you have:

```text
six requests
```

potentially overlapping.

This raises several questions:

```text
Should we request on every keystroke?
Should we debounce?
Should old requests be cancelled?
Should results be cached?
What if response #2 arrives after #6?
```

This is where simply knowing `useEffect` is no longer enough.

---

# 22. Debouncing

Suppose the user types:

```text
l
la
lap
lapt
laptop
```

Instead of immediately requesting each time, you can wait until typing pauses.

Conceptually:

```text
user types
   ↓
wait 300ms
   ↓
did they type again?
   ↓ yes
reset timer
   ↓
eventually they stop
   ↓
send request
```

This is **debouncing**.

We'll later build a custom Hook such as:

```jsx
const debouncedSearch =
  useDebounce(search, 300);
```

Notice how custom Hooks, Effects, state, and data fetching are starting to combine.

---

# 23. Server data vs UI state

Now we can finally introduce an important concept.

Your component has:

```text
isModalOpen
```

and:

```text
products
```

These may both be stored using `useState`, but conceptually they are very different.

### `isModalOpen`

Created and controlled by your UI:

```text
client/UI state
```

### `products`

Originated from the server:

```text
server state
```

The server is ultimately the authority over the product data.

That's a fundamentally different situation.

---

# 24. Why server state is different

Suppose:

```text
your React app
```

has:

```text
products = [...]
```

but another user changes a product on the backend.

Your React state doesn't automatically know.

The server's version and your local copy can diverge.

So server state has properties such as:

```text
remote
asynchronous
shared
possibly stale
cacheable
refetchable
```

That's why server-state management eventually becomes its own topic.

---

# 25. Don't immediately think "put API data into Redux"

This is an important architectural point.

A beginner may think:

```text
API data
 ↓
global state
 ↓
Redux
```

But server state has specialized needs:

```text
caching
deduplication
refetching
stale data
background updates
request status
pagination
mutation invalidation
```

A generic client-state store isn't necessarily the best abstraction for all of that.

We'll later look at **TanStack Query** and other server-state approaches.

But first you need to understand the problem itself.

---

# 26. Why our `useState + useEffect` approach becomes repetitive

Imagine five pages:

```text
Users
Products
Orders
Comments
Posts
```

Each contains:

```text
data state
loading state
error state
Effect
fetch
cleanup
retry
```

You'll start writing the same basic machinery repeatedly.

For example:

```jsx
const [data, setData] = useState(null);
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);

useEffect(() => {
  ...
}, []);
```

That's a sign that reusable abstractions may help.

---

# 27. Extracting the fetch logic

You can create a custom Hook:

```jsx
function useProducts() {
  const [products, setProducts] =
    useState([]);

  const [loading, setLoading] =
    useState(true);

  const [error, setError] =
    useState(null);

  useEffect(() => {
    async function loadProducts() {
      try {
        const response =
          await fetch("/api/products");

        if (!response.ok) {
          throw new Error(
            "Failed to fetch products"
          );
        }

        const data =
          await response.json();

        setProducts(data);
      } catch (error) {
        setError(error.message);
      } finally {
        setLoading(false);
      }
    }

    loadProducts();
  }, []);

  return {
    products,
    loading,
    error
  };
}
```

Then:

```jsx
function Products() {
  const {
    products,
    loading,
    error
  } = useProducts();

  // UI only
}
```

Now your component focuses more on rendering.

---

# 28. Notice the separation

Before:

```text
Products component
├── request logic
├── loading logic
├── error logic
├── state
└── UI
```

After:

```text
useProducts
├── request
├── state
├── loading
└── error

Products
└── UI
```

This is one of the places where custom Hooks become genuinely useful.

---

# 29. But don't conclude "all API calls belong in custom Hooks"

That's too simplistic.

A custom Hook is useful for reusable **client-side behavior**.

For example:

```text
useProducts
useUser
useOnlineStatus
useDebounce
```

But as applications become more advanced, server-state libraries and route data APIs can handle concerns that a hand-written Hook would otherwise have to reinvent.

That's why we're learning the underlying problem first.

---

# 30. One crucial distinction: fetching vs rendering

Consider:

```jsx
const products = await fetchProducts();
```

That is data access.

Then:

```jsx
<ProductList products={products} />
```

is rendering.

Keeping those concepts conceptually separate helps you build cleaner applications:

```text
data layer
   ↓
state/data management
   ↓
UI components
```

You'll eventually build a structure more like:

```text
ProductsPage
    ↓
useProducts()
    ↓
API/server-state layer
    ↓
backend
```

---

# 31. Data fetching and route state

Remember our previous lesson:

```text
/products?search=laptop&category=electronics
```

Now combine that with fetching.

```text
URL
 ↓
search/category
 ↓
request parameters
 ↓
API
 ↓
products
 ↓
UI
```

So changing the URL can cause a new data request.

For example:

```text
/products?category=electronics
```

becomes:

```text
/products?category=accessories
```

The component receives new search params.

That can cause your data-loading logic to request a different dataset.

This is one of the main reasons URL state is so useful.

---

# 32. Example

```jsx
function Products() {
  const [searchParams] =
    useSearchParams();

  const category =
    searchParams.get("category") ?? "all";

  const [products, setProducts] =
    useState([]);

  useEffect(() => {
    async function loadProducts() {
      const response =
        await fetch(
          `/api/products?category=${category}`
        );

      const data =
        await response.json();

      setProducts(data);
    }

    loadProducts();
  }, [category]);

  // ...
}
```

Now:

```text
URL changes
 ↓
category changes
 ↓
Effect dependency changes
 ↓
new request
 ↓
new server data
 ↓
render
```

That's a complete end-to-end application flow.

---

# 33. This is where React really starts to become systematic

You can now reason about a real page:

```text
                    URL
                     ↓
                search params
                     ↓
                component
                     ↓
                data request
                     ↓
          ┌──────────┼──────────┐
          ↓          ↓          ↓
       loading      data       error
                     ↓
              derived values
                     ↓
                   JSX
                     ↓
                    UI
```

And interaction can start the cycle again:

```text
user changes filter
       ↓
URL changes
       ↓
request changes
       ↓
server responds
       ↓
new data
       ↓
new UI
```

This is the kind of flow you should learn to see instinctively.

---

# 34. What I want you to notice about `useEffect`

At this point, you should **not** think:

> "useEffect is the thing we use whenever we fetch."

The deeper model is:

```text
Effect
→ synchronization with something outside React
```

Fetching happens to be one important example.

The data flow is:

```text
React state/props
      ↓
Effect
      ↓
network
      ↓
response
      ↓
state
      ↓
render
```

That explains why the Effect exists.

---

# 35. One more practical improvement: initial data

Sometimes you already have data.

For example:

```jsx
const [products, setProducts] =
  useState(initialProducts);
```

Then the UI can render immediately while a refresh happens in the background.

That leads to concepts such as:

```text
stale data
background refetch
refreshing
revalidation
```

Those are central to server-state management.

We won't fake our way through them with more and more `useState` variables yet.

We'll build the conceptual foundation first.

---

# 36. The state machine you should start seeing

For a simple request:

```text
         START
           ↓
        LOADING
       /       \
      ↓         ↓
  SUCCESS      ERROR
      ↓
   ┌───────┐
   ↓       ↓
 EMPTY    DATA
```

And after a refetch:

```text
existing data
      ↓
refreshing
      ↓
new data
```

This distinction is important.

You don't necessarily want to erase the old data every time a request starts.

For example:

```text
Products from previous request
+
small loading indicator
```

can be a better user experience than:

```text
blank screen
+
large spinner
```

We'll explore this in server-state management.

---

# 37. A practical fetch checklist

When writing client-side data fetching, mentally check:

```text
Where does the data come from?
What triggers the request?
What are the dependencies?
What is the loading state?
What is the error state?
What is the empty state?
What happens if inputs change?
Can requests overlap?
Should stale requests be cancelled?
What happens on retry?
Is this data local or server-owned?
```

You don't need all these answers for a tiny demo.

But you should start asking them in real applications.

---

# 38. Today's big mental model

This is what I want you to internalize:

```text
                COMPONENT
                    │
              props/state/URL
                    │
                    ↓
                useEffect
                    │
                    ↓
                 NETWORK
                    │
                    ↓
                 RESPONSE
                    │
                    ↓
                setState()
                    │
                    ↓
                 RENDER
                    │
                    ↓
                    UI
```

And one level deeper:

```text
UI state
   ≠
server state
```

They may both appear as JavaScript values in your component, but they have very different ownership and lifecycle.

---

# What you should practice

Build a `Users` page using:

```text
GET /api/users
```

Your UI should support these states:

```text
Loading...
```

```text
Error: Could not load users
```

```text
No users found
```

or:

```text
Alex
Sarah
Mike
```

Then add:

```text
Retry
```

When Retry is clicked, make another request.

Don't worry about building a perfect reusable data-fetching system yet.

Focus on understanding:

```text
request
 ↓
loading
 ↓
success/error
 ↓
state
 ↓
render
```

---

## Next lesson — Server State and TanStack Query

This is where we'll answer the question that naturally comes up after today's lesson:

> "Why am I manually maintaining `data`, `loading`, `error`, cancellation, refetching, and caching every time I talk to an API?"

We'll first understand **what server state actually is**, then see what a server-state library gives you that plain `useState + useEffect` doesn't.

The important part will not be memorizing a library API. We'll learn **the problems the library is solving**, so later tools make sense rather than feeling like more React magic.