
# Lesson 40 — React Performance: How to Find the Real Bottleneck

You've now learned a lot of performance-related concepts separately:

```text
memo
useMemo
useCallback
React Compiler
startTransition
useDeferredValue
Suspense
code splitting
lazy loading
Server Components
```

Now we need to step back.

The real professional skill is **not knowing more optimization APIs**.

It's being able to answer:

> **Why is this React application slow, and what evidence proves it?**

That's what this lesson is about.

React 19.3 currently exposes React-specific Performance Tracks inside the browser Performance panel, alongside network, JavaScript, and event-loop activity. React DevTools also provides component-level profiling, and the `<Profiler>` API can measure specific subtrees programmatically. ([React](https://react.dev/reference/dev-tools/react-performance-tracks?utm_source=chatgpt.com "React Performance tracks – React"))

---

# 1. First: "React is slow" is almost never specific enough

Suppose a user says:

> "The dashboard is slow."

That could mean very different things:

```text
A. Initial page takes too long to appear.

B. Clicking a button feels delayed.

C. Typing into an input lags.

D. Navigation takes too long.

E. Scrolling is janky.

F. Data arrives slowly.

G. The page downloads too much JavaScript.

H. A component rerenders unnecessarily.

I. A calculation is expensive.

J. The browser spends too long painting/layout.
```

These are completely different problems.

So performance debugging starts with:

```text
symptom
  ↓
measure
  ↓
identify bottleneck
  ↓
change one thing
  ↓
measure again
```

Not:

```text
symptom
  ↓
useMemo everywhere
```

---

# 2. Three different kinds of "rendering cost"

Let's simplify the browser work.

A React update can involve:

```text
React render
    ↓
DOM commit
    ↓
browser rendering
```

The browser then performs work such as:

```text
style calculation
layout
paint
compositing
```

So a slow interaction could come from:

```text
React
OR
JavaScript outside React
OR
browser rendering
OR
network
```

That distinction is crucial.

---

# 3. Example: slow button click

Suppose:

```jsx
function App() {
  const [open, setOpen] = useState(false);

  return (
    <button onClick={() => setOpen(x => !x)}>
      Toggle
    </button>
  );
}
```

This looks trivial.

But when the user clicks, the browser may have to do:

```text
click event
   ↓
React state update
   ↓
React render
   ↓
React commit
   ↓
layout
   ↓
paint
```

If the interface still feels slow, the problem could be anywhere in that chain.

So don't immediately blame:

```text
useState
```

or:

```text
React rerender
```

---

# 4. Rerendering is not automatically a performance problem

This is one of the biggest lessons.

Suppose:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button
        onClick={() => setCount(c => c + 1)}
      >
        {count}
      </button>

      <SmallText />
    </>
  );
}
```

When `count` changes:

```text
Counter rerenders
SmallText may rerender
```

People often react:

> "Oh no, SmallText rerendered."

But what matters is:

```text
How expensive was that render?
```

If:

```text
SmallText render = 0.02 ms
```

there may be nothing worth fixing.

Compare:

```text
HugeTable render = 90 ms
```

Now you have a real problem.

So:

```text
unnecessary render
≠
slow render
```

They're related concepts, not identical ones.

---

# 5. There are two broad performance problems

Think of:

## Problem A — too much work

```text
Something renders more often than necessary.
```

Examples:

```text
parent update
 ↓
1000 children rerender
```

## Problem B — expensive work

```text
Even one render is expensive.
```

Examples:

```text
sort 200,000 records
parse large document
render giant chart
complex calculation
```

The optimizations are different.

---

# 6. Example: unnecessary work

```jsx
function Dashboard() {
  const [count, setCount] =
    useState(0);

  return (
    <>
      <button
        onClick={() => setCount(c => c + 1)}
      >
        {count}
      </button>

      <HugeList />
    </>
  );
}
```

Suppose:

```text
HugeList
```

doesn't depend on:

```text
count
```

but still rerenders because its parent rerenders.

That may be unnecessary work.

Potential solutions include:

```text
state colocation
memoization
component restructuring
React Compiler
```

---

# 7. State colocation is often the first optimization

Consider:

```jsx
function App() {
  const [search, setSearch] =
    useState("");

  return (
    <>
      <Header />
      <Sidebar />
      <Search onChange={setSearch} />
      <HugeDashboard />
    </>
  );
}
```

If:

```text
search changes
```

the entire `App` rerenders.

That could cause:

```text
Header
Sidebar
HugeDashboard
```

to participate in rendering work.

Instead:

```jsx
function SearchArea() {
  const [search, setSearch] =
    useState("");

  return (
    <Search
      value={search}
      onChange={setSearch}
    />
  );
}
```

Now the state is closer to the component that actually needs it.

Conceptually:

### Before

```text
App
 ├── search state
 ├── Header
 ├── Sidebar
 ├── Search
 └── HugeDashboard
