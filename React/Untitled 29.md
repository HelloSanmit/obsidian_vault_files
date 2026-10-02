
# Lesson 27 — Suspense: Loading Without Manually Wiring Every Loading State

We've just learned Error Boundaries.

Now we're going to look at their close relative:

```text
Error Boundary
    ↓
something failed

Suspense
    ↓
something isn't ready yet
```

Suspense is one of the concepts that feels strange until you understand what React is actually doing.

And there's one especially important point for our current React understanding:

> **Suspense is not simply a nicer `isLoading` boolean.**

It represents a different way of coordinating rendering around something that isn't ready yet.

Current React's `<Suspense>` can show fallback UI when its children suspend, and current React can suspend for lazy-loaded component code, reading Promises with `use`, and several other React-integrated loading mechanisms. It does **not** automatically detect arbitrary fetching done inside `useEffect`. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

---

# 1. Start with the familiar pattern

We've already written code like:

```jsx
function ProductPage() {
  const query = useQuery({
    queryKey: ["product", 42],
    queryFn: fetchProduct,
  });

  if (query.isPending) {
    return <Spinner />;
  }

  if (query.isError) {
    return <ErrorMessage />;
  }

  return <Product product={query.data} />;
}
```

The thinking is:

```text
request starts
    ↓
query state = pending
    ↓
render spinner
    ↓
request finishes
    ↓
render actual content
```

This is perfectly valid.

Suspense approaches the problem differently.

---

# 2. The basic Suspense idea

You can write:

```jsx
<Suspense fallback={<Spinner />}>
  <ProductPage />
</Suspense>
```

Conceptually:

```text
Try to render ProductPage
        ↓
Is everything needed to render it ready?
      ↙               ↘
    yes                no
     ↓                  ↓
render content      suspend
                        ↓
                  show fallback
                        ↓
                  wait for readiness
                        ↓
                  try rendering again
```

That's the core mental model.

The component doesn't necessarily say:

```text
"loading = true"
```

Instead, something during rendering effectively says:

> "I can't finish rendering this part yet."

React catches that suspension and displays the nearest Suspense fallback. Current React describes this as the children "suspending while rendering." ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

---

# 3. "Suspend" doesn't mean "crash"

This distinction matters.

An ordinary error is:

```text
something is wrong
```

Suspension is:

```text
nothing is wrong,
I'm just not ready yet
```

Think:

```text
Error
 ↓
stop because something failed

Suspension
 ↓
pause/retry because something isn't ready
```

That's why Suspense and Error Boundaries are complementary.

---

# 4. The easiest place to see Suspense: lazy loading

Suppose your app has:

```text
Home
Products
Admin
Analytics
```

Maybe Analytics is large and users don't visit it often.

Instead of loading its JavaScript immediately, we can use:

```jsx
import { lazy } from "react";

const Analytics = lazy(
  () => import("./Analytics")
);
```

Notice where this declaration lives:

```text
outside the component
```

That's important. React's current `lazy` API expects the lazy component to be declared at module level; declaring it inside a component can cause state to reset unexpectedly because a new lazy component is created across renders. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

---

# 5. What does `lazy()` actually do?

Normally:

```jsx
import Analytics from "./Analytics";
```

means the module is part of the normal module loading process.

With:

```jsx
const Analytics = lazy(
  () => import("./Analytics")
);
```

the code for Analytics can be loaded when React first attempts to render it.

So conceptually:

```text
initial application
    ↓
Analytics code not yet loaded

user navigates to Analytics
    ↓
render Analytics
    ↓
React needs its component code
    ↓
dynamic import starts
    ↓
component suspends
    ↓
Suspense fallback
    ↓
module finishes loading
    ↓
React retries
    ↓
Analytics renders
```

Current React's `lazy` behavior caches the Promise and resolved module, and a rejected lazy import is passed to the nearest Error Boundary. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

---

# 6. `lazy` and `Suspense` naturally belong together

For example:

```jsx
import {
  lazy,
  Suspense,
} from "react";

const Analytics = lazy(
  () => import("./Analytics")
);

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Analytics />
    </Suspense>
  );
}
```

Now the responsibilities are nicely separated:

```text
lazy
 ↓
load component code later

Suspense
 ↓
what should we show while it isn't ready?

Error Boundary
 ↓
what should we show if loading that component fails?
```

