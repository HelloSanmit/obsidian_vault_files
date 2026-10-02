
# Lesson 15 — Routing with React Router

Until now, we've mostly built a single React page:

```text
App
 ├── Navbar
 ├── Sidebar
 └── Main
```

Real applications need URLs:

```text
/
 /dashboard
 /products
 /products/42
 /settings
```

and the UI should change based on the URL.

That's **routing**.

For the current React Router release, the project provides three modes—Declarative, Data, and Framework. We'll start with **Declarative mode** because it exposes the routing fundamentals with the least machinery. The current docs use the `react-router` package for this mode. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

---

# 1. What problem does routing solve?

Suppose you're building an admin application.

You want:

```text
URL                         UI
────────────────────────────────────────
/                           Home
/dashboard                  Dashboard
/products                   Product list
/products/42                Product #42
/settings                   Settings
```

Without a router, you'd have to inspect:

```js
window.location.pathname
```

and manually decide what to render.

You could do that:

```jsx
function App() {
  const path = window.location.pathname;

  if (path === "/") {
    return <Home />;
  }

  if (path === "/dashboard") {
    return <Dashboard />;
  }

  return <NotFound />;
}
```

But as your application grows, this gets unpleasant.

React Router gives you a system for:

```text
URL
 ↓
route matching
 ↓
component/UI
```

and for navigation between routes. The official docs describe Declarative mode as providing basic route matching, navigation, and active-state APIs such as `Link`, `NavLink`, `useNavigate`, and `useLocation`. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

---

# 2. Install React Router

With a typical Vite React project:

```bash
npm install react-router
```