```

### After

```text
App
 ├── Header
 ├── Sidebar
 ├── SearchArea
 │    └── search state
 └── HugeDashboard
```

This reduces the scope of updates without any memoization API.

That's why **architecture often beats optimization tricks**.

---

# 8. A common bad pattern

```jsx
function App() {
  const [name, setName] =
    useState("");

  const [email, setEmail] =
    useState("");

  const [search, setSearch] =
    useState("");

  const [modalOpen, setModalOpen] =
    useState(false);

  // massive application
}
```

Now one tiny state change can rerender a very large subtree.

Sometimes the solution isn't:

```text
memoize 17 components
```

It's:

```text
move the state closer to where it's used
```

---

# 9. Composition can also reduce work

Imagine:

```jsx
function Layout() {
  const [color, setColor] =
    useState("blue");

  return (
    <PageContent />
  );
}
```

If the state affects only:

```text
ThemePicker
```

don't put it at the highest possible parent.

A more localized architecture:

```text
App
 ├── PageContent
 └── ThemePicker
       └── color state
```

often means fewer components need to rerender.

This is why we've repeatedly emphasized **state ownership** throughout the course.

Performance is another reason.

---

# 10. Context can create broad render propagation

Suppose:

```jsx
<ThemeContext value={theme}>
  <Application />
</ThemeContext>
```

and:

```text
theme changes
```

components reading that context need to update.

If a single broad context contains:

```js
{
  user,
  theme,
  settings,
  notifications,
  cart
}
```

then changing one part can create a large invalidation area for consumers.

A better design may be:

```text
UserContext
ThemeContext
CartContext
NotificationContext
```

or a more targeted external-store/query architecture depending on the problem.

Again:

```text
performance
```

is often a consequence of:

```text
state ownership design
```

rather than missing `useMemo`.

---

# 11. `React.memo` solves only one kind of problem

Suppose:

```jsx
const UserCard = memo(function UserCard({
  user
}) {
  return <p>{user.name}</p>;
});
```

`memo` can skip rendering when props are considered unchanged.

But suppose:

```jsx
<UserCard
  user={{
    id: user.id,
    name: user.name
  }}
/>
```

creates a new object every parent render.

Then:

```text
old object
   ≠
new object
```

by reference identity.

So the memoized component may still render.

This is why the lesson on referential equality was so important.

---

# 12. Function identity has the same issue

```jsx
<UserCard
  onSelect={() => selectUser(user.id)}
/>
```

creates a new function on each render.

So a memoized child can receive:

```text
old function ≠ new function
```

even though behavior is logically identical.

Historically:

```text
useCallback
```

was often used to stabilize that identity.

But React Compiler changes this landscape substantially.

---

# 13. React Compiler changes the optimization baseline

React Compiler 1.0 became stable in 2025 and automatically performs memoization based on compiler analysis. The React team recommends relying on the compiler for normal memoization in new code, while retaining `useMemo`, `useCallback`, and `memo` when you need explicit control or for existing code where changing memoization could alter behavior. ([React](https://react.dev/blog/2025/10/07/react-compiler-1?utm_source=chatgpt.com "React Compiler v1.0 – React"))

So older advice like:

```text
"Wrap every component in memo."
```

is increasingly poor advice.

Modern thinking is more like:

```text
write correct declarative React
        ↓
let compiler optimize where appropriate
        ↓
profile
        ↓
manually optimize actual bottlenecks
```

---

# 14. But the Compiler doesn't make architecture irrelevant

Suppose you have:

```text
App
 └── 10,000-item list
