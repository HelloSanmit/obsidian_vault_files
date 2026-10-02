
# Lesson 6 — State Ownership, Lifting State Up, and Component Communication

You now have enough pieces to understand one of the most important React questions:

> **Where should a piece of state live?**

This sounds simple, but it affects almost everything about how you structure a React application.

---

# 1. Start with a problem

Imagine this UI:

```text
Products
────────────────────────────

Search: [ laptop             ]

Category: [ All ▼ ]

Laptop
₹80,000

Laptop Stand
₹3,000

Mouse
₹1,500
```

Let's break the UI into components:

```text
App
├── SearchBox
├── CategoryFilter
└── ProductList
    └── ProductCard
```

Now suppose the user types:

```text
laptop
```

The `ProductList` needs to know that the search term is `"laptop"` so it can display matching products.

But the input itself lives in:

```text
SearchBox
```

So the question is:

> Should `SearchBox` own `searchTerm`?

It certainly could.

But then:

```text
SearchBox
     searchTerm
         ↓
     ProductList
```

How does the sibling `ProductList` get it?

React's normal data flow is downward.

So instead, we often move the state upward.

---

# 2. Put shared state in the common parent

Structure:

```text
              App
          /     |     \
         ↓      ↓      ↓
   SearchBox  Filter  ProductList
```

`App` becomes the owner of:

```jsx
const [searchTerm, setSearchTerm] = useState("");
```

Then:

```text
App
├── searchTerm
└── setSearchTerm
```

passes what each child needs.

For example:

```jsx
function App() {
  const [searchTerm, setSearchTerm] = useState("");

  return (
    <>
      <SearchBox
        searchTerm={searchTerm}
        onSearchChange={setSearchTerm}
      />

      <ProductList
        searchTerm={searchTerm}
      />
    </>
  );
}
```

Now:

```text
                 App
                  │
          searchTerm state
             /          \
            ↓            ↓
      SearchBox      ProductList
```

This is called:

# Lifting state up

You're moving state from a child to the nearest parent that needs to coordinate it with other components.

---

# 3. Why not let `SearchBox` own it?

Suppose:

```jsx
function SearchBox() {
  const [searchTerm, setSearchTerm] = useState("");
}
```

The problem is that `ProductList` is a sibling.

React doesn't normally have sibling-to-sibling data flow like:

```text
SearchBox ─────────→ ProductList
```

Instead:

```text
          Parent
         /      \
        ↓        ↓
     Child A  Child B
```

The common parent coordinates them.

So:

```text
SearchBox
   ↑
   │
   App
   │
   ↓
ProductList
```

This preserves the one-way flow.

---

# 4. The parent becomes the source of truth

Once `App` owns the state:

```jsx
const [searchTerm, setSearchTerm] = useState("");
```

there is exactly one source of truth for the search term.

`SearchBox` displays/changes it.

`ProductList` consumes it.

That's a clean architecture:

```text
searchTerm
    ↓
SearchBox     → lets user change it
    ↓
App state
    ↓
ProductList   → uses it
```

---

# 5. The child doesn't directly modify the parent's state

This is subtle.

The child receives:

```jsx
<SearchBox
  searchTerm={searchTerm}
  onSearchChange={setSearchTerm}
/>
```

Then:

```jsx
function SearchBox({ searchTerm, onSearchChange }) {
  return (
    <input
      value={searchTerm}
      onChange={e => onSearchChange(e.target.value)}
    />
  );
}
```

When the user types:

```text
"A"
```

this happens:

```text
input event
   ↓
onChange
   ↓
onSearchChange("A")
   ↓
App's setSearchTerm("A")
   ↓
App state changes
   ↓
App renders again
```

The child isn't reaching into the parent and changing its internal variables.

It is calling a function the parent gave it.

---

# 6. Why use a prop named `onSearchChange` instead of `setSearchTerm`?

You technically could write:

```jsx
<SearchBox setSearchTerm={setSearchTerm} />
```

That's valid.

But this:

```jsx
<SearchBox onSearchChange={setSearchTerm} />
```

communicates the component contract better.

The child doesn't need to know that the parent happens to be using React's `useState`.

It's being told:

> "When the search changes, call this function."

This is a better abstraction.

The child says:

```text
"I report search changes."
```

rather than:

```text
"I manipulate your state setter."
```

As applications grow, this distinction becomes valuable.

---

# 7. Let's build the example

Suppose our data is:

```jsx
const products = [
  {
    id: 1,
    name: "Laptop",
    category: "electronics"
  },
  {
    id: 2,
    name: "Laptop Stand",
    category: "accessories"
  },
  {
    id: 3,
    name: "Mouse",
    category: "electronics"
  }
];
```

Our components:

```text
App
├── SearchBox
└── ProductList
```

Parent:

```jsx
import { useState } from "react";

function App() {
  const [searchTerm, setSearchTerm] = useState("");

  return (
    <main>
      <SearchBox
        searchTerm={searchTerm}
        onSearchChange={setSearchTerm}
      />

      <ProductList
        products={products}
        searchTerm={searchTerm}
      />
    </main>
  );
}
```

SearchBox:

```jsx
function SearchBox({
  searchTerm,
  onSearchChange
}) {
  return (
    <input
      value={searchTerm}
      onChange={e => onSearchChange(e.target.value)}
      placeholder="Search products"
    />
  );
}
```

ProductList:

```jsx
function ProductList({
  products,
  searchTerm
}) {
  const visibleProducts = products.filter(product =>
    product.name
      .toLowerCase()
      .includes(searchTerm.toLowerCase())
  );

  return (
    <ul>
      {visibleProducts.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

---

# 8. Trace one keystroke

Let's say the user types:

```text
l
```

Initial:

```text
searchTerm = ""
```

The input displays:

```text
""
```

User types `l`.

`onChange` runs:

```jsx
e => onSearchChange(e.target.value)
```

Suppose:

```text
e.target.value = "l"
```

So:

```js
onSearchChange("l")
```

And remember:

```jsx
<SearchBox
  onSearchChange={setSearchTerm}
/>
```

Therefore:

```js
setSearchTerm("l")
```

Now App renders again.

New state:

```text
searchTerm = "l"
```

App passes:

```text
"l"
```

to `ProductList`.

Then:

```js
products.filter(...)
```

runs again.

Now only matching products remain.

So the complete chain is:

```text
User types
   ↓
SearchBox event
   ↓
callback prop
   ↓
App state update
   ↓
App re-render
   ↓
ProductList receives new prop
   ↓
filter() recalculates
   ↓
new product list UI
```

This is the architecture you should start seeing when you look at React code.

---

# 9. This is one-way data flow

The state itself flows downward:

```text
App
 ↓
SearchBox
 ↓
UI

App
 ↓
ProductList
 ↓
UI
```

The interaction sends information upward by calling a callback:

```text
SearchBox
   │
   │ call callback
   ↓
 App
   │
   │ update state
   ↓
ProductList
```

So React still has one primary data direction:

```text
parent → child
```

The child isn't pushing state directly into the parent.

The child is **requesting that the parent update its state** by calling a function.

---

# 10. Why this feels slightly backwards at first

Coming from vanilla JavaScript, you might think:

> "The input changed, so I'll just tell the product list."

React asks you to think differently:

> "The search term is application data. Who should own that data?"

Once the parent owns it:

```text
searchTerm
```

both the input and list become representations of the same state.

That's much easier to scale.

---

# 11. State ownership is about dependency

Here's a useful rule.

Suppose state is needed by:

```text
Component A
Component B
Component C
```

Ask:

> What is the nearest component that contains all of them?

That component is a strong candidate for owning the state.

For:

```text
       App
      /   \
     A     B
```

if both A and B need some state:

```text
       App
     state
      /  \
     A    B
```

If only A needs it:

```text
       App
       |
       A
     state
```

There's no reason to push it up unnecessarily.

---

# 12. Don't lift state higher than necessary

This is the opposite mistake.

Suppose:

```text
App
├── Navbar
├── Dashboard
│   ├── Counter
│   └── Chart
└── Footer
```

Only `Counter` needs:

```text
count
```

Don't automatically put:

```js
const [count, setCount] = useState(0);
```

in `App`.

Keep it local:

```text
App
└── Dashboard
    └── Counter
         count
```

The principle is:

> Put state at the lowest level that needs to own/control it, while still keeping it high enough for all consumers that need it.

That's state ownership.

---

# 13. Three common cases

### Case 1 — One component needs the state

```text
Button
```

Keep it local:

```jsx
const [open, setOpen] = useState(false);
```

### Case 2 — Two siblings need the state

```text
      Parent
      /    \
     A      B
