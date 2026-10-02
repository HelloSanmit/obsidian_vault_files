
# Lesson 35 — SSR and Hydration: How React Gets HTML From a Server Into an Interactive Browser

Until now, we've mostly assumed React starts here:

```text
Browser
   ↓
JavaScript loads
   ↓
React renders
   ↓
DOM appears
```

That's the **client-side rendering** model.

Modern React can also start here:

```text
Server
   ↓
React renders HTML
   ↓
Browser receives HTML
   ↓
User can see the page
   ↓
React JavaScript loads
   ↓
React hydrates the HTML
   ↓
Page becomes interactive
```

That second model is **server-side rendering (SSR)**.

And understanding it is extremely important because it changes how you think about:

- rendering
    
- `useEffect`
    
- browser APIs
    
- `useId`
    
- Suspense
    
- data fetching
    
- hydration mismatches
    
- Server Components
    
- streaming
    
- framework architecture
    

Current React 19.3 continues to use `hydrateRoot` on the client and streaming server APIs such as `renderToPipeableStream` for Node.js environments. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

---

# 1. First: CSR vs SSR

Let's make the difference concrete.

## Client-side rendering

Imagine your server responds with approximately:

```html
<div id="root"></div>
<script src="/app.js"></script>
```

The browser gets almost no application HTML initially.

Then JavaScript loads:

```text
app.js
  ↓
React
  ↓
<App />
  ↓
render
  ↓
DOM
```

So the user might initially see:

```text
[blank]
```

then:

```text
Loading...
```

then:

```text
Welcome, John
```

---

## Server-side rendering

The server runs React first.

For example:

```jsx
function App() {
  return (
    <main>
      <h1>Welcome, John</h1>
      <button>Like</button>
    </main>
  );
}
```

The server turns that React tree into HTML:

```html
<main>
  <h1>Welcome, John</h1>
  <button>Like</button>
</main>
```

The browser can display that HTML immediately.

But there is a problem.

The HTML is currently just HTML.

The button isn't automatically a React button.

---

# 2. HTML is visible, but not yet React-interactive

This distinction is crucial.

Suppose the server sends:

```html
<button>Like</button>
```

The browser can display it.

But where is this?

```jsx
onClick={handleLike}
```

It isn't in the HTML as a working JavaScript event handler.

The browser only knows:

```text
<button>
```

It doesn't know:

```text
React component ownership
state
hooks
event handlers
effects
transitions
```

So after the HTML arrives, React needs to connect its component tree to that existing DOM.

That's **hydration**.

---

# 3. What hydration actually means

Think of hydration as:

> "React is taking server-produced HTML and attaching the client-side React behavior to it."

The basic client code is:

```jsx
import { hydrateRoot } from "react-dom/client";

hydrateRoot(
  document,
  <App />
);
```