```

and an unnecessarily broad state update.

The compiler can optimize memoization opportunities.

But it cannot magically turn:

```text
10,000 DOM nodes
```

into:

```text
50 DOM nodes
```

or replace:

```text
poor data architecture
```

with:

```text
good data architecture
```

The compiler optimizes React computation.

It doesn't redesign your application for you.

---

# 15. React Compiler doesn't memoize everything everywhere

The compiler analyzes React components and hooks. It does not magically memoize every arbitrary JavaScript function globally, and React notes that shared expensive calculations may still require a different strategy. ([React](https://react.dev/learn/react-compiler/introduction?utm_source=chatgpt.com "Introduction – React"))

Example:

```js
function expensiveTransform(data) {
  return hugeCalculation(data);
}
```

If this function is called:

```text
ComponentA
ComponentB
ComponentC
```

the compiler doesn't turn it into one universal application-wide cache.

You may need domain-level caching, server-side processing, query caching, or another architecture.

Again:

```text
optimization scope matters
```

---

# 16. Expensive calculation is different from rerender count

Consider:

```jsx
function SearchResults({ items, query }) {
  const results = items.filter(
    item =>
      item.name
        .toLowerCase()
        .includes(query.toLowerCase())
  );

  return ...
}
```

This may run on every render.

But if:

```text
items = 20
```

the cost is probably tiny.

If:

```text
items = 2,000,000
```

now it could matter.

So don't optimize the pattern:

```text
"filter inside render"
```

just because it technically runs on every render.

Measure the actual workload.

---

# 17. `useMemo` is about avoiding expensive recalculation

Historically:

```jsx
const results = useMemo(
  () => expensiveFilter(items, query),
  [items, query]
);
```

means:

```text
if items/query unchanged
    ↓
reuse previous calculation
```

But remember your current React environment may already have compiler-driven memoization.

So don't blindly write:

```text
useMemo
useMemo
useMemo
```

everywhere.

Write it when:

```text
there is a measurable expensive calculation
+
the dependency model is appropriate
```

or when you specifically need stable identity.

React's current recommendation is to use the Compiler as the default optimization mechanism where available and retain manual memoization for cases needing explicit control. ([React](https://react.dev/blog/2025/10/07/react-compiler-1?utm_source=chatgpt.com "React Compiler v1.0 – React"))

---

# 18. Effects can create performance cascades

Consider:

```jsx
function ProductList({ products }) {
  const [sorted, setSorted] =
    useState([]);

  useEffect(() => {
    setSorted(
      [...products].sort(compare)
    );
  }, [products]);

  return ...
}
```

This produces:

```text
render
 ↓
effect
 ↓
setSorted
 ↓
render again
```

The first render didn't even need the state.

The derived value could simply be:

```jsx
const sorted =
  [...products].sort(compare);
```

or otherwise computed/memoized appropriately.

This pattern can create **cascading updates**.

Current React Performance Tracks explicitly expose cascading updates because they can be a source of performance regressions. ([React](https://react.dev/reference/dev-tools/react-performance-tracks?utm_source=chatgpt.com "React Performance tracks – React"))

---

# 19. Cascading updates

Think:

```text
Update A
  ↓
render
  ↓
something schedules Update B
  ↓
another render
```

Instead of:

```text
Update A
  ↓
render
  ↓
commit
```

You accidentally get:

```text
Update A
  ↓
render
  ↓
Update B
  ↓
render again
```

This is especially common with:

```text
setState inside effects
```

when the state is derived from other state/props.

The current React lint ecosystem also includes diagnostics for patterns like `set-state-in-effect`, precisely because these patterns can create unnecessary work. ([React](https://react.dev/blog/2025/10/07/react-compiler-1?utm_source=chatgpt.com "React Compiler v1.0 – React"))

---

# 20. Derive instead of synchronize

This is an incredibly useful React rule.

Bad:

```jsx
const [fullName, setFullName] =
  useState("");

useEffect(() => {
  setFullName(
    firstName + " " + lastName
  );
}, [firstName, lastName]);
```

You have:

```text
render
 ↓
effect
 ↓
setState
 ↓
render
```

Better:

```jsx
const fullName =
  firstName + " " + lastName;
```

Now:

```text
render
 ↓
