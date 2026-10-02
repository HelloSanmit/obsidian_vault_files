
# Lesson 39 — Code Splitting, `lazy`, Suspense, and Route-Level Loading

We just learned how React applications can be split between server and client.

Now there's another important performance question:

> **Even among the code that must run in the browser, do we really need to download all of it immediately?**

Usually, no.

Imagine an application with:

```text
Home
Dashboard
Reports
Admin
Rich Text Editor
Charts
Settings
Help
```

A user visiting `/home` probably doesn't need the code for:

```text
Admin
Reports
Rich Text Editor
```

yet.

So instead of:

```text
Browser
 ↓
download 5 MB JavaScript
 ↓
then use 100 KB of it
```

we can split the application:

```text
Initial bundle
 ↓
Home code

Later:
Reports chunk
Admin chunk
Editor chunk
```

This is **code splitting**.

---

# 1. First understand what a JavaScript bundle is

Suppose your application contains:

```text
App.jsx
Dashboard.jsx
Reports.jsx
Admin.jsx
Chart.jsx
Editor.jsx
```

Your build tool analyzes the import graph:

```text
App
 ├── Dashboard
 │    └── Chart
 ├── Reports
 └── Admin
      └── Editor
```

It can package these modules into browser files.

Without splitting, conceptually:

```text
app.js
├── App
├── Dashboard
├── Chart
├── Reports
├── Admin
└── Editor
```

With splitting:

```text
main.js
├── App
└── routing

dashboard.chunk.js
├── Dashboard
└── Chart

reports.chunk.js
└── Reports

admin.chunk.js
├── Admin
└── Editor
```

The browser doesn't need to download every chunk on startup.

---

# 2. Static import vs dynamic import

This is the JavaScript foundation behind code splitting.

Normal import:

```js
import Reports from "./Reports";
```

This tells the bundler:

```text
Reports is a dependency of this module.
```

A dynamic import:

```js
import("./Reports");
```

returns a Promise.

Conceptually:

```text
import("./Reports")
        ↓
      Promise
        ↓
module arrives later
```

That is the primitive that enables lazy loading.

---

# 3. React's `lazy`

React provides:

```js
lazy
```

Example:

```jsx
import { lazy } from "react";

const Reports =
  lazy(() => import("./Reports"));
```

Now `Reports` doesn't need to be downloaded immediately.

React waits until the component needs to be rendered.

The official React behavior is:

```text
lazy(...)
   ↓
component type
   ↓
first render
   ↓
load module
   ↓
suspend while loading
```

A Suspense boundary can then provide the fallback UI. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

---

# 4. `lazy` doesn't mean "render later"

This distinction matters.

Suppose:

```jsx
const Reports =
  lazy(() => import("./Reports"));
```

and:

```jsx
<Reports />
```

When React reaches:

```jsx
<Reports />
```

it tries to render it.

But the component's code isn't available yet.

So React effectively reaches:

```text
Reports
 ↓
"code not ready"
 ↓
SUSPEND
```

Then the surrounding Suspense boundary takes over.

---

# 5. `lazy` + `Suspense`

The standard pattern is:

```jsx
import { lazy, Suspense } from "react";

const Reports =
  lazy(() => import("./Reports"));

function App() {
  return (
    <Suspense fallback={<p>Loading reports...</p>}>
      <Reports />
    </Suspense>
  );
}
```

Execution:

```text
App renders
   ↓
Reports requested
   ↓
JS chunk isn't loaded
   ↓
Reports suspends
   ↓
Suspense fallback
   ↓
"Loading reports..."
```

Then:

```text
chunk downloads
   ↓
module resolves
   ↓
Reports can render
   ↓
Suspense fallback disappears
```

---

# 6. Suspense is therefore doing two jobs we've now seen

Earlier:

```text
Suspense
→ data/content isn't ready
```

Now:

```text
Suspense
→ component code isn't ready
```

The common idea is:

> Something required to render this part of the tree isn't ready yet.

That something might be:

```text
code
data
browser-only capability
```

depending on the mechanism creating the suspension.

---

# 7. `lazy` itself doesn't display anything

This:

```js
const Reports =
  lazy(() => import("./Reports"));
```

