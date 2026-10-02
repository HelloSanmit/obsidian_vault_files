
# Lesson 2 — State: making React interactive

Now we get to the first concept that really separates React from simply writing JSX.

You already know:

```text
Component ≈ function
Props ≈ inputs
map() = normal JavaScript
```

Now we need to answer:

> **How does a component remember something that can change?**

That's what **state** is for.

---

## 1. Start without React

Imagine a normal JavaScript counter:

```js
let count = 0;

function increment() {
  count++;
}
```

After:

```js
increment();
```

we have:

```text
count = 1
```

The variable changed.

But imagine the screen contains:

```html
<h1 id="count">0</h1>
```

Changing the JavaScript variable doesn't automatically change the HTML.

You'd have to manually do something like:

```js
document.querySelector("#count").textContent = count;
```

So with vanilla JS, you're responsible for both:

```text
change the data
       +
change the DOM
```

---

# 2. React separates those concerns

With React, we can say:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return <h1>{count}</h1>;
}
```

Now our mental model becomes:

```text
state changes
    ↓
React knows about it
    ↓
React renders again
    ↓
UI reflects the new state
```

You don't manually find the `<h1>` and change its text.

---

# 3. `useState`

The syntax:

```jsx
const [count, setCount] = useState(0);
```

contains three ideas.

```text
count
```

is the current state value.

```text
setCount
```

is the function used to request a state update.

```text
0
```

is the initial value.

So initially:

```text
count = 0
```

After:

```jsx
setCount(1);
```

the next render sees:

```text
count = 1
```

---

# 4. Why does it return an array?

Because `useState()` conceptually gives you two things:

```js
[value, updateFunction]
```

For example:

```jsx
const [count, setCount] = useState(0);
```

is using JavaScript array destructuring.

You already know this JavaScript:

```js
const [a, b] = [10, 20];
```

So:

```text
useState(0)
     ↓
[0, someFunction]
     ↓
count     = 0
setCount  = someFunction
```

This is important:

**`useState` isn't introducing some strange new assignment syntax.**

You're using ordinary JavaScript destructuring with a React hook.

---

# 5. Your first interactive component

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount(count + 1);
  }

  return (
    <div>
      <h1>{count}</h1>

      <button onClick={increment}>
        Increase
      </button>
    </div>
  );
}
```

Now let's trace it.

---

## Initial render

React renders:

```jsx
<h1>{count}</h1>
```

and:

```text
count = 0
```

so the screen shows:

```text
0
```

---

## User clicks the button

This runs:

```js
increment()
```

which runs:

```js
setCount(count + 1);
```

Since:

```text
count = 0
```

that becomes:

```js
setCount(1);
```

React is notified that the state needs to change.

---

## React renders again

The component runs again.

This time:

```text
count = 1
```

Therefore:

```jsx
<h1>{count}</h1>
```

describes:

```html
<h1>1</h1>
```

The screen updates.

---

# 6. The most important React loop

Memorize this:

```text
         user action
              ↓
          event handler
              ↓
         state update
              ↓
      React renders again
              ↓
          new UI
```

For our counter:

```text
click
 ↓
increment()
 ↓
setCount(1)
 ↓
Counter renders
 ↓
<h1>1</h1>
```

That's the core interactive React model.

---

# 7. Why can't we use a normal variable?

This is a very common beginner experiment:

```jsx
function Counter() {
  let count = 0;

  function increment() {
    count++;
  }

  return (
    <>
      <h1>{count}</h1>
      <button onClick={increment}>
        Increase
      </button>
    </>
  );
}
```

You might think:

```text
click
 ↓
count++
 ↓
count is now 1
```

But the UI doesn't update.

There are actually two related problems.

### Problem 1: React doesn't know it should render again

Doing:

```js
count++;
```

doesn't tell React:

> "This component's UI needs updating."

### Problem 2: the component's local variables don't persist between renders

This is the deeper issue.

Whenever React renders the component, you're executing:

```js
function Counter() {
```

again.

So:

```js
let count = 0;
```

gets created again.

This is why **state is special**.

React keeps state associated with the component across renders.

---

# 8. This is the part you should really understand

Consider:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  console.log("render");

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

On the first render:

```text
Counter()
  ↓
count = 0
```

Click.

React schedules a state update.

Then conceptually:

```text
Counter()
  ↓
count = 1
```

Click again:

```text
Counter()
  ↓
count = 2
```

So a React component should **not** be thought of as one function execution that lives forever.

Instead, think:

```text
render #1
    ↓
component executes

render #2
    ↓
component executes again

render #3
    ↓
component executes again
```

State survives between those renders.

---

# 9. State is not the same thing as a variable

This distinction is crucial.

A normal variable:

```js
let count = 0;
```

is just a value created during that function execution.

React state:

```jsx
const [count, setCount] = useState(0);
```

is data React manages for that component instance.

That's why React can give you:

```text
render #1 → count = 0
render #2 → count = 1
render #3 → count = 2
```

---

# 10. Multiple state values

A component can have several pieces of state.

```jsx
function UserProfile() {
  const [name, setName] = useState("Alex");
  const [age, setAge] = useState(25);
  const [isOnline, setIsOnline] = useState(true);

  // ...
}
```

Conceptually:

```text
name     → "Alex"
age      → 25
isOnline → true
```

They are independent pieces of state.

---

# 11. State can hold any normal JavaScript value

Strings:

```jsx
const [name, setName] = useState("");
```

Numbers:

```jsx
const [count, setCount] = useState(0);
```

Booleans:

```jsx
const [isOpen, setIsOpen] = useState(false);
```

Arrays:

```jsx
const [todos, setTodos] = useState([]);
```

Objects:

```jsx
const [user, setUser] = useState({
  name: "Alex",
  age: 25
});
```

So `useState` doesn't mean "counter."

It means:

> Give this component a piece of state that React will preserve between renders.

---

# 12. State + conditional rendering

Now things get more interesting.

```jsx
function LoginStatus() {
  const [isLoggedIn, setIsLoggedIn] = useState(false);

  return (
    <div>
      {isLoggedIn ? (
        <p>Welcome back!</p>
      ) : (
        <p>Please log in.</p>
      )}

      <button onClick={() => setIsLoggedIn(!isLoggedIn)}>
        Toggle
      </button>
    </div>
  );
}
```

Initial state:

```text
isLoggedIn = false
```

UI:

```text
Please log in.
[Toggle]
```

Click:

```js
setIsLoggedIn(true)
```

React renders again.

UI:

```text
Welcome back!
[Toggle]
```

So now you can see the bigger relationship:

```text
state
  ↓
conditional logic
  ↓
UI
```

---

# 13. One state update can change lots of UI

Suppose:

```jsx
function Dashboard() {
  const [isDarkMode, setIsDarkMode] = useState(false);

  return (
    <div className={isDarkMode ? "dark" : "light"}>
      <h1>Dashboard</h1>

      <p>
        {isDarkMode ? "Dark mode" : "Light mode"}
      </p>

      <button onClick={() => setIsDarkMode(!isDarkMode)}>
        Toggle theme
      </button>
    </div>
  );
}
```

One state variable controls:

```text
CSS class
text
button behavior
```

You don't manually update all three.

The UI is derived from state.

That's a major React idea:

> **Don't manually synchronize every piece of the DOM. Describe what the UI should look like for the current state.**

---

# 14. A subtle problem: state updates don't immediately change the current variable

Consider:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);

    console.log(count);
  }

  return (
    <button onClick={handleClick}>
      {count}
    </button>
  );
}
```

What will the `console.log` show after the first click?

**0**, not 1.

Why?

Because:

```js
setCount(count + 1);
```

requests an update.

It doesn't mutate the current render's `count` variable.

The current render still has:

```text
count = 0
```

The next render will have:

```text
count = 1
```

This is a very important concept.

---

# 15. Think of each render as a snapshot

This mental model will save you a huge amount of confusion later.

Imagine React gives your component a snapshot:

```text
Render #1

count = 0
```

Every expression in that render sees:

```text
count = 0
```

Even if you call:

```js
setCount(1)
```

the existing render doesn't magically transform into:

```text
count = 1
```

Instead, React eventually gives you another render:

```text
Render #2

count = 1
```

So:

```text
render #1
count = 0

      ↓ setCount(1)

render #2
count = 1
```

This "snapshot" model is much more useful than thinking:

> "setState changes my variable."

It doesn't.

It schedules a new render with new state.

---

# 16. Now the weird example

Look at:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
  }

  // ...
}
```

You might expect:

```text
0 → 3
```

But that's not what happens.

Because all three lines are using the same `count` from the current render.