That is a very clean combination.

---

# 7. Why this is better than manually tracking component-code loading

You could theoretically do something like:

```jsx
const [loading, setLoading] = useState(false);

async function loadAnalytics() {
  setLoading(true);

  const module =
    await import("./Analytics");

  setComponent(() => module.default);

  setLoading(false);
}
```

But now you've manually built:

```text
loading state
dynamic import
component storage
error state
retry
render transition
```

Suspense provides a React-level abstraction for coordinated loading UI.

That's one of the big reasons it exists.

---

# 8. The boundary defines the loading region

Suppose:

```jsx
<Suspense fallback={<FullPageSpinner />}>
  <Dashboard />
</Suspense>
```

If Dashboard suspends:

```text
Dashboard
   ↓
FullPageSpinner
```

The whole region under that boundary is replaced by the fallback.

But you can choose a smaller boundary:

```jsx
<Header />

<Suspense fallback={<ChartSkeleton />}>
  <RevenueChart />
</Suspense>

<ActivityFeed />
```

Now:

```text
Header
ActivityFeed
```

can remain visible while:

```text
RevenueChart
```

is loading.

This is the same "failure blast radius" idea we learned with Error Boundaries, but now it's a **loading blast radius**.

---

# 9. Boundary placement is UX design

Suppose you have:

```text
Product page
├── Product information
├── Reviews
├── Recommendations
└── Related articles
```

You could do:

```jsx
<Suspense fallback={<WholePageSkeleton />}>
  <ProductPage />
</Suspense>
```

or:

```jsx
<ProductInformation />

<Suspense fallback={<ReviewsSkeleton />}>
  <Reviews />
</Suspense>

<Suspense fallback={<RecommendationsSkeleton />}>
  <Recommendations />
</Suspense>
```

The second design lets independent regions become available independently.

Conceptually:

```text
Product information
       ↓
ready immediately

Reviews
       ↓
loading
       ↓
Reviews skeleton

Recommendations
       ↓
loading
       ↓
Recommendations skeleton
```

That's often a better user experience than:

```text
everything hidden
      ↓
giant spinner
      ↓
everything appears simultaneously
```

---

# 10. Nested Suspense boundaries

You can nest them:

```jsx
<Suspense fallback={<PageSkeleton />}>
  <ProductPage>

    <ProductInformation />

    <Suspense fallback={<ReviewsSkeleton />}>
      <Reviews />
    </Suspense>

    <Suspense fallback={<RelatedSkeleton />}>
      <RelatedProducts />
    </Suspense>

  </ProductPage>
</Suspense>
```

Now you effectively have:

```text
Outer boundary
    ↓
whole page

Inner boundary
    ↓
reviews

Inner boundary
    ↓
related products
```

Each boundary can reveal its region independently, depending on what suspends.

This is one of the reasons Suspense is more than "a spinner component."

It's a **coordination mechanism for partial readiness**.

---

# 11. Important: Suspense does not detect ordinary `fetch()` in `useEffect`

This is a common misconception.

Consider:

```jsx
useEffect(() => {
  fetch("/api/products");
}, []);
```

That does **not** automatically trigger Suspense.

You still need:

```text
loading state
```

or a server-state library's loading model.

Current React explicitly states that Suspense does not detect data fetching performed inside an Effect or event handler. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

So:

```text
useEffect + fetch
      ↓
ordinary async state management

Suspense
      ↓
requires a Suspense-enabled source
```

That's a very important boundary.

---

# 12. Then how can data cause Suspense?

Modern React has the `use` API.

For example:

```jsx
import {
  Suspense,
  use,
} from "react";

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);

  return comments.map(comment => (
    <p key={comment.id}>
      {comment.text}
    </p>
  ));
}
```

And:

```jsx
<Suspense fallback={<p>Loading comments...</p>}>
  <Comments
    commentsPromise={commentsPromise}
  />
</Suspense>
```

When:

```js
use(commentsPromise)
```

encounters a Promise that isn't ready yet:

```text
Promise pending
    ↓
component suspends
    ↓
nearest Suspense boundary
    ↓
fallback
```

When the Promise resolves:

```text
Promise resolved
    ↓
React retries render
    ↓
use(...) returns value
    ↓
component completes
```

