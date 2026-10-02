
# Lesson 27 — `startTransition`, `useTransition`, and `useDeferredValue`

The last lesson introduced Suspense.

Now we're going to answer a very practical question:

> **What should React do when a user action starts some expensive or suspending work, but the user should still be able to interact with the interface immediately?**

This is where **Transitions** come in.

The three pieces we'll understand today are:

```text
startTransition
useTransition
useDeferredValue
```

They are related, but they solve slightly different problems.

---

# 1. First understand the problem

Imagine this application:

```text
Search:
[ react____________ ]

Results:
1. ...
2. ...
3. ...
...
5000 items
```

Every keystroke changes:

```js
query
```

and therefore causes:

```text
query changes
   ↓
render
   ↓
large results tree rerenders
   ↓
browser has a lot of work
```

On a fast machine:

```text
typing → feels instant
```

On a slower device:

```text
typing
 ↓
heavy rendering
 ↓
input feels delayed
```

The user doesn't care if the results take another 100 ms to visually catch up.

They **do** care if the character they just typed appears late.

So we have two kinds of work:

```text
URGENT
→ update the text field

NON-URGENT
→ update the giant result list
```

Transitions give React a way to distinguish them.

---

# 2. What a Transition means

Suppose:

```jsx
const [tab, setTab] = useState("about");
```

and:

```jsx
function selectTab(nextTab) {
  startTransition(() => {
    setTab(nextTab);
  });
}
```

You're telling React:

> "This state update matters, but it doesn't have to block urgent user interactions."

Current React describes Transition updates as **non-blocking** and interruptible. If higher-priority updates such as typing arrive while a Transition render is in progress, React can interrupt that work and handle the urgent update first. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

That's the core idea.

---

# 3. Transition does NOT mean "wait 1 second"

This is one of the first things to get out of your head.

You might imagine:

```text
startTransition
   ↓
wait
   ↓
do update
```

That's not what happens.

React doesn't introduce a fixed delay.

Instead:

```text
update marked as non-urgent
        ↓
React can work on it in the background
        ↓
urgent work can interrupt it
```

If the device is fast:

```text
transition may finish almost immediately
```

If the device is slow:

```text
transition work can yield to urgent interactions
```

That's why this is different from `setTimeout`.

---

# 4. Normal update vs Transition update

Imagine:

```jsx
function App() {
  const [tab, setTab] = useState("home");

  function openSettings() {
    setTab("settings");
  }

  return ...
}
```

Normal update:

```text
click
 ↓
setTab("settings")
 ↓
React handles update normally
```

Transition:

```jsx
function openSettings() {
  startTransition(() => {
    setTab("settings");
  });
}
```

Now:

```text
click
 ↓
mark update as Transition
 ↓
React works on it non-blockingly
 ↓
urgent work gets priority
```

The final state is still:

```text
tab = "settings"
```

The difference is **how React schedules the rendering work**.

---

# 5. Why not make every state update a Transition?

Because some updates are genuinely urgent.

Consider:

```jsx
<input
  value={text}
  onChange={e => setText(e.target.value)}
/>
```

The input itself must remain responsive.

You don't want:

```text
user types "a"
 ↓
input waits
 ↓
Transition
 ↓
eventually "a" appears
```

That's exactly the opposite of what we're trying to accomplish.

Current React explicitly says Transition updates can't be used to control text inputs. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

So a useful pattern is:

```text
urgent state:
text

transition state:
expensive result view
```

---

# 6. `startTransition` is a function

You can import:

```jsx
import { startTransition } from "react";
```

and:

```jsx
startTransition(() => {
  setTab("posts");
});
```

This is useful when you already have access to the state setter.

For example:

```jsx
function navigate(url) {
  startTransition(() => {
    setPage(url);
  });
}
```

You don't need `useTransition` merely to mark an update as a Transition.

---

# 7. Why does `useTransition` exist then?

Because sometimes you also want to know:

> **"Is this Transition currently in progress?"**

That's what `useTransition` gives you.

```jsx
const [isPending, startTransition] =
  useTransition();
```

You get:

```text
isPending
startTransition
```