The current React Router installation docs use this package and then wrap the app in `<BrowserRouter>`. ([React Router](https://reactrouter.com/start/declarative/installation?utm_source=chatgpt.com "Installation | React Router"))

You'll encounter older tutorials using:

```bash
npm install react-router-dom
```

That's worth recognizing when reading old material, but for our current lessons we'll follow the current React Router documentation and use `react-router`. ([React Router](https://reactrouter.com/start/declarative/installation?utm_source=chatgpt.com "Installation | React Router"))

---

# 3. `BrowserRouter`

At the root of your application:

```jsx
import ReactDOM from "react-dom/client";
import { BrowserRouter } from "react-router";
import App from "./App";

ReactDOM.createRoot(
  document.getElementById("root")
).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

`BrowserRouter` provides routing context using the browser's History API. ([React Router](https://reactrouter.com/api/declarative-routers/BrowserRouter?utm_source=chatgpt.com "BrowserRouter | React Router"))

Think of it as:

```text
BrowserRouter
     │
     └── makes routing information available
         to everything underneath
```

You generally put it near the top of the application.

---

# 4. Define routes

Now:

```jsx
import {
  Routes,
  Route
} from "react-router";

function App() {
  return (
    <Routes>
      <Route
        path="/"
        element={<Home />}
      />

      <Route
        path="/dashboard"
        element={<Dashboard />}
      />

      <Route
        path="/products"
        element={<Products />}
      />

      <Route
        path="/settings"
        element={<Settings />}
      />
    </Routes>
  );
}
```

The relationship is:

```text
path                 element
────────────────────────────────
"/"              →   <Home />
"/dashboard"     →   <Dashboard />
"/products"      →   <Products />
"/settings"      →   <Settings />
```

React Router matches the current location against your route configuration and renders the matching route element. ([React Router](https://reactrouter.com/start/declarative/routing?utm_source=chatgpt.com "Routing | React Router"))

---

# 5. `Route` is basically a mapping

Conceptually:

```jsx
<Route
  path="/products"
  element={<Products />}
/>
```

means:

> When the URL matches `/products`, render this UI.

That's the basic idea.

---

# 6. Navigation

Now suppose you're on:

```text
/
```

and you want a link to:

```text
/products
```

You could technically use:

```html
<a href="/products">Products</a>
```

but React Router gives you:

```jsx
import { Link } from "react-router";

<Link to="/products">
  Products
</Link>
```

The official docs recommend `Link` and `NavLink` for declarative navigation. ([React Router](https://reactrouter.com/start/declarative/routing?utm_source=chatgpt.com "Routing | React Router"))

---

# 7. Why use `Link` instead of `<a>`?

This is an important SPA concept.

With a traditional anchor:

```html
<a href="/products">
```

the browser can perform a normal document navigation.

A router-aware:

```jsx
<Link to="/products">
```

lets React Router handle the navigation within the application.

Conceptually:

```text
Link
 ↓
router changes location
 ↓
matching route changes
 ↓
appropriate React UI renders
```

React Router uses browser navigation/history mechanisms underneath, but your React application remains in control of the route transition. ([React Router](https://reactrouter.com/api/declarative-routers/BrowserRouter?utm_source=chatgpt.com "BrowserRouter | React Router"))

---

# 8. `NavLink`

`NavLink` is useful when you want to know whether the link is currently active.

```jsx
import { NavLink } from "react-router";

<NavLink
  to="/dashboard"
  className={({ isActive }) =>
    isActive ? "active" : ""
  }
>
  Dashboard
</NavLink>
```

The callback receives information about the link's current state.

Conceptually:

```text
current URL = /dashboard
             ↓
NavLink("/dashboard")
             ↓
isActive = true
```

React Router specifically provides `NavLink` for active-state styling. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

---

# 9. Build a navbar

Let's make something realistic:

```jsx
import { NavLink } from "react-router";

function Navbar() {
  return (
    <nav>
      <NavLink to="/">
        Home
      </NavLink>

      <NavLink to="/dashboard">
        Dashboard
      </NavLink>

      <NavLink to="/products">
        Products
      </NavLink>

      <NavLink to="/settings">
        Settings
      </NavLink>
    </nav>
  );
}
```

Then:

```jsx
function App() {
  return (
    <>
      <Navbar />

      <Routes>
        <Route
          path="/"
          element={<Home />}
        />

        <Route
          path="/dashboard"
          element={<Dashboard />}
        />

        <Route
          path="/products"
          element={<Products />}
        />

        <Route
          path="/settings"
          element={<Settings />}
        />
      </Routes>
    </>
  );
}
```

Now the architecture is:

```text
Browser URL
     ↓
BrowserRouter
     ↓
Routes
     ↓
matching Route
     ↓
component
```

---

# 10. Route parameters

Now things get more interesting.

Suppose you have:

```text
/products/42
```

and:

```text
/products/73
```

You don't want to define:

```jsx
<Route path="/products/42" ... />
<Route path="/products/73" ... />
```

Instead:

```jsx
<Route
  path="/products/:productId"
  element={<ProductDetails />}
/>
```

`:productId` is a **dynamic segment**.

React Router supports dynamic segments and exposes their values through `useParams`. ([React Router](https://reactrouter.com/start/declarative/routing?utm_source=chatgpt.com "Routing | React Router"))

---

# 11. Read the parameter

Inside the component:

```jsx
import { useParams } from "react-router";

function ProductDetails() {
  const { productId } = useParams();

  return (
    <h1>
      Product {productId}
    </h1>
  );
}
```

Visit:

```text
/products/42
```

and:

```js
productId
```

is:

```text
"42"
```

Notice it's URL data, so you should treat it appropriately. If you need a number:

```js
const id = Number(productId);
```

---

# 12. Trace what's happening

URL:

```text
/products/42
```

Route:

```jsx
<Route
  path="/products/:productId"
  element={<ProductDetails />}
/>
```

Router matches:

```text
/product/42
       ↓
productId = "42"
```

Then:

```jsx
useParams()
```

returns something conceptually like:

```js
{
  productId: "42"
}
```

Then your component can use it:

```jsx
<ProductDetails />
```

This is just route information becoming component input.

---

# 13. Route parameters are not props

This is a subtle point.

You might expect:

```jsx
<ProductDetails productId="42" />
```

but that's not what's happening.

Instead:

```text
URL
 ↓
router
 ↓
useParams()
 ↓
component gets route information
```

The router provides the parameter through its routing context/hooks.

That's why:

```jsx
const { productId } = useParams();
```

works without passing a prop yourself.

---

# 14. Product details + fetching

Suppose:

```text
/products/42
```

means:

```text
fetch product 42
```

A simple client-side pattern might be:

```jsx
function ProductDetails() {
  const { productId } = useParams();

  const [product, setProduct] =
    useState(null);

  useEffect(() => {
    fetch(`/api/products/${productId}`)
      .then(res => res.json())
      .then(data => setProduct(data));
  }, [productId]);

  if (!product) {
    return <p>Loading...</p>;
  }

  return (
    <h1>{product.name}</h1>
  );
}
```

Now multiple concepts interact:

```text
URL parameter
      ↓
useParams
      ↓
productId
      ↓
Effect
      ↓
API
      ↓
state
      ↓
UI
```

Later we'll improve this significantly using React Router's Data APIs and/or dedicated server-state tools.

---

# 15. Nested routes

Now suppose your application has:

```text
/dashboard
/dashboard/overview
/dashboard/analytics
/dashboard/settings
```

We could define them independently, but React Router allows nested route structures.

Conceptually:

```text
Dashboard
├── Overview
├── Analytics
└── Settings
```

Configuration:

```jsx
<Routes>
  <Route
    path="/dashboard"
    element={<DashboardLayout />}
  >
    <Route
      index
      element={<Overview />}
    />

    <Route
      path="analytics"
      element={<Analytics />}
    />

    <Route
      path="settings"
      element={<DashboardSettings />}
    />
  </Route>
</Routes>
```

Notice:

```jsx
path="analytics"
```

rather than:

```jsx
path="/dashboard/analytics"
```

The child route is relative to its parent.

React Router's routing docs support nested routes and index routes in this manner. ([React Router](https://reactrouter.com/start/declarative/routing?utm_source=chatgpt.com "Routing | React Router"))

---

# 16. But where does the child actually render?

This is what `<Outlet />` is for.

Parent:

```jsx
import { Outlet } from "react-router";

function DashboardLayout() {
  return (
    <div>
      <h1>Dashboard</h1>

      <nav>
        <NavLink to="/dashboard">
          Overview
        </NavLink>

        <NavLink to="/dashboard/analytics">
          Analytics
        </NavLink>

        <NavLink to="/dashboard/settings">
          Settings
        </NavLink>
      </nav>

      <Outlet />
    </div>
  );
}
```

`<Outlet />` is essentially:

> "Render the currently matched child route here."

React Router's route APIs expose `Outlet` specifically for nested route rendering. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

---

# 17. Visualize `Outlet`

Imagine:

```text
DashboardLayout
┌─────────────────────────────┐
│ Dashboard                   │
│                             │
│ Overview | Analytics | ...  │
│                             │
│         <Outlet />           │
│              ↓              │
│       child route appears   │
│                             │
└─────────────────────────────┘
```

If URL is:

```text
/dashboard
```

then:

```text
Outlet → Overview
```

If:

```text
/dashboard/analytics
```

then:

```text
Outlet → Analytics
```

If:

```text
/dashboard/settings
```

then:

```text
Outlet → DashboardSettings
```

This is one of the most useful routing patterns.

---

# 18. Why nested routes are powerful

Without nesting, you might duplicate:

```text
Dashboard header
Dashboard sidebar
Dashboard navigation
```

inside every child page.

With a layout route:

```text
DashboardLayout
      ↓
     Outlet
      ↓
 ┌────┼─────┐
 ↓    ↓     ↓
Home Analytics Settings
```

the shared shell is rendered once conceptually around the child route.

This naturally leads into route layouts and protected sections.

---

# 19. Index routes

This:

```jsx
<Route
  path="dashboard"
  element={<DashboardLayout />}
>
  <Route
    index
    element={<Overview />}
  />
</Route>
```

means:

> When `/dashboard` itself is matched, render `Overview` in the parent's `<Outlet />`.

So:

```text
/dashboard
   ↓
DashboardLayout
   ↓
index route
   ↓
Overview
```

The current routing docs use `index` routes in exactly this way. ([React Router](https://reactrouter.com/start/declarative/routing?utm_source=chatgpt.com "Routing | React Router"))

---

# 20. Navigation from code

Sometimes you don't have a visible link.

Suppose a form submits successfully and you want to navigate:

```text
/login
   ↓
success
   ↓
/dashboard
```

React Router provides:

```jsx
import { useNavigate } from "react-router";

function Login() {
  const navigate = useNavigate();

  function handleSuccess() {
    navigate("/dashboard");
  }

  return (
    <button onClick={handleSuccess}>
      Continue
    </button>
  );
}
```

`useNavigate` is one of React Router's navigation APIs in Declarative mode. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

---

# 21. `Link` vs `useNavigate`

This is another useful distinction.

### `Link`

Use when the user is intentionally navigating through a UI element:

```jsx
<Link to="/settings">
  Settings
</Link>
```

### `useNavigate`

Use when some code needs to cause navigation:

```jsx
navigate("/dashboard");
```

For example:

```text
login succeeded
   ↓
navigate(...)
```

or:

```text
logout
   ↓
navigate("/login")
```

Don't use `useNavigate` when a normal `Link` expresses the interaction more naturally.

---

# 22. `useLocation`

You may also need the current location:

```jsx
import { useLocation } from "react-router";

function Component() {
  const location = useLocation();

  console.log(location);
}
```

This gives you information about the current URL/location.

The current React Router docs list `useLocation` among the APIs available in Declarative mode. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

---

# 23. Query parameters

Now suppose the URL is:

```text
/products?category=laptop&sort=price
```

The pathname:

```text
/products
```

and query string:

```text
?category=laptop&sort=price
```

are different pieces of the URL.

Route params:

```text
/products/:productId
```

are matched as part of the route path.

Query parameters:

```text
/products?category=laptop
```

are typically used for filtering, sorting, pagination, search, and similar UI state.

React Router has APIs for reading/updating search parameters; we'll cover those in the next routing lesson when we build a real filterable page.

---

# 24. 404 routes

What if the user visits:

```text
/completely/random/url
```

You can define a catch-all:

```jsx
<Route
  path="*"
  element={<NotFound />}
/>
```

Then:

```text
unknown URL
    ↓
NotFound
```

This is an essential production pattern.

---

# 25. Route layout vs ordinary layout component

Consider:

```text
App
└── DashboardLayout
     ├── sidebar
     ├── navbar
     └── Outlet
```

The router itself knows:

```text
/dashboard/*
```

belongs inside `DashboardLayout`.

This is different from just manually doing:

```jsx
<DashboardLayout>
  <Dashboard />
</DashboardLayout>
```

The route structure can now control which child page appears inside the layout.

That becomes extremely useful for:

```text
auth sections
dashboard sections
admin sections
settings sections
```

---

# 26. Protected routes

Suppose:

```text
/dashboard
```

should only be visible to authenticated users.

A common conceptual pattern is:

```text
/dashboard
      ↓
authentication check
      ↓
authenticated?
   /       \
 yes        no
 ↓           ↓
Dashboard   Login
```

In simple declarative routing, people often create a wrapper component:

```jsx
function ProtectedRoute() {
  const { user } = useAuth();

  if (!user) {
    return <Navigate to="/login" />;
  }

  return <Outlet />;
}
```

Then:

```jsx
<Route element={<ProtectedRoute />}>
  <Route
    path="/dashboard"
    element={<Dashboard />}
  />
</Route>
```

`Navigate` is part of React Router's Declarative APIs. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

We'll revisit authentication later because real authentication requires more than merely checking whether a JavaScript variable exists.

---

# 27. Why I don't want you to learn only `Routes` and `Route`

The current React Router ecosystem is broader.

The official project has:

```text
Declarative mode
Data mode
Framework mode
```

Data mode adds features such as:

```text
loader
action
pending states
useFetcher
```

and moves route configuration outside ordinary React rendering. Framework mode builds on even more functionality. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

We're deliberately learning Declarative mode first because the underlying concepts are easier to understand there.

Then we'll learn Data mode once you understand:

```text
URL
route
nested route
params
navigation
layout
```

---

# 28. Why Data Routers matter later

Suppose:

```text
/products/42
```

requires data.

A simple Declarative approach might be:

```text
route
 ↓
component
 ↓
useEffect
 ↓
fetch
 ↓
state
```

Data routers can instead associate data loading with the route itself:

```text
route
 ↓
loader
 ↓
route data
 ↓
component
```

React Router's current Data Mode documentation specifically adds loaders, actions, pending states, and `useFetcher` on top of Declarative routing. ([React Router](https://reactrouter.com/start/modes?utm_source=chatgpt.com "Picking a Mode | React Router"))

We'll learn that later, after the fundamentals are comfortable.

---

# 29. Current React Router vocabulary

You will encounter these terms frequently:

```text
Router
Route
Routes
Link
NavLink
Outlet
Navigate
useNavigate
useLocation
useParams
```

And later:

```text
loader
action
useLoaderData
useActionData
useFetcher
deferred data
route.lazy
```

You don't need the second group yet.

---

# 30. A complete small application

Let's combine the fundamentals.

```jsx
import {
  BrowserRouter,
  Routes,
  Route,
  NavLink,
  Outlet,
  useParams
} from "react-router";

function Layout() {
  return (
    <div>
      <nav>
        <NavLink to="/">
          Home
        </NavLink>

        <NavLink to="/products">
          Products
        </NavLink>

        <NavLink to="/settings">
          Settings
        </NavLink>
      </nav>

      <Outlet />
    </div>
  );
}

function Home() {
  return <h1>Home</h1>;
}

function Products() {
  const products = [
    { id: 1, name: "Laptop" },
    { id: 2, name: "Phone" }
  ];

  return (
    <div>
      <h1>Products</h1>

      {products.map(product => (
        <div key={product.id}>
          <Link
            to={`/products/${product.id}`}
          >
            {product.name}
          </Link>
        </div>
      ))}
    </div>
  );
}

function ProductDetails() {
  const { productId } = useParams();

  return (
    <h1>
      Product ID: {productId}
    </h1>
  );
}

function Settings() {
  return <h1>Settings</h1>;
}

function NotFound() {
  return <h1>404 — Not found</h1>;
}
```

Then:

```jsx
function App() {
  return (
    <Routes>
      <Route
        element={<Layout />}
      >
        <Route
          path="/"
          element={<Home />}
        />

        <Route
          path="/products"
          element={<Products />}
        />

        <Route
          path="/products/:productId"
          element={<ProductDetails />}
        />

        <Route
          path="/settings"
          element={<Settings />}
        />

        <Route
          path="*"
          element={<NotFound />}
        />
      </Route>
    </Routes>
  );
}
```

And root:

```jsx
ReactDOM.createRoot(
  document.getElementById("root")
).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

Now the application looks like:

```text
                    BrowserRouter
                         │
                       Routes
                         │
                       Layout
                         │
                    ┌────┴────┐
                    │  Outlet │
                    └────┬────┘
                         │
          ┌──────────────┼───────────────┐
          ↓              ↓               ↓
         Home        Products         Settings
                         │
                         ↓
                /products/:productId
                         │
                         ↓
                  ProductDetails
```

---

# 31. One subtle thing about nesting

When you write:

```jsx
<Route
  element={<Layout />}
>
```

without a `path`, it acts as a layout route rather than matching some URL segment itself. React Router's `Route` API explicitly supports routes without a path as layout routes. ([React Router](https://reactrouter.com/api/components/Route?utm_source=chatgpt.com "Route | React Router"))

Then:

```jsx
<Outlet />
```

marks where the matched child appears.

This combination is worth remembering:

```text
layout route
   +
Outlet
   =
nested page architecture
```

---

# 32. Your routing mental model

You can now think:

```text
                 URL
                  ↓
             React Router
                  ↓
             route matching
                  ↓
        ┌─────────┴─────────┐
        ↓                   ↓
   route component       route params
        ↓                   ↓
      JSX               useParams()
        ↓
       UI
```

Navigation:

```text
<Link>
   ↓
URL changes
   ↓
route matching
   ↓
different UI
```

Programmatic navigation:

```text
event
 ↓
navigate("/dashboard")
 ↓
URL changes
 ↓
new route
```

Nested routes:

```text
Parent route
     ↓
  <Outlet />
     ↓
Child route
```

That's the core routing system.

---

# 33. Don't make this mistake

Don't think:

> "Routing means React creates a new HTML page."

In a typical client-side SPA setup, the browser document remains the application shell while the route determines which React UI is rendered within it.

That's why you can navigate:

```text
/products
     ↓
/products/42
```

without treating every transition like loading an entirely separate document.

There are more advanced server/framework considerations around routing, SSR, and deployment, which we'll cover later.

---

# 34. What to practice

Build this:

```text
/
├── Home
│
├── /products
│   ├── Laptop
│   ├── Phone
│   └── Keyboard
│
├── /products/:id
│
├── /about
│
└── *
    └── 404
```

Then add a layout:

```text
┌───────────────────────────────┐
│ Navbar                        │
├───────────────────────────────┤
│                               │
│        <Outlet />              │
│                               │
└───────────────────────────────┘
```

Then make each product link to:

```text
/products/1
/products/2
/products/3
```

and use:

```jsx
useParams()
```

to display the ID.

The exercise isn't about making the page pretty.

It's about getting this flow into muscle memory:

```text
URL
 ↓
Route
 ↓
Component
 ↓
useParams
 ↓
data
 ↓
UI
```

---

# One important update to your React roadmap

We're now entering the **application-building phase**.

The remaining major sequence is:

```text
Routing
   ↓
URL/search params
   ↓
Data fetching
   ↓
Loading/error states
   ↓
Route loaders/actions
   ↓
Authentication
   ↓
Server state
   ↓
Forms + mutations
   ↓
Advanced state management
   ↓
Testing
   ↓
Performance
   ↓
Production architecture
   ↓
Modern React / framework patterns
```

The next lesson should be **routing in practice + URL state**, where we'll build a product application with:

```text
/products?search=laptop&category=electronics&page=2
```

and learn why the URL itself can be a form of application state rather than putting everything into `useState`. That concept becomes surprisingly important once you start building real applications.