fullName
```

One render.

The broader principle:

> If a value can be calculated from current props/state during rendering, it often should not be stored separately as state.

This improves correctness **and** performance.

---

# 21. Browser rendering can dominate React

Here's a different problem.

Suppose React renders a component in:

```text
2 ms
```

but changing it causes:

```text
layout = 30 ms
paint = 20 ms
```

Your application feels slow.

React isn't the main bottleneck.

The browser rendering pipeline is.

For example, changing:

```css
width
height
top
left
```

in certain patterns can trigger layout calculations.

Animation-heavy UIs may benefit from compositor-friendly properties such as:

```css
transform
opacity
```

depending on the exact rendering workload.

So Chrome's Performance panel is important because it shows more than React.

---

# 22. The browser Performance panel vs React Profiler

This distinction matters.

## React Profiler

Answers:

```text
Which React components rendered?
How long did React spend rendering?
What committed?
```

## Browser Performance panel

Answers more broadly:

```text
JavaScript
React
network
layout
paint
event handlers
long tasks
browser rendering
```

Current React Performance Tracks add React-specific timelines directly inside the browser Performance panel, so you can line up React work with other browser activity. ([React](https://react.dev/reference/dev-tools/react-performance-tracks?utm_source=chatgpt.com "React Performance tracks – React"))

This is extremely valuable.

---

# 23. React Performance Tracks

React 19.2 introduced React-specific Performance Tracks, and they are still part of the current React tooling.

They include tracks for:

```text
Scheduler
Components
Server
```

among others.

The Scheduler track distinguishes priorities such as:

```text
Blocking
Transition
Suspense
Idle
```

and the Components track shows component render/effect durations. ([React](https://react.dev/reference/dev-tools/react-performance-tracks?utm_source=chatgpt.com "React Performance tracks – React"))

So instead of:

```text
"Something felt slow."
```

you can inspect a timeline like:

```text
click
│
├── Blocking work
│
├── React render
│   ├── App
│   ├── Dashboard
│   └── Table
│
├── Commit
│
├── Effects
│
└── Browser paint
```

Now you're debugging evidence rather than guessing.

---

# 24. Example Performance trace

Suppose clicking a filter causes:

```text
User click
  ↓
App render 1 ms
  ↓
FilterList render 2 ms
  ↓
HugeTable render 120 ms
  ↓
commit 4 ms
  ↓
paint 12 ms
```

The evidence says:

```text
HugeTable
```

is the obvious React bottleneck.

You now investigate:

```text
Why does HugeTable rerender?
Why is its render 120 ms?
Can its state be localized?
Can the calculation be reduced?
Can rows be virtualized?
Can the data representation change?
```

That is real performance engineering.

---

# 25. What if the trace says React only takes 3 ms?

Suppose:

```text
React render = 3 ms
```

but:

```text
JavaScript = 80 ms
```

from another library.

Then optimizing:

```text
React.memo
```

may do almost nothing.

The bottleneck might be:

```text
charting library
syntax highlighter
JSON processing
date library
analytics SDK
```

That's why the whole-browser timeline matters.

---

# 26. What if JavaScript is fast but interaction still lags?

Suppose:

```text
React = 3 ms
JS = 5 ms
layout + paint = 90 ms
```

Then the problem is likely browser rendering.

For example:

```text
huge DOM
expensive CSS
layout thrashing
complex shadows
massive repaint
```

Again:

```text
don't optimize React
when React isn't the problem
```

---

# 27. Long tasks

A useful browser concept is a **long task**.

Very roughly:

```text
main-thread task
```

that runs long enough to interfere with responsiveness.

Imagine:

```text
click
 ↓
300 ms JavaScript
```

The browser can't respond smoothly during that blocking work.

Even if:

```text
React render = 5 ms
```

some unrelated computation could freeze the UI.

This is why performance is a system problem, not a "render count" problem.

---

# 28. Input lag is especially important

Imagine:

```jsx
<input
  value={query}
  onChange={e => setQuery(e.target.value)}
/>
```

Suppose every keystroke causes:

```text
query state update
 ↓
huge list render
 ↓
expensive filtering
 ↓
chart recalculation
```

Then typing becomes:

```text
R
  ↓
lag

Re
  ↓
lag

Rea
  ↓