If:

```text
count = 0
```

then you're effectively asking three times:

```text
setCount(1)
setCount(1)
setCount(1)
```

So the resulting state is generally:

```text
1
```

not 3.

This is where **functional state updates** become important.

---

# 17. Functional state updates

Instead:

```jsx
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
```

Now each update says:

> Take whatever the latest state value is and increment it.

So React can process:

```text
0 → 1
1 → 2
2 → 3
```

Result:

```text
3
```

The function:

```js
c => c + 1
```

is called an **updater function**.

This is especially useful when your next state depends on the previous state.

---

# 18. Practical rule

When the new state depends on the old state:

```jsx
setCount(count + 1);
```

is often fine for simple one-off updates.

But:

```jsx
setCount(c => c + 1);
```

is the safer/general pattern when the new value depends on the previous state, particularly when multiple updates may be queued.

For example:

```jsx
setItems(items => [...items, newItem]);
```

or:

```jsx
setIsOpen(open => !open);
```

---

# 19. Don't mutate state

Suppose you have:

```jsx
const [user, setUser] = useState({
  name: "Alex",
  age: 25
});
```

Don't do:

```js
user.age = 26;
```

and expect React state management to work correctly.

Instead:

```jsx
setUser({
  ...user,
  age: 26
});
```

You're creating a new object.

Why this matters becomes much clearer when we discuss **immutability, references, and reconciliation**.

For now, remember:

> Treat state as immutable. Create new values when updating it.

---

# 20. Arrays in state

Suppose:

```jsx
const [todos, setTodos] = useState([]);
```

Add a todo:

```jsx
setTodos([
  ...todos,
  {
    id: 1,
    text: "Learn React"
  }
]);
```

Remove a todo:

```jsx
setTodos(
  todos.filter(todo => todo.id !== id)
);
```

Update one:

```jsx
setTodos(
  todos.map(todo =>
    todo.id === id
      ? { ...todo, completed: true }
      : todo
  )
);
```

Notice something familiar?

```text
map()
filter()
spread syntax
```

All of that is just JavaScript.

React's role is that **the new array is supplied to state**, causing a new render.

---

# 21. State can be passed as props

Now combine today's topic with the previous one.

Parent:

```jsx
function App() {
  const [count, setCount] = useState(0);

  return (
    <CounterDisplay count={count} />
  );
}
```

Child:

```jsx
function CounterDisplay({ count }) {
  return <h1>{count}</h1>;
}
```

The flow:

```text
App owns state
    ↓
count passed as prop
    ↓
CounterDisplay receives count
    ↓
UI displays it
```

This is the beginning of understanding **state ownership**.

---

# 22. State can also be updated through a child

Suppose:

```jsx
function App() {
  const [count, setCount] = useState(0);

  return (
    <Counter
      count={count}
      setCount={setCount}
    />
  );
}
```

Child:

```jsx
function Counter({ count, setCount }) {
  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
}
```

The state still belongs to `App`.

The child merely received a function that can request a change.

Later we'll improve this pattern by passing an intentionally named callback such as:

```jsx
onIncrement
```

rather than exposing `setCount` directly.

That leads into a more important lesson about component design.

---

# The core mental model for state

At this point, I want you to be able to visualize this:

```text
                  STATE
                    │
                    ↓
                Component
                    │
                    ↓
                   JSX
                    │
                    ↓
                    UI
                    │
                    │ user interacts
                    ↓
                  EVENT
                    │
                    ↓
              STATE UPDATE
                    │
                    └──────────────→ new render
```

And remember:

```text
State ≠ ordinary variable
```

State is information React preserves between renders and uses to determine the UI.

---

# Small exercise

Don't code yet. Just reason through this:

```jsx
function Counter() {
  const [count, setCount] = useState(10);

  function handleClick() {
    setCount(count + 5);
  }

  return (
    <>
      <h1>{count}</h1>
      <button onClick={handleClick}>
        Add 5
      </button>
    </>
  );
}
```

Starting at:

```text
count = 10
```

After one click:

```text
?
```

After two:

```text
?
```

After three:

```text
?
```

Then consider:

```jsx
function handleClick() {
  setCount(c => c + 5);
  setCount(c => c + 5);
}
```

Starting from `10`, what should the resulting count be?

Once that model is clear, the next topic is **events + state + forms**, where we'll build a small interactive form and see how data flows through a React application.