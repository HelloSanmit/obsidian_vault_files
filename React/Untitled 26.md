
# Lesson 24 — Rerenders: What Actually Causes Them, What `memo` Can Stop, and How to Find Real Performance Problems

Last lesson was about:

```text
referential equality
memo
useMemo
useCallback
```

Today we're going to correct a common misunderstanding that usually appears immediately afterward:

> **“A component rerendered, therefore something is wrong.”**

No.

Rerendering is normal.

The interesting questions are:

```text
Why did it rerender?
Was that rerender necessary?
How expensive was it?
Can we avoid it?
Should we avoid it?
```

Those are different questions.

---

# 1. First: what does “rerender” actually mean?

Suppose:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  console.log("Counter rendered");

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

Clicking the button:

```text
id="2o9ryh"
state update
   ↓
Counter function executes again
   ↓
new JSX result is calculated
   ↓
React determines what needs changing
   ↓
DOM is updated if necessary
```

The component function executing again is the **render**.

It doesn't mean:

```text
“throw away the entire DOM and rebuild it”
```

That distinction from our rendering-model lesson is essential.

---

# 2. The first major rule: state changes can trigger a render

Suppose:

```jsx
function Profile() {
  const [name, setName] = useState("Alice");

  return (
    <input
      value={name}
      onChange={e => setName(e.target.value)}
    />
  );
}
```

When:

```js
setName("Bob");
```

happens:

```text
state changes
   ↓
Profile gets another render
```

That's expected.

State is one of the major sources of updates.

---

# 3. The second major rule: parent updates can cause child renders

Suppose:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child />
    </>
  );
}

function Child() {
  console.log("Child rendered");

  return <div>Hello</div>;
}
```

Click:

```text
Parent state changes
       ↓
Parent renders
       ↓
React processes Child
       ↓
Child may render
```

Notice:

```text
Child's props didn't change.
Child's own state didn't change.
```

Yet without memoization, a child can still be rendered when its parent renders.

That's normal React behavior. `memo` gives React an opportunity to skip that child when its props remain unchanged. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 4. This is where `memo` becomes meaningful

```jsx
const Child = memo(function Child() {
  console.log("Child rendered");

  return <div>Hello</div>;
});
```

Now:

```text
Parent state changes
       ↓
Parent renders
       ↓
Child receives same props
       ↓
memo says:
"these props haven't changed"
       ↓
Child can be skipped
```

So the key idea is not:

> “memo stops rerenders.”

It's:

> **“memo can allow React to skip a child render when parent rendering doesn't require new props for that child.”**

React's current behavior describes `memo` as a performance optimization rather than an absolute guarantee. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 5. But here's something important: a component can render and nothing visible changes

Suppose:

```jsx
function Example() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(0)}>
      {count}
    </button>
  );
}
```

We're already at:

```text
count = 0
```

Clicking the button asks for:

```text
0 → 0
```

React can recognize that the state remains the same and avoid committing a visible update. React may still call the component in some circumstances, but the result can be discarded. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

This is a subtle but valuable distinction:

```text
component function executed
        ≠
DOM changed
```

Again:

```text
render
```

and:

```text
commit
```

are different phases.

---

# 6. This distinction prevents a lot of bad optimization

Imagine:

```jsx
console.log("Child rendered");
```

and you see the message twice.

You might panic:

> "The DOM must be rebuilding!"

Not necessarily.

You need to determine:

```text
Was the component function executed?
Did React commit a DOM update?
Did the user experience any slowdown?
Was the work expensive?
```

Those are separate observations.

---

# 7. Context is another source of renders

Suppose:

```jsx
const ThemeContext = createContext("light");
```

and:

```jsx
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext value={theme}>
      <Toolbar />
    </ThemeContext>
  );
}
```

Then:

```jsx
function Toolbar() {
  const theme = useContext(ThemeContext);

  return <div className={theme}>Toolbar</div>;
}
```

When the context value changes:

```text
theme:
light → dark

       ↓

context consumers that use it
       ↓

update
```

Even if the component is wrapped in `memo`, a component that **reads that context** still updates when the context value it observes changes. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

So:

```text
memo
```

doesn't mean:

```text
ignore context changes
```

---

# 8. This creates a very useful optimization technique

Suppose:

```jsx
const UserContext = createContext(null);
```

and:

```jsx
const ExpensiveList = memo(function ExpensiveList({
  name,
}) {
  // expensive rendering
});
```

You could have:

```jsx
function ExpensiveComponent() {
  const user = useContext(UserContext);

  return (
    <ExpensiveList
      name={user.name}
    />
  );
}
```

Now the context-reading component is separate from the expensive memoized child.

The architecture becomes:

```text
Context
  ↓