lag
```

The user experiences this as:

> "The input is slow."

But the input itself may be perfectly fine.

The real issue is:

```text
high-frequency state update
        +
too much downstream work
```

---

# 29. This is where `useDeferredValue` helps

You learned:

```js
const deferredQuery =
  useDeferredValue(query);
```

Architecture:

```text
query
 ↓
urgent
 ↓
input updates immediately

deferredQuery
 ↓
non-urgent
 ↓
expensive results update later
```

So:

```text
typing
   ↓
fast

search results
   ↓
can lag slightly
```

This is a **scheduling** solution.

It's different from:

```text
debounce
```

because you're not primarily waiting for a quiet period before doing work.

You're telling React:

> Keep urgent interaction responsive while lower-priority rendering catches up.

---

# 30. But deferred rendering doesn't make expensive work disappear

Suppose:

```text
render Results = 500 ms
```

`useDeferredValue` doesn't turn that into:

```text
5 ms
```

It changes:

```text
when the work is allowed to compete
```

not:

```text
how much CPU the work requires
```

So the hierarchy is:

```text
useDeferredValue
→ scheduling

memoization
→ avoid repeated work

algorithm/data structure
→ reduce amount of work

virtualization
→ reduce rendered DOM

server processing
→ move work elsewhere
```

These solve different problems.

---

# 31. Virtualization

Suppose a table contains:

```text
100,000 rows
```

Rendering all of them is often unnecessary.

The viewport might show only:

```text
30 rows
```

A virtualization system can render approximately:

```text
visible rows
+
small buffer
```

rather than:

```text
100,000 DOM nodes
```

So:

```text
100,000 data records
≠
100,000 DOM nodes
```

This is a profound performance principle.

Sometimes the correct optimization isn't:

```text
"make React render faster."
```

It's:

```text
"don't ask React to render things the user can't see."
```

---

# 32. Algorithmic complexity matters more than hooks

Suppose:

```js
items.sort(...)
```

runs on:

```text
1,000,000 items
```

You might spend an hour optimizing:

```text
useMemo
useCallback
memo
```

while the true problem is:

```text
O(n log n)
```

sorting a million records on every keystroke.

An algorithmic improvement could be worth more than dozens of React optimizations.

For example:

```text
filtering dataset
↓
server query
```

might be better than:

```text
download everything
↓
filter 1 million records in browser
```

This is where frontend architecture meets computer science.

---

# 33. Bundle performance

We've already covered code splitting.

But now think diagnostically.

Suppose first load is slow.

Check:

```text
HTML
CSS
JavaScript
fonts
images
network requests
```

Then:

```text
main.js = 3.2 MB
```

and:

```text
admin.js = 1.4 MB
editor.js = 900 KB
charts.js = 700 KB
```

but the user is visiting:

```text
/home
```

That's a bundle architecture problem.

Code splitting may have a much bigger impact than `memo`.

---

# 34. Bundle size is not the same as runtime cost

Suppose:

```text
bundle = 500 KB
```

but it includes:

```text
huge initialization logic
```

or:

```text
heavy parsing
```

Conversely:

```text
bundle = 2 MB
```

might not all execute immediately.

So you should distinguish:

```text
download cost
parse cost
compile cost
execute cost
render cost
```

Performance is multidimensional.

---

# 35. Images can dominate the experience too

Imagine:

```text
JavaScript = 300 KB
images = 8 MB
```

You could optimize React for days and barely move the needle.

Performance debugging therefore needs to stay broader than:

```text
React renders
```

For initial-load problems, inspect:

```text
network
image sizes
format
dimensions
lazy loading
fonts
preloading
```

---

# 36. Server-side performance matters too

Now connect this to SSR and RSC.

Suppose:

```text
browser waits 2 seconds
```

and you discover:

```text
Server Component data request = 1.6 s
```

Optimizing:

```text
React.memo
```

on the client won't fix that.

You need to inspect:

```text
database query
server fetch
waterfall
cache
serialization
server rendering
```

React 19's Performance Tracks include server request/component information in development builds, making server-side React work visible in the same performance-investigation model. ([React](https://react.dev/reference/dev-tools/react-performance-tracks?utm_source=chatgpt.com "React Performance tracks – React"))

---

# 37. Server waterfalls

Suppose:

```js
const user = await getUser();
const orders = await getOrders(user.id);
const recommendations =
  await getRecommendations();