doesn't specify:

```text
Loading...
Spinner...
Skeleton...
```

The fallback belongs to:

```jsx
<Suspense fallback={...}>
```

That's because React separates:

```text
"What should happen when this subtree isn't ready?"
```

from:

```text
"What resource is currently missing?"
```

This separation is one of Suspense's strengths.

---

# 8. A common mistake: putting `lazy()` inside the component

Don't do this:

```jsx
function App() {
  const Reports =
    lazy(() => import("./Reports"));

  return <Reports />;
}
```

Why?

Every render creates a new lazy component type.

That interferes with React's component identity expectations and can cause state to reset. React's `lazy` guidance expects the lazy declaration to be outside the component. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

Do this:

```jsx
const Reports =
  lazy(() => import("./Reports"));

function App() {
  return <Reports />;
}
```

The component type is stable.

---

# 9. Let's trace the browser

Suppose:

```jsx
const Admin =
  lazy(() => import("./Admin"));
```

and:

```jsx
<Suspense fallback="Loading admin...">
  <Admin />
</Suspense>
```

Initial page:

```text
main.js
```

loads.

Browser renders:

```text
App
 ↓
Admin
 ↓
chunk missing
```

React suspends.

Browser shows:

```text
Loading admin...
```

Then browser requests:

```text
/assets/Admin-X7K2.js
```

Chunk downloads.

Then:

```text
Admin module
 ↓
resolve lazy component
 ↓
render Admin
 ↓
commit
```

The fallback disappears.

That's code splitting in action.

---

# 10. Lazy loading is only useful when the split is meaningful

Suppose your component is:

```text
TinyButton.jsx = 700 bytes
```

Splitting it into a separate chunk can be counterproductive.

You introduce:

```text
extra request
chunk metadata
loading coordination
cache overhead
```

for almost no benefit.

But:

```text
RichEditor
Charting library
PDF viewer
Admin dashboard
Map
Large visualization
```

can be excellent candidates.

So code splitting isn't:

> "Split every component."

It's:

> "Split expensive or infrequently needed code at meaningful boundaries."

---

# 11. Route-level code splitting

This becomes much more powerful when combined with routing.

Suppose:

```text
/home
/dashboard
/reports
/admin
```

The user enters:

```text
/home
```

Why should the browser download:

```text
admin code
reports code
```

immediately?

It often shouldn't.

So routing provides a natural split boundary:

```text
/home
   ↓
home chunk

/reports
   ↓
reports chunk

/admin
   ↓
admin chunk
```