small adapter component
  ↓
extract only what is needed
  ↓
memoized child
  ↓
expensive rendering
```

So instead of allowing a deep expensive component to consume a large context object directly, you can isolate context consumption and pass smaller props.

This is a current React recommendation for making context-driven rendering more selective. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 9. Context values themselves can accidentally change all the time

Remember this:

```jsx
function App() {
  const [user, setUser] = useState(null);

  const login = () => {
    // ...
  };

  return (
    <AuthContext
      value={{
        user,
        login,
      }}
    >
      <Page />
    </AuthContext>
  );
}
```

Every render creates:

```text
new object
```

and:

```text
new function
```

So even if:

```text
user hasn't changed
```

the context value itself may have a new identity.

React compares context values using `Object.is`, so consumers can be notified when that value identity changes. ([React](https://react.dev/reference/react/useContext?utm_source=chatgpt.com "useContext – React"))

This connects directly to our previous lesson.

You could stabilize it:

```jsx
const login = useCallback(() => {
  // ...
}, []);

const contextValue = useMemo(() => {
  return {
    user,
    login,
  };
}, [user, login]);
```

Then:

```text
Parent rerenders
      ↓
user unchanged
      ↓
login unchanged
      ↓
contextValue remains same reference
      ↓
context consumers don't need to update merely because
the provider parent rerendered
```

This is one situation where `useMemo` and `useCallback` have a concrete architectural purpose. ([React](https://react.dev/reference/react/useContext?utm_source=chatgpt.com "useContext – React"))

---

# 10. But let's step back

We now have several things that can trigger updates:

```text
state changes
props change
context changes
external/store subscriptions
```

And then there are updates caused indirectly by effects.

For example:

```jsx
function Component() {
  const [count, setCount] = useState(0);
  const [double, setDouble] = useState(0);

  useEffect(() => {
    setDouble(count * 2);
  }, [count]);

  return ...
}
```

Suppose:

```text
count = 1
```

The component renders.

Then:

```text
effect runs
   ↓
setDouble(2)
   ↓
another render
```

You have created a chain:

```text
render
 ↓
effect
 ↓
setState
 ↓
render
```

Sometimes that is necessary.

But very often it means we stored something that should have been **derived during rendering**.

---

# 11. Derived state can create unnecessary renders

Instead of:

```jsx
const [count, setCount] = useState(0);
const [double, setDouble] = useState(0);

useEffect(() => {
  setDouble(count * 2);
}, [count]);
```

just write:

```jsx
const [count, setCount] = useState(0);

const double = count * 2;
```

Now:

```text
count changes
 ↓
one render
 ↓
double calculated
```

instead of:

```text
count changes
 ↓
render
 ↓
effect
 ↓
setDouble
 ↓
second render
```

This is an extremely important performance lesson:

> **Sometimes the best optimization is removing an unnecessary state/effect relationship, not adding memoization.**

Current React guidance explicitly calls out unnecessary Effects that update state as a common source of repeated render chains. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 12. Another common chain

Consider:

```jsx
function SearchResults({ products }) {
  const [filtered, setFiltered] = useState([]);

  useEffect(() => {
    setFiltered(
      products.filter(product =>
        product.name.includes("phone")
      )
    );
  }, [products]);

  return <List items={filtered} />;
}
```

This is unnecessary if `filtered` is simply derived from `products`.

Instead:

```jsx
function SearchResults({ products }) {
  const filtered = products.filter(product =>
    product.name.includes("phone")
  );

  return <List items={filtered} />;
}
```

Now there is no:

```text
render
 ↓
effect
 ↓
state update
 ↓
render
```

Just:

```text
render
 ↓
calculate filtered list
 ↓
render List
```

And if that filtering later proves expensive:

```jsx
const filtered = useMemo(() => {
  return products.filter(product =>
    product.name.includes("phone")
  );
}, [products]);
```

Only then are we using memoization to solve an actual calculation/identity issue.

---

# 13. This gives us three different optimization levels

When you see unnecessary work, think in this order.

### Level 1 — Architecture

Can I avoid the update entirely?

For example:

```text
localize state
remove unnecessary Effect
derive instead of storing
split context
```

### Level 2 — Component boundaries

Can I prevent unrelated parts of the tree from being involved?

For example:

```text
split component
move state downward
extract a memoized child
```

### Level 3 — Memoization

Only then:

```text
memo
useMemo
useCallback
```

This ordering is much healthier than:

```text
Problem?
→ useMemo
→ useCallback
→ memo
→ pray
```

---

# 14. State placement is a performance decision

Suppose this entire application is:

```text
App
├── Header
├── Sidebar
├── SearchBox
├── ProductList
└── Footer
```

And you have:

```jsx
const [search, setSearch] = useState("");
```

at the top-level `App`.

Every keystroke:

```text
search changes
 ↓