```

Potentially:

```text
user
  ↓
orders
  ↓
recommendations
```

If recommendations doesn't depend on user:

```js
const userPromise = getUser();
const recommendationsPromise =
  getRecommendations();

const [user, recommendations] =
  await Promise.all([
    userPromise,
    recommendationsPromise
  ]);

const orders =
  await getOrders(user.id);
```

Now:

```text
user ───────┐
            ├──→ continue
recommendations ─┘
                ↓
             orders
```

That's reducing a server waterfall.

Performance optimization isn't always about React components.

---

# 38. The most useful performance equation

Think:

```text
Total perceived latency
≈
network
+
server
+
JavaScript
+
React
+
browser rendering
```

Not literally as a universal additive formula—some work overlaps—but as an investigation checklist.

When someone says:

> "The page is slow."

Ask which stage is expensive.

---

# 39. `<Profiler>`

React gives you a programmatic measurement tool:

```jsx
import { Profiler } from "react";

function onRender(
  id,
  phase,
  actualDuration,
  baseDuration,
  startTime,
  commitTime
) {
  console.log({
    id,
    phase,
    actualDuration,
    baseDuration
  });
}

function App() {
  return (
    <Profiler
      id="Dashboard"
      onRender={onRender}
    >
      <Dashboard />
    </Profiler>
  );
}
```

React invokes the callback when the profiled subtree commits. The callback provides measurements such as `actualDuration` and `baseDuration`. ([React](https://react.dev/reference/react/Profiler?utm_source=chatgpt.com "<Profiler> – React"))

---

# 40. `actualDuration`

Suppose:

```text
actualDuration = 15 ms
```

That means roughly:

> React spent about 15 ms rendering this subtree for this update.

Now:

```text
baseDuration = 100 ms
```

can be interpreted as an estimate of how expensive the subtree could be without memoization optimizations, based on recent render durations. ([React](https://react.dev/reference/react/Profiler?utm_source=chatgpt.com "<Profiler> – React"))

So:

```text
actual = 15 ms
base   = 100 ms
```

suggests:

```text
some optimization is successfully avoiding work
```

But don't treat the numbers as universal "speed scores."

They're measurement tools for investigation.

---

# 41. Why profiling adds overhead

Profiling itself requires instrumentation.

So:

```text
production
```

and:

```text
development/profiling
```

don't necessarily behave identically.

React currently disables profiling instrumentation in ordinary production builds by default because it adds overhead. ([React](https://react.dev/reference/dev-tools/react-performance-tracks?utm_source=chatgpt.com "React Performance tracks – React"))

Therefore:

> Don't conclude that development profiler timing equals exact real-world production timing.

Use it to identify patterns and bottlenecks, then validate with production-like measurements.

---

# 42. React DevTools

React DevTools gives you:

```text
Components
Profiler
```

and the current tooling can also surface whether the React Compiler optimized components through a `Memo ✨` indicator. ([React](https://react.dev/learn/react-developer-tools?utm_source=chatgpt.com "React Developer Tools – React"))

The practical workflow is:

```text
reproduce interaction
   ↓
record
   ↓
identify expensive component/work
   ↓
inspect why it rendered
   ↓
fix
   ↓
record again
```

---

# 43. Don't optimize based on render count alone

Suppose DevTools tells you:

```text
Component rendered 200 times
```

That sounds terrible.

But:

```text
200 × 0.01 ms
=
2 ms
```

Maybe irrelevant.

Meanwhile:

```text
Component B rendered 3 times
```

but:

```text
3 × 120 ms
=
360 ms
```

That's much more interesting.

So always consider:

```text
frequency
×
cost per occurrence
```

rather than frequency alone.

---

# 44. What about `useCallback`?

Imagine:

```jsx
const handleSelect = useCallback(
  id => {
    select(id);
  },
  [select]
);
```

Was performance improved?

Only if:

```text
stable function identity
```

allows some downstream optimization.

For example:

```text
memoized child
```

or:

```text
effect dependency
```

depends on that stable identity.

Otherwise:

```text
useCallback
```

might simply add complexity without measurable value.

And with React Compiler, ordinary manual memoization is less frequently necessary than older React advice suggested. ([React](https://react.dev/blog/2025/10/07/react-compiler-1?utm_source=chatgpt.com "React Compiler v1.0 – React"))

---

# 45. The same applies to `useMemo`

Bad reasoning:

```text
"Object creation is expensive."
```

Usually:

```js
const options = {
  sort: "asc"
};
```

is cheap.

The real concern is:

```text
Does this object identity matter?
Is a memoized child depending on it?
Is it an effect dependency?
Is the calculation actually expensive?
```

Optimize the observed consequence, not the syntax.

---

# 46. Performance has an architecture hierarchy

When fixing a slow application, investigate roughly in this order:

```text
1. Is the wrong amount of data being processed?

