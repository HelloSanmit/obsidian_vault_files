
# Lesson 11 — `useReducer`: Managing More Complex State

So far we've used:

```jsx
const [count, setCount] = useState(0);
```

and:

```jsx
setCount(count + 1);
```

That's excellent when state changes are simple.

But imagine a task application where you have:

```text
add task
edit task
delete task
toggle task
clear completed
reset everything
```

With `useState`, you might end up with many handlers, each containing increasingly complicated state-update logic.

React's current docs recommend `useReducer` for cases where state updates are spread across many event handlers and that logic is becoming difficult to understand. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

The core idea is:

```text
event
  ↓
action
  ↓
reducer
  ↓
new state
```

Let's build that from scratch.

---

# 1. First understand what a reducer is

Forget React for one minute.

A reducer is just a function that takes:

```text
current result
+
new information
↓
next result
```

You actually already encountered this with JavaScript `reduce()`.

For example:

```js
const numbers = [1, 2, 3, 4];

const total = numbers.reduce(
  (sum, number) => sum + number,
  0
);
```

The callback:

```js
(sum, number) => sum + number
```

takes:

```text
current accumulated result
+
current item
```

and produces:

```text
new accumulated result
```

React's reducer concept uses a similar idea:

```text
current state
+
action
↓
next state
```

React's documentation explicitly explains that this is where the name "reducer" comes from. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 2. The basic React reducer

Suppose our state is:

```js
const initialState = {
  count: 0
};
```

Our reducer:

```js
function reducer(state, action) {
  if (action.type === "increment") {
    return {
      count: state.count + 1
    };
  }

  return state;
}
```

Read that as:

> Given the current state and what happened, calculate the next state.

So:

```text
state:
{ count: 0 }

action:
{ type: "increment" }

        ↓

reducer

        ↓

{ count: 1 }
```

---

# 3. What is an action?

An action is usually just a JavaScript object describing **what happened**. React's docs recommend that actions generally contain a `type` plus whatever minimal data is needed to process that event. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

For example:

```js
{
  type: "increment"
}
```

or:

```js
{
  type: "add_task",
  text: "Learn React"
}
```

or:

```js
{
  type: "delete_task",
  id: 42
}
```

The action describes the event.

It does not usually contain the new state.

That's an important distinction.

---

# 4. Think "what happened", not "how to update state"

This is one of the biggest ideas behind reducers.

With `useState`, you might do:

```jsx
function handleDelete(id) {
  setTasks(tasks =>
    tasks.filter(task => task.id !== id)
  );
}
```

The event handler knows **how** to update state.

With a reducer, the event handler can simply say:

```jsx
function handleDelete(id) {
  dispatch({
    type: "delete_task",
    id
  });
}
```

The handler is now saying:

> "The user deleted task 42."

It isn't saying:

> "Here's the precise array manipulation you should perform."

The reducer decides that.

This separation is one of the primary reasons reducers can make complex state logic easier to reason about. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 5. `dispatch`

This is the piece that connects your component to the reducer.

```jsx
const [state, dispatch] = useReducer(
  reducer,
  initialState
);
```

It's similar to `useState`:

```jsx
const [state, setState] = useState(initialState);
```

but instead of:

```text
state + setter
```

you get:

```text
state + dispatch
```

The difference is:

```text
useState
---------
setState(newValue)


useReducer
-----------
dispatch(action)
```

---

# 6. What happens when you call `dispatch()`?

Suppose:

```jsx
dispatch({
  type: "increment"
});
```

Conceptually:

```text
dispatch(action)
      ↓
React sends action to reducer
      ↓
reducer(currentState, action)
      ↓
returns newState
      ↓
React stores newState
      ↓
component renders again
```

So:

```text
dispatch
   ↓
reducer
   ↓
state update
   ↓
render
```

That's the main loop.

---

# 7. The smallest complete example

```jsx
import { useReducer } from "react";

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return {
        count: state.count + 1
      };

    case "decrement":
      return {
        count: state.count - 1
      };

    default:
      throw new Error(
        `Unknown action: ${action.type}`
      );
  }
}

function Counter() {
  const [state, dispatch] = useReducer(
    reducer,
    { count: 0 }
  );

  return (
    <div>
      <p>{state.count}</p>

      <button
        onClick={() =>
          dispatch({ type: "increment" })
        }
      >
        +
      </button>

      <button
        onClick={() =>
          dispatch({ type: "decrement" })
        }
      >
        -
      </button>
    </div>
  );
}
```

Let's trace the `+` button.

---

# 8. Click `+`

This executes:

```jsx
dispatch({
  type: "increment"
});
```

The action is:

```js
{
  type: "increment"
}
```

The reducer receives:

```js
reducer(
  { count: 0 },
  { type: "increment" }
)
```

