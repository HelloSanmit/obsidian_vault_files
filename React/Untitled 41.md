
# Lesson 38 — React Router Data APIs: Loaders, Actions, Revalidation, and Fetchers

You've already learned React Router's basic model:

```text
URL
 ↓
route matching
 ↓
component
```

Now we're going to upgrade that mental model.

With **Data Routers**, a route isn't just:

```text
"path → component"
```

It can become:

```text
"path
 + loader
 + action
 + pending state
 + revalidation
 + error handling
 + fetchers"
```

React Router's Data/Framework modes are built around this model. A `loader` provides route data before rendering, an `action` handles mutations, and after successful mutations React Router automatically revalidates relevant loader data so the UI can converge back to server truth. ([React Router](https://reactrouter.com/start/data/data-loading?utm_source=chatgpt.com "Data Loading | React Router"))

This is important because it gives us another way to solve problems you previously solved with:

```text
useEffect
fetch
useState
useNavigate
TanStack Query
manual invalidation
```

The goal is not to declare one approach universally better.

The goal is to understand **when each data model fits**.

---

# 1. The old Router mental model

Earlier, you learned:

```jsx
<Routes>
  <Route
    path="/products/:id"
    element={<Product />}
  />
</Routes>
```

The router answers:

```text
URL:
 /products/42

matches:
 Product
```

Then your component might do:

```jsx
function Product() {
  const { id } = useParams();

  const [product, setProduct] =
    useState(null);

  useEffect(() => {
    fetch(`/api/products/${id}`)
      .then(r => r.json())
      .then(setProduct);
  }, [id]);

  ...
}
```

Notice what's happening.

The router knows:

```text
"/products/:id"
```

but **the component owns the data-loading lifecycle**.

---

# 2. Data Router moves data ownership toward the route

Instead:

```js
{
  path: "/products/:id",
  loader: async ({ params }) => {
    return getProduct(params.id);
  },
  Component: Product
}
```

Now the route itself says:

```text
"This URL requires this data."
```

The component becomes:

```jsx
function Product() {
  const product = useLoaderData();

  return (
    <h1>{product.name}</h1>
  );
}
```

The conceptual difference is huge:

### Old

```text
Route
  ↓
Component
  ↓
effect
  ↓
request
```

### Data Router

```text
Route
  ↓
loader
  ↓
data
  ↓
Component
```