2. Is the wrong amount of UI being rendered?

3. Is state owned too high in the tree?

4. Are there request/render waterfalls?

5. Is expensive work repeated unnecessarily?

6. Is browser layout/paint expensive?

7. Is client JavaScript too large?

8. Only then:
   do targeted memoization.
```

The exact order depends on the symptom, but this prevents premature micro-optimization.

---

# 47. A real debugging example

Suppose:

> "Typing into search freezes for half a second."

You investigate.

### Step 1

Profile the typing interaction.

You discover:

```text
input update
 ↓
SearchPage render = 4 ms
 ↓
Results render = 180 ms
```

Good.

Now ask:

```text
Why are Results recalculated on every keystroke?
```

You inspect:

```text
2 million items
```

being filtered client-side.

Now the solution might be:

```text
server-side search
```

or:

```text
indexed/local data structure
```

rather than:

```text
useCallback()
```

That's the kind of judgment you want to develop.

---

# 48. Another debugging example

Suppose:

> "Opening the modal is slow."

Profiler:

```text
Modal render = 2 ms
```

Browser Performance:

```text
layout = 90 ms
paint = 70 ms
```

Now you investigate:

```text
huge DOM
complex backdrop
massive box-shadow
layout changes
```

The React tree isn't your primary bottleneck.

So:

```text
useMemo
```

would be the wrong fix.

---

# 49. Another example: slow navigation

User:

> "Dashboard navigation takes 2 seconds."

You inspect.

```text
navigation
 ↓
route discovery = 150 ms
 ↓
chunk download = 500 ms
 ↓
server loader = 900 ms
 ↓
React render = 20 ms
```

Now we know:

```text
React render is not the problem.
```

Potential improvements might involve:

```text
prefetching
route splitting strategy
server caching
parallel data fetching
database optimization
loader architecture
```

Again, measurement tells you where to look.

---

# 50. The "waterfall detector"

Whenever you see:

```text
A
 ↓
B
 ↓
C
 ↓
D
```

ask:

> Could B, C, or D start earlier?

This applies everywhere:

```text
network requests
server data
dynamic imports
route discovery
database queries
React rendering
```

For example:

```text
route discovery
 ↓
chunk load
 ↓
loader
 ↓
render
```

might be a waterfall.

Performance engineering often means turning:

```text
serial
```

into:

```text
parallel
```

where dependencies allow it.

---

# 51. Measure before and after

Suppose you change:

```text
state ownership
```

from:

```text
App
```

to:

```text
SearchPanel
```

Don't stop at:

> "It should be faster."

Record again.

Maybe:

```text
Before
120 ms

After
8 ms
```

Excellent.

Or:

```text
Before
12 ms

After
11 ms
```

Then the optimization wasn't important.

This is why profiling is not a final step.

It's part of the entire optimization loop.

---

# 52. A useful optimization loop

```text
          USER REPORT
               │
               ▼
            REPRODUCE
               │
               ▼
             MEASURE
               │
               ▼
          FIND BOTTLENECK
               │
               ▼
        FORM A HYPOTHESIS
               │
               ▼
          MAKE ONE CHANGE
               │
               ▼
             MEASURE
               │
          ┌────┴────┐
          │         │
        better     same
          │         │
          ▼         ▼
       keep       rethink