So:

```text
startTransition
=
mark update as Transition

useTransition
=
mark update as Transition
+
know whether it is pending
```

Current React documents exactly these two return values. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

---

# 8. Example with a tab

```jsx
function Tabs() {
  const [tab, setTab] = useState("about");

  const [isPending, startTransition] =
    useTransition();

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);
    });
  }

  return (
    <>
      <button
        onClick={() => selectTab("about")}
      >
        About
      </button>

      <button
        onClick={() => selectTab("posts")}
      >
        Posts
      </button>

      {isPending && (
        <span>Loading...</span>
      )}

      {tab === "about" && <About />}
      {tab === "posts" && <Posts />}
    </>
  );
}
```

Now:

```text
click Posts
 ↓
startTransition
 ↓
isPending = true
 ↓
React works on Posts
 ↓
Posts becomes ready
 ↓
isPending = false
```

The UI can show a subtle pending indicator without replacing the whole interface with a giant spinner.

---

# 9. Why `isPending` is different from `isLoading`

This is another important distinction.

We've already learned server-state loading:

```text
isPending
isFetching
```

Those belong to:

```text
TanStack Query
```

`useTransition`'s:

```js
isPending
```

means:

> "A React Transition is currently pending."

Not:

> "The server request is definitely pending."

Not:

> "Every async operation in the app is loading."

Not:

> "This query is stale."

Different systems.

So:

```text
TanStack Query isFetching
        ≠
React useTransition isPending
```

They can sometimes overlap, but they describe different things.

---

# 10. A Transition can contain more than one state update

Suppose:

```jsx
startTransition(() => {
  setTab("posts");
  setSelectedPost(null);
  setViewMode("compact");
});
```

All synchronous state updates scheduled inside the callback are marked as part of the Transition. Current React documents this behavior explicitly. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

So conceptually:

```text
Transition
├── state update A
├── state update B
└── state update C
```

That lets React reason about them as one non-urgent UI transition.

---

# 11. What makes a Transition powerful is interruption

Suppose rendering the new page is expensive.

Timeline:

```text
User clicks Posts

        ↓

Transition rendering starts

        ↓

rendering is expensive...

        ↓

User clicks Contact

        ↓

urgent event handled

        ↓

React can abandon/restart the previous Transition work

        ↓

render Contact
```

So the user isn't trapped waiting for React to finish work they no longer care about.

Current React explicitly describes Transition rendering as interruptible. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

That's the key advantage.

---

# 12. This is fundamentally different from debouncing

Suppose search input changes rapidly:

```text
r
re
rea
reac
react
```

Debouncing says:

```text
"Wait until the user stops typing."
```

For example:

```text
wait 300 ms
```

Transition says:

```text
"Start the work, but don't let it block urgent interaction."
```

So:

```text
debounce
→ reduce frequency

transition
→ change rendering priority
```

Those are completely different.

React's current documentation explicitly distinguishes `useDeferredValue` from debounce/throttle for this reason: deferred rendering is interruptible and doesn't introduce a fixed delay. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

---

# 13. `useDeferredValue` solves a different shape of problem

This is where people often confuse the APIs.

Suppose you have:

```jsx
function App() {
  const [query, setQuery] = useState("");

  return (
    <>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
      />

      <SlowResults query={query} />
    </>
  );
}
```

The problem:

```text
query changes
 ↓
SlowResults gets new prop
 ↓
SlowResults rerenders
 ↓
input becomes janky
```

Now:

```jsx
const deferredQuery =
  useDeferredValue(query);
```

and:

```jsx
<SlowResults query={deferredQuery} />
```

Now:

```text
query
 ↓
updates immediately

deferredQuery
 ↓
lags behind
```