```

Put it in the parent.

### Case 3 — Many distant components need it

Now lifting it through five or ten levels can become awkward.

This is where we'll eventually discuss:

```text
Context
global client state
server state
URL state
```

But don't jump to those solutions prematurely.

---

# 14. Prop drilling

Imagine:

```text
App
 ↓
Layout
 ↓
Dashboard
 ↓
Sidebar
 ↓
SearchBox
```

and the search state originates in `App`.

You may end up with:

```jsx
<Layout searchTerm={searchTerm} />
```

then:

```jsx
<Dashboard searchTerm={searchTerm} />
```

then:

```jsx
<Sidebar searchTerm={searchTerm} />
```

then:

```jsx
<SearchBox searchTerm={searchTerm} />
```

even though:

```text
Layout
Dashboard
Sidebar
```

don't actually care about the search term.

This is called:

# Prop drilling

It's not automatically bad.

Passing props through one or two layers is perfectly normal.

But if intermediate components exist primarily to forward props, the design may become inconvenient.

Later, we'll see how Context can solve some of these situations.

---

# 15. Callback props are not special React objects

This is related to the callback confusion you mentioned earlier.

When you see:

```jsx
<SearchBox
  onSearchChange={setSearchTerm}
/>
```

remember:

```text
onSearchChange
```

is simply a prop.

And its value happens to be a function.

Exactly like:

```jsx
<SearchBox
  title="Search"
/>
```

except:

```text
title → string
onSearchChange → function
```

Props can hold any JavaScript value.

---

# 16. The child doesn't have to know what the callback does

For example:

```jsx
function SearchBox({ onSearchChange }) {
  return (
    <input
      onChange={e => onSearchChange(e.target.value)}
    />
  );
}
```

The child doesn't know whether the parent will:

```text
update search state
send analytics
update URL
trigger a request
```

It just knows:

```text
"When the search changes, call this function."
```

That's excellent component design.

---

# 17. A callback can carry data upward

Here's an important distinction.

People often summarize React as:

> "Props go down."

Correct.

But callbacks allow a child to **send information upward indirectly**.

Example:

```jsx
function Child({ onSelect }) {
  return (
    <button onClick={() => onSelect("laptop")}>
      Select laptop
    </button>
  );
}
```

Parent:

```jsx
function Parent() {
  function handleSelect(productName) {
    console.log(productName);
  }

  return (
    <Child onSelect={handleSelect} />
  );
}
```

Flow:

```text
Child
  │
  │ onSelect("laptop")
  ↓
Parent's function
```

The data isn't literally flowing upward as a prop.

The child is **calling a function provided by the parent**.

That distinction is useful.

---

# 18. A better naming convention

For callback props, names often begin with `on`:

```jsx
onClick
onSubmit
onChange
onDelete
onSelect
onToggle
onSearchChange
onClose
```

These usually mean:

> "When this thing happens, invoke this function."

Then inside the child:

```jsx
onClick={...}
```

or:

```jsx
onDelete(id)
```

This convention makes component APIs easier to understand.

---

# 19. Let's revisit your task example

You had:

```jsx
<TaskItem
  key={task.id}
  task={task}
  onToggle={toggleTask}
/>
```

Think of it now as:

```text
TaskItem receives:

task
  ↓
information

onToggle
  ↓
function it can call
```

The child:

```jsx
onChange={() => onToggle(task.id)}
```

calls the parent's function.

Parent:

```jsx
function toggleTask(id) {
  setTasks(tasks =>
    tasks.map(...)
  );
}
```

updates its state.

Then everything downstream recalculates:

```text
tasks
 ↓
visibleTasks
 ↓
map()
 ↓
TaskItem
```

So what looked like a complicated chain is really just:

```text
state owner
   ↓
passes data
   ↓
passes callback
   ↓
child event
   ↓
callback
   ↓
state owner changes state
   ↓
new render
```

---

# 20. State ownership and derived data together

Let's make the product example slightly more realistic.

```jsx
function App() {
  const [searchTerm, setSearchTerm] = useState("");
  const [category, setCategory] = useState("all");

  const visibleProducts = products.filter(product => {
    const matchesSearch =
      product.name
        .toLowerCase()
        .includes(searchTerm.toLowerCase());

    const matchesCategory =
      category === "all" ||
      product.category === category;

    return matchesSearch && matchesCategory;
  });

  return (
    <>
      <SearchBox
        value={searchTerm}
        onChange={setSearchTerm}
      />

      <CategoryFilter
        value={category}
        onChange={setCategory}
      />

      <ProductList
        products={visibleProducts}
      />
    </>
  );
}
```

Notice the architecture:

```text
                  App
        ┌──────────┼──────────┐
        ↓          ↓          ↓
 searchTerm    category    products
        │          │
        └────┬─────┘
             ↓
      visibleProducts
             ↓
       ProductList
