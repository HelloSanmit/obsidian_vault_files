
# Lesson 5 — Callbacks, Closures, `map()`, `filter()`, `find()`, and `reduce()`

Before we go deeper into React architecture, let's make **JavaScript callbacks feel completely natural**.

This is worth doing because React code often looks like:

```jsx
items.map(item => ...)
```

```jsx
items.filter(item => ...)
```

```jsx
onClick={() => doSomething(id)}
```

```jsx
setItems(items => ...)
```

These all involve functions being passed around, but each function has a different job.

Once that becomes intuitive, React code becomes much easier to read.

---

# 1. What is a callback?

A callback is simply:

> **A function that you give to another function so that the other function can call it later.**

For example:

```js
function greet() {
  console.log("Hello");
}

function runSomething(fn) {
  fn();
}

runSomething(greet);
```

The important part:

```js
runSomething(greet);
```

We are **not** doing:

```js
runSomething(greet());
```

We're passing the function itself.

So:

```text
greet
```

means:

> "Here is the function."

while:

```text
greet()
```

means:

> "Run the function now."

That's the same distinction you saw with React:

```jsx
onClick={handleClick}
```

versus:

```jsx
onClick={handleClick()}
```

---

# 2. Why are functions so useful as values?

JavaScript lets us store functions in variables:

```js
const sayHello = () => {
  console.log("Hello");
};
```

put them in objects:

```js
const user = {
  name: "Alex",
  greet: sayHello
};
```

put them in arrays:

```js
const actions = [
  sayHello,
  someOtherFunction
];
```

and pass them to other functions:

```js
runSomething(sayHello);
```

This idea is fundamental to React because a lot of React APIs accept functions.

---

# 3. Let's understand `map()` one more time

Suppose:

```js
const numbers = [1, 2, 3];
```

You write:

```js
const result = numbers.map(number => number * 10);
```

What's happening?

`map()` takes the callback:

```js
number => number * 10
```

and calls it for every item.

Conceptually:

```text
callback(1) → 10
callback(2) → 20
callback(3) → 30
```

So:

```js
result = [10, 20, 30]
```

The callback's job is:

> **Transform one item into another value.**

Therefore:

```text
map callback → one output value
map()        → new array of those output values
```

---

# 4. The callback doesn't need to return the same type

For example:

```js
const numbers = [1, 2, 3];

const words = numbers.map(number => {
  return `Number: ${number}`;
});
```

Result:

```js
[
  "Number: 1",
  "Number: 2",
  "Number: 3"
]
```

Input type:

```text
number
```

Output type:

```text
string
```

You can even return objects:

```js
const users = [
  { name: "Alex", age: 25 },
  { name: "Sarah", age: 30 }
];

const names = users.map(user => ({
  name: user.name
}));
```

Result:

```js
[
  { name: "Alex" },
  { name: "Sarah" }
]
```

---

# 5. Why `map()` fits React so well

Now:

```jsx
const users = [
  { id: 1, name: "Alex" },
  { id: 2, name: "Sarah" }
];
```

and:

```jsx
users.map(user => (
  <UserCard
    key={user.id}
    user={user}
  />
))
```

The callback takes:

```text
one user
```

and transforms it into:

```text
one React element
```

Therefore:

```text
users array
    ↓
map
    ↓
array of React elements
```

React can render that array.

---

# 6. `filter()` is different

Now:

```js
const numbers = [1, 2, 3, 4, 5];

const result = numbers.filter(number => number > 3);
```

The callback returns:

```text
true / false
```

For each number:

```text
1 → false
2 → false
3 → false
4 → true
5 → true
```

`filter()` keeps the values for which the callback returned `true`.

Result:

```js
[4, 5]
```

So:

```text
map:
item → transformed item

filter:
item → true/false
```

This distinction is extremely important.

---

# 7. `filter()` does not transform items

Suppose:

```js
const users = [
  { name: "Alex", age: 20 },
  { name: "Sarah", age: 30 },
  { name: "Mike", age: 17 }
];
```

Then:

```js
const adults = users.filter(user => user.age >= 18);
```

The callback returns:

```text
Alex  → true
Sarah → true
Mike  → false
```

The resulting array contains the **original objects**:

```js
[
  { name: "Alex", age: 20 },
  { name: "Sarah", age: 30 }
]
```

It doesn't turn each object into some new thing.

Its job is simply deciding:

> Keep or discard?

---

# 8. `find()` is another cousin

Suppose:

```js
const users = [
  { id: 1, name: "Alex" },
  { id: 2, name: "Sarah" },
  { id: 3, name: "Mike" }
];
```

You want user `2`.

```js
const user = users.find(user => user.id === 2);
```

The callback asks:

```text
Does this item match?
```

React/JavaScript roughly checks:

```text
Alex  → false
Sarah → true
```

Then `find()` stops and returns:

```js
{ id: 2, name: "Sarah" }
```

So:

```text
filter → all matching items
find   → first matching item
```

And if nothing matches:

```js
undefined
```

---

# 9. `map`, `filter`, and `find` side by side

Take:

```js
const numbers = [1, 2, 3, 4, 5];
```

### `map`

```js
numbers.map(n => n * 2);
```

returns:

```js
[2, 4, 6, 8, 10]
```

Question being answered:

> "What should each item become?"

---

### `filter`

```js
numbers.filter(n => n > 3);
```

returns:

```js
[4, 5]
```

Question:

> "Which items should remain?"

---

### `find`

```js
numbers.find(n => n > 3);
```

returns:

```js
4
```

Question:

> "What is the first item that matches?"

---

# 10. `reduce()` is more general

Now we get to the one that often looks scary:

```js
const numbers = [1, 2, 3, 4];

const total = numbers.reduce(
  (sum, number) => sum + number,
  0
);
```

Result:

```text
10
```

The easiest way to understand `reduce()` is:

> **Take many values and progressively combine them into one result.**

Let's trace it.

Initial accumulator:

```text
sum = 0
```

Then:

```text
0 + 1 = 1
```

Then:

```text
1 + 2 = 3
```

Then:

```text
3 + 3 = 6
```

Then:

```text
6 + 4 = 10
```

Final result:

```text
10
```

---

# 11. What are the arguments to these callbacks?

This is another source of confusion.

For:

```js
numbers.map((number, index) => ...)
```

the callback can receive:

```text
number → current item
index  → current position
```

For example:

```js
numbers.map((number, index) => {
  console.log(number, index);
});
```

might produce:

```text
1 0
2 1
3 2
```

There is also the whole array available as another callback argument:

```js
numbers.map((number, index, array) => ...)
```

You usually won't need the third argument.

Same idea applies to `filter()`:

```js
numbers.filter((number, index) => ...)
```

---

# 12. This is why React code can look dense

Look at:

```jsx
tasks.map((task, index) => (
  <TaskItem
    key={task.id}
    task={task}
  />
))
```

There are actually **three separate concepts** packed together:

### 1. `map()`

```text
Take every task.
```

### 2. Callback

```js
task => ...
```

```text
Here's what to produce for one task.
```

### 3. Component

```jsx
<TaskItem ... />
```

```text
The output for that task is a React element.
```

So don't mentally process that as one mysterious React construct.

Break it apart:

```text
array
 ↓
map
 ↓
callback receives one item
 ↓
callback returns JSX
 ↓
new array of JSX
```

---

# 13. Now let's talk about closures

This is the other JavaScript concept that causes a lot of React confusion.

Consider:

```js
function createGreeting(name) {
  return function() {
    console.log(`Hello ${name}`);
  };
}
```

Then:

```js
const greetAlex = createGreeting("Alex");
```

Later:

```js
greetAlex();
```

outputs:

```text
Hello Alex
```

How does the returned function still know `name`?

Because the function **closed over** `name`.

That's a closure.

---

# 14. Why closures matter in React

Look at:

```jsx
function TaskItem({ task, onToggle }) {
  return (
    <input
      type="checkbox"
      onChange={() => onToggle(task.id)}
    />
  );
}
```

That arrow function:

```js
() => onToggle(task.id)
```

uses:

```text
onToggle
task.id
```

from the surrounding component function.

That's a closure.

The function remembers the values from the render in which it was created.

This becomes extremely important when we talk about state.

---

# 15. Let's make the closure visible

Suppose:

```jsx
function Button() {
  const name = "Alex";

  function handleClick() {
    console.log(name);
  }

  return (
    <button onClick={handleClick}>
      Click
    </button>
  );
}
```

You don't call:

```js
handleClick()
```

while rendering.

You give React the function:

```jsx
onClick={handleClick}
```

Later, when the user clicks, React invokes it.

The function still has access to:

```text
name = "Alex"
```

because of the closure.

---

# 16. Closures + state

This is where React's "snapshot" model becomes much easier to understand.

Suppose:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log(count);
  }

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

On the first render:

```text
count = 0
```

and `handleClick` is created in that render.

That function closes over:

```text
count = 0
```

After:

```js
setCount(1)
```

React renders the component again.

Now a **new** `handleClick` function is created, associated with the new render:

```text
count = 1
```

So you can think:

```text
Render #1
    count = 0
    handleClick₁ remembers 0

Render #2
    count = 1
    handleClick₂ remembers 1
```

That's one reason the "render is a snapshot" model is so powerful.

---

# 17. This explains a common React surprise

Consider:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);

    setTimeout(() => {
      console.log(count);
    }, 1000);
  }

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

Suppose count is `0`.

You click once.

The timeout callback was created inside the render/event execution where `count` was `0`.

So after a second, it logs:

```text
0
```

even though the screen may now show:

```text
1
```

That's because the callback closed over the value from that particular execution.

This is a huge part of understanding React's behavior with async code.

We will revisit this when we study `useEffect`.

---

# 18. Now connect callbacks to React props

Suppose parent:

```jsx
function App() {
  function handleDelete(id) {
    console.log("Deleting", id);
  }

  return (
    <TaskItem
      task={task}
      onDelete={handleDelete}
    />
  );
}
```

The parent passes a function as a prop.

The child receives it:

```jsx
function TaskItem({ task, onDelete }) {
  return (
    <button onClick={() => onDelete(task.id)}>
      Delete
    </button>
  );
}
```

There are now **two callbacks**.

First:

```js
() => onDelete(task.id)
```

belongs to the click event.

Second:

```js
onDelete
```

is the function passed from the parent.

Flow:

```text
User clicks
    ↓
child's arrow function
    ↓
onDelete(task.id)
    ↓
parent's handleDelete()
```

Once you can trace that chain, child → parent communication becomes straightforward.

---

# 19. A useful debugging technique

When React code looks confusing, temporarily rewrite callbacks with named functions.

Instead of:

```jsx
onChange={e => setFilter(e.target.value)}
```

mentally expand it:

```js
function handleFilterChange(e) {
  setFilter(e.target.value);
}
```

then:

```jsx
onChange={handleFilterChange}
```

It's easier to understand.

Likewise:

```jsx
tasks.map(task => (
  <TaskItem task={task} />
))
```

mentally become:

```js
function convertTaskToUI(task) {
  return <TaskItem task={task} />;
}

tasks.map(convertTaskToUI);
```

You don't have to literally rewrite your code every time.

Just use the expanded form when reading.

---

# 20. A reading strategy I want you to adopt

Whenever you see:

```js
something.someMethod(callback)
```

stop.

Identify:

### What is `something`?

```text
array?
object?
React state?
```

### What is `someMethod`?

```text
map?
filter?
find?
reduce?
```

### What does that method expect the callback to return?

For example:

```text
map    → transformed value
filter → boolean
find   → boolean
reduce → accumulated value
```

### What does the entire method return?

```text
map    → array
filter → array
find   → item / undefined
reduce → whatever final accumulator is
```

This four-step analysis will make a lot of code immediately readable.

---

# 21. Apply that to your original example

Take:

```js
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

Read it mechanically:

### `tasks`

An array.

### `.filter()`

Keep some elements.

### Callback

```js
task => {
  ...
}
```

Receives one task.

### Callback return

Boolean:

```text
true → keep
false → discard
```

### Whole `filter()` result

Array.

### Therefore:

```text
visibleTasks = array of tasks
```

No React magic required.

---

Now:

```jsx
visibleTasks.map(task => (
  <TaskItem
    key={task.id}
    task={task}
    onToggle={toggleTask}
  />
))
```

### `visibleTasks`

Array.

### `.map()`

Transform every element.

### Callback

```text
task → ...
```

### Callback result

A React element:

```jsx
<TaskItem ... />
```

### Whole `map()` result

An array of React elements.

React renders them.

Again, no magic.

---

# 22. And then this:

```jsx
onChange={() => onToggle(task.id)}
```

is different.

There's no `map`, `filter`, or `reduce`.

This is an **event callback**.

Its job is simply:

```text
wait for event
    ↓
execute callback
    ↓
call onToggle(task.id)
```

That's why you should never lump all callbacks together.

They are all functions, but they have different contracts.

---

# The callback mental model

Start seeing code like this:

```text
┌────────────────┬──────────────────────┐
│ Function       │ Job                  │
├────────────────┼──────────────────────┤
│ map callback   │ transform each item  │
│ filter callback│ keep/discard item    │
│ find callback  │ test for a match     │
│ reduce callback│ accumulate result    │
│ event callback │ respond to an event  │
│ state updater  │ calculate next state │
│ component      │ describe UI          │
└────────────────┴──────────────────────┘
```

They're all just JavaScript functions.

The surrounding API determines what role they play.

---

# One last React-specific insight

You're probably noticing that React code often contains something like:

```jsx
<Component
  data={data}
  onSomething={handleSomething}
/>
```

Notice the asymmetry:

```text
data
  ↓
component

function
  ↑
component
```

The first prop gives the child **information**.

The second gives the child a way to **trigger behavior**.

That's the foundation for the next topic.

---

# Next Lesson — State Ownership & Lifting State Up

We'll take this exact idea and answer a genuinely important React design question:

```text
App
├── SearchBox
└── ProductList
```

The `SearchBox` knows what the user typed, but `ProductList` needs that information.

Should `SearchBox` own the state?

Should `ProductList` own it?

Should `App` own it?

We'll learn how to decide **where state belongs**, how callbacks move information upward, and why React's one-way data flow leads to a very specific component architecture.