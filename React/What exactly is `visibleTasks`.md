
Yes. These are exactly the right questions to ask, because you're now hitting the point where React code starts looking confusing **not because React is difficult, but because several layers of ordinary JavaScript + React are nested together**.

Let's take this code apart very slowly.

---

# 1. What exactly is `visibleTasks`?

You have:

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

You asked:

> `visibleTasks` is a `const`, so what is it returning? Is it a boolean or the completed/active tasks?

The key distinction is:

### The callback returns a boolean.

### `filter()` returns an array.

### `visibleTasks` stores that array.

That's the whole thing.

Let's separate those three levels.

---

## 2. What does `filter()` do in JavaScript?

Forget React for a moment.

Suppose:

```js
const numbers = [1, 2, 3, 4, 5];
```

Now:

```js
const result = numbers.filter(number => number > 2);
```

The callback:

```js
number => number > 2
```

returns a boolean for each item.

Let's manually evaluate it:

```text
1 → 1 > 2 → false
2 → 2 > 2 → false
3 → 3 > 2 → true
4 → 4 > 2 → true
5 → 5 > 2 → true
```

Then `filter()` keeps the items where the callback returned `true`.

So:

```js
result
```

becomes:

```js
[3, 4, 5]
```

Therefore:

```text
callback return value → boolean
filter return value   → array
```

That's the distinction you were missing.

---

# 3. Now apply exactly the same idea to your tasks

Your tasks are:

```js
[
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
]
```

And:

```js
tasks.filter(task => {
  ...
});
```

`filter()` takes each task one at a time.

Let's pretend:

```text
task = Learn React
```

Then your callback runs:

```js
if (filter === "completed") {
  return task.completed;
}
```

Suppose:

```text
filter = "completed"
```

Then:

```js
return task.completed;
```

becomes:

```js
return true;
```

So `Learn React` is kept.

Then the second task:

```text
task = Build a project
```

has:

```js
task.completed === false
```

so:

```js
return false;
```

That task is discarded.

Third:

```text
Practice JavaScript
```

also returns:

```js
false
```

So the result is:

```js
[
  {
    id: 1,
    name: "Learn React",
    completed: true
  }
]
```

And **that entire array** is assigned to:

```js
const visibleTasks = ...
```

So:

```text
task callback returns → true/false

filter() returns → array of tasks

visibleTasks = that array
```

---

# 4. So `visibleTasks` is not a function

This is worth emphasizing.

You have:

```js
const visibleTasks = tasks.filter(...)
```

The `filter()` function has already executed.

So `visibleTasks` is just a value.

In this case, an array.

For example:

```js
visibleTasks
```

might currently be:

```js
[
  { id: 1, name: "Learn React", completed: true }
]
```

Or:

```js
[
  { id: 2, name: "Build a project", completed: false },
  { id: 3, name: "Practice JavaScript", completed: false }
]
```

depending on `filter`.

---

# 5. Then what is `filter` doing?

This is where your second question comes in.

You said:

> How is `filter` ever going to be `"completed"` or `"active"` because it is not changing?

It **is** changing.

Look at this:

```jsx
<select
  value={filter}
  onChange={e => setFilter(e.target.value)}
>
```

Initially:

```js
const [filter, setFilter] = useState("all");
```

So:

```text
filter = "all"
```

Then the user selects:

```text
Completed
```

The browser fires the `change` event.

This executes:

```js
e => setFilter(e.target.value)
```

Suppose:

```js
e.target.value === "completed"
```

So this happens:

```js
setFilter("completed");
```

React updates the state.

Then the component renders again.

And this line runs again:

```js
const visibleTasks = tasks.filter(...)
```

But now:

```text
filter = "completed"
```

So the filtering logic behaves differently.

---

# 6. Let's trace it from start to finish

This is the part I really want you to get comfortable with.

Initial state:

```js
filter = "all"
```

and:

```js
tasks = [...]
```

During render:

```js
visibleTasks = tasks.filter(...)
```

Since:

```js
filter === "all"
```

neither of these runs:

```js
filter === "completed"
filter === "active"
```

So we reach:

```js
return true;
```

for every task.

Therefore:

```js
visibleTasks = all tasks
```

Then the `<select>` is displayed.

User chooses:

```text
Completed
```

That triggers:

```js
onChange={e => setFilter(e.target.value)}
```

Suppose:

```js
e.target.value === "completed"
```

So:

```js
setFilter("completed");
```

React schedules another render.

Now the component runs again.

This line executes again:

```js
const visibleTasks = tasks.filter(...)
```

But now:

```js
filter === "completed"
```

So for each task:

```js
return task.completed;
```

which gives either `true` or `false`.

`filter()` keeps only the `true`s.

Then:

```jsx
visibleTasks.map(...)
```

renders those tasks.

So the full flow is:

```text
Initial:
filter = "all"
       ↓
visibleTasks = all tasks
       ↓
UI shows all tasks

User selects "completed"
       ↓
onChange fires
       ↓
setFilter("completed")
       ↓
React renders App again
       ↓
filter = "completed"
       ↓
visibleTasks is recalculated
       ↓
only completed tasks remain
       ↓
UI shows completed tasks
```