```

Only the **minimal source-of-truth state** is stored.

Everything else is derived.

This is a pattern you'll use constantly.

---

# 21. Don't store `visibleProducts` as state

A beginner might write:

```jsx
const [visibleProducts, setVisibleProducts] =
  useState(products);
```

Then whenever search changes, they'd have to remember to update:

```jsx
setVisibleProducts(...)
```

and whenever category changes, update it again.

Now you have synchronization problems.

Instead:

```jsx
const visibleProducts = products.filter(...);
```

means:

> "Whenever the source state changes, calculate the result again."

That's much simpler.

---

# 22. A powerful design question

When designing a component tree, ask these questions in order:

### 1. What data can change?

Those are candidates for state.

### 2. Who needs that data?

List the components.

### 3. What's the nearest common parent?

That is a candidate state owner.

### 4. What can be calculated from that data?

Those values are derived.

### 5. Which child needs to report an event?

Give it a callback prop.

This is basically the beginning of **Thinking in React**.

---

# 23. Your mental model is expanding

Previously you had:

```text
state
 ↓
UI
```

Now:

```text
                    state owner
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
          data props           callback props
              ↓                     ↑
           children             events
              ↓                     │
              UI ────────────────┘
```

And then:

```text
event
 ↓
callback
 ↓
state update
 ↓
new render
 ↓
new props
 ↓
new UI
```

That is a major piece of React.

---

# 24. One common mistake

Don't do this unnecessarily:

```jsx
function Parent() {
  const [value, setValue] = useState("");

  return (
    <Child
      value={value}
      setValue={setValue}
    />
  );
}
```

unless the child actually needs to work with the generic setter.

Prefer:

```jsx
<Child
  value={value}
  onChange={setValue}
/>
```

or, even more semantically:

```jsx
<Child
  value={value}
  onSearchChange={setValue}
/>
```

The child should ideally interact with the **concept**, not the parent's implementation detail.

Later, when we study reusable component APIs, this distinction becomes very useful.

---

# 25. Another common mistake: duplicating state

Avoid:

```jsx
const [name, setName] = useState("Alex");
const [displayName, setDisplayName] = useState("Alex");
```

when:

```jsx
const displayName = name;
```

would be enough.

Also avoid:

```jsx
const [selectedProduct, setSelectedProduct] = useState(null);

const [selectedProductName, setSelectedProductName] =
  useState("");
```

if the name can simply be:

```js
selectedProduct?.name
```

The more duplicated state you have, the more synchronization work you create.

---

# 26. What "lifting state up" really means

It does **not** mean:

> "Always put state in the parent."

It means:

> **Move state upward only when multiple components need to coordinate around the same source of truth.**

So:

```text
Only A needs it
→ state in A
```

```text
A and B need it
→ state in their common parent
```

```text
A, B, C and many distant parts need it
→ consider another state-sharing mechanism
```

This principle will eventually lead us to Context and external state libraries.

---

# 27. Practice problem

Imagine:

```text
App
├── TemperatureInput
├── TemperatureInput
└── TemperatureDisplay
```

You have:

```text
Celsius:    [ 25 ]
Fahrenheit: [ 77 ]
```

The two inputs represent the same underlying temperature.

Question:

> Where should the temperature state live?

Think through:

```text
CelsiusInput
FahrenheitInput
TemperatureDisplay
```

Which components need access to the same source of truth?

The answer leads directly to lifting state up.

---

# 28. The architectural pattern to remember

For shared state:

```text
              Parent
            state owner
           /    |     \
          ↓     ↓      ↓
       Child  Child  Child
          ↑            |
          └ callbacks ┘
```

State/data goes down.

Events are communicated back up through callbacks.

The parent remains the source of truth.

---

## Next lesson — Forms in depth + state design

We've touched forms already, but now we'll build them properly.

We'll cover:

```text
controlled vs uncontrolled inputs
multiple fields
checkboxes
selects
textarea
form submission
validation
field-level errors
form-level state
```

More importantly, we'll use forms to reinforce **state ownership and data flow**, because forms are one of the best places to learn whether you genuinely understand React rather than just recognizing syntax.