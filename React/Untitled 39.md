
# Lesson 36 — React Server Components: The Server/Client Boundary

SSR answered this question:

> **Where can React render HTML?**

Server Components answer a different question:

> **Where should a React component's code execute at all?**

That distinction is absolutely fundamental.

A lot of React developers initially think:

```text
SSR
=
Server Components
```

They are **not the same thing**.

You can have:

```text
Server Components + SSR
Server Components without SSR
SSR without Server Components
```

Server Components are an **application architecture and module-boundary model**, while SSR is a way of producing HTML. React's current RSC model has stable component semantics in React 19.x, while the underlying bundler/framework implementation APIs remain framework-dependent. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

---

# 1. Start with ordinary React

Without Server Components, imagine your application like this:

```text
Browser
  │
  ▼
JavaScript bundle
  │
  ├── App
  ├── ProductPage
  ├── Header
  ├── Reviews
  ├── MarkdownRenderer
  ├── Chart
  ├── Editor
  └── ...
```

The browser gets the JavaScript needed to run those components.

Whether the initial HTML was produced by:

```text
CSR
or
SSR
```

is a separate concern.

With Server Components, we introduce another division:

```text
                  React application
                       │
             ┌─────────┴─────────┐
             │                   │
          SERVER               CLIENT
             │                   │
      Server Components    Client Components
```

Now some component code **never needs to be shipped to the browser as component code**.

That is the major idea.

---

# 2. Server Component

A Server Component is a component whose usage runs in the server environment.

For example:

```jsx
async function ProductPage() {
  const product = await db.products.get(42);

  return (
    <main>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
    </main>
  );
}
```

Notice something unusual:

```js
async function ProductPage()
```

and:

```js
await db.products.get(42)
```

happen directly during component rendering.

Server Components can do this because the rendering is happening in a server environment where server-only resources can be accessed. React's current Server Components model supports asynchronous rendering and server-side data access. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

---

# 3. There is no `"use server"` for Server Components

This is one of the most important facts.

You might incorrectly imagine:

```jsx
"use server";

function ProductPage() {
  ...
}
```

means:

> "This is a Server Component."

It does not.

There is **no directive that marks a component as a Server Component**.

Server Components are the default in an RSC environment unless the module is marked for the client. `"use server"` is for **Server Functions**, which we'll cover separately. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

So:

```jsx
function ProductPage() {
  ...
}
```

can be a Server Component.

While:

```jsx
"use client";

function ProductEditor() {
  ...
}
```

defines a client boundary.

---

# 4. `"use client"` is a module boundary

This is where people often form the wrong mental model.

Suppose:

```jsx
// ProductEditor.jsx

"use client";

import { useState } from "react";

export default function ProductEditor() {
  const [quantity, setQuantity] = useState(1);

  return (
    <button onClick={() => setQuantity(q => q + 1)}>
      Quantity: {quantity}
    </button>
  );
}
```

The important part isn't simply:

```text
"This component runs in browser."
```

The deeper rule is:

> `"use client"` marks a boundary in the **module dependency graph**.

Once you import:

```js
import ProductEditor from "./ProductEditor";
```

