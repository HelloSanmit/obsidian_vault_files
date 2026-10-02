
# Lesson 4 — Conditional Rendering, Lists, Keys, and Derived Data

Today we're going to combine several things you've already learned:

```text
components
props
state
events
map()
```

and use them to build a realistic UI.

The main concepts are:

1. Conditional rendering
    
2. Rendering lists
    
3. `key`
    
4. Derived data
    
5. Why you shouldn't put every value into state
    

---

# 1. Conditional rendering

You already know JavaScript has:

```js
if (condition) {
  ...
}
```

React uses the same JavaScript logic to decide what UI to return.

For example:

```jsx
function Greeting({ isLoggedIn }) {
  if (isLoggedIn) {
    return <h1>Welcome back!</h1>;
  }

  return <h1>Please log in.</h1>;
}
```

If:

```jsx
<Greeting isLoggedIn={true} />
```

we get:

```text
Welcome back!
```

If:

```jsx
<Greeting isLoggedIn={false} />
```

we get:

```text
Please log in.
```

Nothing special is happening here.

It's ordinary JavaScript controlling which JSX gets returned.

---

# 2. Ternary operator

Often you want two alternatives inside JSX.

```jsx
function Greeting({ isLoggedIn }) {
  return (
    <h1>
      {isLoggedIn ? "Welcome back!" : "Please log in."}
    </h1>
  );
}
```

The structure is:

```text
condition ? valueIfTrue : valueIfFalse
```

So:

```jsx
isLoggedIn
  ? "Welcome back!"
  : "Please log in."
```

means:

```text
if true  → Welcome back!
if false → Please log in.
```

You can return entire components too:

```jsx
{isLoggedIn ? <Dashboard /> : <Login />}
```

---

# 3. `&&` conditional rendering

Suppose you want to show something only when a condition is true.

```jsx
{hasError && <p>Something went wrong.</p>}
```

Think of JavaScript:

```js
hasError && something
```

If `hasError` is false, the expression doesn't produce the right-hand side.

So:

```text
hasError = true
        ↓
<p>Something went wrong.</p>
```

and:

```text
hasError = false
        ↓
nothing is rendered
```

This is extremely common.

---

# 4. Be careful with numbers and `&&`

Here's a classic gotcha.

Suppose:

```jsx
const tasks = [];
```

and you write:

```jsx
{tasks.length && <p>Tasks exist</p>}
```

You might think:

```text
0 → nothing
```

But JavaScript actually evaluates:

```text
0
```

and React can render `0`.

So you might see:

```text
0
```

instead of nothing.

A safer expression is:

```jsx
{tasks.length > 0 && (
  <p>Tasks exist</p>
)}
```

This matters enough to remember.

---

# 5. Conditional UI is everywhere

Real applications often have several states.

Imagine a page loading users from a server:

```text
loading
   ↓
success / error
```

Your UI might be:

```jsx
if (loading) {
  return <Spinner />;
}

if (error) {
  return <ErrorMessage />;
}

if (users.length === 0) {
  return <EmptyState />;
}

return <UserList users={users} />;
```

Notice something important:

We're describing **what UI corresponds to each state**.

That's very much the React way of thinking.

---

# 6. Lists

Now let's return to `map()`.

Suppose:

```js
const tasks = [
  { id: 1, name: "Learn React" },
  { id: 2, name: "Build a project" },
  { id: 3, name: "Learn TypeScript" }
];
```

You can render them:

```jsx
function TaskList({ tasks }) {
  return (
    <ul>
      {tasks.map(task => (
        <li key={task.id}>
          {task.name}
        </li>
      ))}
    </ul>
  );
}
```

You already understand the `map()` part:

```text
task 1 → <li>Learn React</li>
task 2 → <li>Build a project</li>
task 3 → <li>Learn TypeScript</li>
```

The new part is:

```jsx
key={task.id}
```

So let's understand that properly.

---

# 7. What is a `key`?

Imagine React rendered:

```text
Learn React
Build a project
Learn TypeScript
```

Then on the next render the array becomes:

```text
Learn React
Learn TypeScript
```

The middle item disappeared.

React needs to reconcile the old list with the new list.