That's the actual mechanism.

---

# 7. This is an extremely important React idea

This:

```js
const visibleTasks = tasks.filter(...)
```

does **not** mean:

> "Create a permanent value once."

`const` only means:

> Within this particular execution of the component, this variable cannot be reassigned.

Remember what we learned about rendering:

```text
render #1
App() executes
    ↓
filter = "all"
    ↓
visibleTasks = ...

render #2
App() executes again
    ↓
filter = "completed"
    ↓
visibleTasks = ...
```

So `visibleTasks` is recalculated on each render.

That's why it reacts to the changed `filter` state.

---

# 8. This connects directly to your earlier confusion about state

This is subtle but very important.

You may have been thinking:

```text
const visibleTasks = ...
```

means:

> "This stays fixed."

Not quite.

It means:

> "During this one execution of `App`, this local variable can't be reassigned."

React can execute `App()` again later, creating a completely new `visibleTasks` variable for that render.

So:

```text
Render 1:
const visibleTasks = [all tasks]

Render 2:
const visibleTasks = [completed tasks]
```

There are effectively two separate executions.

---

# 9. Now let's address your bigger concern

You said:

> Sometimes I understand the entire flow of data and state, but not on the first glance. I get confused by the different callbacks happening either to functions or components and then what are they returning?

**This is completely normal**, and I think the solution is not "learn more React."

The solution is learning to **classify functions by role**.

When you see this:

```jsx
{visibleTasks.map(task => (
  <TaskItem
    key={task.id}
    task={task}
    onToggle={toggleTask}
  />
))}
```

there are multiple functions involved, but they are doing very different jobs.

Let's label them.

---

# 10. Type 1: Component function

This:

```jsx
function TaskItem({ task, onToggle }) {
  return (
    <li>
      ...
    </li>
  );
}
```

is a **component function**.

Its job:

```text
receive props
   ↓
return JSX
```

So:

```text
TaskItem
   ↓
JSX
```

Think:

> "This function describes a piece of UI."

---

# 11. Type 2: `map()` callback

This:

```js
tasks.map(task => ...)
```

has a callback:

```js
task => ...
```

Its job is **not to be a component**.

Its job is:

> "Given one task, tell me what the corresponding output should be."

So:

```js
tasks.map(task => ...)
```

means:

```text
task 1 → output 1
task 2 → output 2
task 3 → output 3
```

In our case, the output happens to be JSX:

```jsx
<TaskItem task={task} />
```

So:

```text
map callback
     ↓
React element
```

---

# 12. Type 3: `filter()` callback

This:

```js
tasks.filter(task => {
  return task.completed;
});
```

has a callback too.

But its job is completely different.

Its job is:

> "Should this item stay in the array?"

Therefore it must return a boolean.

```text
task → true/false
```

So:

```text
filter callback
       ↓
   boolean
```

This is why learning the JavaScript methods themselves matters so much.

---

# 13. Type 4: Event handler

This:

```jsx
onChange={e => setFilter(e.target.value)}
```

is another callback.

Its job:

> "When this event happens, do something."

So:

```text
event
  ↓
callback
  ↓
state update
```

It doesn't exist to return a transformed array.

It doesn't exist to return JSX.

It exists to handle an event.

---

# 14. Type 5: State updater function

Then we have this:

```jsx
setTasks(tasks =>
  tasks.map(...)
);
```

This is another function:

```js
tasks => tasks.map(...)
```

But its role is:

> "Given the previous state, calculate the new state."

So:

```text
previous state
     ↓
updater function
     ↓
new state
```

Different job again.

---

# 15. Here's the cheat sheet I want you to use

When you see a callback, ask:

|Where is the function being passed?|What should it return/do?|
|---|---|
|Component definition|JSX/UI|
|`.map()`|One transformed item|
|`.filter()`|`true` or `false`|
|`.find()`|An item or `undefined`|
|`.reduce()`|Accumulated result|
|`onClick` / `onChange`|Handle an event|
|`setState(prev => ...)`|New state|
|`setTimeout()`|Code to execute later|

This is **mostly JavaScript**, not React.

That's a big insight.

---

# 16. Let's annotate your code

Your original code:

```jsx
const visibleTasks = tasks.filter(task => {
```

Read as:

> For every task, run this callback and decide whether to keep it.

Then:

```jsx
if (filter === "completed") {
  return task.completed;
}
```

means:

> If we're viewing completed tasks, return `true` for completed ones and `false` for the others.

Then:

```jsx
if (filter === "active") {
  return !task.completed;
}
```

means:

> If we're viewing active tasks, return `true` for incomplete ones and `false` for completed ones.

Then:

```jsx
return true;
```

means:

> Otherwise keep every task.

Then the result of the entire `filter()`:

```text
array of tasks
```

gets stored in:

```js
visibleTasks
```

Then:

```jsx
visibleTasks.map(task => (
```

means:

> For each visible task, create one `TaskItem`.

And:

```jsx
<TaskItem
  key={task.id}
  task={task}
  onToggle={toggleTask}
/>
```