App rerenders
 ↓
large portion of tree gets processed
```

Maybe that's perfectly fine.

But what if only:

```text
SearchBox
ProductList
```

need the search state?

It might be better to move state closer to the components that actually need it.

This is why **state colocation** is a useful performance principle.

Current React guidance also recommends keeping transient state local rather than lifting it higher than necessary. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 15. Here's a powerful example of state colocation

Bad architecture—not necessarily “broken,” just unnecessarily broad:

```jsx
function App() {
  const [isFormOpen, setIsFormOpen] = useState(false);

  return (
    <>
      <Header />
      <Sidebar />
      <Main />
      <Footer />

      {isFormOpen && <Form />}
    </>
  );
}
```

If `App` is huge, opening the form updates the top of the tree.

Instead:

```jsx
function App() {
  return (
    <>
      <Header />
      <Sidebar />
      <Main />
      <Footer />
    </>
  );
}

function Main() {
  const [isFormOpen, setIsFormOpen] = useState(false);

  return ...
}
```

Now the state exists closer to where it matters.

This isn't only about speed.

It also improves ownership and makes the application easier to reason about.

---

# 16. There is another surprisingly powerful technique: `children`

Consider:

```jsx
function Wrapper() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <ExpensiveTree />
    </div>
  );
}
```

You might think:

```text
Wrapper rerenders
 ↓
ExpensiveTree must rerender
```

But composition can produce a useful boundary.

For example:

```jsx
function App() {
  return (
    <Wrapper>
      <ExpensiveTree />
    </Wrapper>
  );
}
```

And:

```jsx
function Wrapper({ children }) {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      {children}
    </div>
  );
}
```

The important idea is that `children` is a JSX value created by the parent and passed into `Wrapper`; when `Wrapper` updates its own state, React can preserve the already-created child subtree rather than treating it as newly described by the rerendering wrapper.

This is one reason good component composition can reduce the need for memoization. Current React guidance specifically recommends letting visual wrapper components accept JSX as `children` when appropriate. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

This connects directly to our earlier composition lesson.

---

# 17. Notice what just happened

Earlier, composition looked like an API-design topic:

```text
children
slots
component boundaries
```

Now we're seeing another benefit:

```text
composition
      ↓
state ownership becomes more local
      ↓
fewer unrelated updates
      ↓
less need for memoization
```

That's why good React architecture often beats clever optimization.

---

# 18. “Every parent rerender rerenders the whole tree” is an oversimplification

You'll hear:

> "When a parent changes, the whole component tree rerenders."

Not literally.

React processes the subtree, but it can skip work based on:

```text
memoization
component identity
unchanged state
context behavior
composition
other reconciliation decisions
```

And even when component functions execute, React can still avoid committing DOM changes where nothing changed.

So don't imagine:

```text
state update
 ↓
destroy DOM
 ↓
rebuild everything
```

The actual process is closer to:

```text
state update
 ↓
React schedules update
 ↓
render phase
 ↓
reconciliation
 ↓
determine necessary changes
 ↓
commit required changes
```

That distinction is fundamental.

---

# 19. How do you know what's actually slow?

Don't guess.

React provides the `<Profiler>` component for measuring rendering performance programmatically, and React Developer Tools also provide profiling capabilities. The Profiler reports information such as render duration and whether the update was a mount or update. ([React](https://react.dev/reference/react/Profiler?utm_source=chatgpt.com "<Profiler> – React"))

For example:

```jsx
import { Profiler } from "react";

function onRender(
  id,
  phase,
  actualDuration,
  baseDuration
) {
  console.log({
    id,
    phase,
    actualDuration,
    baseDuration,
  });
}