It matches:

```js
case "increment":
```

and returns:

```js
{
  count: 1
}
```

React uses that as the new state.

Then the component renders again.

Now:

```jsx
{state.count}
```

is:

```text
1
```

---

# 9. Notice the separation

We now have three separate things.

### Event handler

```jsx
onClick={() =>
  dispatch({ type: "increment" })
}
```

Its job:

> Report what happened.

### Action

```js
{
  type: "increment"
}
```

Its job:

> Describe what happened.

### Reducer

```js
function reducer(state, action) {
  ...
}
```

Its job:

> Decide what the next state should be.

That's a very clean separation.

---

# 10. Let's make the state more realistic

Suppose:

```js
const initialState = {
  tasks: [
    {
      id: 1,
      text: "Learn React",
      completed: false
    }
  ]
};
```

We want:

```text
add
delete
toggle
```

Our reducer:

```jsx
function tasksReducer(state, action) {
  switch (action.type) {
    case "add": {
      return {
        ...state,
        tasks: [
          ...state.tasks,
          {
            id: action.id,
            text: action.text,
            completed: false
          }
        ]
      };
    }

    case "delete": {
      return {
        ...state,
        tasks: state.tasks.filter(
          task => task.id !== action.id
        )
      };
    }

    case "toggle": {
      return {
        ...state,
        tasks: state.tasks.map(task =>
          task.id === action.id
            ? {
                ...task,
                completed: !task.completed
              }
            : task
        )
      };
    }

    default:
      throw new Error(
        `Unknown action: ${action.type}`
      );
  }
}
```

This looks larger than `useState`.

That's intentional.

The benefit comes when there are many related state transitions: the update rules are now centralized. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 11. The event handlers become simpler

Instead of:

```jsx
function handleAdd(text) {
  setTasks(tasks => [
    ...tasks,
    {
      id: Date.now(),
      text,
      completed: false
    }
  ]);
}
```

we can write:

```jsx
function handleAdd(text) {
  dispatch({
    type: "add",
    id: Date.now(),
    text
  });
}
```

Delete:

```jsx
function handleDelete(id) {
  dispatch({
    type: "delete",
    id
  });
}
```

Toggle:

```jsx
function handleToggle(id) {
  dispatch({
    type: "toggle",
    id
  });
}
```

Now the handlers are very easy to scan.

You can look at them and immediately see:

```text
handleAdd    → add
handleDelete → delete
handleToggle → toggle
```

The complicated update logic lives elsewhere.

---

# 12. This is especially useful when there are many transitions

Imagine a form with:

```text
idle
loading
success
error
```

and transitions such as:

```text
SUBMIT
SUCCESS
FAILURE
RESET
```

Instead of having a bunch of scattered `setState()` calls, you can model it as a state machine-like set of transitions:

```text
idle
 │
 │ SUBMIT
 ↓
loading
 ├── SUCCESS → success
 └── FAILURE → error
```

A reducer is a natural place to describe those transitions.

For example:

```js
function reducer(state, action) {
  switch (action.type) {
    case "submit":
      return {
        ...state,
        status: "loading"
      };

    case "success":
      return {
        ...state,
        status: "success",
        data: action.data
      };

    case "failure":
      return {
        ...state,
        status: "error",
        error: action.error
      };

    case "reset":
      return initialState;

    default:
      throw new Error("Unknown action");
  }
}
```

Now the valid state transitions are easy to inspect.

---

# 13. Reducers must be pure

This is extremely important.

A reducer:

```jsx
function reducer(state, action) {
  ...
}
```

should not:

```text
make network requests
start timers
modify the DOM
write to localStorage
perform random side effects
```

The reducer's job is:

```text
state + action → next state
```

React's current documentation explicitly says reducers must be pure and should not perform side effects. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

So this is good:

```js
function reducer(state, action) {
  return {
    ...state,
    count: state.count + 1
  };
}
```

This is not:

```js
function reducer(state, action) {
  fetch("/api");
  return ...
}
```

If you need external synchronization, that's where the other mechanisms we learned come in.

---

# 14. Reducers must not mutate state

This:

```js
state.tasks.push(newTask);
```

is wrong.

Similarly:

```js
state.user.name = "Sarah";
```

is wrong.

Instead:

```js
return {
  ...state,
  tasks: [
    ...state.tasks,
    newTask
  ]
};
```

or:

```js
return {
  ...state,
  user: {
    ...state.user,
    name: "Sarah"
  }
};
```

Same immutable-state principle you learned with `useState`.

React's documentation explicitly calls out avoiding state mutation in reducers. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 15. The reducer doesn't mutate the old state into the new state

This is a really useful mental model:

```text
OLD STATE
   +
ACTION
   ↓
REDUCER
   ↓
NEW STATE
```

not:

```text
OLD STATE
   ↓
mutate it
   ↓
same object, changed internally
```

For example:

```js
function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return {
        ...state,
        count: state.count + 1
      };

    default:
      return state;
  }
}
```

You create the next state.

---

# 16. Why use `switch`?

You could write:

```js
function reducer(state, action) {
  if (action.type === "increment") {
    ...
  }

  if (action.type === "decrement") {
    ...
  }

  return state;
}
```

That's valid.

React's documentation uses `switch` as a common convention because when there are many action types, it can be easier to scan. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

So:

```js
switch (action.type) {
  case "add":
    ...
  case "delete":
    ...
  case "toggle":
    ...
}
```

is mostly a readability convention.

---

# 17. Why `{}` around each `case`?

You'll often see:

```js
switch (action.type) {
  case "add": {
    const newTask = ...;

    return ...;
  }

  case "delete": {
    ...
  }
}
```

Those braces create a block scope for that case.

It's especially useful if different cases declare variables with the same names.

The React docs recommend this style in their reducer examples. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 18. Actions can carry data

This is one of the most useful parts.

For example:

```js
dispatch({
  type: "add",
  text: "Learn reducers",
  id: 7
});
```

The reducer receives the whole action:

```js
function reducer(state, action) {
  console.log(action);
}
```

which is:

```js
{
  type: "add",
  text: "Learn reducers",
  id: 7
}
```

So:

```js
action.text
```

is available inside the reducer.

---

# 19. Think of an action as an event message

A very useful analogy:

```text
Action ≈ message describing something that happened
```

Examples:

```text
{ type: "task_added" }
{ type: "task_deleted", id: 4 }
{ type: "task_toggled", id: 4 }
{ type: "form_submitted" }
{ type: "login_succeeded", user: ... }
```

Then your reducer is the system that says:

> "Given this event, what should the application's state become?"

That makes reducers conceptually easier to understand.

---

# 20. `useState` vs `useReducer`

Let's compare.

### `useState`

```jsx
const [count, setCount] = useState(0);
```

Great for:

```text
simple values
simple transitions
local UI state
```

### `useReducer`

```jsx
const [state, dispatch] =
  useReducer(reducer, initialState);
```

Useful when:

```text
many related state updates
complex transition logic
many event handlers
state shape is structured
```

React's documentation makes the same tradeoff: `useState` generally involves less code upfront, while `useReducer` can make complex state-update logic easier to see and centralize. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 21. Don't replace every `useState` with `useReducer`

This is important.

Don't write:

```jsx
const [isOpen, dispatch] = useReducer(...);
```

for a simple modal if all you need is:

```jsx
const [isOpen, setIsOpen] = useState(false);
```

That's overkill.

The goal isn't:

```text
useReducer = advanced = better
```

The goal is:

```text
simple state
→ useState

complex related transitions
→ maybe useReducer
```

React itself recommends mixing and matching; reducers are not required for everything. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 22. A useful transformation: `useState` → `useReducer`

Suppose you have:

```jsx
const [tasks, setTasks] = useState([]);
```

and:

```jsx
function addTask(text) {
  setTasks(tasks => [
    ...tasks,
    {
      id: Date.now(),
      text,
      completed: false
    }
  ]);
}

function deleteTask(id) {
  setTasks(tasks =>
    tasks.filter(task => task.id !== id)
  );
}
```

You can refactor to:

```jsx
const [tasks, dispatch] =
  useReducer(tasksReducer, []);
```

Then:

```jsx
function addTask(text) {
  dispatch({
    type: "added",
    id: Date.now(),
    text
  });
}

function deleteTask(id) {
  dispatch({
    type: "deleted",
    id
  });
}
```

And:

```jsx
function tasksReducer(tasks, action) {
  switch (action.type) {
    case "added":
      return [
        ...tasks,
        {
          id: action.id,
          text: action.text,
          completed: false
        }
      ];

    case "deleted":
      return tasks.filter(
        task => task.id !== action.id
      );

    default:
      throw new Error("Unknown action");
  }
}
```

This is exactly the refactoring pattern React's documentation recommends: dispatch actions from handlers, move update logic into a reducer, then replace `useState` with `useReducer`. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 23. Notice what becomes cleaner

Before:

```text
handleAdd
  └── array update logic

handleDelete
  └── filter logic

handleToggle
  └── map logic

handleEdit
  └── map logic

handleClear
  └── filter logic
```

After:

```text
handleAdd
  └── dispatch("added")

handleDelete
  └── dispatch("deleted")

handleToggle
  └── dispatch("toggled")

handleEdit
  └── dispatch("edited")

handleClear
  └── dispatch("cleared")


        ↓

     reducer
     ├── added
     ├── deleted
     ├── toggled
     ├── edited
     └── cleared
```