Current React Router Framework Mode performs automatic code splitting for route modules, so routes become natural browser bundle boundaries. ([React Router](https://reactrouter.com/explanation/code-splitting?utm_source=chatgpt.com "Automatic Code Splitting | React Router"))

---

# 12. Old-school route lazy loading

With a Data Router, you can explicitly lazy-load route implementations.

For example:

```jsx
const router =
  createBrowserRouter([
    {
      path: "/",
      Component: Home
    },

    {
      path: "/reports",
      lazy: async () =>
        import("./routes/reports")
    }
  ]);
```

The precise route-lazy API supports different forms depending on your React Router mode/version, but the architectural idea is:

```text
route definition
   ↓
route implementation loaded later
```

React Router's `route.lazy` API can lazy-load route properties such as `loader`, `action`, and `Component`. ([React Router](https://reactrouter.com/changelog?utm_source=chatgpt.com "CHANGELOG.md | React Router"))

---

# 13. Why route splitting is better than random component splitting

Imagine:

```text
Reports
 ├── Header
 ├── Chart
 ├── Filters
 ├── Table
 └── Export
```

You could lazy-load each component independently:

```text
Chart chunk
Filters chunk
Table chunk
Export chunk
```

That might create unnecessary loading complexity.

A route-level boundary:

```text
reports.chunk.js
```

may be a better unit:

```text
user navigates to Reports
      ↓
download report-related code
      ↓
render route
```

The route itself represents a meaningful product-level boundary.

---

# 14. React Router's automatic code splitting

Current React Router Framework Mode can inspect the route graph and split application code automatically.

Conceptually:

```text
routes
├── home.tsx
├── about.tsx
├── dashboard.tsx
└── admin.tsx
```

becomes roughly:

```text
initial
├── app shell
└── routes needed initially

later
├── dashboard chunk
├── about chunk
└── admin chunk
```

So you don't necessarily have to manually write:

```js
lazy(() => import(...))
```

for every route in a Framework Mode application.

React Router explicitly describes route modules as entry points for automatic code splitting. ([React Router](https://reactrouter.com/explanation/code-splitting?utm_source=chatgpt.com "Automatic Code Splitting | React Router"))

---

# 15. Route discovery is a separate concept

Here's a subtle React Router concept.

There are actually two kinds of information:

```text
route metadata
+
route implementation
```

React Router can progressively discover routes instead of putting every route's complete manifest information into the initial payload.

Current Framework Mode uses lazy route discovery by default, where additional route metadata can be fetched as the user navigates. It also eagerly discovers links on the current page to reduce navigation waterfalls. ([React Router](https://reactrouter.com/explanation/lazy-route-discovery?utm_source=chatgpt.com "Lazy Route Discovery | React Router"))

Conceptually:

```text
initial page
   ↓
know enough about current routes
   ↓
<Link to="/admin">
   ↓
discover admin route metadata
   ↓
user clicks
   ↓
load actual admin code
```

This is a subtle but important optimization.

---

# 16. Why eager discovery exists

Without preparation, navigation might become:

```text
click Admin
   ↓
discover route
   ↓
learn chunk URL
   ↓
download chunk
   ↓
render
```

That's a waterfall.

Instead, React Router can discover routes referenced by links ahead of time.

So:

```text
page loaded
   ↓
<Link to="/admin">
   ↓
route discovered in background
```

Then:

```text
user clicks
   ↓
actual route code already easier to locate/load
```

This is one example of performance engineering that isn't visible in your component code. ([React Router](https://reactrouter.com/explanation/lazy-route-discovery?utm_source=chatgpt.com "Lazy Route Discovery | React Router"))

---

# 17. Code splitting and Suspense fit together

Let's make the hierarchy explicit:

```text
Route
  │
  ▼
lazy component
  │
  ▼
chunk request
  │
  ▼
Suspense
  │
  ▼
fallback
  │
  ▼
chunk ready
  │
  ▼
component render
```

So:

```text
code splitting
→ changes when code is downloaded

lazy
→ tells React how to represent delayed component code

Suspense
→ defines what UI appears while it's unavailable
```

Those are separate responsibilities.

---

# 18. Loading code isn't the same as loading data

Suppose:

```text
Reports component code
```

takes 300 ms.

And:

```text
Reports data
```

takes 800 ms.

You may have:

```text
code
   ↓
300 ms

data
   ↓
800 ms
```

The user could theoretically need to wait for both.

That's why modern React's Suspense model is useful.

You can reason about:

```text
code not ready
data not ready
```

as separate asynchronous concerns.

---

# 19. Nested Suspense boundaries

Suppose:

```jsx
<Suspense fallback={<PageSkeleton />}>
  <Reports />

  <Suspense fallback={<ChartSkeleton />}>
    <Chart />
  </Suspense>
</Suspense>
```

Potentially:

```text
Page
 ├── Reports
 └── Chart
```

If Reports code/data becomes available but Chart is still pending:

```text
Reports content
Chart skeleton
```

instead of replacing the entire page with:

```text
Page skeleton
```

This is why Suspense boundaries are fundamentally **UX boundaries**, not just technical loading wrappers.

---

# 20. Poor Suspense boundaries produce poor UX

Consider:

```jsx
<Suspense fallback={<FullScreenSpinner />}>
  <EntireApplication />
</Suspense>
```

Now any suspension anywhere could potentially replace the whole application with:

```text
Loading...
```

That may be technically correct but unpleasant.

Better:

```text
Application shell
 ├── navigation
 ├── page content
 │     ├── main content
 │     └── slow widget
 └── footer
```

with a local boundary around the slow region.

This gives:

```text
shell remains visible
slow region shows fallback
```

---

# 21. `lazy` preserves component-level state rules

Suppose:

```jsx
const Settings =
  lazy(() => import("./Settings"));
```

and Settings contains:

```jsx
const [tab, setTab] =
  useState("profile");
```

Once the lazy component is loaded, it behaves like an ordinary component.

Its state is governed by the same identity rules we learned:

```text
same component type
+
same key
+
same position
→ state preserved
```

Code splitting doesn't create a second state model.

---

# 22. Dynamic import and named exports

`lazy` expects the dynamically imported module to resolve to a default export.

So:

```js
export default function Reports() {
  ...
}
```

works directly:

```js
const Reports =
  lazy(() => import("./Reports"));
```

But if your module only has:

```js
export function Reports() {
  ...
}
```

you can adapt it:

```js
const Reports = lazy(() =>
  import("./Reports").then(module => ({
    default: module.Reports
  }))
);
```

The key is that React `lazy` wants:

```text
Promise
 ↓
{ default: Component }
```

React's current `lazy` contract explicitly expects the resolved module's `default` export to be a valid component. ([React](https://react.dev/reference/react/lazy?utm_source=chatgpt.com "lazy – React"))

---

# 23. A subtle cache behavior of `lazy`

React caches the Promise returned by the `load` function and the resolved component module.

So:

```js
const Reports =
  lazy(() => import("./Reports"));
```

doesn't mean:

```text
every render
 ↓
download Reports
```

Instead, conceptually:

```text
first use
 ↓
start import
 ↓
cache Promise/result
```

Later renders reuse the loaded component.

This is another reason you define the `lazy` component outside the render function.

---

# 24. Error handling for lazy components

What happens if:

```text
Reports chunk
```

fails to load?

Maybe:

```text
network failure
CDN issue
404
deployment mismatch
```

Then Suspense isn't the right abstraction for the final error.

Remember the distinction:

```text
Suspense
→ "not ready"

Error Boundary
→ "failed"
```

So a robust architecture can be:

```jsx
<ErrorBoundary>
  <Suspense fallback={<ReportsSkeleton />}>
    <Reports />
  </Suspense>
</ErrorBoundary>
```

Conceptually:

```text
chunk pending
→ Suspense

chunk failed
→ Error Boundary
```

Exactly the same distinction we learned in Lesson 27.

---

# 25. This is why `Suspense + ErrorBoundary` is such a useful pair

Think:

```text
                 resource
                    │
             ┌──────┴──────┐
             │             │
          pending         failed
             │             │
             ▼             ▼
          Suspense     Error Boundary
             │             │
          fallback        error UI
```

Then:

```text
ready
 ↓
normal UI
```

This is a general asynchronous rendering pattern, not just a lazy-loading trick.

---

# 26. Preloading code

There is another performance idea:

> What if we know we're probably going to need a component soon?

Suppose:

```text
user hovers "Reports"
```

You may want to start downloading Reports code before the click.

React and modern bundlers provide mechanisms around preloading resources; React also exposes resource APIs such as `preload`, `preinit`, and module-oriented variants. The exact API is environment/build-tool dependent, so think of this as **starting required resource work earlier**, not as something every application must manually implement.

The strategic idea is:

```text
current state
   ↓
predict likely next state
   ↓
start loading early
   ↓
user action
   ↓
code arrives faster
```

This is called **preloading/prefetching**, depending on what exactly you're doing.

---

# 27. Prefetching is different from lazy loading

Lazy loading:

```text
don't load until needed
```

Prefetching:

```text
load before needed because we expect it soon
```

They can work together.

Imagine:

```text
Dashboard
```

contains:

```text
<Link to="/reports">
```

You can have:

```text
initial:
Dashboard code

likely next:
Reports chunk prefetch
```

Then when the user clicks:

```text
Reports already downloading or cached
```

This can reduce perceived navigation latency.

But over-prefetching wastes bandwidth.

---

# 28. The performance tradeoff

Every optimization has a cost.

Prefetching:

```text
pro:
faster future navigation

con:
uses bandwidth now
```

Code splitting:

```text
pro:
smaller initial JavaScript

con:
additional chunks later
```

Aggressive splitting:

```text
pro:
tiny initial bundle

con:
too many requests/chunks
complex loading
```

Large bundle:

```text
pro:
fewer requests
simple startup dependency graph

con:
larger initial download/parse/execute work
```

So performance isn't:

```text
smaller bundle = automatically faster
```

It's a resource scheduling problem.

---

# 29. JavaScript performance has multiple stages

Suppose you download:

```text
2 MB JavaScript
```

The cost isn't just:

```text
download 2 MB
```

The browser also has work around:

```text
network
 ↓
decompression
 ↓
parse
 ↓
compile
 ↓
execute
 ↓
React startup
```

So reducing unnecessary JavaScript can improve more than network transfer size.

This is one reason keeping server-only modules out of the client graph matters.

---

# 30. Now connect this to Server Components

We learned:

```text
Server Components
→ server-side code doesn't need to become client component code
```

and now:

```text
Code splitting
→ client code can be divided into chunks
```

Combine them:

```text
Server
├── Product processing
├── Markdown parser
├── Database code
└── Server Components

Client
├── AddToCart chunk
├── ReviewsForm chunk
└── Chart chunk
```

That's powerful because you're optimizing at two different levels:

```text
server/client boundary
        +
client code splitting
```

---

# 31. A realistic application graph

Imagine:

```text
App
│
├── Header
│
├── Home
│
├── Dashboard
│   ├── Chart
│   └── Table
│
├── Admin
│   ├── UserManager
│   └── RichEditor
│
└── Settings
```

A sensible strategy could be:

```text
Initial
├── App
├── Header
└── Home

Dashboard chunk
├── Dashboard
├── Chart
└── Table

Admin chunk
├── Admin
├── UserManager
└── RichEditor

Settings chunk
└── Settings
```

Now the user doesn't pay for Admin functionality just to visit Home.

---

# 32. Route splitting and nested routes

Suppose:

```text
/dashboard
   ├── overview
   ├── analytics
   └── settings
```

You don't necessarily need:

```text
one gigantic dashboard chunk
```

You can potentially split:

```text
dashboard-shell
overview
analytics
settings
```

Then:

```text
/dashboard/overview
```

loads:

```text
shell + overview
```

while:

```text
/dashboard/analytics
```

loads:

```text
shell + analytics
```

This matches the user's actual navigation path.

Current React Router Framework Mode uses route modules as code-splitting boundaries and can split route modules further so client-side route exports and component code don't all have to arrive together. ([React Router](https://reactrouter.com/explanation/code-splitting?utm_source=chatgpt.com "Automatic Code Splitting | React Router"))

---

# 33. A subtle React Router optimization

Consider:

```text
route component
+
clientLoader
```

If both live in the same large module, naive splitting could require:

```text
download entire route module
 ↓
execute module
 ↓
run client loader
 ↓
render component
```

React Router can split route modules so some route exports can be loaded independently from the component code.

Conceptually:

```text
clientLoader chunk
      ↓
load/run

component chunk
      ↓
load
```

This can reduce certain waterfalls. Current Framework Mode performs this route-module splitting behavior by default. ([React Router](https://reactrouter.com/explanation/code-splitting?utm_source=chatgpt.com "Automatic Code Splitting | React Router"))

That's an advanced optimization, but the principle is simple:

> Don't make code dependencies larger than the behavior that actually needs them.

---

# 34. `React.lazy` vs route lazy loading

These are related but not identical.

### `React.lazy`

Component-level:

```js
const Chart =
  lazy(() => import("./Chart"));
```

Meaning:

```text
delay loading this component's code
```

### React Router route lazy

Route-level:

```text
delay loading route implementation
```

Meaning:

```text
route definition is known
implementation arrives when needed
```

So:

```text
React.lazy
→ component boundary

route.lazy / automatic route splitting
→ route boundary
```

---

# 35. `<Routes>` vs Data/Framework routes

Recall from earlier:

```jsx
<Routes>
  <Route ... />
</Routes>
```

The current React Router API notes that these declarative `<Routes>/<Route>` trees do **not** participate in Data Router route-module features such as data loading, actions, or route code splitting. ([React Router](https://reactrouter.com/api/components/Routes?utm_source=chatgpt.com "Routes | React Router"))

This is worth remembering because we've just learned all these Data Router capabilities.

So:

```text
plain <Routes>
→ basic rendering-based routing

Data/Framework route modules
→ data + actions + revalidation + code splitting
```

That's another architecture distinction.

---

# 36. Don't confuse loading UI with application loading UI

Suppose:

```jsx
<Suspense fallback={<Spinner />}>
  <Chart />
</Suspense>
```

The fallback represents:

```text
Chart isn't ready.
```

It doesn't necessarily mean:

```text
Application isn't ready.
```

This seems obvious, but poor Suspense architecture often violates it.

You want loading states that match the scope of the pending work.

Examples:

```text
page-level navigation
→ page transition indicator

chart code loading
→ chart skeleton

table data loading
→ table skeleton

button mutation
→ button pending state
```

The pending UI should correspond to the actual asynchronous boundary.

---

# 37. Code splitting and user perception

Consider two architectures.

### Architecture A

```text
click Reports
 ↓
blank screen
 ↓
2 seconds
 ↓
Reports
```

### Architecture B

```text
click Reports
 ↓
existing shell remains
 ↓
Reports skeleton
 ↓
Reports
```

Both might have identical network times.

But B usually feels better because the interface preserves context.

This is why:

```text
Suspense boundary placement
```

is a UX decision, not just a technical implementation detail.

---

# 38. Code splitting and `startTransition`

You learned earlier:

```js
startTransition(...)
```

for non-urgent UI updates.

Now imagine navigation triggers a lazy chunk.

If the UI change is handled as a transition, React can avoid unnecessarily replacing already-visible content with fallback during some transitions, depending on the rendering architecture.

The deeper connection is:

```text
navigation/render work
+
code availability
+
Suspense
+
transition priority
```

can all interact.

You don't need to memorize every fallback rule.

The core idea is:

> React knows that a UI transition can temporarily be waiting for code, and transitions can help preserve the currently visible interface while the next state becomes ready.

---

# 39. Don't use `lazy` to hide poor application architecture

Suppose your entire app is:

```text
App.jsx
  ↓
50,000 lines
```

and you split it into:

```text
lazy chunks
```

You may reduce initial JavaScript, but the underlying architecture remains hard to reason about.

Code splitting is a performance technique.

It doesn't replace:

```text
good component boundaries
good module boundaries
good route boundaries
server/client separation
```

In fact, good architecture makes code splitting easier.

---

# 40. How to decide a split boundary

Ask:

### Is this functionality rarely needed?

```text
Admin
PDF viewer
advanced editor
```

Good candidate.

### Is it large?

```text
chart library
map library
syntax highlighter
```

Good candidate.

### Is it route-specific?

```text
/reporting
/settings
/admin
```

Excellent candidate.

### Is it needed above-the-fold on almost every visit?

Probably don't lazily load it.

The point is not to minimize every chunk.

The point is to minimize **unnecessary initial work**.

---

# 41. A complete lazy component example

```jsx
import {
  lazy,
  Suspense
} from "react";

import LoadingPanel from "./LoadingPanel";

const MarkdownEditor = lazy(
  () => import("./MarkdownEditor")
);

function CreatePost() {
  return (
    <main>
      <h1>Create Post</h1>

      <Suspense
        fallback={
          <LoadingPanel />
        }
      >
        <MarkdownEditor />
      </Suspense>
    </main>
  );
}
```

Flow:

```text
CreatePost loads
   ↓
MarkdownEditor not needed yet
   ↓
chunk deferred
```

When React actually renders:

```text
MarkdownEditor
   ↓
dynamic import
   ↓
Suspense
   ↓
LoadingPanel
   ↓
chunk arrives
   ↓
MarkdownEditor renders
```

---

# 42. What if the user never opens the editor?

Then:

```text
MarkdownEditor chunk
```

may never be requested.

That's the entire point.

You aren't asking users to download functionality they never use.

---

# 43. One advanced example: modal

Imagine:

```jsx
function Settings() {
  const [showEditor, setShowEditor] =
    useState(false);

  return (
    <>
      <button
        onClick={() =>
          setShowEditor(true)
        }
      >
        Edit
      </button>

      {showEditor && (
        <Suspense fallback="Loading editor...">
          <RichEditor />
        </Suspense>
      )}
    </>
  );
}
```

And:

```js
const RichEditor = lazy(
  () => import("./RichEditor")
);
```

Now:

```text
user visits Settings
   ↓
editor code NOT downloaded

user clicks Edit
   ↓
editor chunk requested
   ↓
fallback
   ↓
editor appears
```

That's an excellent use of component-level lazy loading.

---

# 44. Route-level example

Imagine:

```text
/dashboard
/admin
```

The Admin route includes:

```text
large table
permissions editor
audit viewer
rich editor
```

You definitely don't want all of that on the initial `/dashboard` page.

A route split lets you have:

```text
Dashboard bundle
```

and:

```text
Admin bundle
```

The user pays for Admin only when entering Admin.

React Router Framework Mode's route-module code splitting is designed to make this kind of route-level separation automatic. ([React Router](https://reactrouter.com/explanation/code-splitting?utm_source=chatgpt.com "Automatic Code Splitting | React Router"))

---

# 45. One last distinction: bundle splitting vs rendering splitting

This is subtle.

### Code splitting

```text
When is JavaScript downloaded?
```

### Suspense

```text
What UI appears while required rendering work isn't ready?
```

### Server Components

```text
Where does this component's code execute?
```

### SSR

```text
How is initial HTML produced?
```

These are four different questions.

They can combine:

```text
Server Component
    ↓
server execution

Client Component
    ↓
lazy-loaded chunk

Suspense
    ↓
loading UI

SSR
    ↓
initial HTML
```

---

# 46. The complete modern performance picture

You now have:

```text
                    APPLICATION
                         │
              ┌──────────┴───────────┐
              │                      │
           SERVER                  CLIENT
              │                      │
      Server Components       Client Components
              │                      │
      server-only code          initial bundle
              │                      │
              │                code splitting
              │                      │
              │                ┌─────┴─────┐
              │                │           │
              │             lazy chunk  route chunk
              │                │           │
              └───────┬────────┴───────────┘
                      │
                 Suspense
                      │
                 ready/fallback
                      │
                    UI
```

And React Router can add:

```text
URL
 ↓
route discovery
 ↓
route chunk
 ↓
route data
 ↓
render
```

with automatic route code splitting in Framework Mode. ([React Router](https://reactrouter.com/explanation/code-splitting?utm_source=chatgpt.com "Automatic Code Splitting | React Router"))

---

# 47. The performance questions to ask now

When looking at a slow React application, don't immediately say:

```text
"React is rerendering too much."
```

Instead ask:

```text
1. How much JavaScript is downloaded initially?

2. How much of that JavaScript is actually needed?

3. Are large routes split?

4. Are expensive libraries loaded only when needed?

5. Are we creating waterfalls?

6. Are Suspense boundaries placed around the correct units of work?

7. Are we prefetching intelligently?

8. Is server-only logic accidentally shipped to the browser?
```

These questions often matter more than sprinkling `useMemo` everywhere.

---

# 48. Your mental map after this lesson

```text
Dynamic import
     ↓
bundler can create separate chunk
     ↓
lazy()
     ↓
React component may suspend
     ↓
Suspense fallback
     ↓
chunk loads
     ↓
component renders
```

And with routing:

```text
URL
 ↓
route
 ↓
route chunk
 ↓
route data
 ↓
component
```

And with the server:

```text
Server Component
     ↓
stays server-side

Client Component
     ↓
client bundle
     ↓
possibly lazy/code-split
```

---

# 49. What to permanently remember

```text
dynamic import()
→ JavaScript-level delayed module loading

lazy()
→ turn a delayed module into a React component

Suspense
→ define UI while required work isn't ready

Error Boundary
→ define UI when that work fails

code splitting
→ divide browser JavaScript into independently loaded chunks

route splitting
→ make navigation boundaries become code boundaries

prefetching
→ start loading likely-needed code before it's actually needed

Server Components
→ avoid shipping server-only component logic to the browser
```

The most important conceptual sentence is:

> **Code splitting controls when code arrives; Suspense controls what the user sees while required work isn't ready.**

And the next topic that follows naturally is **React's performance model at production scale: profiling, browser rendering cost, expensive renders vs unnecessary renders, memoization strategy, scheduling, bundle analysis, and how to systematically find the real bottleneck instead of guessing.**