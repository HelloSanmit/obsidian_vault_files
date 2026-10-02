
# Lesson 13 — How React Actually Renders

Up until now, we've repeatedly said:

```text
state changes
   ↓
component re-renders
   ↓
UI updates
```

That's correct, but incomplete.

Now we're going to open the black box.

You want to understand:

```text
What exactly is a render?
What causes it?
Does re-render mean the DOM is rebuilt?
What does React compare?
Why do keys matter?
Why does component identity matter?
Why can state sometimes reset?
```

This lesson is the foundation for understanding React performance later.

React describes the process as three broad steps:

```text
1. Trigger
2. Render
3. Commit
```

([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

---

# 1. The three-step model

Think of React like this:

```text
        something happens
               ↓
            TRIGGER
               ↓
            RENDER
               ↓
            COMMIT
               ↓
          Browser DOM
```

Let's understand each.

---

# 2. Step 1 — Trigger

Something needs React to render.

For example:

### Initial render

When your application starts:

```jsx
root.render(<App />);
```

React performs the initial render. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

### State update

For example:

```jsx
setCount(1);
```

This schedules an update.

### An ancestor updates

If a parent component updates, React normally renders the subtree beneath it as part of processing that update. There are optimizations that can skip some work, which we'll discuss later. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

So a very simplified trigger model is:

```text
initial app startup
        OR
state update
        ↓
React has work to do
```

---

# 3. Step 2 — Render

Here's where people often misunderstand React.

When React says:

> "render the component"

it primarily means:

> **Call the component functions and figure out what UI they describe.**

For example:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button>
      {count}
    </button>
  );
}
```

Rendering means React executes:

```text
Counter()
```

and gets back something representing:

```text
<button>0</button>
```

It is essentially calculating the next UI description.

React's docs explicitly describe rendering as React calling your components to figure out what should be displayed. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

---

# 4. Render does NOT mean "change the DOM"

This distinction is crucial.

Suppose:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <h1>{count}</h1>
  );
}
```

Initial render:

```text
render
 ↓
<h1>0</h1>
```

Then:

```text
setCount(1)
```

React renders again:

```text
render
 ↓
<h1>1</h1>
```

But **the render itself isn't the DOM update**.

After React determines what the new UI should be, it commits the necessary DOM changes.

That's why React describes render and commit as separate stages. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

---

# 5. Step 3 — Commit

Now React takes the result of rendering and applies the necessary changes to the actual DOM.

Initial render:

```text
React description
      ↓
create DOM nodes
      ↓
browser
```

Later render:

```text
old UI description
        +
new UI description
        ↓
determine necessary changes
        ↓
DOM updates
```

React says that on re-renders it applies the minimal necessary DOM operations to make the DOM match the latest rendering output. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

---

# 6. Example

Initial:

```jsx
function App() {
  return (
    <div>
      <h1>Hello</h1>
      <p>Count: 0</p>
    </div>
  );
}
```

Conceptually:

```text
render
 ↓
<div>
  <h1>Hello</h1>
  <p>Count: 0</p>
</div>

 ↓

commit
 ↓

browser DOM
```

Now state changes and the component renders:

```jsx
function App() {
  return (
    <div>
      <h1>Hello</h1>
      <p>Count: 1</p>
    </div>
  );
}
```

React doesn't need to throw away everything.

It can determine:

```text
<h1>Hello</h1> → unchanged
<p>Count: 0</p> → <p>Count: 1</p>
```

So the relevant DOM update is made.

---

# 7. This is why "re-render" is a misleading phrase for beginners

When you hear:

> "The component re-rendered."

don't imagine:

```text
delete entire webpage
 ↓
rebuild entire webpage
 ↓
put it back
```

A render is primarily React recalculating the component tree/output.

The commit phase applies the necessary DOM changes. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

This distinction will become extremely important when we discuss performance.

---

# 8. Let's look at a component tree

Suppose:

```text
App
├── Navbar
├── Sidebar
└── Main
    ├── SearchBox
    └── ProductList
```

`App` has state:

```jsx
function App() {
  const [search, setSearch] =
    useState("");

  return (
    <>
      <Navbar />
      <Sidebar />
      <Main search={search} />
    </>
  );
}
```

Suppose:

```text
search
```

changes.

React starts from the update and evaluates the relevant component tree.

By default, child components under an updated parent can be rendered as part of that work. `memo` can allow React to skip a child when its props are unchanged, subject to the normal conditions and current React optimizations. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

This leads to an important distinction.

---

# 9. "Rendered" and "DOM changed" are NOT the same thing

Suppose:

```jsx
function Child() {
  console.log("Child rendered");

  return <p>Hello</p>;
}
```

Parent changes:

```text
parent state
 ↓
parent renders
 ↓
child may render
```

But Child's output may still be:

```text
<p>Hello</p>
```

So even if Child is rendered again, React might find:

```text
old output = new output
```

and make no DOM change for it.

React's documentation explicitly notes that rendering does not necessarily result in a DOM update. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

This is a crucial distinction:

```text
component render
      ≠
DOM mutation
```

---

# 10. The component tree

React maintains a representation of your UI as a tree.

For example:

```text
App
├── Header
│   ├── Logo
│   └── UserMenu
├── Main
│   ├── SearchBar
│   └── ProductList
│       ├── ProductCard
│       ├── ProductCard
│       └── ProductCard
└── Footer
```

As state and props change, React calculates a new rendering result for this tree.

Then it reconciles the new result with what was there before.

This is where identity becomes important.

---

# 11. Component identity

Consider:

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

Suppose it displays:

```text
5
```

Now your parent renders:

```jsx
<Counter />
```

again.

Does the counter start at 0?

No.

React preserves the state because this is still the same component at the same position in the UI tree. React explicitly describes state as being associated with a component's position in the render tree. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

So:

```text
Counter at position X
      ↓
state belongs to that position/identity
```

---

# 12. State doesn't literally live inside the function

This connects to something we discussed much earlier.

You wrote:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  ...
}
```

It can feel as though:

```text
count
```

is sitting inside the function permanently.

It isn't.

The function gets called again during later renders.

React keeps track of the state associated with the component's position/identity in the UI tree and supplies that state back to the component when it renders again. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

So:

```text
render 1
Counter()
count = 0

render 2
Counter()
count = 1

render 3
Counter()
count = 2
```

The function executions are new.

The state is preserved by React.

---

# 13. Position matters

Consider:

```jsx
function App() {
  return (
    <div>
      <Counter />
      <Counter />
    </div>
  );
}
```

There are two counters.

React sees:

```text
div
├── Counter at position 1
└── Counter at position 2
```

They each get separate state.

If the first is at `5` and the second at `2`:

```text
Counter #1 → 5
Counter #2 → 2
```

They don't share state.

React's state-preservation documentation shows this exact principle: state is isolated per component occurrence in the tree. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 14. This explains why keys matter

Now imagine:

```jsx
{isPlayerA ? (
  <Counter person="Taylor" />
) : (
  <Counter person="Sarah" />
)}
```

Both branches contain:

```jsx
<Counter />
```

in the same position.

React can therefore treat this as the same component identity and preserve its state when the person changes.

That may be undesirable.

Add:

```jsx
{isPlayerA ? (
  <Counter
    key="Taylor"
    person="Taylor"
  />
) : (
  <Counter
    key="Sarah"
    person="Sarah"
  />
)}
```

Now React sees:

```text
Taylor Counter
```

and:

```text
Sarah Counter
```

as different identities.

Switching between them causes their state to be reset rather than shared. React's documentation explicitly demonstrates using different keys to control state preservation/reset. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

This is much deeper than:

> "keys remove the warning."

Keys are about identity.

---

# 15. Keys are not just for lists

This is an important upgrade to your understanding.

You previously learned:

```jsx
items.map(item => (
  <Item key={item.id} />
))
```

and thought:

> "key is something lists require."

More accurately:

> **Keys help React distinguish component/element identities among siblings.**

Lists are simply the most common place where you need them.

React's current documentation explicitly notes that keys aren't just for lists and can be used to force state to reset between otherwise similar components. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 16. Why a different `key` resets state

Consider:

```jsx
<Chat contact={selectedContact} />
```

Suppose Chat contains:

```jsx
const [message, setMessage] = useState("");
```

You type:

```text
Hello!
```

Then select a different contact.

If React sees the same `Chat` component identity at the same position, it can preserve:

```text
message = "Hello!"
```

That may not be what you want.

Instead:

```jsx
<Chat
  key={selectedContact.id}
  contact={selectedContact}