function App() {
  return (
    <Profiler
      id="ProductPage"
      onRender={onRender}
    >
      <ProductPage />
    </Profiler>
  );
}
```

The important measurements include:

```text
actualDuration
```

which reflects the time spent rendering the profiled tree for that update, and:

```text
baseDuration
```

which estimates the cost of rendering that subtree without the optimizations that currently help it. ([React](https://react.dev/reference/react/Profiler?utm_source=chatgpt.com "<Profiler> – React"))

You don't need to memorize the metrics yet.

The important engineering habit is:

> **Measure before optimizing.**

---

# 20. Why `console.log()` is not a profiler

Suppose:

```jsx
console.log("ExpensiveComponent rendered");
```

appears 20 times.

That tells you:

```text
the function executed
```

It does **not** tell you:

```text
the component is slow
```

A component could execute 100 times and be essentially free.

Another component could execute once and take 200 ms.

So:

```text
render count
≠
render cost
```

This distinction is incredibly useful.

---

# 21. A useful performance equation

Think roughly:

```text
performance problem
=
frequency of work
×
cost of each piece of work
```

A component that renders:

```text
1000 times
```

but each render takes:

```text
0.01 ms
```

may be less concerning than one that renders:

```text
10 times
```

at:

```text
100 ms
```

So the right question isn't:

> "How many rerenders?"

It's:

> **"How much useful work is React doing, how often, and why?"**

---

# 22. A classic mistake with `memo`

Imagine:

```jsx
const BigList = memo(function BigList({ items }) {
  ...
});
```

But the parent does:

```jsx
const items = [...products];
```

every render.

Then:

```text
old items → Array A
new items → Array B
```

so:

```text
A !== B
```

`memo` cannot help.

You might then add:

```jsx
const items = useMemo(
  () => [...products],
  [products]
);
```

Now the array identity is stable when `products` is stable.

But ask another question first:

> Why are we creating `[...]` at all?

Perhaps the better code is simply:

```jsx
<BigList items={products} />
```

Again:

```text
architecture
   ↓
memoization
```

not the other way around.

---

# 23. Another classic mistake: unstable callback + memo

```jsx
const Row = memo(function Row({ onDelete }) {
  ...
});

function List({ items }) {
  return items.map(item => (
    <Row
      key={item.id}
      onDelete={() => deleteItem(item.id)}
    />
  ));
}
```

Each row gets a new function every render:

```text
Row 1 → Function A
Row 1 next render → Function B

Row 2 → Function C
Row 2 next render → Function D
```

So `memo(Row)` can't necessarily skip those rows based on `onDelete`.

You might restructure:

```jsx
const handleDelete = useCallback((id) => {
  deleteItem(id);
}, []);
```

and then pass:

```jsx
<Row
  key={item.id}
  item={item}
  onDelete={handleDelete}
/>
```

Now the row receives a stable callback, and the item identity determines whether its specific data changed.

But again, you only care if this rendering is actually expensive enough to matter.

---

# 24. `memo` doesn't make Context cheap

Suppose:

```jsx
const AppContext = createContext(null);
```

with:

```jsx
<AppContext
  value={{
    theme,
    user,
    notifications,
    settings,
    ...
  }}
>
```

Now one huge context value changes frequently.

Any component consuming that context is tied to changes of that context value.

You may improve the architecture by splitting contexts:

```text
ThemeContext
UserContext
NotificationContext
```

or by separating context-reading components from expensive children.

This is an example where the best optimization isn't:

```text
memo everything
```

but rather:

```text
design narrower reactive boundaries
```

---

# 25. Why splitting a component can improve performance

Suppose:

```jsx
function HugeComponent() {
  const theme = useContext(ThemeContext);

  // 500 lines of expensive UI
}
```

You could instead have:

```jsx
function ThemeReader() {
  const theme = useContext(ThemeContext);

  return (
    <HugeComponent theme={theme} />
  );
}
```

and:

```jsx
const HugeComponent = memo(function HugeComponent({
  theme,
}) {
  ...
});
```

Now the boundary is explicit:

```text
context changes
 ↓
ThemeReader
 ↓
HugeComponent receives props
```

You can now decide whether the expensive child truly needs to update for every context change.

It's a form of **reactive boundary design**.

---

# 26. This is a deeper way to think about React performance

Instead of:

```text
"How do I stop rerenders?"
```

ask:

```text
"Where should reactivity exist?"
```

For example:

```text
Search input
   ↓
Search component

Page number
   ↓
URL

Server products
   ↓
Query cache

Theme
   ↓
Theme context

Hover state
   ↓