from the server side, that module and its transitive client dependencies become part of the client side. ([React](https://react.dev/reference/rsc/use-client?utm_source=chatgpt.com "'use client' directive – React"))

Think in terms of modules.

```text
ProductPage.jsx
      │
      ├── Header.jsx
      │
      ├── ProductDetails.jsx
      │
      └── ProductEditor.jsx
                │
                ├── useState
                ├── Button.jsx
                └── editor-utils.js
```

If `ProductEditor.jsx` says:

```jsx
"use client";
```

the client graph begins there.

Conceptually:

```text
SERVER MODULE GRAPH

ProductPage
 ├── Header
 ├── ProductDetails
 └── ProductEditor ──────┐
                         │ boundary
                         ▼
CLIENT MODULE GRAPH
 ProductEditor
 ├── Button
 └── editor-utils
```

This is much more precise than thinking only about the visible component hierarchy. ([React](https://react.dev/reference/rsc/use-client?utm_source=chatgpt.com "'use client' directive – React"))

---

# 5. Why "module tree" matters more than "component tree"

Here's a subtle example.

```jsx
// App.jsx

import InteractivePanel from "./InteractivePanel";
import Copyright from "./Copyright";

export default function App() {
  return (
    <>
      <InteractivePanel>
        <Copyright />
      </InteractivePanel>
    </>
  );
}
```

And:

```jsx
// InteractivePanel.jsx

"use client";

export default function InteractivePanel({ children }) {
  // client component
  return <section>{children}</section>;
}
```

And:

```jsx
// Copyright.jsx

export default function Copyright() {
  return <small>© 2026</small>;
}
```

Many people see:

```text
App
 └── InteractivePanel
      └── Copyright
```

and conclude:

```text
Copyright must be a Client Component
```

That isn't necessarily true.

Why?

Because the client boundary comes from the **module dependency relationship**, not simply from the fact that one component appears inside another component's rendered output.

`App` imports `Copyright` and constructs that element itself:

```jsx
<Copyright />
```

Then passes the already-created element as:

```jsx
children
```

to the client component.

Conceptually:

```text
App (server)
   │
   ├── creates <Copyright />
   │
   ▼
children
   │
   ▼
InteractivePanel (client)
```

So the `Copyright` usage can remain server-rendered.

React explicitly calls out this distinction: the parent/child relationship in the render tree does not by itself determine the server/client environment. ([React](https://react.dev/reference/rsc/use-client?utm_source=chatgpt.com "'use client' directive – React"))

This is one of the deepest RSC concepts.

---

# 6. Think "module graph", not "DOM tree"

Before:

```text
React component tree
```

was your primary mental model.

With Server Components, you now need **two trees**.

## Render tree

```text
App
 └── ProductPage
      ├── Header
      ├── ProductDetails
      └── ProductEditor
```

## Module dependency graph

```text
ProductPage.js
   │
   ├── Header.js
   ├── ProductDetails.js
   └── ProductEditor.js
          │
          └── editor-utils.js
```

`"use client"` operates on the **module graph**.

That's why a component's rendering environment can change depending on where it is imported from.

---

# 7. What can a Server Component do?

Because it runs on the server, it can do things such as:

```js
await database.query(...)
```

or:

```js
await filesystem.readFile(...)
```

or:

```js
await fetch(...)
```

against server resources.

For example:

```jsx
async function NotesPage() {
  const notes = await db.notes.getAll();

  return (
    <ul>
      {notes.map(note => (
        <li key={note.id}>
          {note.title}
        </li>
      ))}
    </ul>
  );
}
```

Notice what we didn't need:

```text
useEffect
useState
loading state
fetch inside effect
extra client request after mount
```

The data is obtained as part of server rendering.

That can significantly change application architecture.

---

# 8. What a Server Component cannot do

Server Components aren't interactive browser components.

They cannot use ordinary client interaction APIs such as:

```js
useState
useReducer
useEffect
useRef
```

for client-side behavior.

They also cannot directly depend on browser-only APIs such as:

```js
window
document
localStorage
navigator
```

because they execute outside the browser.

To make something interactive, you cross into a Client Component. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

---

# 9. The classic architecture

Imagine a product page.

You could have:

```jsx
async function ProductPage({ id }) {
  const product = await db.products.get(id);

  return (
    <main>
      <ProductDetails product={product} />
      <AddToCart productId={product.id} />
    </main>
  );
}
```

and:

```jsx
"use client";

import { useState } from "react";

function AddToCart({ productId }) {
  const [adding, setAdding] = useState(false);

  async function handleClick() {
    setAdding(true);

    await addToCart(productId);

    setAdding(false);
  }

  return (
    <button
      onClick={handleClick}
      disabled={adding}
    >
      {adding ? "Adding..." : "Add to cart"}
    </button>
  );
}
```

Now:

```text
                ProductPage
              Server Component
                    │
            database access
                    │
                    ▼
             ProductDetails
             Server Component

                    │
                    ▼
               AddToCart
             Client Component
```

This is extremely powerful.

The majority of the page can stay server-side.

Only the interactive island needs client JavaScript.

---

# 10. "Client Component" does NOT mean "server never sees it"

This is another common misconception.

Suppose:

```jsx
"use client";

function AddToCart() {
  return <button>Add to cart</button>;
}
```

It doesn't mean:

```text
server ignores it completely
```

A framework can still include its rendered output in the initial page generation.

The important distinction is:

```text
Server Component
→ its component code is not shipped to the browser as client component code

Client Component
→ its code is available to the browser so it can become interactive
```

So `"use client"` primarily determines **where the client-side code boundary exists**, not simply whether some HTML can be produced on the server. ([React](https://react.dev/reference/rsc/use-client?utm_source=chatgpt.com "'use client' directive – React"))

---

# 11. Server Components reduce client JavaScript

Imagine a page with 50 components.

Without a server/client split:

```text
Browser JavaScript
├── 50 components
├── database-related utilities
├── markdown parsing
├── formatting libraries
├── internal helpers
└── application code
```

With Server Components:

```text
Server
├── 35 server components
├── server-only libraries
└── database logic

Browser
├── 15 client components
└── their client dependencies
```

The browser doesn't need to execute all of the server-side component logic.

This is one of the major motivations behind RSC.

---

# 12. This does not mean "more components should become client"

In fact, generally the opposite.

Suppose:

```jsx
"use client";

import HugeMarkdownParser from "huge-markdown-parser";
import { useState } from "react";

function Article() {
  ...
}
```

Because `Article` is client code, its dependency graph may pull that parser into client code too.

But if article rendering doesn't need client interactivity:

```jsx
async function Article() {
  const content = await getArticle();

  return <RenderedArticle content={content} />;
}
```

the heavy server-only processing can remain on the server.

That gives you:

```text
less client JavaScript
+
server-only dependencies
+
direct server data access
```

---

# 13. Server Components and data fetching

This creates a major architectural difference from the `useEffect` pattern you learned earlier.

Old pattern:

```jsx
function ProductPage() {
  const [product, setProduct] = useState(null);

  useEffect(() => {
    fetch("/api/products/42")
      .then(res => res.json())
      .then(setProduct);
  }, []);

  if (!product) {
    return <Spinner />;
  }

  return <Product product={product} />;
}
```

Lifecycle:

```text
browser loads
   ↓
React renders
   ↓
loading UI
   ↓
effect runs
   ↓
request
   ↓
response
   ↓
state update
   ↓
render product
```

Server Component version:

```jsx
async function ProductPage() {
  const product =
    await db.products.get(42);

  return <Product product={product} />;
}
```

Lifecycle:

```text
request
   ↓
server component starts
   ↓
database request
   ↓
component resolves
   ↓
React produces result
   ↓
browser receives result
```

The request doesn't have to originate from browser JavaScript.

That's a fundamental shift.

---

# 14. "Then why do I need TanStack Query?"

Excellent question.

Because they solve different problems.

TanStack Query is excellent for:

```text
client-side server state
caching
refetching
mutations
background synchronization
pagination
stale data
invalidation
```

Server Components are excellent for:

```text
server-side rendering/data access
keeping server-only logic on server
reducing client JavaScript
composing server and interactive client UI
```

You can combine them.

For example:

```text
Server Component
    ↓
fetch initial product data
    ↓
Client Component
    ↓
TanStack Query
    ↓
interactive updates / refetching
```

So don't think:

```text
RSC replaced TanStack Query.
```

The architecture is more nuanced than that.

---

# 15. The boundary requires serializable data

Now we hit an important constraint.

Imagine:

```jsx
<ProductEditor
  product={product}
  onSave={saveProduct}
/>
```

where `ProductEditor` is a Client Component.

What crosses the server → client boundary?

The data:

```text
product
```

needs to be representable in the protocol connecting the two environments.

But this:

```js
onSave
```

is a normal JavaScript function living on the server.

You can't simply serialize an arbitrary function and send its implementation to the browser.

So ordinary functions are not generally valid props across this boundary.

Instead, you use mechanisms specifically designed for server/client communication, such as Server Functions, which we'll reach shortly.

---

# 16. Think of the boundary as a network boundary

This is a fantastic mental model.

Even if the server and browser are part of one application, conceptually imagine:

```text
SERVER
   │
   │ data / references
   ▼
──────────── boundary ────────────
   ▲
   │
CLIENT
```

That means:

```text
Server
↓
send data

Client
↓
receive data
```

You cannot behave as though:

```text
server function === local browser function
```

The boundary behaves much more like an RPC/network boundary than like an ordinary function call.

That explains many RSC rules.

---

# 17. Why this is different from ordinary props

Normally:

```jsx
function Parent() {
  const handleSave = () => {};

  return <Child onSave={handleSave} />;
}
```

Everything lives inside one JavaScript runtime.

So passing:

```js
handleSave
```

is trivial.

With RSC:

```text
server runtime
     ↓
 client runtime
```

those aren't the same runtime.

You need a representation the receiving side understands.

That's why Server Components encourage you to think:

```text
data crosses boundary
code stays where it belongs
```

---

# 18. The server/client composition pattern

A very common structure is:

```jsx
// Server Component
async function Dashboard() {
  const user = await getUser();
  const stats = await getStats(user.id);

  return (
    <>
      <DashboardHeader user={user} />
      <InteractiveChart stats={stats} />
    </>
  );
}
```

Then:

```jsx
// Client Component
"use client";

import { useState } from "react";

function InteractiveChart({ stats }) {
  const [range, setRange] = useState("7d");

  return (
    <>
      <select
        value={range}
        onChange={e => setRange(e.target.value)}
      >
        <option value="7d">7 days</option>
        <option value="30d">30 days</option>
      </select>

      <Chart
        stats={stats}
        range={range}
      />
    </>
  );
}
```

The architecture is:

```text
SERVER
 └── Dashboard
      ├── fetch user
      ├── fetch stats
      └── InteractiveChart
              │
              ▼
           CLIENT
            ├── state
            ├── events
            └── browser interaction
```

This is often what people mean by an **interactive island**.

---

# 19. Server Components can contain Client Components

Absolutely.

```jsx
async function Page() {
  const products = await db.products.getAll();

  return (
    <main>
      <h1>Products</h1>

      <ProductList
        products={products}
      />
    </main>
  );
}
```

where:

```jsx
"use client";

function ProductList({ products }) {
  ...
}
```

The server component can render the client component as part of the overall tree.

The important part is the boundary:

```text
SERVER
Page
  ↓
CLIENT
ProductList
```

---

# 20. Client Components can also receive Server Components as `children`

This is the subtle pattern from earlier.

Suppose:

```jsx
// Client
"use client";

function Modal({ children }) {
  const [open, setOpen] = useState(false);

  return ...
}
```

Then:

```jsx
// Server
async function Page() {
  const post = await getPost();

  return (
    <Modal>
      <PostContent post={post} />
    </Modal>
  );
}
```

`PostContent` can remain a Server Component because the server created that JSX before passing it into the Client Component.

Conceptually:

```text
Server:
   create <PostContent />

          ↓

   pass JSX as children

          ↓

Client:
   Modal renders children
```

This lets a Client Component provide interactivity **around** server-rendered content without forcing all of that content into the client bundle. React specifically documents this composition pattern. ([React](https://react.dev/reference/rsc/use-client?utm_source=chatgpt.com "'use client' directive – React"))

---

# 21. This gives us "shell around server content"

For example:

```jsx
"use client";

function Expandable({ children }) {
  const [expanded, setExpanded] = useState(false);

  return (
    <section>
      <button
        onClick={() => setExpanded(x => !x)}
      >
        Toggle
      </button>

      {expanded && children}
    </section>
  );
}
```

Server:

```jsx
async function Notes() {
  const notes = await db.notes.getAll();

  return notes.map(note => (
    <Expandable key={note.id}>
      <article>
        {note.content}
      </article>
    </Expandable>
  ));
}
```

Now:

```text
database work
   ↓
server

interaction
   ↓
client

article content
   ↓
server
```

That's a surprisingly powerful composition pattern.

---

# 22. Server Components vs SSR — the critical distinction

Let's put them side by side.

### SSR

Question:

> How do I produce HTML for the browser?

```text
React
 ↓
HTML
```

### Server Components

Question:

> Which component code should execute in the server environment rather than in the client environment?

```text
Server Component
 ↓
server execution

Client Component
 ↓
browser execution
```

They can coexist:

```text
Server Components
       ↓
React output
       ↓
SSR
       ↓
HTML
```

So:

```text
RSC ≠ SSR
```

but:

```text
RSC can participate in SSR.
```

React's current RSC model can also operate without requiring a traditional always-running SSR server, including build-time execution in some setups. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

---

# 23. Server Components vs `"use server"`

Now another distinction:

```text
Server Component
```

and:

```text
"use server"
```

are not the same concept.

### Server Component

```jsx
async function Page() {
  const user = await getUser();

  return <Profile user={user} />;
}
```

This is a component that executes on the server.

### Server Function

```js
async function saveProfile(formData) {
  "use server";

  await db.users.update(...);
}
```

This is a server-side function designed to be invoked remotely from client-capable code.

React calls these **Server Functions**. When used from a Client Component, the framework creates a reference that causes a request to the server when invoked. ([React](https://react.dev/reference/rsc/server-functions?utm_source=chatgpt.com "Server Functions – React"))

We'll make this distinction extremely concrete in the next lesson.

---

# 24. Why Server Components are interesting for security

There's an architectural advantage here.

Suppose you have:

```js
import db from "./database";
import stripe from "./stripe";
```

You don't want:

```text
database credentials
Stripe secret key
private server logic
```

to end up in browser code.

A properly server-side component can execute:

```js
const user =
  await db.users.findById(id);
```

on the server without shipping the database library and secret credentials to the browser as part of that component's client code.

This is an architectural boundary, not just a rendering optimization.

But don't confuse that with automatic security of the entire application. Authorization still needs to be enforced on the server for every privileged operation.

---

# 25. Server Component data access can be colocated

This is another interesting improvement.

Old architecture often encourages:

```text
Component
   ↓
API route
   ↓
service
   ↓
database
```

For a Server Component, you can conceptually do:

```jsx
async function Profile() {
  const user = await db.users.find(...);

  return <h1>{user.name}</h1>;
}
```

The data requirement is now near the component that needs it.

React's Server Components model explicitly supports asynchronous components that fetch data during render. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

That can simplify some application designs considerably.

---

# 26. Parallel data fetching

You should immediately connect this to your previous lesson on dependent queries.

Suppose:

```jsx
async function Dashboard() {
  const userPromise = getUser();
  const statsPromise = getStats();

  const [user, stats] =
    await Promise.all([
      userPromise,
      statsPromise
    ]);

  return (
    <>
      <User user={user} />
      <Stats stats={stats} />
    </>
  );
}
```

The requests can start together.

This is the same principle you already learned with client-side async work:

```text
Don't accidentally create:

A
 ↓
wait
 ↓
B
 ↓
wait
 ↓
C
```

when you really need:

```text
A ──────┐
B ──────┼──→ results
C ──────┘
```

The React environment has changed, but the underlying asynchronous reasoning hasn't.

---

# 27. `use()` becomes especially interesting here

Remember the `use` API from the Suspense lesson.

A Server Component can create a promise:

```jsx
async function Page() {
  const commentsPromise =
    getComments();

  return (
    <Suspense fallback="Loading comments...">
      <Comments
        commentsPromise={commentsPromise}
      />
    </Suspense>
  );
}
```

Then a Client Component can unwrap it:

```jsx
"use client";

import { use } from "react";

function Comments({
  commentsPromise
}) {
  const comments =
    use(commentsPromise);

  return (
    <ul>
      {comments.map(comment => (
        <li key={comment.id}>
          {comment.text}
        </li>
      ))}
    </ul>
  );
}
```

The server starts the work.

The promise crosses the boundary.

The client reads it with `use()` and Suspense handles the waiting.

React documents this exact server-to-client Promise pattern. ([React](https://react.dev/reference/rsc/server-components?utm_source=chatgpt.com "Server Components – React"))

---

# 28. Why not simply send the data instead?

Sometimes you should.

Sometimes you don't want:

```js
await commentsPromise
```

on the server because the content is lower priority.

Instead:

```text
start request early
       ↓
don't block main server render
       ↓
pass Promise
       ↓
client waits where necessary
```

This gives you control over **where the suspension happens**.

It's another place where:

```text
Suspense
+
streaming
+
Server Components
+
use()
```

come together.

---

# 29. The architecture is now getting much richer

Your application is no longer just:

```text
React → browser
```

It can be:

```text
                  SERVER
                    │
           ┌────────┴─────────┐
           │                  │
      Server Components   Server Functions
           │                  │
           │                  │
       database          mutations
       filesystem        privileged work
       server APIs            │
           │                  │
           └────────┬─────────┘
                    │
               boundary
                    │
                    ▼
                  CLIENT
                    │
             Client Components
                    │
            state/events/effects
                    │
             browser APIs
```

And potentially:

```text
                React
                  │
        ┌─────────┼─────────┐
        │         │         │
      Server     HTML     Client
   Components    stream   Components
        │                   │
        │                   │
      DB/API           browser state
```

This is modern React architecture.

---

# 30. A realistic e-commerce example

Let's build the mental architecture.

```jsx
// ProductPage — Server Component

async function ProductPage({ id }) {
  const product =
    await db.products.get(id);

  return (
    <main>
      <ProductInfo product={product} />

      <AddToCart
        productId={product.id}
      />

      <Suspense fallback={<ReviewsSkeleton />}>
        <Reviews productId={product.id} />
      </Suspense>
    </main>
  );
}
```

Now:

```text
ProductPage
     │
     ├── ProductInfo
     │      ↓
     │    SERVER
     │
     ├── AddToCart
     │      ↓
     │    CLIENT
     │
     └── Reviews
            ↓
          SERVER
```

The page can therefore use:

```text
SERVER
├── database access
├── product rendering
├── review retrieval
│
CLIENT
├── quantity controls
├── buttons
├── transitions
└── local interaction
```

Only the necessary interactive behavior crosses into the client.

---

# 31. What happens to the browser's JavaScript bundle?

This is where the architecture has a major payoff.

Imagine:

```text
ProductPage
 ├── expensive markdown parser
 ├── database utilities
 ├── review formatting
 ├── product formatting
 └── AddToCart
```

If all of that were client-side code, the browser might receive much more JavaScript.

With Server Components:

```text
Server-only code
├── database
├── markdown parser
├── review processing
└── product processing

Client code
└── AddToCart
```

The browser needs only the client part.

That's one reason RSC can improve the JavaScript footprint of an application.

---

# 32. A rule worth memorizing

When deciding whether something should be client-side, ask:

> **Does this component need browser interactivity or browser-only capabilities?**

For example:

```text
useState?
→ client

onClick?
→ client

window?
→ client

localStorage?
→ client

useEffect?
→ client

database access?
→ server

filesystem?
→ server

secret API key?
→ server

static transformation?
→ server can be appropriate
```

There are exceptions and optimization reasons, but this is an excellent first mental filter.

---

# 33. Another important rule: don't put `"use client"` everywhere

Bad architecture:

```jsx
"use client";

export default function EntireApplication() {
  ...
}
```

Now you've potentially made a huge section of your module graph client-side.

Better:

```text
Server
   │
   ├── Layout
   ├── ProductPage
   ├── ProductDetails
   └── ReviewList
             │
             ▼
          Client
             │
        ReviewForm
             │
             └── interactive bits
```

Push the client boundary **toward the leaves** when practical.

That can preserve more server-side execution and reduce client JavaScript.

---

# 34. But don't obsess over "smallest possible client boundary"

Architecture should serve behavior.

Suppose three components always operate together:

```text
RichEditor
Toolbar
PreviewControls
```

and all require:

```text
state
keyboard events
DOM interaction
```

Creating tiny client boundaries everywhere may make the architecture harder to reason about.

The real goal is:

```text
Put server work on the server.
Put browser-dependent interaction on the client.
Cross the boundary deliberately.
```

Not:

```text
Minimize number of client components at all costs.
```

---

# 35. Where hydration fits now

Now connect this lesson with Lesson 35.

You might have:

```text
Server Components
       ↓
produce React output
       ↓
SSR / streaming
       ↓
HTML sent to browser
       ↓
client components hydrate
```

So the two ideas are layered:

```text
RSC
"What runs where?"

SSR
"How do we produce initial HTML?"

Hydration
"How does client React connect to the HTML?"
```

That's the clean separation.

---

# 36. Current React 19.3 adds an interesting Context capability

This connects directly to our previous Context lesson.

In React 19.3, a Server Component can render a Context imported from a `"use client"` module directly:

```jsx
<UserContext value={user}>
  {children}
</UserContext>
```

without needing a separate client-only Provider wrapper component. Client Components below it can read the value. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

So the architecture can be:

```text
Server Component
      │
      ├── fetch current user
      │
      ▼
<UserContext value={user}>
      │
      ▼
Client Component
      │
      └── useContext(UserContext)
```

That is a useful example of server data feeding client UI.

---

# 37. The three boundaries you now need to distinguish

At this stage, React has three different conceptual boundaries.

## Render boundary

```text
component render
   ↓
render / commit
```

Your standard React mental model.

## Server/client boundary

```text
Server Component
      │
      ▼
Client Component
```

RSC.

## Browser/React interaction boundary

```text
HTML
 ↓
hydration
 ↓
interactive React
```

SSR + hydration.

These are related but different.

---

# 38. The grand mental model

Put everything together:

```text
                         REQUEST
                            │
                            ▼
                         SERVER
                            │
                   Server Components
                            │
              ┌─────────────┼─────────────┐
              │             │             │
             DB          filesystem    server APIs
              │             │             │
              └─────────────┼─────────────┘
                            │
                       React output
                            │
                     Suspense boundaries
                            │
                     streaming / SSR
                            │
                            ▼
                          HTML
                            │
                            ▼
                         BROWSER
                            │
                     hydrate client code
                            │
                 ┌──────────┴──────────┐
                 │                     │
          Client Components      browser APIs
                 │                     │
          useState/events         DOM/storage
                 │
                 ▼
             Interactive UI
```

And alongside it:

```text
Server Function
     ↑
     │ request
     │
Client Component
```

for mutations or other server-side operations.

---

# 39. What you've just unlocked

Before Server Components, you primarily thought:

```text
React component
+
state
+
effects
```

Now your architectural question becomes:

```text
Where should this logic execute?
```

For each piece of functionality:

```text
Can it run on server?
Should it run on server?
Does it need browser APIs?
Does it need interactivity?
What data crosses the boundary?
What code needs to ship to the browser?
```

That is a much more senior-level React question.

---

# 40. One final example

Suppose we're building:

```text
Social post page
```

We might architect it as:

```text
SERVER
│
├── PostPage
│     └── fetch post
│
├── PostContent
│
├── Author
│
├── Comments
│
└── UserContext
      │
      ▼
CLIENT
│
├── LikeButton
│
├── CommentForm
│
├── ShareButton
│
└── CharacterCounter
```

Then:

```text
PostPage
   ↓
server database access

LikeButton
   ↓
useState / useOptimistic

CommentForm
   ↓
Actions / form status

Comments
   ↓
server data + Suspense

UserContext
   ↓
server obtains user
   ↓
client reads user context
```

Now the pieces you learned over the previous 35 lessons aren't isolated APIs anymore.

They're beginning to fit into one architecture.

---

## What to remember from this lesson

```text
Server Component
→ component executes on server

Client Component
→ component belongs to client module graph

"use client"
→ creates a server/client module boundary

"use server"
→ marks a Server Function, not a Server Component

SSR
→ generates HTML

Hydration
→ connects client React to existing HTML

RSC
→ decides where component code executes

Suspense + RSC
→ allows asynchronous server rendering and streaming

Server → Client
→ data/references cross an explicit boundary
```

The next major piece is **Server Functions (`"use server"`)**. That's where the architecture becomes truly interesting: a Client Component can trigger server-side code, how the call crosses the network boundary, how this connects to the Actions lesson, form Actions, `useActionState`, security, mutations, and why `"use server"` is fundamentally different from `"use client"`. ([React](https://react.dev/reference/rsc/server-functions?utm_source=chatgpt.com "Server Functions – React"))