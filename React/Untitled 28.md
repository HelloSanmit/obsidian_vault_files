
# Lesson 26 — Error Boundaries and Error Handling in React

So far we've learned how to make React applications work.

Now we need to learn how to make them **fail gracefully**.

Because in a real application, something eventually goes wrong:

```text
component crashes
API fails
data is malformed
third-party code throws
unexpected state appears
a rendering bug happens
```

And there's a huge difference between:

```text
one small part of the page breaks
```

and:

```text
the entire application becomes unusable
```

React's **Error Boundary** mechanism exists largely to control that blast radius. Current React still uses Error Boundaries for errors thrown during rendering of child components; there is still no direct function-component equivalent of an Error Boundary built into React itself. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

---

# 1. Start with ordinary JavaScript errors

Suppose:

```jsx
function Product() {
  const product = null;

  return <h1>{product.name}</h1>;
}
```

At runtime:

```text
product = null
       ↓
product.name
       ↓
error
```

Something has thrown during rendering.

If nothing catches it appropriately, React cannot successfully produce that subtree.

---

# 2. Why `try/catch` doesn't solve render errors

A beginner may try:

```jsx
function App() {
  try {
    return <Product />;
  } catch (error) {
    return <p>Something went wrong</p>;
  }
}
```

That does **not** work as a general mechanism for errors thrown while React renders `Product`.

Why?

Because:

```jsx
<Product />
```

doesn't execute `Product()` directly inside your `try` block in the ordinary JavaScript sense.

You're describing a React element.

React later processes that element during its rendering work.

So:

```text
try/catch around JSX
        ≠
catch errors from React's rendering work
```