It needs to answer:

```text
Which old item corresponds to which new item?
Which item was removed?
Which one stayed?
Which one was inserted?
```

Keys give React an identity for each item.

```text
Learn React       → key 1
Build a project   → key 2
Learn TypeScript  → key 3
```

After deletion:

```text
Learn React       → key 1
Learn TypeScript  → key 3
```

React can recognize:

```text
key 2 disappeared
key 1 remains
key 3 remains
```

That's fundamentally what the key is helping with.

---

# 8. Key is about identity, not position

Consider:

```text
Before:

A
B
C
```

Then we insert:

```text
X
A
B
C
```

If React thinks primarily in terms of position:

```text
position 0 → A
position 1 → B
position 2 → C
```

then inserting something at the beginning changes every position.

But with stable keys:

```text
A → key A
B → key B
C → key C
X → key X
```

React can understand that:

```text
A is still A
B is still B
C is still C
```

even though their positions changed.

This becomes particularly important when list items contain state.

We'll revisit that in the rendering/reconciliation section.

---

# 9. Why not use the array index?

You can technically write:

```jsx
{tasks.map((task, index) => (
  <li key={index}>
    {task.name}
  </li>
))}
```

Sometimes this is fine, particularly for a genuinely static list whose order never changes.

But consider a list where items can be inserted, deleted, or reordered.

Initial list:

```text
A
B
C
```

keys based on index:

```text
A → 0
B → 1
C → 2
```

Remove A:

```text
B → 0
C → 1
```

Now React sees:

```text
old key 0 = A
new key 0 = B
```

The identity associated with that position has changed.

That can cause subtle problems when list items contain local state, input fields, focus, animations, etc.

So when your data has a stable ID:

```jsx
key={task.id}
```

is usually what you want.

---

# 10. Keys should be stable

Avoid:

```jsx
key={Math.random()}
```

because it creates a new identity every render.

Also avoid generating a new random ID solely for the key during rendering.

The key should usually come from the data itself:

```js
{
  id: 123,
  name: "Learn React"
}
```

Then:

```jsx
key={task.id}
```

---

# 11. A key is not a normal prop

This is a subtle point.

Suppose:

```jsx
<UserCard key={user.id} user={user} />
```

Inside `UserCard`:

```jsx
function UserCard(props) {
  console.log(props.key);
}
```

you should not expect `key` to behave like an ordinary prop.

That's because `key` is special information React uses for reconciliation.

If the component itself needs an identifier, pass it explicitly:

```jsx
<UserCard
  key={user.id}
  userId={user.id}
/>
```

Then:

```jsx
function UserCard({ userId }) {
  // ...
}
```

So:

```text
key
```

is for React's identity mechanism.

```text
userId
```

is application data.

---

# 12. Now the interesting part: derived data

Let's create a task app.

Suppose state is:

```jsx
const [tasks, setTasks] = useState([
  { id: 1, name: "Learn React", completed: true },
  { id: 2, name: "Build a project", completed: false },
  { id: 3, name: "Learn TypeScript", completed: false }
]);
```

And we have:

```jsx
const [filter, setFilter] = useState("all");
```

We want:

```text
All
Active
Completed
```

The visible tasks depend on:

```text
tasks
+
filter
```

So we could write:

```jsx
const visibleTasks = tasks.filter(task => {
  if (filter === "completed") {
    return task.completed;
  }

  if (filter === "active") {
    return !task.completed;
  }

  return true;
});
```

Then:

```jsx
{visibleTasks.map(task => (
  <TaskItem key={task.id} task={task} />
))}
```

Now notice something:

`visibleTasks` is not state.

Why?

Because we can **calculate it from existing state**.

---

# 13. Don't store what you can calculate

This is one of the most important React design principles.

We already have:

```text
tasks
filter
```

From those we can calculate:

```text
visibleTasks
```

So storing:

```jsx
const [visibleTasks, setVisibleTasks] = useState([]);
```

would create a second source of truth.

Now you'd have to keep these synchronized:

```text
tasks
   ↓
visibleTasks
```

That's unnecessary complexity.

Instead:

```text
tasks ─────┐
           ├──→ visibleTasks
filter ────┘
```