/>
```

Now:

```text
contact 1 → key 1
contact 2 → key 2
```

Switching contacts creates a different component identity, so the form state resets.

This exact pattern appears in React's state-preservation guidance. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 17. So what does React "compare"?

At a high level, React compares the new rendering result with the previous one and determines what needs to change.

But it doesn't just compare raw HTML strings.

Identity comes from things such as:

```text
element type
position
key
props/state/context changes
```

For example:

```jsx
<div />
```

is fundamentally a different element type from:

```jsx
<span />
```

Likewise:

```jsx
<Counter key="A" />
```

and:

```jsx
<Counter key="B" />
```

represent different keyed identities in the same parent context.

This is why changing types or keys can cause existing state to be discarded. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 18. Example: same type

Suppose:

```jsx
<div>
  <Counter />
</div>
```

becomes:

```jsx
<div>
  <Counter />
</div>
```

Same surrounding structure and same component identity.

React can preserve state.

---

# 19. Example: different type

Suppose:

```jsx
<div>
  <Counter />
</div>
```

becomes:

```jsx
<div>
  <Profile />
</div>
```

Now React sees a different component type in that position.

The previous `Counter` is removed and `Profile` is created.

Therefore Counter's state doesn't remain attached there. State is tied to the component identity/tree position, not to the JSX tag text itself. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 20. This explains a weird form bug

Imagine:

```jsx
function App() {
  const [showHint, setShowHint] =
    useState(false);

  if (showHint) {
    return (
      <div>
        <p>Hint</p>
        <Form />
        <button onClick={() => setShowHint(false)}>
          Hide
        </button>
      </div>
    );
  }

  return (
    <div>
      <Form />
      <button onClick={() => setShowHint(true)}>
        Show hint
      </button>
    </div>
  );
}
```

At first glance you might think:

```text
<Form />
```

is the same component in both cases.

But React sees changes in the surrounding tree structure, and depending on the exact structure the identity/position can change in a way that causes the form state to reset. React's official state-preservation examples include this kind of accidentally remounted component. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

This is one reason component structure matters.

---

# 21. Component identity is why nested component definitions are dangerous

Don't do this:

```jsx
function App() {
  function Counter() {
    const [count, setCount] =
      useState(0);

    return <button>{count}</button>;
  }

  return <Counter />;
}
```

If the component definition itself is recreated on each execution of `App`, React can treat it as a different component type and reset state in ways you didn't intend.

Instead:

```jsx
function Counter() {
  const [count, setCount] =
    useState(0);

  return <button>{count}</button>;
}

function App() {
  return <Counter />;
}
```

React's current state-preservation documentation explicitly warns against nesting component definitions for this reason. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 22. Now let's connect rendering to props

Suppose:

```jsx
function Parent() {
  const [name, setName] = useState("Alex");

  return <Child name={name} />;
}
```

When:

```jsx
setName("Sarah");
```

Parent renders again.

New result:

```jsx
<Child name="Sarah" />
```

So the child receives new props.

Conceptually:

```text
state
 ↓
Parent render
 ↓