React Router calls route loaders before rendering the matching route component during navigation. ([React Router](https://reactrouter.com/start/data/data-loading?utm_source=chatgpt.com "Data Loading | React Router"))

---

# 3. What is a loader?

A loader is a function associated with a route that provides the data needed by that route.

For example:

```jsx
const router = createBrowserRouter([
  {
    path: "/products/:id",
    loader: async ({ params }) => {
      const product =
        await getProduct(params.id);

      return product;
    },
    Component: ProductPage
  }
]);
```

Then:

```jsx
function ProductPage() {
  const product = useLoaderData();

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </main>
  );
}
```

The data flow becomes:

```text
URL /products/42
       ↓
route matches
       ↓
loader({ params })
       ↓
getProduct("42")
       ↓
loader resolves
       ↓
ProductPage renders
       ↓
useLoaderData()
```

That's the central Data Router pattern.

---

# 4. `loader` knows about the URL

The loader receives information about the navigation/request.

For example:

```js
loader: async ({ params }) => {
  const product =
    await getProduct(params.id);

  return product;
}
```

With:

```text
/products/42
```

we get:

```js
params.id === "42"
```

So the route itself becomes the bridge between:

```text
URL
 ↓
data requirement
 ↓
component
```

This is one reason route-level data loading is so useful.

---

# 5. Query parameters are available too

Suppose:

```text
/products?category=books&page=2
```

The loader gets the request:

```js
loader: async ({ request }) => {
  const url = new URL(request.url);

  const category =
    url.searchParams.get("category");

  const page =
    url.searchParams.get("page");

  return getProducts({
    category,
    page
  });
}
```

So your URL becomes the input:

```text
URL state
   ↓
loader
   ↓
server/data source
   ↓
route data
```

This connects directly to Lesson 16.

You already learned:

```text
URL state is state
```

Data Routers let routes directly consume that URL state to load the corresponding data.

---

# 6. Why this is better than an effect for route data

Consider:

```jsx
function ProductPage() {
  const [product, setProduct] =
    useState(null);

  useEffect(() => {
    fetch(...)
  }, []);

  if (!product) {
    return <Spinner />;
  }

  return ...
}
```

The component has to represent two things:

```text
UI
+
data-loading machinery
```

With a loader:

```jsx
function ProductPage() {
  const product = useLoaderData();

  return ...
}
```

the component mostly represents:

```text
UI
```

and the route owns:

```text
data requirements
```

This separation often makes route-driven applications easier to reason about.

---

# 7. Route loading happens before the component is rendered

Conceptually:

```text
User clicks
/products/42
       ↓
Router matches route
       ↓
loader()
       ↓
data resolves
       ↓
Product component renders
```

Instead of:

```text
User clicks
       ↓
component renders
       ↓
loading UI
       ↓
useEffect
       ↓
request
       ↓
state update
       ↓
component renders again
```

This doesn't mean the page never needs a loading UI.

It means the loading lifecycle is now associated with the **navigation/data router** instead of being manually created inside the component.

---

# 8. Pending UI

When navigation is underway, React Router exposes navigation state through:

```js
useNavigation()
```

For example:

```jsx
function Navigation() {
  const navigation =
    useNavigation();

  const loading =
    navigation.state === "loading";

  return (
    <nav>
      {loading && <p>Loading...</p>}
    </nav>
  );
}
```

Conceptually:

```text
navigation.state

"idle"
   ↓
"loading"
   ↓
"idle"
```

This is very different from maintaining:

```js
const [loading, setLoading] = useState(false);
```

for every route transition.

The router already knows whether a navigation is pending.

---

# 9. Navigation state is not component state

This distinction matters.

Your component might have:

```js
const [menuOpen, setMenuOpen] =
  useState(false);
```

That's local UI state.

But:

```js
const navigation =
  useNavigation();
```

describes the router's state.

So:

```text
useState
→ component/application state

useNavigation
→ router state
```

That's another example of state ownership.

---

# 10. The route hierarchy becomes important

Suppose:

```text
/
└── dashboard
    ├── profile
    ├── settings
    └── reports
```

A parent route can have:

```js
loader: loadDashboard
```

and a child route can have:

```js
loader: loadReports
```

When `/dashboard/reports` is active, React Router has multiple matched routes.

So there can be multiple loaders:

```text
/dashboard
    ↓ loader

/dashboard/reports
    ↓ loader
```

React Router's Data Router executes matched loaders in parallel by default. ([React Router](https://reactrouter.com/how-to/data-strategy?utm_source=chatgpt.com "Data Strategy | React Router"))

That matters.

You don't automatically get:

```text
parent loader
    ↓ wait
child loader
    ↓ wait
render
```

Instead, conceptually:

```text
parent loader ───────┐
                     ├──→ route data
child loader ────────┘
```

unless your own data dependencies require sequencing.

---

# 11. Parent and child route data

Imagine:

```jsx
const router = createBrowserRouter([
  {
    path: "/dashboard",
    loader: loadDashboard,
    Component: Dashboard,
    children: [
      {
        path: "reports",
        loader: loadReports,
        Component: Reports
      }
    ]
  }
]);
```

The route tree:

```text
/dashboard
    │
    └── /reports
```

has data ownership:

```text
Dashboard
→ dashboard data

Reports
→ report data
```

This is much more intentional than having every nested component independently execute arbitrary effects.

---

# 12. Route-level data ownership

This gives us a useful design principle:

> Data required by a route should often be owned by that route's loader.

For example:

```text
/users/:userId
```

may require:

```text
user
permissions
activity summary
```

The route can express those requirements.

Then the UI consumes the resulting data.

This creates a strong connection:

```text
route
 ↓
data requirements
 ↓
UI
```

---

# 13. Now the interesting part: `action`

Loader:

```text
READ
```

Action:

```text
WRITE
```

Conceptually:

```text
loader
→ GET-like work

action
→ POST/PUT/PATCH/DELETE-like work
```

For example:

```js
{
  path: "/todos",
  loader: loadTodos,

  action: async ({ request }) => {
    const formData =
      await request.formData();

    await createTodo(
      formData.get("title")
    );

    return { ok: true };
  },

  Component: Todos
}
```

Route actions are the Data Router's mutation mechanism. React Router automatically revalidates loader data after an action completes successfully. ([React Router](https://reactrouter.com/start/data/actions?utm_source=chatgpt.com "Actions | React Router"))

---

# 14. Why automatic revalidation is such a big deal

Let's say:

```text
loader → fetch todos
```

returns:

```text
[A, B, C]
```

Then user submits:

```text
"D"
```

The action performs:

```text
INSERT D
```

Server now has:

```text
[A, B, C, D]
```

Without automatic revalidation, the browser might still display:

```text
[A, B, C]
```

You'd have to manually say:

```text
"Refetch todos!"
```

Data Router does this automatically after a successful action.

Conceptually:

```text
loader
   ↓
[A B C]

action
   ↓
create D
   ↓
success
   ↓
revalidate loader
   ↓
[A B C D]
```

This is one of the defining features of Data Routers. ([React Router](https://reactrouter.com/start/data/actions?utm_source=chatgpt.com "Actions | React Router"))

---

# 15. This solves the consistency problem we studied earlier

Recall our Lesson 21 discussion.

A mutation changes server state.

Then cached/loaded representations may become stale.

Previously we discussed:

```text
mutation
 ↓
invalidateQueries()
 ↓
refetch
```

React Router's model is:

```text
action
 ↓
revalidate loaders
 ↓
fresh route data
```

These are philosophically similar:

```text
mutation
 ↓
known stale read model
 ↓
refresh
```

The difference is where the read state lives and who owns the invalidation logic.

---

# 16. `<Form>` in React Router

React Router provides its own:

```jsx
<Form>
```

which can submit to route actions.

For example:

```jsx
import { Form } from "react-router";

function TodoForm() {
  return (
    <Form method="post">
      <input name="title" />
      <button type="submit">
        Add
      </button>
    </Form>
  );
}
```

You don't write:

```js
event.preventDefault();
fetch(...);
```

The router handles the submission lifecycle.

The action receives:

```js
request
```

and can read:

```js
const formData =
  await request.formData();
```

React Router currently supports declarative submission through `<Form>` and imperative submission through APIs such as `useSubmit`. ([React Router](https://reactrouter.com/start/data/actions?utm_source=chatgpt.com "Actions | React Router"))

---

# 17. Route action trace

Let's trace it.

Initial:

```text
loader
 ↓
[A, B]
```

User types:

```text
Learn React
```

and submits.

### Step 1

```jsx
<Form method="post">
```

submits.

### Step 2

Router finds the relevant route action:

```js
action({ request })
```

### Step 3

Action reads:

```js
const formData =
  await request.formData();

const title =
  formData.get("title");
```

### Step 4

Server/data source mutation:

```js
await db.todos.insert({
  title
});
```

### Step 5

Action completes.

### Step 6

React Router revalidates active loaders.

### Step 7

Loader returns:

```text
[A, B, Learn React]
```

### Step 8

Route component renders fresh data.

So:

```text
FORM
 ↓
ACTION
 ↓
MUTATION
 ↓
REVALIDATION
 ↓
LOADER
 ↓
FRESH UI
```

That's the Data Router mutation loop.

---

# 18. Why the action doesn't directly update the todo list

This is an intentional design choice.

You might think:

```js
action() {
  const todo = createTodo(...);

  return {
    todos: [...]
  };
}
```

But your route loader remains the source of truth.

That's useful because the server may do things you don't know about:

```text
generate ID
set timestamps
apply permissions
normalize text
trigger business logic
sort data
attach metadata
```

Rather than making the client recreate the server's truth:

```text
perform mutation
 ↓
reload authoritative route data
```

The action can simply say:

```text
"The mutation happened."
```

and the loader reconstructs current state.

---

# 19. Action data vs loader data

These are different.

### Loader data

```js
useLoaderData()
```

means:

```text
"What data does this route currently have?"
```

### Action data

```js
useActionData()
```

means:

```text
"What did the most recent action return?"
```

For example:

```js
return {
  ok: false,
  error: "Title required"
};
```

Then:

```jsx
const actionData =
  useActionData();

{actionData?.error && (
  <p>{actionData.error}</p>
)}
```

So:

```text
loader data
→ current resource state

action data
→ result of mutation attempt
```

---

# 20. A complete Todo route

```jsx
const router = createBrowserRouter([
  {
    path: "/todos",

    loader: async () => {
      return db.todos.getAll();
    },

    action: async ({ request }) => {
      const formData =
        await request.formData();

      const title =
        formData.get("title");

      if (
        typeof title !== "string" ||
        !title.trim()
      ) {
        return {
          ok: false,
          error: "Title required"
        };
      }

      await db.todos.create({
        title: title.trim()
      });

      return {
        ok: true
      };
    },

    Component: Todos
  }
]);
```

Component:

```jsx
function Todos() {
  const todos =
    useLoaderData();

  const actionData =
    useActionData();

  return (
    <>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            {todo.title}
          </li>
        ))}
      </ul>

      <Form method="post">
        <input name="title" />

        <button type="submit">
          Add
        </button>
      </Form>

      {actionData?.error && (
        <p>{actionData.error}</p>
      )}
    </>
  );
}
```

Notice what isn't here:

```text
useEffect
fetch
setTodos
setLoading
refetchTodos
```

The route machinery owns the data lifecycle.

---

# 21. But what if I don't want navigation?

This is where **fetchers** become extremely useful.

Suppose you're looking at:

```text
/todos
```

and each todo has a delete button.

Deleting a todo should not navigate you somewhere else.

You can use:

```js
useFetcher()
```

A fetcher can load data or submit mutations **without causing navigation**, and each fetcher tracks its own independent state. ([React Router](https://reactrouter.com/how-to/fetchers?utm_source=chatgpt.com "Using Fetchers | React Router"))

---

# 22. `useFetcher`

Example:

```jsx
function DeleteButton({ todoId }) {
  const fetcher = useFetcher();

  return (
    <fetcher.Form
      method="post"
      action={`/todos/${todoId}/delete`}
    >
      <button
        disabled={fetcher.state !== "idle"}
      >
        {fetcher.state === "submitting"
          ? "Deleting..."
          : "Delete"}
      </button>
    </fetcher.Form>
  );
}
```

Now:

```text
Delete
 ↓
fetcher
 ↓
route action
 ↓
mutation
 ↓
revalidation
```

but:

```text
URL stays /todos
```

That's the defining difference.

---

# 23. Navigation vs fetcher

This distinction is worth memorizing.

### Navigation

```text
Link
Form
useSubmit
```

can represent:

```text
"I am moving through the application."
```

### Fetcher

```text
useFetcher
```

means:

```text
"I want to interact with route data,
but I don't want to navigate."
```

For example:

```text
navigation:
click product
→ /products/42

fetcher:
toggle favorite
→ stay on current page
```

---

# 24. Why fetchers are powerful

Imagine a dashboard:

```text
Project A     [Archive]
Project B     [Archive]
Project C     [Archive]
Project D     [Archive]
```

If each button uses a fetcher:

```text
Fetcher A → independent state
Fetcher B → independent state
Fetcher C → independent state
Fetcher D → independent state
```

You can have:

```text
Project A → Archiving...
Project B → idle
Project C → Archiving...
Project D → idle
```

simultaneously.

React Router explicitly describes fetchers as useful for multiple concurrent data interactions without navigation. ([React Router](https://reactrouter.com/how-to/fetchers?utm_source=chatgpt.com "Using Fetchers | React Router"))

This connects beautifully to the concurrency concepts you've already learned.

---

# 25. Fetcher state

A fetcher has a state similar to:

```text
"idle"
"loading"
"submitting"
```

The exact state depends on whether it is loading data or submitting a mutation.

So:

```jsx
if (fetcher.state === "submitting") {
  // mutation in progress
}
```

You don't need:

```js
const [isDeleting, setIsDeleting] =
  useState(false);
```

for every button.

The fetcher already owns that interaction's pending state.

---

# 26. Multiple fetchers solve a coordination problem

Suppose we have:

```text
A delete button
B rename button
C toggle favorite button
```

Without fetchers, you might accidentally build:

```js
const [pendingAction, setPendingAction] =
  useState(null);
```

Then:

```text
which row is pending?
which request is pending?
which button should disable?
```

With fetchers:

```text
row A
 ↓
fetcher A

row B
 ↓
fetcher B

row C
 ↓
fetcher C
```

Each interaction gets its own state machine.

That's a much more scalable mental model.

---

# 27. Automatic revalidation after fetcher mutations

This is another powerful part.

Suppose:

```text
loader → todos = [A, B, C]
```

Fetcher submits:

```text
delete B
```

Action succeeds.

React Router revalidates the relevant loaders.

Then:

```text
loader → [A, C]
```

The UI updates.

So the flow is:

```text
fetcher.Form
   ↓
route action
   ↓
database mutation
   ↓
automatic revalidation
   ↓
loader
   ↓
fresh UI
```

No manual:

```js
setTodos(...)
```

is required for the canonical server result. ([React Router](https://reactrouter.com/start/data/actions?utm_source=chatgpt.com "Actions | React Router"))

---

# 28. This is almost a state machine

You can model a fetcher as:

```text
        idle
         │
         │ submit
         ▼
     submitting
         │
         ▼
      loading
         │
         ▼
        idle
```

And navigation similarly has states.

This is one reason React Router's Data APIs feel different from hand-written `useEffect` code.

The router has an explicit understanding of:

```text
navigation lifecycle
data loading lifecycle
mutation lifecycle
revalidation lifecycle
```

rather than leaving you to reconstruct them manually.

---

# 29. Revalidation: the heart of Data Router architecture

Let's zoom out.

Suppose we have:

```text
Route loader
   ↓
server data snapshot
```

Then:

```text
Action
   ↓
changes server
```

Now the snapshot might be stale.

React Router says:

```text
"After a successful mutation,
let's ask the loaders again."
```

Thus:

```text
READ
 ↓
MUTATE
 ↓
READ AGAIN
 ↓
UI
```

That's a very clean model.

---

# 30. But revalidating everything isn't always necessary

Suppose:

```text
/dashboard
  loader → expensive analytics query
```

and a form submits:

```text
change user's avatar
```

Re-running expensive analytics may be unnecessary.

React Router provides:

```js
shouldRevalidate
```

to let you customize whether a loader should run again. ([React Router](https://reactrouter.com/how-to/optimize-revalidation?utm_source=chatgpt.com "Revalidation Optimization | React Router"))

For example:

```js
shouldRevalidate({
  formMethod,
  formAction,
  defaultShouldRevalidate
}) {
  if (
    formAction?.endsWith("/avatar")
  ) {
    return false;
  }

  return defaultShouldRevalidate;
}
```

The exact conditions depend on your route structure.

The important principle:

```text
automatic revalidation
        ↓
safe default

custom revalidation
        ↓
optimization
```

React Router itself cautions that skipping revalidation can leave the UI stale, so this optimization should be targeted rather than blindly turning revalidation off. ([React Router](https://reactrouter.com/how-to/optimize-revalidation?utm_source=chatgpt.com "Revalidation Optimization | React Router"))

---

# 31. Why "always return false" is dangerous

Suppose:

```js
shouldRevalidate() {
  return false;
}
```

Now a mutation succeeds:

```text
database:
new value
```

but your loader doesn't run again.

UI:

```text
old value
```

You have created your own consistency bug.

That's why the default should generally be trusted until you have a concrete reason to optimize.

This is exactly the same lesson we learned with TanStack Query:

> Performance optimization that breaks data consistency is not a performance win.

---

# 32. `useRevalidator`

Sometimes data becomes stale for a reason that isn't a normal mutation.

For example:

```text
window regains focus
polling
external event
WebSocket message
browser storage change
```

React Router provides:

```js
useRevalidator()
```

which lets you explicitly request route-data revalidation. ([React Router](https://reactrouter.com/api/hooks/useRevalidator?utm_source=chatgpt.com "useRevalidator | React Router"))

Example:

```jsx
function RefreshButton() {
  const revalidator =
    useRevalidator();

  return (
    <button
      onClick={() =>
        revalidator.revalidate()
      }
    >
      Refresh
    </button>
  );
}
```

Notice the distinction:

```text
normal mutation
→ automatic revalidation

external reason
→ useRevalidator
```

React Router specifically recommends not using `useRevalidator` for ordinary CRUD interactions because forms/fetchers/actions already handle those revalidation flows. ([React Router](https://reactrouter.com/api/hooks/useRevalidator?utm_source=chatgpt.com "useRevalidator | React Router"))

---

# 33. Route loaders vs TanStack Query

Now the comparison you've been waiting for.

Suppose:

```text
/product/42
```

needs product data.

You have two plausible architectures.

### React Router

```text
route
 ↓
loader
 ↓
product data
 ↓
component
```

### TanStack Query

```text
component
 ↓
useQuery
 ↓
query cache
 ↓
product data
```

Neither is inherently "React's one true way."

They solve different forms of server-state management.

---

# 34. React Router is route-oriented

React Router's Data APIs naturally think:

```text
"What data does this route require?"
```

For example:

```text
/products/42
```

has:

```text
loader → product 42
```

And navigation automatically coordinates with that data.

This is a very natural architecture for route-driven applications.

---

# 35. TanStack Query is cache-oriented

TanStack Query thinks more like:

```text
"What server data do I currently have cached,
and how fresh is it?"
```

For example:

```js
useQuery({
  queryKey: ["product", id],
  queryFn: ...
});
```

Its architecture is especially useful when many different parts of the application share the same server data.

For example:

```text
header
sidebar
product page
recommendations
modal
```

may all access related queries.

The query cache becomes a dedicated server-state system.

---

# 36. React Router's loader is not exactly a global cache

This is an important distinction.

A loader is fundamentally associated with route data.

```text
route match
 ↓
loader
 ↓
route component
```

TanStack Query has a first-class cache:

```text
query key
 ↓
cache
 ↓
many consumers
```

So asking:

> "Which one caches better?"

misses the architectural difference.

The better question is:

> "Who owns this server data, and what lifecycle do I need?"

---

# 37. A useful architecture decision

For route-specific data:

```text
route-specific
navigation-dependent
URL-driven
```

React Router loaders can be a natural fit.

For:

```text
widely shared server data
background refresh
stale/fresh semantics
complex cache relationships
highly interactive server state
```

TanStack Query can be a strong fit.

And applications can combine them.

---

# 38. Combining React Router and TanStack Query

You could have:

```text
Route loader
   ↓
initial data
   ↓
Client Component
   ↓
TanStack Query
   ↓
background synchronization
```

For example, a loader can provide initial server data while the client query cache takes over ongoing synchronization.

Or you might keep route-critical data in loaders and use TanStack Query only for highly dynamic widgets.

There isn't a requirement that every server-state problem in an app use exactly one abstraction.

---

# 39. React Router loaders vs `useEffect`

This comparison is even simpler.

### `useEffect` approach

```text
component mounts
 ↓
effect
 ↓
fetch
 ↓
state
 ↓
render
```

### loader approach

```text
navigation
 ↓
loader
 ↓
route data
 ↓
render
```

For data whose existence is tied directly to a route, the loader model often gives the router more information about the lifecycle.

The router knows:

```text
"I'm navigating to this route and need this data."
```

That lets it coordinate loading and navigation together.

---

# 40. Pending navigation becomes part of the UX

Suppose we're at:

```text
/products
```

and click:

```text
/products/42
```

The router can know:

```text
target route = /products/42
loader pending
```

So UI can respond:

```text
link click
 ↓
navigation pending
 ↓
show pending indicator
 ↓
loader completes
 ↓
new route rendered
```

This is better than independently guessing:

```text
"Maybe we should show a spinner because some component probably just requested data."
```

The router knows the actual navigation lifecycle.

---

# 41. `<NavLink>` and pending UI

You already learned `NavLink` for active navigation.

It can also participate in pending navigation UX.

Conceptually:

```jsx
<NavLink to="/reports">
  Reports
</NavLink>
```

can render differently while the target navigation is pending.

This means navigation UI can represent:

```text
active route
+
pending target
```

which is a natural fit for route-level loading.

---

# 42. Actions vs React 19 Server Functions

You've just learned Server Functions.

Now compare:

### React Server Function

```text
Client
 ↓
Server Function
 ↓
server mutation
```

### React Router route action

```text
Router interaction
 ↓
route action
 ↓
server mutation
 ↓
loader revalidation
```

The second one is more tightly connected to **routing and route data**.

That's the important difference.

---

# 43. React Router actions are route-owned mutations

Imagine:

```text
/routes
  /todos
  /todos/:id
```

The action is associated with the route.

Therefore:

```text
URL
 +
mutation
 +
affected route data
```

are all closely connected.

That's very useful when the UI architecture itself is strongly route-driven.

---

# 44. Server Function vs route action

You can think about the choice like this:

```text
"I need to mutate server state."
        ↓
Server Function

"I need this route's UI/form
to mutate its route-owned data
and then revalidate."
        ↓
Route action
```

Again, this isn't a prohibition.

A framework may combine these mechanisms.

But the ownership distinction is useful.

---

# 45. Framework Mode adds even more server behavior

React Router's current Framework Mode supports server-side `loader` and `action` functions, while `clientLoader` and `clientAction` explicitly move that work to the browser. ([React Router](https://reactrouter.com/start/framework/actions?utm_source=chatgpt.com "Actions | React Router"))

So the architecture can look like:

```text
Framework route
      │
      ├── loader
      │      ↓
      │    server
      │
      ├── clientLoader
      │      ↓
      │    browser
      │
      ├── action
      │      ↓
      │    server
      │
      └── clientAction
             ↓
           browser
```

That makes the server/client distinction we just studied in RSC immediately relevant.

---

# 46. `clientLoader`

Suppose a page needs browser-only data:

```text
localStorage
IndexedDB
browser APIs
client-only cache
```

You may use:

```js
export async function clientLoader() {
  const theme =
    localStorage.getItem("theme");

  return { theme };
}
```

This is explicitly client-side data loading. React Router currently documents `clientLoader` as the browser-side alternative when data should be loaded on the client. ([React Router](https://reactrouter.com/start/framework/data-loading?utm_source=chatgpt.com "Data Loading | React Router"))

So:

```text
loader
→ server-side route data

clientLoader
→ browser-side route data
```

This is another example of explicitly choosing an execution environment.

---

# 47. A route's data architecture can therefore be layered

Imagine:

```text
Product route
      │
      ├── loader
      │     ↓
      │   server product
      │
      ├── clientLoader
      │     ↓
      │   local preferences
      │
      ├── action
      │     ↓
      │   server mutation
      │
      └── clientAction
            ↓
          local/browser mutation
```

This is quite different from:

```text
component
   ↓
useEffect
   ↓
random request
```

The route now describes its own data lifecycle.

---

# 48. React Router's route object is becoming an architectural unit

Recall:

```js
{
  path,
  loader,
  action,
  Component
}
```

This one object can describe:

```text
URL
data requirements
mutations
UI
```

Current React Router describes route modules as the foundation for features including data loading, actions, revalidation, and error boundaries. ([React Router](https://reactrouter.com/start/data/route-object?utm_source=chatgpt.com "Route Object | React Router"))

That's a powerful design concept.

---

# 49. Error handling fits here too

Suppose:

```js
loader: async ({ params }) => {
  const product =
    await getProduct(params.id);

  if (!product) {
    throw new Response(
      "Not found",
      { status: 404 }
    );
  }

  return product;
}
```

Now the route's error UI can handle the failure.

This is architecturally cleaner than:

```text
component
 ↓
effect
 ↓
request
 ↓
setError
 ↓
hope the component renders correct error UI
```

The route itself knows:

```text
"This data failed."
```

and the router knows which UI boundary corresponds to that route.

---

# 50. Nested error boundaries become especially valuable

Imagine:

```text
/app
 ├── loader
 └── component
      │
      ├── /profile
      │
      └── /reports
```

A report loader can fail without necessarily destroying the entire application shell.

Conceptually:

```text
App shell
 ├── navigation
 ├── profile
 └── reports
       ↓
     loader fails
       ↓
     reports error UI
```

This fits perfectly with the Error Boundary concepts we learned earlier.

---

# 51. Route-level data ownership reduces waterfalls

Imagine:

```text
Dashboard
 ↓
fetch user
 ↓
render
 ↓
fetch reports
 ↓
render
```

A route hierarchy can often identify all active loaders up front.

React Router executes active loaders in parallel by default. ([React Router](https://reactrouter.com/how-to/data-strategy?utm_source=chatgpt.com "Data Strategy | React Router"))

So:

```text
user loader ───────────┐
reports loader ────────┼──→ render route tree
stats loader ──────────┘
```

This is exactly the same parallelism principle we learned with:

```text
Promise.all
```

but integrated into routing.

---

# 52. Fetcher vs Action vs Loader

Keep this table:

|API|Question|
|---|---|
|`loader`|What data does this route need?|
|`action`|How does this route mutate data?|
|`useLoaderData`|Give me this route's loaded data|
|`useActionData`|What did the last action return?|
|`useNavigation`|What is navigation doing?|
|`useFetcher`|Interact with route data without navigation|
|`useRevalidator`|Re-run route data loading for an external reason|

This is the Data Router vocabulary.

---

# 53. A complete CRUD example

Let's imagine:

```text
/users
/users/:id
```

### `/users`

```text
loader
 ↓
all users
```

### `/users/:id`

```text
loader
 ↓
one user
```

### edit form

```text
action
 ↓
update user
 ↓
revalidate
```

### delete button

```text
fetcher
 ↓
delete action
 ↓
revalidate
```

So:

```text
READ
→ loader

CREATE
→ action

UPDATE
→ action

DELETE
→ action

NON-NAVIGATING interaction
→ fetcher
```

That is a very clean route architecture.

---

# 54. Where optimistic UI fits

React Router can participate in pending-state UI, while you can also build optimistic interfaces.

For example:

```text
fetcher.state === "submitting"
```

can tell you:

```text
"the mutation is underway."
```

You can then temporarily render:

```text
Deleting...
```

or optimistically hide the row before the server confirms.

The same conceptual distinction from Lesson 34 remains:

```text
optimistic state
≠
server truth
```

The router's revalidation is what eventually brings you back to authoritative route data.

---

# 55. The router's version of "server consistency"

Remember TanStack Query:

```text
mutation
 ↓
invalidate
 ↓
refetch
```

Data Router:

```text
action
 ↓
revalidate
 ↓
reload loader data
```

This is a powerful pattern because your server remains authoritative.

Instead of trying to reconstruct every affected representation manually:

```text
setList(...)
setDetail(...)
setCount(...)
setSidebar(...)
```

you can say:

```text
mutation succeeded
 ↓
ask route loaders again
```

That can dramatically simplify CRUD-heavy applications.

---

# 56. But direct cache updates can still be useful

Suppose a mutation returns enough information to update UI immediately.

You may still use optimistic UI or local state.

The important question is:

```text
Is this temporary UI
or authoritative server state?
```

For example:

```text
temporary "Saving..."
→ UI state

confirmed user's name
→ server-derived data
```

Don't accidentally make local UI state the authoritative source of server data.

---

# 57. A comparison with the architecture you've learned so far

You now have three substantial server-data models.

### `useEffect` fetching

```text
component owns fetching
```

### TanStack Query

```text
query cache owns server state
```

### React Router Data APIs

```text
route owns route-associated data
```

And now Server Components:

```text
server-rendered component owns server-side execution
```

These are different abstractions.

A good React engineer doesn't ask:

> "Which library is best?"

They ask:

> "Who owns this piece of state, what lifecycle does it have, and what boundary does it cross?"

---

# 58. The strongest mental model for React Router

Think of a route as a mini state machine.

For:

```text
/products/42
```

the route may have:

```text
IDLE
 │
 │ navigation
 ▼
LOADING
 │
 │ loader resolves
 ▼
READY
 │
 │ form submit
 ▼
SUBMITTING
 │
 │ action resolves
 ▼
REVALIDATING
 │
 │ loader resolves
 ▼
READY
```

That is much richer than:

```text
URL → component
```

The router is effectively coordinating:

```text
navigation
data fetching
mutation
pending UI
revalidation
error boundaries
```

---

# 59. One complete flow

Let's imagine editing a project title.

Current page:

```text
/projects/42
```

Loader:

```text
getProject(42)
```

returns:

```js
{
  id: 42,
  title: "React Course"
}
```

User submits:

```text
React Course — Advanced
```

Action:

```text
request.formData()
 ↓
updateProject(42, title)
 ↓
database
```

Action succeeds.

React Router:

```text
successful action
 ↓
revalidate loaders
```

Loader runs again:

```text
getProject(42)
```

returns:

```js
{
  id: 42,
  title: "React Course — Advanced"
}
```

Component renders.

The entire consistency loop:

```text
                READ
                 ↓
             loader
                 ↓
               UI
                 ↓
              submit
                 ↓
              action
                 ↓
             mutation
                 ↓
           revalidation
                 ↓
               loader
                 ↓
              fresh UI
```

That's the central Data Router pattern.

---

# 60. The bigger React architecture we're building

Look at how everything now fits.

```text
                     APPLICATION
                          │
             ┌────────────┴─────────────┐
             │                          │
          ROUTER                     COMPONENT
             │                          │
      ┌──────┼──────┐             ┌────┴─────┐
      │      │      │             │          │
   loader  action fetcher       local       context
      │      │      │             state
      │      │      │
      ▼      ▼      ▼
    server data / mutations
             │
             ▼
        revalidation
```

Then, depending on the architecture:

```text
Server Components
        ↓
server-side execution

Server Functions
        ↓
client → server mutation

TanStack Query
        ↓
shared server-state cache

React Router
        ↓
route-owned data lifecycle
```

These can coexist.

---

# 61. When should you choose a loader?

A loader is particularly natural when:

```text
the data is required by a route
+
the URL determines the data
+
navigation should coordinate with loading
```

Examples:

```text
/profile
/products/42
/projects/17/settings
/search?q=react
```

---

# 62. When should you choose a fetcher?

A fetcher is natural when:

```text
interaction affects server data
+
you don't want navigation
```

Examples:

```text
like button
delete row
toggle subscription
inline edit
archive item
mark notification read
```

---

# 63. When should you choose TanStack Query?

It becomes especially attractive when:

```text
server data is shared widely
+
you need sophisticated client cache behavior
+
background refresh matters
+
staleness matters
+
multiple components independently consume the data
```

Examples:

```text
dashboard widgets
real-time-ish data
complex filters
pagination
infinite scrolling
shared entities
```

---

# 64. When should you choose Server Components?

When:

```text
data can be obtained on server
+
the component doesn't need browser interaction
+
you want to keep server-only logic/dependencies out of client code
```

Examples:

```text
article content
product details
server-generated navigation
database-backed content
```

---

# 65. When should you use plain `useEffect`?

Still plenty of cases.

Especially when you're synchronizing with an external browser system:

```text
WebSocket
event listener
third-party widget
DOM API
browser subscription
analytics integration
```

Remember:

```text
useEffect
→ synchronize with external systems
```

Not:

```text
useEffect
→ generic data fetching mechanism for everything
```

---

# 66. One important modern React Router detail

The current React Router ecosystem has three broad modes:

```text
Declarative
Data
Framework
```

Declarative gives you basic routing.

Data mode adds route data loading, actions, pending state, fetchers, and related behavior.

Framework mode adds more application-level server/build/runtime capabilities. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

You don't need to memorize the modes yet.

The important conceptual progression is:

```text
routing
   ↓
routing + data
   ↓
routing + data + application/server framework
```

---

# 67. Why this topic matters after SSR/RSC

Remember the previous lessons:

```text
SSR
→ server produces HTML

RSC
→ some components execute on server

Server Functions
→ client can invoke server work
```

Now React Router can add:

```text
route
 ↓
server loader
 ↓
server action
 ↓
revalidation
```

So modern React applications may have several server-aware abstractions simultaneously.

That can look overwhelming until you understand what each one owns:

```text
SSR
→ initial rendering

RSC
→ component execution boundary

Server Function
→ callable server capability

Route loader
→ route data

Route action
→ route mutation

Fetcher
→ non-navigation route interaction

TanStack Query
→ client-side server cache
```

Now they stop looking like duplicate APIs.

---

# 68. The biggest takeaway

The old mental model:

```text
component
  ↓
effect
  ↓
fetch
  ↓
state
```

is only one architecture.

A Data Router lets you say:

```text
route
  ↓
loader
  ↓
data
  ↓
component
```

and for mutations:

```text
form/fetcher
  ↓
action
  ↓
server mutation
  ↓
automatic revalidation
  ↓
loader
  ↓
fresh UI
```

That is a very different way of designing the application.

---

# 69. Keep this diagram

```text
                    URL
                     │
                     ▼
                  ROUTER
                     │
          ┌──────────┴──────────┐
          │                     │
        LOADER                ACTION
          │                     │
          ▼                     ▼
      READ DATA             MUTATE DATA
          │                     │
          │                     ▼
          │                SUCCESS
          │                     │
          │                     ▼
          │               REVALIDATE
          │                     │
          └──────────┬──────────┘
                     ▼
                FRESH DATA
                     │
                     ▼
                COMPONENT
```

And when the interaction shouldn't navigate:

```text
COMPONENT
   │
   ▼
useFetcher()
   │
   ▼
loader / action
   │
   ▼
revalidation
   │
   ▼
fresh route data
```

That's React Router Data APIs.

---

## What to remember

```text
loader
→ route-level data loading

action
→ route-level mutation

useLoaderData
→ read loader result

useActionData
→ read latest action result

useNavigation
→ navigation lifecycle

<Form>
→ router-managed form submission

useFetcher
→ load/mutate route data without navigation

useRevalidator
→ manually revalidate for external reasons

revalidation
→ bring route data back in sync after mutations

shouldRevalidate
→ selectively optimize that behavior
```

The next major topic should be **code splitting, `lazy`, route-level lazy loading, and how Suspense interacts with JavaScript bundles**. That will connect directly to everything you've learned about Suspense, rendering performance, Server Components, and route architecture.