All state-transition logic is in one place.

That's the primary architectural benefit.

---

# 24. Reducer + Context

This combination is extremely important.

You can eventually have:

```text
Context
   +
useReducer
```

Then you can make:

```text
state
dispatch
```

available to a deep component tree.

Conceptually:

```text
Provider
   │
   ├── state
   └── dispatch
        │
        ↓
   descendants
```

This is a common pattern for application-wide or subtree-wide client state.

Later, we'll build this combination properly.

---

# 25. One subtle but important idea: dispatch describes intent

Compare:

```js
dispatch({
  type: "set_tasks",
  tasks: newTasks
});
```

with:

```js
dispatch({
  type: "task_deleted",
  id
});
```

The second one is generally more descriptive.

It says:

> "A task was deleted."

rather than:

> "Here's an array, please replace state with it."

React's documentation encourages actions to describe what happened, and to contain the minimal information necessary. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

That makes debugging easier.

---

# 26. Reducers are easy to test

This is another advantage.

The reducer is just a function:

```js
function reducer(state, action) {
  ...
}
```

So you can theoretically test:

```js
const state = {
  count: 0
};

const action = {
  type: "increment"
};

const nextState =
  reducer(state, action);
```

and expect:

```js
{
  count: 1
}
```

No browser required.

No component required.

No button click required.

That's possible because reducers are pure functions. React's documentation specifically points to this as a benefit for testing complex state logic. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

# 27. A good action naming style

You might see:

```text
add
delete
toggle
```

or:

```text
added
deleted
toggled
```

A useful convention is to phrase the action as something that **happened**:

```text
task_added
task_deleted
task_toggled
form_submitted
login_succeeded
```

The exact naming is your choice; React doesn't impose a required action shape. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

What matters is consistency and clarity.

---

# 28. Reducer vs callback confusion

This also connects directly to the problem you told me about earlier.

Look at:

```jsx
dispatch({
  type: "delete",
  id
});
```

No callback here.

`dispatch` is a function you're calling.

Then React internally invokes your reducer:

```js
reducer(state, action)
```

The reducer is another function.

So mentally:

```text
event handler
    ↓
dispatch(action)
    ↓
React
    ↓
reducer(state, action)
    ↓
new state
```

There are several function calls, but each has a very specific role.

---

# 29. Your callback-role cheat sheet is growing

You can now recognize:

```text
Component
→ receives props, returns JSX

map callback
→ transforms one item

filter callback
→ true/false: keep/discard

event handler
→ responds to user/browser event

state updater
→ computes next state from previous state

reducer
→ computes next state from state + action

useEffect callback
→ synchronizes with external system
```

This is the classification system I wanted you to build.

Once you see the role, the syntax becomes much less intimidating.

---

# 30. The whole `useReducer` model

Here's the final picture:

```text
                     STATE
                       │
                       ↓
                    render
                       │
                       ↓
                       UI
                       │
                  user action
                       │
                       ↓
                 event handler
                       │
                       ↓
                dispatch(action)
                       │
                       ↓
                    reducer
                 /           \
           current state     action
                 \           /
                    ↓
                 new state
                    │
                    ↓
                  render
```

And the reducer itself is conceptually just:

```js
function reducer(state, action) {
  // calculate next state

  return nextState;
}
```

That's it.

No magic.

---

# 31. When you should reach for `useReducer`

Ask:

> Are there several related ways my state can change, and is the update logic becoming difficult to understand when spread across event handlers?

If yes:

```text
useReducer may help
```

If the state is simple:

```text
useState is probably clearer
```

The goal is **clarity**, not sophistication. React's docs emphasize exactly this tradeoff. ([React](https://react.dev/learn/extracting-state-logic-into-a-reducer?utm_source=chatgpt.com "Extracting State Logic into a Reducer – React"))

---

## Practice challenge

Take the task app we've been building and change it from:

```jsx
const [tasks, setTasks] = useState(...)
```

to:

```jsx
const [tasks, dispatch] =
  useReducer(tasksReducer, initialTasks);
```

Your reducer should support:

```text
"added"
"deleted"
"toggled"
```

Your event handlers should **only dispatch actions**.

For example:

```jsx
function handleDelete(id) {
  dispatch({
    type: "deleted",
    id
  });
}
```

and the reducer should decide how the array changes.

Once you've done that, the next lesson is **custom hooks**, where we'll take logic like state + effects + handlers and learn how to package it into reusable behavior such as:

```jsx
const {
  tasks,
  addTask,
  deleteTask,
  toggleTask
} = useTasks();
```

That is the point where React code starts becoming much more reusable and architectural rather than just component-by-component scripting.