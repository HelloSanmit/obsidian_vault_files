
# Lesson 3 — Events, State, and Forms

You now understand:

```text
Component ≈ function
Props ≈ inputs
map() = JavaScript array transformation
State = React-managed data that persists across renders
```

The next step is to connect **user actions → state changes → UI updates**.

This is where React starts feeling like an actual application framework.

---

# 1. React events

You already saw:

```jsx
<button onClick={handleClick}>
  Click me
</button>
```

The important thing is that:

```jsx
onClick={handleClick}
```

means:

> "When this button is clicked, call `handleClick`."

For example:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      Count: {count}
    </button>
  );
}
```

The flow is:

```text
click
  ↓
handleClick()
  ↓
setCount(...)
  ↓
state changes
  ↓
component renders again
  ↓
new count appears
```

---

# 2. Don't call the function yourself

This is a very common mistake.

Correct:

```jsx
<button onClick={handleClick}>
```

Incorrect:

```jsx
<button onClick={handleClick()}>
```

Why?

The first means:

```text
"Give React this function."
```

The second means:

```text
"Call this function right now,
and give the result to onClick."
```

For example:

```jsx
function handleClick() {
  console.log("clicked");
}
```

This:

```jsx
<button onClick={handleClick}>
```

waits for the click.

This:

```jsx
<button onClick={handleClick()}>
```

calls `handleClick()` during rendering.

---

# 3. What if I need to pass an argument?

Suppose:

```jsx
function deleteUser(id) {
  console.log(id);
}
```

You might try:

```jsx
<button onClick={deleteUser(5)}>
```

That's wrong for the same reason: you're executing it immediately.

Instead:

```jsx
<button onClick={() => deleteUser(5)}>
```

Now React receives a function:

```text
click
 ↓
arrow function executes
 ↓
deleteUser(5)
```

So this pattern is extremely common:

```jsx
onClick={() => doSomething(value)}
```

---

# 4. Event objects

React gives event handlers an event object.

```jsx
function handleClick(event) {
  console.log(event);
}
```

Then:

```jsx
<button onClick={handleClick}>
  Click
</button>
```

The event contains information about what happened.

For example:

```jsx
function handleClick(event) {
  console.log(event.target);
}
```

`event.target` refers to the element that triggered the event.

You don't need to memorize every property. Learn them as needed.

---

# 5. Input events

This is where events and state become particularly useful.

Consider:

```jsx
function NameInput() {
  const [name, setName] = useState("");

  return (
    <input
      value={name}
      onChange={event => {
        setName(event.target.value);
      }}
    />
  );
}
```

Let's trace a single keystroke.

Suppose the user types:

```text
A
```

The browser generates an input/change event.

React calls:

```jsx
event => {
  setName(event.target.value);
}
```

Suppose:

```text
event.target.value = "A"
```

So:

```jsx
setName("A");
```

React renders again.

Now:

```jsx
value={name}
```

is:

```text
value="A"
```

The input displays `A`.

---

# 6. This is called a controlled input

This is an important React term.

```jsx
<input
  value={name}
  onChange={e => setName(e.target.value)}
/>
```

The input's value is controlled by React state.

Think of it as:

```text
               state
                 ↓
              value
                 ↓
              <input>
                 │
                 │ user types
                 ↓
              onChange
                 │
                 ↓
              setState
                 │
                 └────────→ state
```

That's a complete feedback loop.

---

# 7. Why call it "controlled"?

Because React is controlling the displayed value.

Consider:

```jsx
const [name, setName] = useState("");
```

and:

```jsx
value={name}
```

The input is essentially saying:

> "Whatever `name` currently is, that's what my value should be."

Then:

```jsx
onChange={e => setName(e.target.value)}
```

says:

> "Whenever the user edits me, update `name`."

So state and input stay synchronized.

---

# 8. A simple form

Let's build one.

```jsx
function SignupForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  function handleSubmit(event) {
    event.preventDefault();

    console.log(name);
    console.log(email);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}
        onChange={e => setName(e.target.value)}
        placeholder="Name"
      />

      <input
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="Email"
      />

      <button type="submit">
        Sign up
      </button>
    </form>
  );
}
```

There are several React concepts here.

---

# 9. `onSubmit`

Instead of putting logic directly on the button:

```jsx
<button onClick={...}>
```

forms generally use:

```jsx
<form onSubmit={handleSubmit}>
```

Then:

```jsx
<button type="submit">
```

causes the form's submit event.

This gives you a clean structure:

```text
form
 └── submit
      ↓
 handleSubmit()