new props
 ↓
Child render/work
```

This is the same data-flow model you've already learned, now viewed through rendering.

---

# 23. What about a child whose props don't change?

Suppose:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button
        onClick={() => setCount(c => c + 1)}
      >
        {count}
      </button>

      <Child name="Alex" />
    </>
  );
}
```

When `count` changes:

```text
Parent changes
```

By default, the subtree can be rendered again as part of the update.

So:

```text
Parent
 ↓
Child
```

may both execute.

But:

```jsx
const Child = memo(function Child({ name }) {
  ...
});
```

can allow React to skip the child when its props have not changed. `memo` is a performance optimization, not a correctness mechanism. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

We'll study this properly later.

---

# 24. Don't jump to `memo` yet

This is important.

Right now, don't start thinking:

> "Every child should use `memo`."

No.

React's current docs explicitly recommend using memoization only where it provides a meaningful performance benefit and note that it can become unnecessary or ineffective in many situations. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

For now, understand **why** extra render work can happen.

Optimization comes later.

---

# 25. The "Virtual DOM" explanation

You've probably heard:

> "React uses a Virtual DOM."

This phrase is often taught poorly.

A more useful mental model is:

```text
Your component code
      ↓
React's internal representation of UI
      ↓
compare/reconcile with previous result
      ↓
DOM commit
```

Don't imagine there is literally a second full browser DOM living somewhere that React constantly replaces.

The important practical concept is:

> React computes what the UI should be and commits the necessary DOM changes.

The official React docs focus on the more useful "render and commit" model rather than relying on the oversimplified Virtual DOM explanation. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

---

# 26. Render purity

Now we can understand another important React rule.

A component's rendering logic should be pure.

For example:

```jsx
function Price({ amount }) {
  return <p>₹{amount * 1.18}</p>;
}
```

Given the same:

```text
props
state
context
```

it should produce the same UI description.

You should not have rendering secretly perform things like:

```text
API request
DOM mutation
random global mutation
```

because React may render more than once or discard a render before commit.

This is one of the fundamental reasons React emphasizes pure rendering. React's current `memo` documentation also explicitly describes components as needing pure rendering logic. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 27. This makes `useEffect` make more sense

Remember the last lesson?

We said:

```text
render
 ↓
useEffect
 ↓
external system
```

Now you can see why.

Rendering should calculate the UI.

Effects handle synchronization that shouldn't happen as part of that pure calculation.

So:

```text
render
→ "what should UI look like?"

effect
→ "now synchronize with the outside world"
```

That distinction is becoming much more coherent now.

---

# 28. Why React might call your component more than once

One reason you'll notice this during development is Strict Mode.

React's current render documentation says that in Strict Mode, React may call component functions twice in development to help surface impure rendering logic. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

For example:

```jsx
function App() {
  console.log("render");

  return <h1>Hello</h1>;
}
```

You may see:

```text
render
render
```

during development.

Don't respond by making rendering code depend on:

```text
"this function must only execute once"
```

Instead, keep rendering pure.

---

# 29. A complete render trace

Let's trace a real component:

```jsx
function Counter() {
  const [count, setCount] =
    useState(0);

  return (
    <button
      onClick={() =>
        setCount(c => c + 1)
      }
    >
      {count}
    </button>
  );
}
```

### Initial

```text
Trigger
 ↓
Counter renders
 ↓
count = 0
 ↓
React gets:
<button>0</button>
 ↓
Commit
 ↓
browser shows 0
```

### User clicks

```text
click
 ↓
event handler
 ↓
setCount(c => c + 1)
 ↓
Trigger
```

### Second render

```text
Counter renders
 ↓
count = 1
 ↓
React gets:
<button>1</button>
 ↓
Commit
 ↓
browser updates text
```

That is the complete basic cycle.

---

# 30. Now add a child