means:

> Render a `TaskItem`, give it this task as data, and give it this function so it can request a toggle.

Then inside `TaskItem`:

```jsx
onChange={() => onToggle(task.id)}
```

means:

> When the checkbox changes, call the function we received from the parent, passing this task's ID.

That's the entire chain.

---

# 17. Follow the chain in reverse when reading React

This is my biggest recommendation for you.

When React code looks confusing, **don't read it top-to-bottom blindly**.

Instead, trace a single piece of data.

Suppose you see:

```jsx
onToggle={toggleTask}
```

Ask:

### Where does `toggleTask` come from?

```js
function toggleTask(id) {
  ...
}
```

### Who receives it?

```jsx
<TaskItem onToggle={toggleTask} />
```

### How does `TaskItem` use it?

```jsx
function TaskItem({ onToggle }) {
```

### When is it called?

```jsx
onChange={() => onToggle(task.id)}
```

### What does that eventually do?

```js
setTasks(...)
```

### What does that change?

`tasks`

### What depends on `tasks`?

```js
visibleTasks
```

### What depends on `visibleTasks`?

```jsx
visibleTasks.map(...)
```

### What gets rendered?

`TaskItem`s.

That is far easier than trying to mentally understand the whole file at once.

---

# 18. Think in "input → output" for every function

This is probably the single best habit for you.

When you see a function, immediately ask:

```text
What goes IN?
What comes OUT?
Why is this function being passed here?
Who calls it?
```

For example:

### `filter` callback

```js
task => task.completed
```

Input:

```text
task
```

Output:

```text
boolean
```

Purpose:

```text
decide whether task stays
```

---

### `map` callback

```jsx
task => <TaskItem task={task} />
```

Input:

```text
task
```

Output:

```text
React element
```

Purpose:

```text
turn one task into UI
```

---

### Event handler

```js
e => setFilter(e.target.value)
```

Input:

```text
event
```

Output:

```text
usually nothing meaningful to the caller
```

Purpose:

```text
respond to user action
```

---

### State updater

```js
tasks => tasks.filter(...)
```

Input:

```text
previous tasks
```

Output:

```text
new tasks array
```

Purpose:

```text
calculate next state
```

---

### Component

```jsx
function TaskItem({ task }) {
  return <li>...</li>;
}
```

Input:

```text
props
```

Output:

```text
JSX/UI description
```

Purpose:

```text
describe UI
```

Once you start seeing functions this way, React code becomes much easier to parse.

---

# 19. One more thing: callbacks don't always "return" something useful

You may be expecting:

```text
function → input → return something
```

But callbacks are often supplied because someone else will call them for a particular purpose.

For example:

```jsx
<button onClick={handleClick}>
```

`handleClick` doesn't have to return anything.

It's being passed so React can call it later.

Likewise:

```js
tasks.forEach(task => {
  console.log(task);
});
```

The callback doesn't return anything useful.

It just performs an action.

So don't ask only:

> "What does this function return?"

Also ask:

> **"Why is this function being passed here?"**

That question is often more useful.

---

# 20. Your new mental model

When reading React, classify the code into these layers:

```text
                    COMPONENT
                        ↓
                 receives props
                        ↓
                 uses state/data
                        ↓
       ┌────────────────┴────────────────┐
       ↓                                 ↓
   calculate                         return JSX
   derived data                           ↓
       ↓                            React renders UI
   map/filter
       ↓
   UI elements

User interaction
       ↓
   event handler
       ↓
   state updater
       ↓
   new render
```

And remember:

**React isn't replacing your JavaScript knowledge.**

It's composing normal JavaScript concepts like:

```text
functions
closures
callbacks
arrays
map
filter
objects
destructuring
spread
```

with a system for rendering UI and managing state.

That's why being solid with JavaScript makes React much easier.

---

# One correction to your intuition about `const`

Don't think:

> "It's a `const`, therefore it doesn't change."

Think:

> "That variable cannot be reassigned during this execution of the function."

For example:

```js
const x = 10;

// x = 20; ❌
```

But React can execute:

```js
App()
```

again, and during that new execution:

```js
const x = 20;
```

can be created.

That's exactly why:

```js
const visibleTasks = ...
```

can produce a different value on a later render.

---

## A very good exercise for you

Take this:

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

and mentally rewrite it as plain English:

```text
Take tasks.
For each task:
    if filter is "completed":
        keep task if completed is true

    otherwise if filter is "active":
        keep task if completed is false

    otherwise:
        keep task

Give me the resulting array.
```

Then this:

```jsx
visibleTasks.map(task => (
  <TaskItem
    key={task.id}
    task={task}
    onToggle={toggleTask}
  />
))
```

as:

```text
Take the visible tasks.
For each task:
    create a TaskItem for it.
```

That is essentially what the code is saying.

**Next, we should do state ownership and lifting state up**, but I would actually spend one small lesson first on **JavaScript callbacks, closures, `map`, `filter`, `find`, and `reduce` in the context of React**, because that will remove most of the "what function is calling what?" friction you're experiencing.