Current React uses `hydrateRoot` for hydrating server-rendered HTML; the old `hydrate` API was removed in React 19. ([React](https://react.dev/reference/react-dom?utm_source=chatgpt.com "React DOM APIs – React"))

So instead of:

```text
HTML exists
   ↓
throw it away
   ↓
create everything again
```

React tries to work with the existing HTML.

---

# 4. The complete SSR lifecycle

Let's look at the whole thing.

```text
                 SERVER
                   │
                   ▼
              React render
                   │
                   ▼
                 HTML
                   │
                   │ HTTP response
                   ▼
                BROWSER
                   │
                   ▼
             HTML parsed
                   │
                   ▼
              page visible
                   │
                   ▼
            JavaScript loads
                   │
                   ▼
             hydrateRoot()
                   │
                   ▼
           React connects to DOM
                   │
                   ▼
            page interactive
```

This means **initial rendering happens twice in two environments**:

```text
Server render
+
Client render during hydration
```

That is the source of many SSR bugs.

---

# 5. The server and client must agree

Suppose the server renders:

```jsx
<h1>Hello John</h1>
```

but when the browser hydrates, the client renders:

```jsx
<h1>Hello Sarah</h1>
```

React now sees:

```text
SERVER HTML:
Hello John

CLIENT INITIAL RENDER:
Hello Sarah
```

The trees don't match.

That's a **hydration mismatch**.

The underlying principle is:

```text
server initial output
        ≈
client initial output
```

They need to correspond so React can hydrate reliably. React explicitly describes components in SSR as needing matching initial HTML between server and client. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

---

# 6. Why does this happen?

The most common reason is using browser-specific information during render.

For example:

```jsx
function App() {
  return (
    <p>
      {window.innerWidth}
    </p>
  );
}
```

On the server:

```text
window
```

doesn't exist.

There is no browser there.

Even when code is made to avoid crashing, the server and browser may produce different values.

Another example:

```jsx
function Clock() {
  return (
    <p>
      {new Date().toLocaleTimeString()}
    </p>
  );
}
```

Suppose server render happens at:

```text
10:00:00
```

and hydration occurs at:

```text
10:00:01
```

Then:

```text
SERVER:
10:00:00

CLIENT:
10:00:01
```

Mismatch.

---

# 7. Why `typeof window !== "undefined"` isn't a complete solution

You may see:

```jsx
function Component() {
  const isBrowser =
    typeof window !== "undefined";

  return (
    <p>
      {isBrowser
        ? window.innerWidth
        : "Loading..."}
    </p>
  );
}
```

This prevents the server from directly accessing `window`.

But the initial output is now deliberately different:

```text
SERVER:
Loading...

CLIENT:
1024
```

You're still dealing with a server/client rendering difference.

Sometimes that's intentional and manageable, but it must be handled as part of your SSR design.

---

# 8. The role of `useEffect`

You already learned:

> `useEffect` synchronizes with external systems after rendering.

That becomes particularly useful with SSR.

For example:

```jsx
function Width() {
  const [width, setWidth] = useState(null);

  useEffect(() => {
    setWidth(window.innerWidth);
  }, []);

  return (
    <p>
      {width ?? "Loading..."}
    </p>
  );
}
```

Server:

```text
Loading...
```

Initial client render:

```text
Loading...
```

Hydration matches.

Then after hydration:

```text
useEffect()
   ↓
window.innerWidth
   ↓
setWidth(...)
   ↓
render again
```

Now:

```text
1024
```

This produces:

```text
SERVER
Loading...

CLIENT HYDRATION
Loading...

AFTER EFFECT
1024
```

No initial mismatch.

---

# 9. But don't turn this into a superstition

You may now be tempted to think:

```text
browser API?
→ useEffect
```

Not always.

The real question is:

> Does this value need to exist during the server's initial render?

For something like:

```text
document.title
window.addEventListener
localStorage
resize observer
```

these are naturally browser-side effects.

But for application content, blindly pushing everything into `useEffect` can make SSR less useful.

For example, imagine:

```jsx
useEffect(() => {
  fetch("/api/products")
    .then(...)
}, []);
```

That means:

```text
server:
HTML doesn't contain products

browser:
JavaScript loads

effect:
request starts

response:
products appear
```

You have effectively delayed the content until after hydration.

That's often exactly what SSR was supposed to help avoid.

---

# 10. SSR isn't just "render React on server"

This is where the topic gets more interesting.

There are several different concepts:

```text
SSR
SSG
Streaming SSR
Prerendering
Partial prerendering
Hydration
Server Components
```

Don't mix them together.

---

# 11. SSR

Traditional SSR means:

```text
request
  ↓
server executes React
  ↓
HTML generated
  ↓
HTML sent to browser
```

For every request, the server can potentially render fresh content.

For example:

```text
GET /profile/123
        ↓
server
        ↓
database
        ↓
React
        ↓
HTML
```

---

# 12. Static generation

Static generation is different.

Instead of:

```text
request
 ↓
render React
```

you might do:

```text
build time
   ↓
render React
   ↓
HTML file
```

Then:

```text
request
   ↓
CDN/static server
   ↓
existing HTML
```

No React render needs to happen for every request.

React 19 added `prerender` and `prerenderToNodeStream` for generating static HTML while waiting for data needed to complete the prerender. ([React](https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

So:

```text
SSR
request-time rendering

SSG/prerendering
build-time or pre-request rendering
```

---

# 13. Streaming SSR

Here's where React gets really powerful.

Suppose the page contains:

```text
Header
Product details
Recommendations
Reviews
Personalized ads
```

Imagine reviews take 1.5 seconds to load.

Naive SSR:

```text
wait for EVERYTHING
       ↓
generate entire HTML
       ↓
send response
```

The browser waits.

Streaming changes the model.

React can send HTML progressively:

```text
Header
Product details
...
```

then later:

```text
Reviews
...
```

React's streaming server APIs are designed to send the rendered result incrementally rather than waiting for the entire tree. For Node.js, the current API is `renderToPipeableStream`. ([React](https://react.dev/reference/react-dom/server?utm_source=chatgpt.com "Server React DOM APIs – React"))

---

# 14. Suspense is the bridge

Remember our previous lesson:

```jsx
<Suspense fallback={<Spinner />}>
  <SlowComponent />
</Suspense>
```

On the client, we discussed Suspense as a way to say:

```text
"I'm not ready yet."
```

On the server, Suspense can become a streaming boundary.

Imagine:

```jsx
<App>
  <Header />

  <Suspense fallback={<ReviewsSkeleton />}>
    <Reviews />
  </Suspense>
</App>
```

React can conceptually send:

```html
<header>...</header>

<section>
  ReviewsSkeleton
</section>
```

initially.

Then later send the completed Reviews content.

So Suspense isn't just:

```text
loading spinner
```

It can define **where progressively revealed UI is allowed to appear**.

---

# 15. Streaming lifecycle

Conceptually:

```text
request
   ↓
React begins rendering
   ↓
Shell becomes ready
   ↓
send HTML
   ↓
browser displays shell
   ↓
slow boundary completes
   ↓
stream more HTML
   ↓
browser inserts completed content
```

This is significantly different from:

```text
render entire tree
   ↓
send one giant HTML response
```

The current Node streaming API exposes callbacks such as `onShellReady` so the server can start sending the shell when the initial shell is ready. ([React](https://react.dev/reference/react-dom/server/renderToPipeableStream?utm_source=chatgpt.com "renderToPipeableStream – React"))

---

# 16. What exactly is "the shell"?

Suppose:

```jsx
<App>
  <Header />

  <Suspense fallback="Loading products...">
    <Products />
  </Suspense>

  <Footer />
</App>
```

The shell is roughly the part React can render without waiting for the suspended content.

Conceptually:

```text
┌──────────────────────────┐
│ Header                   │
│                          │
│ Loading products...      │ ← fallback
│                          │
│ Footer                   │
└──────────────────────────┘
```

That can be sent early.

Then Products can arrive later.

This gives you a strong user-perceived loading structure.

---

# 17. Hydration after streaming

Now think about the browser.

It receives:

```text
HTML chunk 1
HTML chunk 2
HTML chunk 3
```

and starts displaying the page.

Then React loads.

Eventually:

```js
hydrateRoot(document, <App />);
```

React connects its component model and behavior to the server-generated HTML.

So:

```text
Server
──────
render
stream
HTML
 │
 ▼
Browser
───────
display
 │
 ▼
hydrate
 │
 ▼
interactive
```

This is why SSR doesn't eliminate client JavaScript automatically.

SSR can make the **initial page visible faster**, while hydration makes it **interactive**.

---

# 18. A very common misconception

People sometimes say:

> "SSR means React runs only on the server."

No.

Traditional SSR + hydration means React runs in both places.

```text
SERVER
React render
   ↓
HTML

CLIENT
React render
   ↓
hydration
```

Both environments matter.

And that's why render code must be carefully written.

---

# 19. What does not happen during server rendering?

This is extremely important for your previous knowledge of hooks.

Your render function runs:

```jsx
function Profile() {
  const [name, setName] = useState("John");

  useEffect(() => {
    console.log("connected");
  }, []);

  return <h1>{name}</h1>;
}
```

The server needs the result of:

```jsx
<h1>{name}</h1>
```

But there is no browser environment for the effect.

`useEffect` is a client-side synchronization mechanism; you should not think of it as part of producing server HTML.

So mentally:

```text
SERVER
render component
   ↓
HTML

CLIENT
render component
   ↓
hydrate
   ↓
effects run
```

This is one reason `useEffect` is not an SSR data-fetching mechanism.

---

# 20. Why server rendering makes render purity even more important

Remember the rule:

> Rendering should be pure.

SSR reinforces this.

Bad:

```jsx
function Product() {
  globalCounter++;

  return <p>{globalCounter}</p>;
}
```

Why is this problematic?

Because now:

```text
Server render
```

changes external state.

Then:

```text
Client render
```

runs separately.

You can get different output depending on execution order and environment.

Instead:

```jsx
function Product({ count }) {
  return <p>{count}</p>;
}
```

Now:

```text
same input
   ↓
same output
```

which is exactly what SSR/hydration needs.

---

# 21. SSR and `localStorage`

A classic example:

```jsx
function Theme() {
  const [theme] = useState(
    localStorage.getItem("theme")
  );

  return <p>{theme}</p>;
}
```

This crashes on the server because:

```text
localStorage
```

doesn't exist there.

You might move it:

```jsx
function Theme() {
  const [theme, setTheme] = useState("light");

  useEffect(() => {
    setTheme(
      localStorage.getItem("theme") ?? "light"
    );
  }, []);

  return <p>{theme}</p>;
}
```

Now it can safely render:

```text
SERVER → light
CLIENT → light
EFFECT → actual preference
```

That's a valid strategy.

But React 19.3 also introduced a `browser()` API that can explicitly mark a component as browser-only during server rendering. `use(browser())` causes a Suspense boundary to render its fallback on the server, then allows the component to render normally after hydration. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

For example:

```jsx
import { use } from "react";
import { browser } from "react-dom";

function LocalTimeZone() {
  use(browser());

  const timezone =
    new Intl.DateTimeFormat()
      .resolvedOptions()
      .timeZone;

  return <p>{timezone}</p>;
}
```

wrapped in:

```jsx
<Suspense fallback="Loading timezone...">
  <LocalTimeZone />
</Suspense>
```

Conceptually:

```text
SERVER
LocalTimeZone
   ↓
browser() says:
"wait until browser"
   ↓
Suspense fallback

CLIENT
hydration
   ↓
browser() resolves
   ↓
LocalTimeZone renders
```

This is a newer React 19.3 capability and is particularly useful for genuinely browser-only components. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

---

# 22. Why hydration bugs are so annoying

Suppose your component is:

```jsx
function Greeting() {
  const hour = new Date().getHours();

  return (
    <h1>
      {hour < 12 ? "Good morning" : "Good evening"}
    </h1>
  );
}
```

You don't immediately see the problem.

But imagine:

```text
Server timezone → UTC
Browser timezone → IST
```

or even just a request crossing midnight.

Server:

```text
Good morning
```

Client:

```text
Good evening
```

Now the initial trees disagree.

This is why environment-dependent values are particularly dangerous in SSR.

---

# 23. The `useId` issue

You learned `useId` earlier for accessibility.

It becomes even more important with SSR.

Suppose you have:

```jsx
function EmailField() {
  const id = useId();

  return (
    <>
      <label htmlFor={id}>
        Email
      </label>

      <input id={id} />
    </>
  );
}
```

You want the server and client to generate corresponding IDs.

React specifically designed `useId` to work across server rendering and hydration.

That's much safer than manually generating IDs with:

```js
Math.random()
```

because:

```text
Server random value
       ≠
Client random value
```

which can produce mismatches.

---

# 24. `Math.random()` in render

This is bad even without SSR:

```jsx
function Example() {
  return <p>{Math.random()}</p>;
}
```

With SSR it becomes even more obvious.

Server:

```text
0.412
```

Client:

```text
0.937
```

Different render output.

Similarly:

```js
Date.now()
crypto.randomUUID()
window.innerWidth
localStorage.getItem(...)
navigator.language
```

can introduce environment-dependent rendering.

The rule is not:

> "Never use these APIs."

The rule is:

> Don't let nondeterministic or browser-only values accidentally determine the server/client initial render.

---

# 25. Hydration is not an event listener attachment exercise

It's tempting to describe hydration as:

```text
"React adds event listeners."
```

That's incomplete.

React is reconstructing/associating its component tree with the existing DOM and taking ownership of updates.

For example:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      {count}
    </button>
  );
}
```

Server HTML:

```html
<button>0</button>
```

After hydration, React knows:

```text
Counter component
   ↓
state = 0
   ↓
button
   ↓
click handler
```

Now:

```text
click
 ↓
setCount
 ↓
render
 ↓
commit
 ↓
button becomes 1
```

So hydration reconnects the React application model to existing markup.

---

# 26. SSR doesn't change React's rendering model

Your mental model from Lesson 13 still works.

Normally:

```text
trigger
  ↓
render
  ↓
commit
```

With SSR, we can have:

```text
SERVER
React render
  ↓
HTML generation
```

then on the client:

```text
HYDRATION
initial React render
  ↓
match existing DOM
  ↓
interactive React tree
```

After that, ordinary React behavior resumes:

```text
state update
  ↓
render
  ↓
commit
```

So SSR is an additional **environment and startup phase**, not a completely different React mental model.

---

# 27. Streaming introduces another layer

Now the full architecture can become:

```text
REQUEST
   │
   ▼
SERVER RENDER
   │
   ├── ready content
   │
   └── suspended content
          │
          ▼
        STREAM
          │
          ▼
       BROWSER
          │
          ├── show HTML immediately
          │
          └── receive more HTML later
                    │
                    ▼
                HYDRATION
                    │
                    ▼
               INTERACTIVE
```

That's where:

```text
SSR
+
Suspense
+
streaming
+
hydration
```

start working as one system.

---

# 28. Static prerendering and partial prerendering

Modern React also has another concept.

Suppose your application has:

```text
Header
Navigation
Marketing content
Product recommendations
User-specific account info
```

Most of the page might be known ahead of time.

You could prerender the stable parts and postpone dynamic work.

React's current platform includes `prerender`, and since React 19.2 it supports a partial prerendering model where rendering can be postponed and later resumed with `resume`. ([React](https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

Conceptually:

```text
PRERENDER
   ↓
static shell
   +
postponed state
```

Then later:

```text
REQUEST
   ↓
resume()
   ↓
dynamic content
```

This gets us toward architectures where:

```text
CDN/static content
        +
dynamic server content
        +
client interactivity
```

can coexist.

You don't need to memorize these APIs yet. Understand the direction.

---

# 29. SSR vs SSG vs streaming

Keep this table in your head:

|Technique|Main idea|
|---|---|
|CSR|Browser generates initial UI|
|SSR|Server generates HTML per request|
|SSG/prerender|HTML generated ahead of request|
|Streaming SSR|Server sends HTML progressively|
|Partial prerendering|Pre-generate some work, resume dynamic work later|
|Hydration|Client React attaches to server HTML|

These are not mutually exclusive concepts.

For example:

```text
prerendered HTML
+
streamed dynamic content
+
hydration
```

can all be part of one application.

---

# 30. A practical example: product page

Imagine:

```jsx
<ProductPage>
  <Header />
  <ProductDetails />
  <Suspense fallback={<ReviewsSkeleton />}>
    <Reviews />
  </Suspense>
  <Footer />
</ProductPage>
```

Request comes in:

```text
GET /products/42
```

Server:

```text
ProductPage
   │
   ├── Header
   ├── ProductDetails
   ├── Reviews → waiting
   └── Footer
```

React can produce a shell:

```text
Header
Product details
Reviews skeleton
Footer
```

The browser receives it and displays it.

Then:

```text
Reviews data ready
   ↓
more HTML streamed
   ↓
browser reveals Reviews
```

Then JavaScript loads:

```text
hydrateRoot
   ↓
React attaches behavior
```

Now:

```text
Add to cart
Buy
Like
Review form
```

can all become interactive.

---

# 31. Why frameworks exist

You may now understand why real React applications often use a framework.

Building this yourself means handling:

```text
HTTP requests
routing
server rendering
streaming
hydration
data fetching
code splitting
bundling
asset loading
caching
error handling
deployment
```

React provides the rendering primitives.

A framework can coordinate the entire application lifecycle around those primitives.

This becomes especially important when we get to **Server Components**, because then you have another distinction:

```text
server-rendered HTML
vs
server component execution
vs
client component execution
```

Those are related, but they're not the same thing.

That will be one of the next major conceptual jumps.

---

# 32. The most important SSR rules

You don't need to memorize the APIs yet. Remember these principles.

### Rule 1

The server and client's initial render should correspond.

```text
server output
      ≈
client initial output
```

### Rule 2

Browser-only APIs don't exist on the server.

```text
window
document
localStorage
navigator
```

must be handled appropriately.

### Rule 3

`useEffect` happens after client rendering.

So don't think:

```text
useEffect = SSR data fetching
```

### Rule 4

Suspense can define streaming boundaries.

```text
fast content
   ↓
send now

slow content
   ↓
send later
```

### Rule 5

SSR does not automatically make the page interactive.

```text
HTML
  ≠
hydrated React
```

### Rule 6

Hydration errors usually mean your server/client initial outputs disagree.

---

# 33. The entire lesson in one picture

```text
                     REQUEST
                        │
                        ▼
                    SERVER
                        │
                   React render
                        │
            ┌───────────┴───────────┐
            │                       │
       ready content          Suspended content
            │                       │
            └──────────┬────────────┘
                       ▼
                    HTML
                       │
                    streamed
                       │
                       ▼
                    BROWSER
                       │
                 display HTML
                       │
                       ▼
                  JS loads
                       │
                       ▼
                 hydrateRoot()
                       │
                       ▼
              React owns the UI
                       │
                       ▼
          state/events/effects/etc.
```

The key distinction is:

```text
SERVER RENDER
    =
produce HTML

HYDRATION
    =
connect client React to that HTML
```

And:

```text
STREAMING
    =
don't wait for the entire tree before sending HTML
```

That gives you the foundation needed for the next major topic: **React Server Components — what runs on the server, what runs in the browser, what `"use client"` actually means, and how Server Components differ from SSR.**