React's own lint guidance explicitly calls out this distinction: ordinary `try/catch` cannot catch errors thrown during React's rendering process; Error Boundaries are the mechanism intended for that job. ([React](https://react.dev/reference/eslint-plugin-react-hooks/lints/error-boundaries?utm_source=chatgpt.com "error-boundaries – React"))

---

# 3. What is an Error Boundary?

An Error Boundary is a component that says:

> "If something below me crashes during React rendering, show this fallback instead of letting that failure take down the whole UI region."

Conceptually:

```text
ErrorBoundary
└── ProductPage
    ├── ProductImage
    ├── ProductInfo
    └── Reviews
```

If:

```text
Reviews
   ↓
throws
```

you can show:

```text
Something went wrong loading this section.
[Try again]
```

while the rest of the page can remain available.

---

# 4. The first important architectural idea: blast radius

Suppose you have:

```text
App
├── Header
├── Sidebar
├── MainContent
└── Footer
```

If the entire application is under one boundary:

```text
ErrorBoundary
└── App
```

then a serious render error anywhere under `App` can result in the whole application area switching to the same fallback.

But if you do:

```text
App
├── Header
├── Sidebar
├── ErrorBoundary
│   └── MainContent
└── Footer
```

then a crash in MainContent doesn't necessarily take down Header, Sidebar, or Footer.

So Error Boundary placement is really a question of:

> **How much UI should fail together?**

Current React guidance explicitly discusses choosing boundary granularity based on where it makes sense to show an error—for example, around a conversation list or individual message rather than around every tiny avatar. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

---

# 5. The unusual thing: Error Boundaries are class components

We have spent almost the entire course writing function components:

```jsx
function Product() {
  ...
}
```

But the built-in Error Boundary mechanism is still based on two class lifecycle methods:

```jsx
static getDerivedStateFromError(error)
```

and:

```jsx
componentDidCatch(error, info)
```

There isn't currently a direct built-in function-component version of those lifecycle methods. React's current guidance explicitly notes this. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

That makes Error Boundaries a useful opportunity to understand a little old-school React.

---

# 6. A minimal Error Boundary

```jsx
import { Component } from "react";

class ErrorBoundary extends Component {
  state = {
    hasError: false,
  };

  static getDerivedStateFromError(error) {
    return {
      hasError: true,
    };
  }

  componentDidCatch(error, info) {
    console.error(error, info);
  }

  render() {
    if (this.state.hasError) {
      return <p>Something went wrong.</p>;
    }

    return this.props.children;
  }
}
```

Use it like:

```jsx
<ErrorBoundary>
  <ProductPage />
</ErrorBoundary>
```

Now:

```text
ProductPage crashes
       ↓
ErrorBoundary catches it
       ↓
state becomes:
hasError = true
       ↓
boundary rerenders
       ↓
fallback appears
```

That's the core mechanism.

---

# 7. Why `getDerivedStateFromError` exists

Look at:

```jsx
static getDerivedStateFromError(error) {
  return {
    hasError: true,
  };
}
```

It converts:

```text
error occurred
```

into:

```text
state saying:
"render the fallback"
```

Then `render()` sees:

```jsx
if (this.state.hasError) {
  return <p>Something went wrong.</p>;
}
```

So the sequence is:

```text
child throws
   ↓
getDerivedStateFromError
   ↓
state update
   ↓
render boundary
   ↓
fallback
```

The method is expected to be pure; it determines what state should be displayed in response to the error. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

---

# 8. Why `componentDidCatch` exists separately

Now:

```jsx
componentDidCatch(error, info) {
  console.error(error, info);
}
```

This is the place for side effects such as error reporting.

For example:

```text
error
 ↓
send to monitoring service
 ↓
log component stack
 ↓
record diagnostics
```

That's why React separates:

```text
getDerivedStateFromError
=
what should the UI display?

componentDidCatch
=
what side effect should we perform because this happened?
```

React's current API guidance recommends `componentDidCatch` for reporting/logging and `getDerivedStateFromError` for switching the displayed UI to the fallback. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

---

# 9. Think of an Error Boundary like a circuit breaker

This analogy is useful.

Imagine an electrical system:

```text
Power source
   ↓
Circuit breaker
   ↓
Section A
Section B
Section C
```

If Section B has a fault, the breaker prevents the failure from destroying everything.

Likewise:

```text
React tree
    ↓
Error Boundary
    ↓
feature
```

If the feature crashes:

```text
feature fails
   ↓
boundary isolates failure
   ↓
fallback UI
```

The boundary is defining a **failure containment region**.

---

# 10. Nested Error Boundaries

This becomes more interesting when boundaries are nested:

```text
App
└── Boundary A
    ├── Header
    └── Boundary B
        ├── Main
        └── Boundary C
            └── Reviews
```

Suppose:

```text
Reviews throws
```

React looks for the nearest boundary capable of handling that error.

That might be:

```text
Boundary C
```

So:

```text
Reviews
 ↓
Boundary C
 ↓
Reviews fallback
```

The outer boundaries don't necessarily need to take over.

This lets you design progressively larger fallback regions.

---

# 11. A realistic application structure

Imagine:

```text
App
├── Header
├── Navigation
│
├── ErrorBoundary
│   └── Dashboard
│       ├── RevenueChart
│       ├── SalesTable
│       └── ActivityFeed
│
└── Footer
```

Now suppose `RevenueChart` throws.

You could show:

```text
Revenue chart unavailable.
```

while:

```text
SalesTable
ActivityFeed
```

still work.

That's much better than:

```text
The whole dashboard disappeared.
```

---

# 12. Error Boundaries are not API error states

This distinction is extremely important given everything we've learned about TanStack Query.

Suppose:

```jsx
const query = useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
});
```

and the server returns:

```text
500 Internal Server Error
```

Your query function might do:

```js
if (!response.ok) {
  throw new Error("Failed to fetch products");
}
```

Then TanStack Query gives you:

```text
isError = true
```

You might render:

```jsx
if (query.isError) {
  return <p>Could not load products.</p>;
}
```

That is **application-level data-fetching error handling**.

It's different from a component crashing during rendering.

---

# 13. Two very different failure modes

### Server/data error

```text
GET /products
 ↓
500
 ↓
query function throws
 ↓
TanStack Query state:
isError = true
 ↓
component renders error UI
```

### Render error

```text
component receives malformed/unexpected data
 ↓
component itself throws during render
 ↓
Error Boundary
 ↓
fallback
```

These should not be mentally collapsed into one thing.

---

# 14. Example combining both

```jsx
function ProductList() {
  const query = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });

  if (query.isPending) {
    return <Spinner />;
  }

  if (query.isError) {
    return (
      <p>
        Couldn't load products.
      </p>
    );
  }

  return <Products products={query.data} />;
}
```

Now:

```text
network/API failure
       ↓
query handles it
       ↓
controlled UI
```

But suppose:

```jsx
function Products({ products }) {
  return products.map(product => (
    <ProductCard
      key={product.id}
      product={product}
    />
  ));
}
```

and one `ProductCard` contains a bug:

```jsx
function ProductCard({ product }) {
  return <h2>{product.details.name.toUpperCase()}</h2>;
}
```

but:

```js
product.details === undefined
```

Now the component can throw.

That is a different category of failure:

```text
query succeeded
        ↓
data arrived
        ↓
rendering code is buggy
        ↓
Error Boundary needed
```

This distinction is fundamental.

---

# 15. Error Boundaries don't catch everything

This is one of the most important parts of the topic.

An Error Boundary does **not** catch errors from every piece of JavaScript.

Current React explicitly lists several categories it doesn't catch automatically, including errors in event handlers, server-side rendering, asynchronous callbacks such as `setTimeout`, and errors thrown by the boundary itself. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

Let's go through those.

---

# 16. Event-handler errors

Suppose:

```jsx
function Button() {
  function handleClick() {
    throw new Error("Boom");
  }

  return (
    <button onClick={handleClick}>
      Click
    </button>
  );
}
```

The error happens when the user performs an event.

This:

```text
click
 ↓
event handler
 ↓
throw
```

is not the same as:

```text
render
 ↓
throw
```

Error Boundaries are designed primarily for rendering/lifecycle failures, not arbitrary event-handler exceptions. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

So for event handlers, ordinary JavaScript handling is appropriate:

```jsx
function handleClick() {
  try {
    riskyOperation();
  } catch (error) {
    setError(error);
  }
}
```

This is exactly the situation where `try/catch` can be appropriate.

---

# 17. Event handler vs render: compare them carefully

### Render

```jsx
function Component() {
  throw new Error("Render failed");

  return <div />;
}
```

Use:

```text
Error Boundary
```

### Event

```jsx
function Component() {
  function handleClick() {
    try {
      riskyOperation();
    } catch (error) {
      ...
    }
  }

  return <button onClick={handleClick} />;
}
```

Use:

```text
try/catch or explicit application error handling
```

This distinction becomes second nature once you identify **when the error occurs**.

---

# 18. Async errors are another category

Consider:

```jsx
useEffect(() => {
  setTimeout(() => {
    throw new Error("Boom");
  }, 1000);
}, []);
```

The error happens later inside the timer callback.

That isn't a render-time child error.

An Error Boundary doesn't magically catch every asynchronous exception. Current React explicitly lists asynchronous code such as `setTimeout` callbacks among errors boundaries don't catch. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

You need the asynchronous operation itself to handle failure appropriately.

For example:

```js
try {
  await somethingAsync();
} catch (error) {
  ...
}
```

or an explicit promise rejection handler.

---

# 19. What about `useEffect`?

This deserves careful thinking.

The effect itself is part of React's lifecycle, but code executed asynchronously from that effect can escape the Error Boundary's rendering-error mechanism.

For example:

```jsx
useEffect(() => {
  someAsyncWork().catch(error => {
    setError(error);
  });
}, []);
```

Here you're deliberately converting the asynchronous failure into application state:

```text
async failure
 ↓
setError(error)
 ↓
render
 ↓
error UI
```

That's often a better design for expected async failures anyway.

For example:

```text
network error
```

isn't necessarily a component crash.

It's a normal operational failure that your UI should be prepared to represent.

---

# 20. Expected errors vs programmer bugs

This distinction is incredibly useful.

### Expected operational error

```text
API unavailable
user not authorized
validation rejected
file upload failed
payment declined
```

You often want:

```text
explicit error state
```

### Unexpected programming error

```text
undefined access
invalid assumption
render logic bug
broken third-party component
```

You often want:

```text
Error Boundary
+
error reporting
```

So:

```text
expected failure
→ model it in application state

unexpected render failure
→ isolate with Error Boundary
```

This distinction is one of the most useful practical lessons from the topic.

---

# 21. A good fallback is specific

Compare:

```text
Something went wrong.
```

with:

```text
We couldn't display your activity right now.

[Try again]
```

The second is much more useful to the user.

The boundary's job is not necessarily to explain the underlying JavaScript error.

It should provide the correct **UI-level fallback for that region**.

The developer-facing details belong in logging/monitoring.

---

# 22. Don't show raw error objects to users

Suppose:

```js
error.message
```

is:

```text
Cannot read properties of undefined (reading 'name')
```

That's useful to a developer.

Not to the user.

So avoid:

```jsx
<p>{error.message}</p>
```

as your default product UI.

Instead:

```jsx
<p>
  This section couldn't be displayed.
</p>
```

and send the actual diagnostic information to your monitoring system.

---

# 23. Logging matters

An Error Boundary can collect:

```text
error
component stack
context
```

and send it to an error-monitoring service.

React's current `componentDidCatch` information includes a component stack describing where the error occurred. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

That gives developers something much more useful than:

```text
"Something broke."
```

You can potentially learn:

```text
ProductPage
 → Reviews
   → ReviewCard
     → Avatar
```

and identify the failing component.

---

# 24. React 19+ has root-level error reporting hooks too

Modern React has expanded root-level error handling.

When creating a root, there are options such as:

```jsx
createRoot(container, {
  onCaughtError(error, errorInfo) {
    ...
  },

  onUncaughtError(error, errorInfo) {
    ...
  },
});
```

React introduced these in React 19 as part of its improved error reporting, and current React continues to support them. ([React](https://react.dev/blog/2024/04/25/react-19-upgrade-guide?utm_source=chatgpt.com "React 19 Upgrade Guide – React"))

Conceptually:

```text
caught by boundary
     ↓
onCaughtError

not caught by boundary
     ↓
onUncaughtError
```

This gives an application-wide place for error reporting.

But it doesn't replace local Error Boundaries.

Think:

```text
local boundary
=
local recovery/fallback

root-level handler
=
global observation/reporting
```

---

# 25. A sophisticated application can combine them

For example:

```text
                        App
                         │
                 global error reporting
                         │
             ┌───────────┼───────────┐
             ↓           ↓           ↓
          Header      Dashboard     Footer
                         │
                  Error Boundary
                         │
                ┌────────┼────────┐
                ↓        ↓        ↓
             Chart     Table    Activity
```

Now if:

```text
Activity
```

crashes:

```text
Activity
 ↓
local Error Boundary
 ↓
activity fallback
```

while:

```text
global reporting
```

can still record the incident.

That's a good production architecture.

---

# 26. What if the Error Boundary itself crashes?

Suppose:

```jsx
class ErrorBoundary extends Component {
  render() {
    if (this.state.hasError) {
      return <BrokenFallback />;
    }
  }
}
```

and `BrokenFallback` itself throws.

The boundary can't catch its own failure.

That's why you may have broader boundaries above smaller boundaries.

Conceptually:

```text
Outer Boundary
└── Inner Boundary
    └── App section
```

If the inner boundary itself fails:

```text
Inner Boundary
      ↓
failure
      ↓
Outer Boundary
```

Current React explicitly notes that an Error Boundary does not catch errors thrown inside the boundary itself. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))

---

# 27. Retry is an interesting problem

Suppose the boundary shows:

```text
Something went wrong.

[Try again]
```

How should Retry work?

One simple strategy is to reset the boundary's state.

For our conceptual class component:

```jsx
handleRetry = () => {
  this.setState({
    hasError: false,
  });
};
```

Then:

```text
click Retry
 ↓
hasError = false
 ↓
children render again
```

If the underlying problem was temporary, the UI may recover.

If the bug is deterministic:

```text
children render
 ↓
same bug
 ↓
same error
```

and the fallback returns.

---

# 28. Why blindly retrying can be useless

Imagine:

```jsx
function BrokenComponent() {
  const user = undefined;

  return <h1>{user.name}</h1>;
}
```

Retrying will not fix that.

The same code still throws.

So Retry is especially meaningful when:

```text
the underlying condition may have changed
```

For example:

```text
lazy-loaded code failed temporarily
data/environment recovered
component's external dependency changed
```

For a deterministic programming bug:

```text
retry
```

is unlikely to solve the underlying issue.

---

# 29. Error Boundary and route-level recovery

This becomes useful in applications with routing.

Imagine:

```text
App
└── Router
    ├── Home
    ├── Products
    ├── Settings
    └── Admin
```

You may want a boundary around an entire route:

```text
Products
   ↓
Error Boundary
   ↓
Product route
```

Then a bug in the product page doesn't necessarily destroy navigation to:

```text
Home
Settings
```

This is a natural continuation of our earlier routing lesson.

---

# 30. Boundary placement is essentially product design

Suppose your messaging app is:

```text
Header
├── ConversationList
├── Conversation
└── Composer
```

Where should the boundary be?

Maybe:

```text
ConversationList
```

gets its own.

Maybe:

```text
Conversation
```

gets another.

Maybe:

```text
Composer
```

has separate error handling.

Why?

Because each region has a different meaningful recovery experience.

This is why Error Boundaries aren't just a technical mechanism.

They're part of UI architecture.

---

# 31. Don't put one around every tiny component

You might theoretically do:

```jsx
<ErrorBoundary>
  <Avatar />
</ErrorBoundary>

<ErrorBoundary>
  <Badge />
</ErrorBoundary>

<ErrorBoundary>
  <Button />
</ErrorBoundary>
```

That's usually excessive.

Now your app has:

```text
many boundaries
many fallback decisions
many recovery surfaces
more complexity
```

Instead, group things by meaningful failure/recovery regions.

The right question is:

> **"If this part breaks, what useful part of the UI should remain?"**

That usually leads to sensible boundary placement.

---

# 32. Error boundaries and Suspense are related but different

We haven't properly taught Suspense yet, but you're about to encounter it.

A useful first distinction:

```text
Suspense
=
"I'm waiting for something."

Error Boundary
=
"Something went wrong."
```

So:

```text
loading
 ↓
Suspense fallback

error
 ↓
Error Boundary fallback
```

They can be composed:

```jsx
<ErrorBoundary fallback={<ErrorUI />}>
  <Suspense fallback={<LoadingUI />}>
    <Page />
  </Suspense>
</ErrorBoundary>
```

Current React uses Error Boundaries for rejected promises passed through `use`, while Suspense handles the pending state. ([React](https://react.dev/reference/react/use?utm_source=chatgpt.com "use – React"))

This will matter a lot when we discuss modern React's async rendering model.

---

# 33. Why Error Boundaries fit our state taxonomy

We've learned:

```text
local UI state
URL state
server state
context
derived data
mutation state
```

Now add:

```text
error state
```

But be careful.

There isn't one universal "error state."

Different failures belong to different systems.

For example:

```text
query error
    → server-state/query concern

form validation error
    → form/application state

rendering crash
    → Error Boundary

event-handler failure
    → event/application handling

authentication failure
    → auth/application state
```

Again, the question is always:

> **What kind of failure is this, and which system owns it?**

---

# 34. A complete example

Imagine:

```jsx
function Dashboard() {
  return (
    <>
      <Header />

      <ErrorBoundary
        fallback={
          <p>
            Dashboard unavailable.
            <button>Try again</button>
          </p>
        }
      >
        <DashboardContent />
      </ErrorBoundary>

      <Footer />
    </>
  );
}
```

Then:

```jsx
function DashboardContent() {
  const query = useQuery({
    queryKey: ["dashboard"],
    queryFn: fetchDashboard,
  });

  if (query.isPending) {
    return <Spinner />;
  }

  if (query.isError) {
    return (
      <p>
        We couldn't load dashboard data.
      </p>
    );
  }

  return (
    <RevenueChart data={query.data.revenue} />
  );
}
```

Notice the two layers:

```text
API fails
 ↓
query.isError
 ↓
expected server-state error UI
```

But:

```text
RevenueChart contains a bug
 ↓
render throws
 ↓
Error Boundary
 ↓
dashboard-level fallback
```

This distinction is exactly what you want in a production application.

---

# 35. The final mental model

Think of React errors as a few different categories:

```text
                    FAILURE
                       │
         ┌─────────────┼─────────────┐
         ↓             ↓             ↓
     expected       rendering      async/event
      failure         crash          failure
         │             │             │
         ↓             ↓             ↓
   application      Error        explicit
     state         Boundary      handling
         │             │             │
         ↓             ↓             ↓
    error UI       fallback      state/catch
```

Then add the server-state layer:

```text
API failure
   ↓
query/mutation error state
   ↓
render appropriate UI
```

And global reporting:

```text
unexpected error
   ↓
Error Boundary/root handler
   ↓
report to monitoring
```

---

# 36. The principles to remember

You don't need to memorize the class syntax right now.

Understand these:

```text
1. Error Boundaries protect a React subtree from rendering failures.

2. They let one part of the UI fail without necessarily destroying
   unrelated parts.

3. The nearest appropriate boundary handles the failure.

4. getDerivedStateFromError determines fallback state.

5. componentDidCatch is useful for side effects such as logging.

6. There is currently no direct built-in function-component
   implementation of Error Boundaries.

7. try/catch is not a replacement for Error Boundaries around JSX/rendering.

8. Event-handler and arbitrary asynchronous errors need their own handling.

9. Expected API failures usually belong in explicit query/application
   error state rather than being treated as component crashes.

10. Boundary placement determines the failure/recovery blast radius.

11. A fallback is a product decision, not merely a developer message.

12. Error Boundaries and Suspense solve different failure/waiting states.
```

One particularly useful distinction to carry forward is:

```text
"Something went wrong with the request"
```

is different from:

```text
"My component crashed while trying to display the result."
```

The first is ordinary application/network state.

The second is a rendering failure.

Once you recognize that difference, you'll know whether to reach for query state, local error state, `try/catch`, or an Error Boundary instead of treating every failure as the same problem. ([React](https://react.dev/reference/react/Component?utm_source=chatgpt.com "Component – React"))