```

This is much more valuable than memorizing a list of "React performance hacks."

---

# 53. Current React's performance philosophy

The current direction of React is increasingly:

```text
declarative code
+
compiler optimization
+
scheduler
+
Suspense
+
server rendering
+
profiling tooling
```

rather than forcing developers to manually optimize every component.

React Compiler 1.0 was explicitly released to automate memoization and reduce the need for manually adding `useMemo`, `useCallback`, and `memo`. ([React](https://react.dev/blog/2025/10/07/react-compiler-1?utm_source=chatgpt.com "React Compiler v1.0 – React"))

So your job increasingly becomes:

```text
write correct architecture
+
follow React's rules
+
measure actual bottlenecks
+
use explicit optimization when it addresses a measured need
```

---

# 54. But performance doesn't mean "make everything synchronous"

You've already learned:

```text
startTransition
```

Now connect that to performance.

Suppose:

```text
urgent:
typing

non-urgent:
render 5,000 search results
```

Then:

```text
startTransition
```

or:

```text
useDeferredValue
```

can tell React:

```text
don't let lower-priority work make urgent interaction feel stuck
```

This is a **responsiveness** optimization.

It doesn't necessarily reduce total CPU.

That's an important distinction:

```text
throughput
≠
latency
≠
responsiveness
```

---

# 55. Performance vocabulary

You should now distinguish:

### Latency

```text
How long until something happens?
```

### Throughput

```text
How much work can the system process?
```

### Responsiveness

```text
Does interaction feel immediate?
```

### Rendering cost

```text
How much React/browser work is needed?
```

### Bundle cost

```text
How much code must be transferred/parsed/executed?
```

### Server cost

```text
How expensive is the server-side work?
```

These are related but not interchangeable.

---

# 56. One final architecture example

Imagine:

```text
Search page
```

with:

```text
input
filters
results
map
analytics
```

A good architecture might be:

```text
Input
 └── urgent local state

Search query
 └── URL state

Results
 └── deferred rendering

Server search
 └── route loader / query

Map
 └── lazy-loaded client component

Analytics
 └── lower-priority/external work

Large result list
 └── virtualization
```

Now notice how many different React concepts cooperate:

```text
useState
URL state
loader / server state
useDeferredValue
lazy
Suspense
virtualization
profiling
```

Performance architecture isn't one API.

---

# 57. The most important performance rule

When you see a problem:

```text
DON'T ASK FIRST:
"Which React optimization hook should I use?"
```

Ask:

```text
"What work is actually taking the time?"
```

Then:

```text
"Why is that work happening?"
```

Then:

```text
"Can I make less work happen?"
```

Then:

```text
"Can I move the work somewhere better?"
```

Then:

```text
"Can I schedule it differently?"
```

And only then:

```text
"Do I need memoization?"
```

That order is extremely valuable.

---

# 58. Final mental model

```text
                  SLOW UI
                     │
          ┌──────────┼───────────┐
          │          │           │
       Network      Server     Browser
          │          │           │
       loading      DB/API     layout
       chunks       SSR        paint
          │          │           │
          └──────────┼───────────┘
                     │
                   React
                     │
          ┌──────────┼───────────┐
          │          │           │
        too many   expensive    cascades
        renders     renders      updates
          │          │           │
          └──────────┼───────────┘
                     │
                architecture
                     │
        ┌────────────┼─────────────┐
        │            │             │
    state scope   data volume   scheduling
        │            │             │
   colocate       paginate      transition
   compose        virtualize     defer
   split          server-side    suspense
```

The final optimization layer:

```text
Compiler
memo
useMemo
useCallback
```

comes after you've understood the actual problem.

---

# 59. What to permanently remember

```text
Rerender
≠
slow

Expensive render
≠
unnecessary render

React performance
≠
whole-browser performance

useMemo
→ avoid/reuse expensive calculations or stabilize identity when needed

memo
→ skip component work when props are unchanged

useCallback
→ stabilize function identity when that identity matters

startTransition
→ schedule non-urgent updates

useDeferredValue
→ let urgent value updates outpace expensive rendering

lazy
→ delay code loading

Suspense
→ define UI while work is unavailable

Profiler
→ measure React subtree performance

Performance panel
→ measure React + browser + network + JS together

React Compiler
→ automatically applies many memoization optimizations
```

And the biggest rule:

> **Measure first. Optimize the bottleneck, not the code pattern that merely looks suspicious.**

That principle will stay useful regardless of which React APIs exist five years from now.