`visibleTasks` is **derived data**.

---

# 14. Another example

Suppose:

```jsx
const [firstName, setFirstName] = useState("Alex");
const [lastName, setLastName] = useState("Morgan");
```

You don't need:

```jsx
const [fullName, setFullName] = useState("Alex Morgan");
```

because:

```jsx
const fullName = `${firstName} ${lastName}`;
```

is enough.

Why create state that duplicates information you already have?

You now have:

```text
firstName = Alex
lastName = Morgan
fullName = Alex Morgan
```

and potentially:

```text
firstName = Sarah
lastName = Morgan
fullName = Alex Morgan
```

if you forget to synchronize them.

Derived values avoid that class of bug.

---

# 15. State vs derived data

A useful question is:

> **Can I calculate this value from props, state, or other existing data during render?**

If yes, you often don't need another state variable.

For example:

```text
products              → state
searchTerm            → state
selectedCategory      → state
filteredProducts      → derived
totalPrice            → derived
isEmpty               → derived
```

Potentially:

```jsx
const filteredProducts =
  products.filter(...);

const totalPrice =
  cart.reduce(...);

const isEmpty =
  filteredProducts.length === 0;
```

That's clean.

---

# 16. Derived data isn't necessarily expensive

People sometimes say:

> "But calculating it again on every render is inefficient."

Usually, start simple.

For:

```js
products.filter(...)
```

on a few dozen or hundred items, the calculation may be trivial.

Don't immediately turn everything into:

```jsx
useMemo(...)
```

We'll learn optimization later, when there's an actual reason for it.

First get the data model right.

---

# 17. Let's build the task filter

State:

```jsx
const [filter, setFilter] = useState("all");

const [tasks, setTasks] = useState([
  { id: 1, name: "Learn React", completed: true },
  { id: 2, name: "Build project", completed: false },
  { id: 3, name: "Practice JavaScript", completed: false }
]);
```

Derived:

```jsx
const visibleTasks = tasks.filter(task => {
  if (filter === "completed") {
    return task.completed;
  }

  if (filter === "active") {
    return !task.completed;
  }

  return true;
});
```

UI:

```jsx
<select
  value={filter}
  onChange={e => setFilter(e.target.value)}
>
  <option value="all">All</option>
  <option value="active">Active</option>
  <option value="completed">Completed</option>
</select>
```

And:

```jsx
<ul>
  {visibleTasks.map(task => (
    <li key={task.id}>
      {task.name}
    </li>
  ))}
</ul>
```

Look at the chain:

```text
filter state
     ↓
visibleTasks derived data
     ↓
map()
     ↓
list of UI elements
```

That's a very typical React pattern.

---

# 18. Add completion

Let's make each task interactive.

```jsx
function TaskItem({ task, onToggle }) {
  return (
    <li>
      <label>
        <input
          type="checkbox"
          checked={task.completed}
          onChange={() => onToggle(task.id)}
        />

        {task.name}
      </label>
    </li>
  );
}
```

The parent:

```jsx
function App() {
  const [tasks, setTasks] = useState([
    { id: 1, name: "Learn React", completed: true },
    { id: 2, name: "Build project", completed: false },
    { id: 3, name: "Practice JavaScript", completed: false }
  ]);

  function toggleTask(id) {
    setTasks(tasks =>
      tasks.map(task =>
        task.id === id
          ? { ...task, completed: !task.completed }
          : task
      )
    );
  }

  // ...
}
```

Notice our familiar JavaScript:

```jsx
tasks.map(...)
```

We create a new array.

For the matching task:

```jsx
{ ...task, completed: !task.completed }
```

we create a new object with the opposite `completed` value.

For every other task:

```jsx
task
```

we keep the existing object.

---

# 19. Why this is such a useful pattern

We now have:

```text
tasks = source of truth
filter = source of truth
visibleTasks = derived
```

When the checkbox is clicked:

```text
click
 ↓
onToggle(id)
 ↓
setTasks(...)
 ↓
tasks changes
 ↓
visibleTasks is recalculated
 ↓
list is rendered again
```

Notice what we didn't do:

```text
manually remove an item from the DOM
manually add a CSS class
manually update the task count
manually move the item
```