```

---

# 10. Why `preventDefault()`?

Normally, submitting a traditional HTML form can cause browser-level form submission/navigation.

In a React single-page application, you often want to handle that submission yourself.

So:

```jsx
event.preventDefault();
```

prevents the browser's default behavior.

Then your React code can do whatever is appropriate:

```jsx
function handleSubmit(event) {
  event.preventDefault();

  // validate
  // send API request
  // show success
}
```

---

# 11. Multiple inputs

Imagine a form:

```text
Name
[____________]

Email
[____________]

Age
[____________]

Password
[____________]

[ Create account ]
```

You could have:

```jsx
const [name, setName] = useState("");
const [email, setEmail] = useState("");
const [age, setAge] = useState("");
const [password, setPassword] = useState("");
```

This is perfectly valid.

Then:

```jsx
<input
  value={name}
  onChange={e => setName(e.target.value)}
/>
```

and similarly for the others.

For small forms, this is often completely reasonable.

---

# 12. One state object for a form

You can also use one object:

```jsx
const [form, setForm] = useState({
  name: "",
  email: "",
  age: "",
  password: ""
});
```

Updating `name`:

```jsx
setForm({
  ...form,
  name: e.target.value
});
```

Updating `email`:

```jsx
setForm({
  ...form,
  email: e.target.value
});
```

Remember why the spread is there:

```jsx
...form
```

preserves the other fields.

Without it:

```jsx
setForm({
  name: e.target.value
});
```

you would replace the entire object and lose:

```text
email
age
password
```

---

# 13. A very useful pattern: dynamic field names

You can make form handling less repetitive.

```jsx
function handleChange(event) {
  const { name, value } = event.target;

  setForm({
    ...form,
    [name]: value
  });
}
```

Then:

```jsx
<input
  name="name"
  value={form.name}
  onChange={handleChange}
/>

<input
  name="email"
  value={form.email}
  onChange={handleChange}
/>
```

Why does this work?

Suppose:

```text
name = "email"
value = "alex@example.com"
```

Then:

```js
[name]: value
```

becomes:

```js
email: "alex@example.com"
```

This is ordinary JavaScript computed property syntax.

React isn't doing anything special here.

---

# 14. Checkboxes are slightly different

For a checkbox:

```jsx
const [isSubscribed, setIsSubscribed] = useState(false);

<input
  type="checkbox"
  checked={isSubscribed}
  onChange={e => setIsSubscribed(e.target.checked)}
/>
```

Notice:

```jsx
checked
```

rather than:

```jsx
value
```

because the important state is boolean:

```text
true / false
```

And:

```jsx
e.target.checked
```

gives you that boolean.

---

# 15. Selects

Example:

```jsx
const [country, setCountry] = useState("");

return (
  <select
    value={country}
    onChange={e => setCountry(e.target.value)}
  >
    <option value="">Choose country</option>
    <option value="india">India</option>
    <option value="japan">Japan</option>
    <option value="germany">Germany</option>
  </select>
);
```

Same fundamental pattern:

```text
state → value
event → state update
```

That's one of the most useful patterns in React.

---

# 16. Validation

Now we can add validation.

```jsx
function SignupForm() {
  const [email, setEmail] = useState("");
  const [error, setError] = useState("");

  function handleSubmit(event) {
    event.preventDefault();

    if (!email.includes("@")) {
      setError("Please enter a valid email.");
      return;
    }

    setError("");
    console.log("Submit!");
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={email}
        onChange={e => setEmail(e.target.value)}
      />

      {error && <p>{error}</p>}

      <button type="submit">
        Submit
      </button>
    </form>
  );
}
```

Notice how naturally today's concepts combine:

```text
input
 ↓
state
 ↓
submit
 ↓
validation
 ↓
state update
 ↓
conditional rendering
```

This is actual React application logic.

---

# 17. Events aren't limited to buttons

You'll use many event types:

```jsx
onClick
onChange
onSubmit
onFocus
onBlur
onKeyDown
onKeyUp
onMouseEnter
```

For example:

```jsx
<input
  onKeyDown={e => {
    if (e.key === "Enter") {
      console.log("Enter pressed");
    }
  }}
/>
```

Again, the important skill isn't memorizing every event.

It's understanding the pattern:

```text
event occurs
    ↓
handler runs
    ↓
read event data
    ↓
update state / perform action
```

---

# 18. A very important distinction: event vs effect

You'll soon encounter `useEffect`.

Before that, understand:

### Event

Something happens because the user or browser triggers an interaction.

```text
click
submit
type
focus
```

For example:

```jsx
function handleSubmit() {
  // do something because the user submitted
}
```

### Effect

Something your component needs to synchronize with an external system after rendering.

We'll study this carefully later.

A common beginner mistake is to put ordinary event logic into `useEffect`.

Don't.

If something happens **because the user clicked a button**, the first place to look is an event handler.

---

# 19. Example: don't use an effect unnecessarily

Suppose clicking "Buy" should submit an order.

Correct conceptual structure:

```jsx
<button onClick={handleBuy}>
  Buy