individual row
```

Now changes naturally stay close to where they matter.

That reduces unnecessary propagation.

---

# 27. A realistic product-page architecture

Imagine:

```text
ProductPage
├── SearchControls
├── FilterSidebar
├── ProductGrid
│   ├── ProductCard
│   ├── ProductCard
│   ├── ProductCard
│   └── ...
├── Pagination
└── CartPanel
```

We might decide:

```text
search/filter/page
    → URL state

products
    → server state

cart
    → application state

hovered card
    → local state inside ProductCard

theme
    → context
```

Now suppose the user hovers over one product.

If hover state is kept inside that card:

```text
hover Card #7
 ↓
Card #7 updates
```

Instead of:

```text
hover Card #7
 ↓
ProductPage updates
 ↓
ProductGrid updates
 ↓
every Card gets processed
```

That's state colocation as a performance tool.

---

# 28. There's a surprising lesson here

A lot of performance optimization in React is actually **state architecture**.

Not:

```text
fancy hooks
```

but:

```text
Where does state live?
Who consumes it?
How large is the reactive boundary?
What needs to update?
```

This is why the lessons about:

```text
state ownership
lifting state
Context
composition
derived data
URL state
server state
```

all matter for performance too.

---

# 29. One more subtlety: Strict Mode

During development, React's Strict Mode can deliberately perform extra development-only checks, including extra render/setup behavior designed to expose bugs in impure rendering and effect cleanup. This can make logs appear more frequently in development than you expected.

So seeing:

```text
Component rendered
Component rendered
```

during development does not automatically prove that production will perform the exact same sequence.

This is one reason `console.log` output isn't a reliable performance measurement by itself.

---

# 30. The practical debugging process

Suppose a page feels slow when typing.

Don't immediately add:

```jsx
useMemo
useCallback
memo
```

Instead:

### Step 1

Ask:

```text
What state changes when I type?
```

Maybe:

```text
search
```

### Step 2

Ask:

```text
Where does that state live?
```

If it's unnecessarily high:

```text
App
```

instead of:

```text
SearchPage
```

consider moving it.

### Step 3

Ask:

```text
Which components actually need search?
```

Maybe only:

```text
SearchBox
ProductList
```

### Step 4

Measure.

Use React profiling tools to find which part of the tree consumes meaningful rendering time. ([React](https://react.dev/reference/react/Profiler?utm_source=chatgpt.com "<Profiler> – React"))

### Step 5

Only then consider:

```text
memo
useMemo
useCallback
```

where the measurements and data flow justify them.

---

# 31. The hierarchy I want you to remember

When you encounter unnecessary rendering:

```text
                 Performance problem
                         │
                         ↓
                Is the work necessary?
                   ↙           ↘
                 no             yes
                 ↓               ↓
         fix architecture     measure cost
                                   ↓
                            identify expensive part
                                   ↓
                         can boundary be improved?
                           ↙               ↘
                         yes               no
                         ↓                  ↓
                   split/localize       memoize if useful
                   state/composition     memo/useMemo/
                                        useCallback
```

That is a much stronger mental model than:

```text
rerender → memo
```

---

# 32. The four things you should now be able to distinguish

### A component rendered

```text
function executed
```

### React committed an update

```text
necessary DOM/UI changes were applied
```

### A component rendered unnecessarily

```text
it did work that could have been avoided
```

### The application is slow

```text
the user actually experiences costly work
```

These are **not equivalent statements**.

You need evidence before moving from the first statement to the fourth.

---

# 33. Final mental model

Put everything together:

```text
                     USER INTERACTION
                            ↓
                      STATE / PROP / CONTEXT
                            ↓
                         UPDATE
                            ↓
                      RENDER PHASE
                            ↓
                 component functions execute
                            ↓
                       reconciliation
                            ↓
                   should this work happen?
                    ↙                 ↘
                 skip                 render
                  ↓                     ↓
             memo/boundary        calculate UI
                                        ↓
                                      commit
                                        ↓
                                  visible changes
```

And performance engineering sits around this:

```text
state placement
composition
derived data
context boundaries
memo
useMemo
useCallback
profiling
```

—not just the last three.

The most important takeaway from this lesson is:

> **A rerender is not a bug. An expensive or unnecessary piece of work is what you investigate.**

Once that mindset is in place, `memo`, `useMemo`, and `useCallback` stop being mysterious “performance hooks” and become tools you reach for for specific reasons. ([React](https://react.dev/reference/react/Profiler?utm_source=chatgpt.com "<Profiler> – React"))

The next topic gets us into another part of React that initially looks mysterious but becomes much easier now: **portals, event propagation, and why a modal can visually live outside your component's DOM location while still behaving like a child in React's tree.**