We change the underlying data.

React renders UI from that data.

---

# 20. A complete example

Here's the whole idea together:

```jsx
import { useState } from "react";

function TaskItem({ task, onToggle }) {
  return (
    <li>
      <label>
        <input
          type="checkbox"
          checked={task.completed}
          onChange={() => onToggle(task.id)}
        />

        {task.name}
      </label>
    </li>
  );
}

function App() {
  const [filter, setFilter] = useState("all");

  const [tasks, setTasks] = useState([
    {
      id: 1,
      name: "Learn React",
      completed: true
    },
    {
      id: 2,
      name: "Build a project",
      completed: false
    },
    {
      id: 3,
      name: "Practice JavaScript",
      completed: false
    }
  ]);

  const visibleTasks = tasks.filter(task => {
    if (filter === "completed") {
      return task.completed;
    }

    if (filter === "active") {
      return !task.completed;
    }

    return true;
  });

  function toggleTask(id) {
    setTasks(tasks =>
      tasks.map(task =>
        task.id === id
          ? {
              ...task,
              completed: !task.completed
            }
          : task
      )
    );
  }

  return (
    <main>
      <h1>Tasks</h1>

      <select
        value={filter}
        onChange={e => setFilter(e.target.value)}
      >
        <option value="all">All</option>
        <option value="active">Active</option>
        <option value="completed">Completed</option>
      </select>

      <ul>
        {visibleTasks.map(task => (
          <TaskItem
            key={task.id}
            task={task}
            onToggle={toggleTask}
          />
        ))}
      </ul>
    </main>
  );
}

export default App;
```

This tiny application already contains a surprisingly large amount of React thinking:

```text
State
 ↓
events
 ↓
state updates
 ↓
derived data
 ↓
map()
 ↓
props
 ↓
components
 ↓
keys
 ↓
UI
```

---

# 21. One more important distinction: state vs props vs derived data

At this stage, start classifying values.

Imagine a product page:

```text
products
searchTerm
selectedCategory
filteredProducts
productCount
isEmpty
```

Ask:

### `products`

Does it change over time?

Maybe yes → **state**

### `searchTerm`

Does the user change it?

Yes → **state**

### `selectedCategory`

Does the user change it?

Yes → **state**

### `filteredProducts`

Can we calculate it?

Yes → **derived data**

### `productCount`

Can we calculate it?

Yes:

```js
filteredProducts.length
```

→ **derived data**

### `isEmpty`

Can we calculate it?

Yes:

```js
filteredProducts.length === 0
```

→ **derived data**

This way of thinking will become increasingly important as applications get complicated.

---

# 22. The deeper principle

React doesn't reward you for putting everything into `useState`.

In fact, a lot of bad React code comes from treating state as a storage box for every piece of data.

Instead, ask:

```text
Is this something that can change independently?
```

If yes, maybe it belongs in state.

Then:

```text
Can I calculate everything else from that state?
```

If yes, derive it.

So conceptually:

```text
                    Source of truth
                          │
                ┌─────────┴─────────┐
                ↓                   ↓
              state               props
                │                   │
                └─────────┬─────────┘
                          ↓
                    derived data
                          ↓
                         JSX
                          ↓
                          UI
```

This is one of the cleanest ways to think about React application state.

---

# 23. What you should know after this lesson

You should now understand why these patterns exist:

```jsx
{condition && <Something />}
```

```jsx
{condition ? <A /> : <B />}
```

```jsx
items.map(item => ...)
```

```jsx
key={item.id}
```

and:

```jsx
const filteredItems = items.filter(...);
```

More importantly:

> **Not everything that appears on the screen needs to be stored in state.**

Some things are simply calculated from the current state/props.

---

## Next lesson — State ownership and lifting state up

Now we're getting into one of the most important skills in React architecture.

We'll have something like:

```text
             App
          /       \
     SearchBox   ProductList
```

and ask:

> Search text affects `ProductList`, but the input lives in `SearchBox`. Who should own the state?

That leads to:

**lifting state up → callbacks → one-way data flow → component communication → deciding where state belongs.**

Once you understand that, you'll be able to design component trees instead of just writing individual components.