Current React describes `useDeferredValue` as returning a deferred version of the value and scheduling a background render to catch up to the latest value. ([React](https://react.dev/reference/react/useDeferredValue?utm_source=chatgpt.com "useDeferredValue – React"))

---

# 14. Visualizing `useDeferredValue`

Suppose the user types:

```text
a
ab
abc
```

Normal:

```text
query:
a → ab → abc

results:
a → ab → abc
```

With:

```js
const deferredQuery =
  useDeferredValue(query);
```

you might temporarily get:

```text
query:
a → ab → abc

deferredQuery:
a → a → ab → abc
```

The deferred value **lags**.

But eventually:

```text
deferredQuery = query
```

Current React describes this as first rendering with the new actual value and old deferred value, then attempting a background render with the updated deferred value. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

---

# 15. Why this is useful

Imagine:

```jsx
<input value={query} ... />

<ExpensiveChart query={deferredQuery} />
```

The input is:

```text
urgent
```

The chart is:

```text
non-urgent
```

So:

```text
typing
 ↓
query updates
 ↓
input changes immediately
 ↓
deferredQuery still old
 ↓
chart can continue showing old result
 ↓
React works on new chart in background
 ↓
new chart catches up
```

That's exactly the user experience we wanted.

---

# 16. `useDeferredValue` doesn't make slow code faster

Very important.

Suppose:

```text
SlowChart takes 200ms to render
```

`useDeferredValue` does **not** transform:

```text
200ms
```

into:

```text
20ms
```

It changes **when that work gets to block the user**.

So:

```text
slow computation
        ↓
still slow

but:

slow computation
        ↓
less likely to block urgent interaction
```

Current React explicitly describes `useDeferredValue` as not making the slow component's rendering faster; it deprioritizes that rendering. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

---

# 17. Why `memo` matters with `useDeferredValue`

This is subtle and extremely useful.

Suppose:

```jsx
function App() {
  const [text, setText] = useState("");

  const deferredText =
    useDeferredValue(text);

  return (
    <>
      <input
        value={text}
        onChange={e => setText(e.target.value)}
      />

      <SlowList text={deferredText} />
    </>
  );
}
```

Now user types.

The first render after typing can have:

```text
text = "ab"
deferredText = "a"
```

If:

```jsx
function SlowList({ text }) {
  ...
}
```

is not memoized, the parent can rerender it anyway.

React's current documentation explicitly notes that the deferred-value optimization is most useful when the slow child is wrapped in `memo`, allowing it to skip the quick render where the deferred value has not changed yet. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

So:

```jsx
const SlowList = memo(function SlowList({
  text,
}) {
  ...
});
```

makes the pattern much more effective.

---

# 18. Trace the entire process

Let's say:

```text
text = "a"
deferredText = "a"
```

User types `b`.

### Step 1

Event handler:

```text
setText("ab")
```

### Step 2

React performs a quick render:

```text
text = "ab"
deferredText = "a"
```

### Step 3

SlowList receives:

```text
text = "a"
```

same prop as before.

Because it's memoized:

```text
SlowList skips that render
```

### Step 4

React schedules background work for:

```text
text = "ab"
deferredText = "ab"
```

### Step 5

SlowList now receives:

```text
"ab"
```

Prop changed.

It renders the new result.

### Step 6

If another keystroke happens during that background render:

```text
"abc"
```

React can interrupt the unfinished background work and prioritize the input again.

This is the core `useDeferredValue` mechanism. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

---

# 19. `useDeferredValue` and Suspense are tightly connected

Now our previous lesson becomes useful.

Suppose:

```jsx
const deferredQuery =
  useDeferredValue(query);

<Suspense fallback={<Spinner />}>
  <SearchResults query={deferredQuery} />
</Suspense>
```

The new deferred render starts.

But suppose the new data isn't ready.

The new rendering attempt suspends.

Instead of immediately replacing the old visible result with:

```text
Spinner
```

React can keep showing the previous deferred result while the new value catches up.

Current React explicitly documents this integration between `useDeferredValue` and Suspense. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

That's a very powerful combination:

```text
input
 ↓
immediate

deferred result
 ↓
background

new data not ready
 ↓
keep old result visible

new data ready
 ↓
switch to new result
```

---

# 20. An excellent real-world example: search

Here's a conceptual search page:

```jsx
function SearchPage() {
  const [query, setQuery] = useState("");

  const deferredQuery =
    useDeferredValue(query);

  return (
    <>
      <input
        value={query}
        onChange={e =>
          setQuery(e.target.value)
        }
      />

      <SearchResults
        query={deferredQuery}
      />
    </>
  );
}
```

Now separate the responsibilities:

```text
query
 ↓
what the user is typing right now

deferredQuery
 ↓
what the expensive result UI should currently use
```

This is an elegant example of having two different notions of "current":

```text
user-current
vs
UI-work-current
```

---

# 21. But here's a trap: this does not debounce network requests

This is one of the most important caveats.

Suppose your search results trigger a network request for each query.

You type:

```text
r
re
rea
reac
react
```

`useDeferredValue` does **not automatically say**:

> "Only request `react`."

It changes rendering priority.

Current React explicitly states that `useDeferredValue` does not by itself prevent extra network requests. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

So:

```text
debounce
→ fewer requests

useDeferredValue
→ deprioritized rendering/display
```

You can combine them.

---

# 22. Transition vs Deferred Value

Now let's put them side by side.

## `startTransition`

You control the state setter:

```jsx
startTransition(() => {
  setPage(nextPage);
});
```

Meaning:

> "This state update is non-urgent."

---

## `useDeferredValue`

You don't necessarily control the setter.

You have some value:

```jsx
const deferredValue =
  useDeferredValue(value);
```

Meaning:

> "Let another part of the UI lag behind this value."

Current React describes this distinction directly: use `startTransition` when you have access to the state setter; use `useDeferredValue` when you want a prop/custom-hook value to catch up in the background. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

---

# 23. An easy rule

Ask:

> **"Do I control the state update?"**

If yes:

```text
startTransition
```

If not, and you want some downstream consumer to lag behind:

```text
useDeferredValue
```

For example:

```jsx
function Parent() {
  const [query, setQuery] = useState("");

  const deferredQuery =
    useDeferredValue(query);

  return (
    <Child query={deferredQuery} />
  );
}
```

You're explicitly saying:

```text
Child doesn't need the newest value immediately.
```

---

# 24. `useTransition` vs `startTransition`

These are even simpler.

```text
startTransition
=
mark update as Transition
```

while:

```text
useTransition
=
mark update as Transition
+
know isPending
```

So:

```jsx
const [isPending, startTransition] =
  useTransition();
```

is useful when your UI needs:

```text
pending indicator
disabled state
pending tab styling
pending navigation appearance
```

Current React explicitly says standalone `startTransition` does not provide a way to track pending state. ([React](https://react.dev/reference/react/startTransition?utm_source=chatgpt.com "startTransition – React"))

---

# 25. Why `startTransition` isn't just `setTimeout`

Compare:

```js
setTimeout(() => {
  setTab("posts");
}, 300);
```

This means:

```text
wait 300 ms
then perform update
```

But:

```js
startTransition(() => {
  setTab("posts");
});
```

means:

```text
perform update now,
but mark its rendering work as non-urgent
```

A fast device may produce:

```text
almost no visible delay
```

A slow device may allow:

```text
typing/clicks to interrupt the rendering work
```

That's fundamentally different.

---

# 26. Transitions and Suspense

Here's where everything we've learned combines.

Suppose:

```jsx
function Router() {
  const [page, setPage] =
    useState("/home");

  const [
    isPending,
    startTransition,
  ] = useTransition();

  function navigate(url) {
    startTransition(() => {
      setPage(url);
    });
  }

  return (
    <Layout isPending={isPending}>
      <Page page={page} />
    </Layout>
  );
}
```

And:

```jsx
<Suspense fallback={<BigSpinner />}>
  <Router />
</Suspense>
```

Now:

```text
click navigation
       ↓
startTransition
       ↓
page state changes
       ↓
new page starts rendering
       ↓
new page suspends
       ↓
because navigation is a Transition,
React can avoid a jarring replacement of
already-visible content in appropriate cases
       ↓
new page becomes ready
       ↓
new page shown
```

Current React recommends that Suspense-aware routers mark navigation updates as Transitions for precisely this reason: interruption, avoiding unwanted loading indicators, and coordinating page transitions. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

---

# 27. This explains a common UX pattern

Suppose you're on:

```text
Home
```

Click:

```text
Products
```

Products take time to become ready.

Bad UX:

```text
Home
 ↓
click Products
 ↓
whole page disappears
 ↓
LOADING...
 ↓
Products
```

Better:

```text
Home
 ↓
click Products
 ↓
existing UI stays usable/visible where appropriate
 ↓
Products work happens as a Transition
 ↓
Products becomes ready
 ↓
Products replaces current content
```

That's the experience Transitions + Suspense are designed to support.

---

# 28. The subtle limitation around async work

Current React's `startTransition` supports Actions that can be asynchronous, and current `useTransition` guidance allows patterns such as:

```jsx
startTransition(async () => {
  const result = await saveSomething();

  startTransition(() => {
    setSomething(result);
  });
});
```

The important subtlety is that state updates **after an `await`** currently need another `startTransition` call if you want those updates themselves marked as Transition updates. This is an existing limitation in current React. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

So don't reason:

```text
startTransition(() => {
  await something;
  setState(...);
});
```

and assume every later state update automatically has the same Transition marking.

The current API has this async boundary to be aware of.

---

# 29. Why React cares about priority

This is the deeper idea behind everything.

A browser can be doing lots of work:

```text
rendering React
handling click
processing keyboard event
layout
paint
network
animations
```

Not every piece of UI work is equally urgent.

Compare:

```text
User typed "a"
```

with:

```text
Update a 10,000-row table
```

The first should feel immediate.

The second can catch up.

React's concurrent rendering model allows it to interrupt lower-priority rendering work and handle more urgent updates first. `startTransition` is the explicit way you tell React an update is lower urgency. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

---

# 30. Don't think "priority" means a fixed queue

Avoid this mental model:

```text
priority 1
priority 2
priority 3
```

with a rigid scheduler you manually control.

Think more conceptually:

```text
urgent user interaction
        ↓
handle promptly

non-urgent UI work
        ↓
can be interrupted
```

The important benefit is **responsiveness**, not manually managing a scheduler.

---

# 31. A practical comparison table

|Tool|What you provide|What it changes|Typical use|
|---|---|---|---|
|`startTransition`|state update|marks update as non-urgent|tab/page navigation|
|`useTransition`|state update + pending UI|same, plus `isPending`|navigation with pending indicator|
|`useDeferredValue`|a value|lets downstream UI lag behind|expensive search results/chart|
|debounce|time delay|reduces frequency|reduce network requests|
|throttle|rate limit|reduces frequency|events/requests at a bounded rate|

This distinction is worth memorizing conceptually.

Especially:

```text
Transition
→ priority

Deferred value
→ lagging consumer

Debounce
→ fewer executions

Throttle
→ bounded execution frequency
```

---

# 32. A common mistake: Transitioning the input itself

Don't do this:

```jsx
startTransition(() => {
  setQuery(e.target.value);
});
```

if `query` is directly controlling the text input.

Current React explicitly says Transition updates cannot be used to control text inputs. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))

Instead:

```jsx
setQuery(e.target.value);
```

should update the input urgently.

Then either:

```jsx
const deferredQuery =
  useDeferredValue(query);
```

or:

```jsx
startTransition(() => {
  setResultsView(...);
});
```

can handle the expensive downstream work.

---

# 33. A realistic search architecture

Imagine:

```text
                      SearchPage
                           │
             ┌─────────────┴─────────────┐
             ↓                           ↓
        actualQuery                deferredQuery
             │                           │
             ↓                           ↓
          input                   expensive results
             │                           │
             ↓                           ↓
      immediate updates            background-ish
                                   catch-up work
```

Potentially:

```jsx
const [query, setQuery] =
  useState("");

const deferredQuery =
  useDeferredValue(query);

return (
  <>
    <SearchInput
      value={query}
      onChange={setQuery}
    />

    <SearchResults
      query={deferredQuery}
    />
  </>
);
```

That's a very clean separation.

---

# 34. Add Suspense

Now:

```jsx
const deferredQuery =
  useDeferredValue(query);

return (
  <>
    <SearchInput
      value={query}
      onChange={setQuery}
    />

    <Suspense
      fallback={<ResultsSkeleton />}
    >
      <SearchResults
        query={deferredQuery}
      />
    </Suspense>
  </>
);
```

Now:

```text
input
 ↓
always immediate

results
 ↓
deferred

data not ready
 ↓
old results can remain visible in the
Suspense-integrated deferred flow
```

That's much closer to a polished search interface. ([React](https://react.dev/reference/react/useDeferredValue "useDeferredValue – React"))

---

# 35. And this connects directly to our server-state lesson

Suppose:

```text
query = "react"
```

causes:

```text
GET /api/products?search=react
```

With TanStack Query:

```text
query
 ↓
query key
 ↓
server-state cache
```

With `useDeferredValue`:

```text
query
 ↓
deferredQuery
 ↓
slower visual consumer
```

So these solve different layers:

```text
TanStack Query
→ server-state management

useDeferredValue
→ rendering responsiveness

Suspense
→ readiness coordination
```

They can work together.

---

# 36. Current React 19.3 adds another interesting piece

As of September 2026, React 19.3 has made `<ViewTransition>` stable.

This is separate from `startTransition`, but it integrates with Transition-marked updates: updates inside a Transition, Suspense reveals, and `useDeferredValue` updates can participate in View Transition animations. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

So:

```text
Transition
 ↓
new UI state
 ↓
ViewTransition
 ↓
animate visual change
```

That's an example of React's newer rendering model building increasingly rich behavior around the concept of **how and when UI changes**, rather than simply immediately replacing the DOM.

We don't need to learn `ViewTransition` yet; the important thing is that it confirms the architectural significance of Transitions.

---

# 37. The deepest mental model of this lesson

Earlier, we had:

```text
state change
 ↓
render
 ↓
commit
```

Now refine it:

```text
state update
      │
      ├── urgent
      │      ↓
      │   prioritize
      │
      └── Transition
             ↓
        background/interruption-friendly
             ↓
          render
             ↓
      may be interrupted
             ↓
           commit
```

And `useDeferredValue` adds:

```text
current value
     ↓
deferred value
     ↓
background catch-up render
```

Suspense adds:

```text
background render
     ↓
not ready
     ↓
keep appropriate UI visible / fallback depending on boundary
     ↓
retry when ready
```

These concepts are all part of the same modern React rendering model.

---

# 38. The rules worth remembering

### `startTransition`

```text
"Treat this state update as non-urgent."
```

### `useTransition`

```text
"Treat this update as non-urgent,
and tell me whether the Transition is pending."
```

### `useDeferredValue`

```text
"Let this consumer lag behind
the latest value."
```

### Suspense

```text
"If this region isn't ready,
here's what to show while waiting."
```

### Error Boundary

```text
"If this region fails while rendering,
here's how to contain the failure."
```

And:

```text
debounce ≠ defer
defer ≠ cache
cache ≠ transition
transition ≠ loading state
```

Each solves a different problem.

---

# 39. The final picture

Imagine you're building a search page:

```text
                    USER TYPES
                        │
                        ↓
                    query state
                        │
                  ┌─────┴─────┐
                  ↓           ↓
             input UI      deferred value
             urgent             │
                                ↓
                          expensive results
                                │
                         ┌──────┴──────┐
                         ↓             ↓
                     not ready       ready
                         ↓             ↓
                      Suspense       render
                         │
                  old result may remain
                  during deferred work
```

And if navigation happens:

```text
navigation
   ↓
startTransition
   ↓
new page render
   ↓
Suspense if needed
   ↓
user can still interact
   ↓
new page becomes ready
   ↓
commit
```

That is the core of **responsive concurrent UI**.

---

The key idea I want you to leave with is:

> **React isn't only deciding what the UI should look like. Modern React can also reason about which UI work is urgent, which can wait, which can be interrupted, and what should remain visible while new UI becomes ready.**

That is a significant step beyond the basic `state → rerender → DOM update` model we've been using so far. ([React](https://react.dev/reference/react/useTransition "useTransition – React"))