React introduced `use` in React 19 specifically to read resources such as Promises during rendering and suspend until they're ready. ([React](https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

---

# 13. But there's a critical rule about Promises

This is important enough to emphasize.

Don't do:

```jsx
function Comments() {
  const comments = use(
    fetchComments()
  );

  ...
}
```

where a new uncached Promise is created on each render.

That creates the equivalent of:

```text
render 1
 ↓
Promise A

render 2
 ↓
Promise B

render 3
 ↓
Promise C
```

and React has no stable resource to coordinate around.

React's current guidance explicitly warns against passing uncached Promises created during render to `use`; the Promise needs to be cached/reused, such as through a Suspense-enabled framework or appropriate resource mechanism. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

This is one reason you shouldn't equate:

```text
use(fetch(...))
```

with:

```text
useQuery(...)
```

They belong to different abstractions.

---

# 14. How TanStack Query fits with Suspense

This is where our previous lessons become relevant again.

We've been doing:

```jsx
const query = useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
});

if (query.isPending) {
  return <Spinner />;
}
```

TanStack Query also has Suspense-oriented APIs/configuration.

The conceptual difference becomes:

### Explicit query-state style

```text
query
 ↓
isPending
 ↓
component decides what to render
```

### Suspense-oriented style

```text
query/resource not ready
 ↓
component suspends
 ↓
Suspense boundary decides fallback
```

That can produce a much more compositional loading architecture.

But I don't want you to conclude:

> "Suspense replaces TanStack Query."

It doesn't.

TanStack Query solves:

```text
server-state caching
freshness
refetching
mutations
invalidation
```

Suspense solves:

```text
how React coordinates rendering around something not yet ready
```

Those are different responsibilities.

---

# 15. The difference between `isPending` and Suspense

Let's compare.

### Explicit state

```jsx
function Product() {
  const query = useQuery(...);

  if (query.isPending) {
    return <Spinner />;
  }

  return <ProductView data={query.data} />;
}
```

The component itself decides:

```text
"I'm loading, so I'll render this."
```

### Suspense

```jsx
<Suspense fallback={<Spinner />}>
  <Product />
</Suspense>
```

The component can conceptually say:

```text
"I need data before I can finish rendering."
```

and the parent boundary decides:

```text
"While you're not ready, I'll show this fallback."
```

That makes loading behavior more composable.

---

# 16. Think of Suspense as moving responsibility upward

Without Suspense:

```text
Product
 ↓
checks loading
 ↓
renders loading UI
```

With Suspense:

```text
Product
 ↓
needs something
 ↓
suspends

Parent
 ↓
decides what loading UI should represent the region
```

So the child doesn't necessarily need to know:

```text
"What spinner does my parent want?"
```

The boundary owns that decision.

This is a major architectural benefit.

---

# 17. Suspense is not "wait for all children"

Suppose:

```jsx
<Suspense fallback={<Spinner />}>
  <Header />
  <Sidebar />
  <ProductList />
  <Recommendations />
</Suspense>
```

If `ProductList` suspends, the boundary's fallback can replace the suspended content region rather than React simply waiting silently for every child.

And with nested boundaries, you can make the regions much more granular.

The important mental model is:

```text
boundary
 ↓
region with readiness requirement
```

not:

```text
entire application
 ↓
one giant loading state
```

---

# 18. Suspense + Error Boundary is a powerful pair

Here's a very useful pattern:

```jsx
<ErrorBoundary fallback={<ErrorUI />}>
  <Suspense fallback={<LoadingUI />}>
    <Analytics />
  </Suspense>
</ErrorBoundary>
```

Think:

```text
Analytics
   │
   ├── not ready
   │      ↓
   │   Suspense
   │      ↓
   │   LoadingUI
   │
   ├── ready
   │      ↓
   │   render
   │
   └── fails
          ↓
      Error Boundary
          ↓
        ErrorUI
```

If a lazy component's import rejects, React throws that rejection for the nearest Error Boundary to handle. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

So you can model:

```text
not ready → Suspense
failed     → Error Boundary
ready      → actual UI
```

That is an extremely clean state model.

---

# 19. Suspense changes the way you think about loading UI

Traditional thinking:

```text
component state
├── loading
├── error
└── data
```

Suspense-oriented thinking can be:

```text
resource
├── not ready → suspend
├── ready     → render
└── failed    → error
```

And the surrounding tree determines:

```text
while suspended → fallback
if error → error boundary
```

It's almost like moving control over these states from the component into the **tree architecture**.

---

# 20. `startTransition` becomes important here

Now we need another concept.

Suppose you're already displaying:

```text
Product Page
```

Then an interaction causes a new piece of UI to suspend.

Without any special coordination, React may need to show the Suspense fallback for the region.

But sometimes that's jarring.

Imagine:

```text
Current page
     ↓
user changes something
     ↓
new content suspends
     ↓
existing content disappears
     ↓
loading fallback
```

You may instead want:

```text
existing content
     ↓
remains visible
     ↓
new content loads in background-ish transition
     ↓
new content becomes ready
     ↓
switch over
```

React provides transitions for this kind of non-urgent update coordination.

This becomes especially important with Suspense.

---

# 21. Why `startTransition` helps

Conceptually:

```jsx
startTransition(() => {
  setTab("analytics");
});
```

You're telling React:

> "This update can be treated as non-urgent."

That gives React more flexibility to keep currently visible content while the new tree becomes ready rather than immediately replacing it with a Suspense fallback in situations where the update would otherwise reveal a fallback.

Current React's Suspense documentation explicitly notes that if already-visible content suspends again, React normally shows the fallback, unless the update causing the suspension was caused by `startTransition` or `useDeferredValue`. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

---

# 22. This is where transitions become different from loading state

Suppose:

```text
urgent:
typing into input

non-urgent:
rendering a huge search result section
```

You don't want:

```text
user types
 ↓
entire UI becomes blocked
```

You can conceptually separate:

```text
urgent interaction
+
non-urgent rendering work
```

And Suspense can coordinate the latter.

We'll go deeper into `startTransition` and `useDeferredValue` separately because they deserve their own lesson.

For now, just understand their relationship to Suspense.

---

# 23. Suspense and streaming server rendering

Now we reach one of the bigger architectural uses.

Imagine server-rendering:

```text
Page
├── Header
├── Main
├── Recommendations
└── Comments
```

Perhaps:

```text
Header
```

is immediately ready.

But:

```text
Comments
```

take longer.

With Suspense boundaries, the server can send parts of the page progressively rather than waiting for absolutely everything to be ready before delivering the whole thing.

Current React integrates Suspense with streaming server rendering and selective hydration. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

Conceptually:

```text
server

Header ready
   ↓
send Header

Main ready
   ↓
send Main

Comments not ready
   ↓
send fallback

Comments ready
   ↓
send/reveal Comments
```

This is a major reason Suspense exists at the architecture level.

It's not merely a client-side spinner API.

---

# 24. But don't jump to SSR yet

You don't need to learn:

```text
renderToPipeableStream
hydration
server components
streaming protocol
```

to understand Suspense.

For now, simply understand:

```text
Suspense gives React a boundary around readiness.
```

That same model can then work with:

```text
lazy-loaded code
Suspense-enabled data
server rendering
streaming
transitions
```

The core concept stays the same.

---

# 25. Suspense does not mean "async functions in components"

Another common misunderstanding.

You might think:

```jsx
async function Product() {
  const product = await fetchProduct();
  return <div>...</div>;
}
```

and:

```jsx
<Suspense>
  <Product />
</Suspense>
```

automatically solves it.

That's not the general client-component model.

Suspense needs React-integrated suspension mechanisms.

Examples include:

```text
lazy
use(Promise/resource)
Suspense-enabled frameworks/libraries
```

not just arbitrary asynchronous component functions.

So don't reduce Suspense to:

```text
"React supports async components."
```

That statement is far too broad.

---

# 26. A mental model that makes `use` much easier

Suppose:

```js
const commentsPromise = getComments();
```

Then:

```jsx
function Comments() {
  const comments = use(commentsPromise);

  return ...
}
```

Think of:

```text
use(promise)
```

as:

> "Give me this resource's value. If it isn't available yet, don't let me finish this render."

So:

```text
resource ready
   ↓
return value

resource pending
   ↓
suspend

resource failed
   ↓
error
```

Then the surrounding React tree decides what to show:

```text
pending
 ↓
Suspense

failure
 ↓
Error Boundary
```