</button>
```

and:

```jsx
function handleBuy() {
  // submit order
}
```

You generally don't need:

```jsx
useEffect(() => {
  if (shouldBuy) {
    // submit order
  }
}, [shouldBuy]);
```

That introduces an unnecessary state/effect chain.

This distinction becomes extremely important when we reach `useEffect`.

---

# 20. Events + props

Suppose the parent wants the child to trigger something.

Parent:

```jsx
function App() {
  function handleDelete() {
    console.log("delete");
  }

  return <DeleteButton onDelete={handleDelete} />;
}
```

Child:

```jsx
function DeleteButton({ onDelete }) {
  return (
    <button onClick={onDelete}>
      Delete
    </button>
  );
}
```

Now the flow is:

```text
App owns behavior
      ↓
passes callback as prop
      ↓
DeleteButton receives callback
      ↓
user clicks
      ↓
callback runs
```

This is one of the most important patterns in React.

---

# 21. Events + props + arguments

Suppose the child needs to tell the parent _which_ product was deleted.

Parent:

```jsx
function App() {
  function handleDelete(productId) {
    console.log("Delete:", productId);
  }

  return (
    <ProductList
      onDelete={handleDelete}
    />
  );
}
```

Child:

```jsx
function ProductList({ onDelete }) {
  const products = [
    { id: 1, name: "Laptop" },
    { id: 2, name: "Phone" }
  ];

  return (
    <div>
      {products.map(product => (
        <button
          key={product.id}
          onClick={() => onDelete(product.id)}
        >
          Delete {product.name}
        </button>
      ))}
    </div>
  );
}
```

The flow:

```text
Product 2 clicked
      ↓
onClick callback
      ↓
onDelete(2)
      ↓
parent's handleDelete(2)
```

This is the practical meaning of:

> **Data goes down; events/callbacks can communicate information back up.**

---

# 22. One complete example

Let's combine everything into a small searchable task application.

```jsx
import { useState } from "react";

function TaskList({ tasks, onDelete }) {
  return (
    <ul>
      {tasks.map(task => (
        <li key={task.id}>
          {task.name}

          <button
            onClick={() => onDelete(task.id)}
          >
            Delete
          </button>
        </li>
      ))}
    </ul>
  );
}

function App() {
  const [text, setText] = useState("");

  const [tasks, setTasks] = useState([
    { id: 1, name: "Learn React" },
    { id: 2, name: "Build a project" }
  ]);

  function addTask(event) {
    event.preventDefault();

    if (!text.trim()) {
      return;
    }

    setTasks(tasks => [
      ...tasks,
      {
        id: Date.now(),
        name: text
      }
    ]);

    setText("");
  }

  function deleteTask(id) {
    setTasks(tasks =>
      tasks.filter(task => task.id !== id)
    );
  }

  return (
    <div>
      <h1>Tasks</h1>

      <form onSubmit={addTask}>
        <input
          value={text}
          onChange={e => setText(e.target.value)}
        />

        <button type="submit">
          Add
        </button>
      </form>

      <TaskList
        tasks={tasks}
        onDelete={deleteTask}
      />
    </div>
  );
}

export default App;
```

Don't worry about understanding every architectural detail yet.

Look at the data flow:

```text
                    App
                     │
            ┌────────┴─────────┐
            ↓                  ↓
        text state          tasks state
            │                  │
            ↓                  ↓
         input             TaskList
            │                  │
        onChange               │
            │                  │
            ↓                  │
       setText()               │
                               │
                         onDelete prop
                               ↑
                          button click
```

That's a real React application in miniature.

---

# The key ideas from this lesson

You should now be comfortable with this pattern:

```jsx
<input
  value={value}
  onChange={e => setValue(e.target.value)}
/>
```

and:

```jsx
<button onClick={handleClick}>
```

and:

```jsx
<form onSubmit={handleSubmit}>
```

More importantly, understand the underlying flow:

```text
USER ACTION
    ↓
EVENT
    ↓
EVENT HANDLER
    ↓
STATE UPDATE
    ↓
RENDER
    ↓
UPDATED UI
```

And when a child needs to trigger parent-owned logic:

```text
parent
  ↓ callback prop
child
  ↓ event
callback
  ↓
parent state update
```

---

## Next: conditional rendering + lists + keys, properly

You already saw `map()`, but now we'll put it together with state and conditional UI.

We'll build something like:

```text
Tasks
─────────────────────────

[ Learn React       ] [✓]

[ Add task ]

Show:
  All | Active | Completed

3 tasks
```

That lesson will introduce an important idea: **derived data** — why something like `filteredTasks` usually does not need its own state. It will also give you the proper mental model for React `key`s rather than treating them as a mysterious warning-silencer.