```jsx
function Label({ count }) {
  return <p>Count: {count}</p>;
}

function Counter() {
  const [count, setCount] =
    useState(0);

  return (
    <>
      <button
        onClick={() =>
          setCount(c => c + 1)
        }
      >
        +
      </button>

      <Label count={count} />
    </>
  );
}
```

Click:

```text
setCount()
 ↓
Counter render
 ↓
count becomes 1
 ↓
Label receives count=1
 ↓
Label render/work
 ↓
commit necessary DOM changes
```

Now you can see:

```text
state → parent → props → child
```

through the rendering pipeline.

---

# 31. One of the most important rules about keys

Suppose:

```jsx
{users.map(user => (
  <UserCard
    key={user.id}
    user={user}
  />
))}
```

`key` should represent the **identity of the item**, not its current position.

For:

```text
id = 42
```

the identity is:

```text
42
```

even if it moves from position:

```text
0 → 5
```

This is why stable IDs are so useful.

React's state-preservation docs explicitly frame keys as part of component position/identity within the parent. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 32. A revealing example with inputs

Imagine:

```jsx
function Item({ item }) {
  const [text, setText] =
    useState("");

  return (
    <input
      value={text}
      onChange={e => setText(e.target.value)}
    />
  );
}
```

Now render:

```jsx
items.map((item, index) => (
  <Item
    key={index}
    item={item}
  />
))
```

If items reorder, the identity associated with a position can change.

So you can accidentally have:

```text
Item state belonging to A
```

become associated with:

```text
B
```

after a reorder.

That's why index keys can cause much more than a console warning when list items are dynamic.

Use a stable item identity:

```jsx
key={item.id}
```

when available.

---

# 33. The mental model I want you to leave with

Think of React as:

```text
          STATE / PROPS / CONTEXT
                    ↓
                 TRIGGER
                    ↓
              render components
                    ↓
            calculate new UI tree
                    ↓
             reconcile changes
                    ↓
                  COMMIT
                    ↓
                DOM updates
```

And state is associated with component identity in that tree:

```text
component identity
        ↓
React-held state
```

Keys and types help determine identity.

This is why:

```text
same identity → preserve state

new identity → reset/create state
```

React's current documentation makes this connection explicit. ([React](https://react.dev/learn/preserving-and-resetting-state?utm_source=chatgpt.com "Preserving and Resetting State – React"))

---

# 34. A much better way to think about "re-render"

From now on, don't think:

> "The screen is being rebuilt."

Think:

> **"React is recalculating what this part of the UI should be, then committing whatever DOM changes are actually necessary."**

That sentence will serve you much better as you progress.

---

# 35. Where `memo`, `useMemo`, and `useCallback` fit

We're intentionally **not** going deep into them today.

But now you have enough foundation to understand their purpose later:

```text
rendering
   ↓
sometimes work is unnecessary
   ↓
memoization techniques
```

For example:

```jsx
memo(...)
```

can help skip a component render when its props haven't changed, and React's current documentation notes that the React Compiler can automatically apply equivalent memoization in supported setups. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

But don't learn that API yet.

First understand the normal rendering model.

Then optimization becomes logical rather than magical.

---

# The four things I want you to remember

### 1.

```text
render ≠ DOM update
```

Rendering calculates UI.

Commit applies DOM changes.

### 2.

```text
state belongs to component identity/tree position
```

not to a function execution.

### 3.

```text
key = identity
```

not merely "something required by map."

### 4.

```text
state update
 → trigger
 → render
 → reconcile
 → commit
```

That is the core React rendering pipeline. ([React](https://react.dev/learn/render-and-commit?utm_source=chatgpt.com "Render and Commit – React"))

---

## Next lesson — Component composition and prop patterns

Now that you understand what React is doing with the component tree, we're going to go back upward and make your component design much stronger.

We'll cover:

```text
children
composition
slots/patterns
prop drilling
callback props
controlled components
compound component patterns
```

Then we'll transition into **routing and real application structure**, where these ideas become much more practical.