That's a very elegant model.

---

# 27. Why `use` is unusual compared with hooks

You've learned the Rules of Hooks:

```text
hooks must be called in consistent positions
```

But `use` has special rules and can be called inside conditions/loops in ways ordinary hooks cannot.

For example, React's current `use` API is specifically documented as being callable inside loops and conditions, unlike ordinary Hooks. It still must be used within a component or hook and must not be wrapped in arbitrary `try/catch` logic that captures React's suspension mechanism. ([React](https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

That's a fairly advanced detail, so don't generalize:

```text
"use can be conditional"
```

into:

```text
"all hooks can be conditional."
```

Absolutely not.

---

# 28. Suspense boundaries behave like architectural boundaries

This is similar to the way we thought about:

```text
Error Boundary
```

A Suspense boundary says:

> "This region can temporarily be replaced because its contents aren't ready."

An Error Boundary says:

> "This region can be replaced because its contents failed."

So you can think:

```text
                 UI region
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
       suspense             error
          ↓                   ↓
       fallback             fallback
```

Both are forms of **failure/readiness containment**.

---

# 29. The fallback itself is real UI

Suppose:

```jsx
<Suspense fallback={<Spinner />}>
  <Product />
</Suspense>
```

When Product suspends, React renders:

```jsx
<Spinner />
```

That isn't a temporary browser-level magic overlay.

It's actual React UI participating in the render.

So:

```text
content tree
   ↓
can't currently render
   ↓
fallback tree
```

Then:

```text
ready
   ↓
content tree
```

Current React also has nuanced behavior when already-visible content suspends again, including cleanup/recreation of layout effects around content hidden by a Suspense fallback. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

That's an advanced detail we'll revisit when discussing transitions and Suspense behavior.

---

# 30. Avoid one giant Suspense boundary

Imagine:

```jsx
<Suspense fallback={<HugeSpinner />}>
  <App />
</Suspense>
```

Now one suspended region can potentially blank out a very large portion of the application.

It's technically valid.

But UX may be poor.

Instead, think:

```text
App
├── Header
├── Navigation
├── Suspense → main content
├── Suspense → sidebar widget
└── Footer
```

This creates smaller loading regions.

Again:

```text
boundary placement
=
UX architecture
```

---

# 31. But don't make every element its own boundary

The opposite extreme is also bad:

```text
Suspense → button
Suspense → heading
Suspense → icon
Suspense → paragraph
Suspense → image
```

Now the page can reveal dozens of tiny fallback states.

That's visually noisy and harder to reason about.

Instead, group components that make sense to become ready together.

For example:

```text
Product information
```

might reasonably be one loading region.

---

# 32. Skeletons work especially well with Suspense

Instead of:

```jsx
fallback={<Spinner />}
```

you can use:

```jsx
fallback={<ProductSkeleton />}
```

For example:

```text
┌──────────────────────────────┐
│ ███████████                  │
│                              │
│ ████████                     │
│ ███████████████              │
│                              │
│ ███████                      │
└──────────────────────────────┘
```

Then:

```text
skeleton
   ↓
actual content
```

The structure remains spatially similar, reducing layout jumping.

Suspense doesn't require skeletons, but the boundary model makes these kinds of loading experiences natural.

---

# 33. The distinction from our previous `useEffect` lesson

We've now seen two different approaches.

### Explicit async state

```jsx
const [loading, setLoading] = useState(false);

useEffect(() => {
  ...
}, []);
```

You control:

```text
loading
error
data
```

### Suspense-oriented resource

```jsx
const value = use(resource);
```

React controls the suspension:

```text
not ready
 ↓
Suspense fallback

ready
 ↓
render

failure
 ↓
Error Boundary
```

Neither is universally "the correct way."

They solve different architectural problems.

In many production applications you'll encounter **both**.

---

# 34. TanStack Query + explicit state + Suspense

It's entirely possible to have all three concepts in one codebase:

```text
legacy/simple fetch
   ↓
explicit loading/error state

server-state query
   ↓
TanStack Query

Suspense-enabled resource
   ↓
Suspense
```

The important thing is not to pick a religion.

Understand what abstraction owns what problem.

---

# 35. The four-state picture

A useful model now is:

```text
                  RESOURCE
                     │
         ┌───────────┼───────────┐
         ↓           ↓           ↓
      pending       ready       error
         │           │           │
         ↓           ↓           ↓
      Suspense      render    Error Boundary
      fallback
```

Then interaction/transitions add another dimension:

```text
already visible
      ↓
new render suspends
      ↓
transition decides whether
to preserve currently visible UI
```

This is where React's rendering model becomes much more sophisticated than:

```text
state changed
→ rerender
```

---

# 36. One complete example

Suppose we have:

```jsx
import {
  lazy,
  Suspense,
} from "react";

const Analytics = lazy(
  () => import("./Analytics")
);

function Dashboard() {
  return (
    <>
      <Header />

      <Suspense
        fallback={<AnalyticsSkeleton />}
      >
        <Analytics />
      </Suspense>

      <ActivityFeed />
    </>
  );
}
```

Now imagine Analytics code hasn't loaded.

The flow:

```text
Dashboard renders
       ↓
Analytics attempted
       ↓
lazy import pending
       ↓
Analytics suspends
       ↓
nearest Suspense boundary
       ↓
AnalyticsSkeleton
       ↓
module resolves
       ↓
React retries Analytics
       ↓
Analytics renders
```

If the import fails:

```text
lazy import rejects
       ↓
Error Boundary above
       ↓
Error UI
```

So:

```text
loading
→ Suspense

failed
→ Error Boundary

ready
→ Analytics
```

That's a very clean architecture. Current React's `lazy` and `Suspense` APIs are designed to work together this way. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

---

# 37. The deeper React concept underneath Suspense

This lesson ties back to our render-model lesson.

We previously learned:

```text
trigger
 ↓
render
 ↓
commit
```

Now we can expand it:

```text
trigger
 ↓
render
 ↓
component needs something not ready
 ↓
suspend
 ↓
nearest Suspense fallback rendered
 ↓
resource becomes ready
 ↓
React retries rendering that region
 ↓
successful render
 ↓
commit
```

So suspension is fundamentally part of React's **rendering coordination system**.

That's why it can't be reduced to:

```text
setLoading(true)
```

---

# 38. The mental model I want you to keep

When you see:

```jsx
<Suspense fallback={<Loading />}>
  <Something />
</Suspense>
```

don't think:

> "This automatically shows Loading while `Something` makes a network request."

Instead think:

> **"If something inside this region suspends during rendering, React can temporarily show this fallback until that region becomes renderable again."**

Then ask:

```text
What is causing the suspension?
```

Possibilities include:

```text
lazy-loaded code
use(resource/Promise)
Suspense-enabled framework/data mechanism
other React-supported resources
```

Current React's Suspense reference lists these supported sources of suspension and explicitly distinguishes them from arbitrary Effect-based fetching. ([React](https://react.dev/reference/react/Suspense?utm_source=chatgpt.com "<Suspense> – React"))

---

# 39. And now the whole family makes sense

We've now covered:

```text
Error Boundary
        ↓
"something failed"

Suspense
        ↓
"something isn't ready"

memo
        ↓
"maybe this work doesn't need to rerun"

useMemo
        ↓
"reuse this calculated value"

useCallback
        ↓
"reuse this function identity"

startTransition
        ↓
"this update isn't urgent"
```

These aren't random React APIs.

They're mechanisms for telling React more about **what kind of work is happening** and **how the UI should behave around it**.

That's the bigger picture.

---

# 40. Final takeaway

The most important distinction from this lesson is:

```text
Loading state:
"I, the component, know that I'm loading."

Suspense:
"I cannot currently finish rendering;
the surrounding tree decides what to show."
```

And:

```text
Error Boundary:
"I failed while rendering."

Suspense:
"I'm not ready yet."
```

Finally:

```text
lazy
 ↓
code isn't loaded yet

use(resource)
 ↓
resource isn't ready yet

Suspense
 ↓
what should the user see while waiting?

Error Boundary
 ↓
what should the user see if it fails?
```

Current React 19.x continues to build more rendering, streaming, and transition behavior around these primitives; React 19.3, released in September 2026, also includes additional Suspense/View Transition integration. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

The next topic naturally follows from this one: **`startTransition` and `useDeferredValue`**—how React lets urgent interactions stay responsive while expensive or suspending UI catches up in the background.