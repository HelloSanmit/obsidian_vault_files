
# Lesson 12 — Custom Hooks

We've learned a lot of individual React tools:

```text
useState
useEffect
useRef
useContext
useReducer
```

Now we're going to learn something more architectural:

> **How do I take React logic that I want to reuse and package it into a reusable function?**

That's what a **custom Hook** is.

React's current documentation describes custom Hooks as a way to **share stateful logic between components**. A custom Hook is simply a JavaScript function whose name starts with `use` and which can call other Hooks. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 1. Start with a problem

Suppose we have two components:

```jsx
function ComponentA() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    function handleOnline() {
      setIsOnline(true);
    }

    function handleOffline() {
      setIsOnline(false);
    }

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return <p>{isOnline ? "Online" : "Offline"}</p>;
}
```

And another component needs exactly the same logic:

```jsx
function ComponentB() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    function handleOnline() {
      setIsOnline(true);
    }

    function handleOffline() {
      setIsOnline(false);
    }

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return <button disabled={!isOnline}>Save</button>;
}
```

We've duplicated a lot of logic.

We want to extract the logic.

---

# 2. Create a custom Hook

We can create:

```jsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    function handleOnline() {
      setIsOnline(true);
    }

    function handleOffline() {
      setIsOnline(false);
    }

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return isOnline;
}
```

Then:

```jsx
function ComponentA() {
  const isOnline = useOnlineStatus();

  return (
    <p>
      {isOnline ? "Online" : "Offline"}
    </p>
  );
}
```

And:

```jsx
function ComponentB() {
  const isOnline = useOnlineStatus();

  return (
    <button disabled={!isOnline}>
      Save
    </button>
  );
}
```

Much cleaner.

---

# 3. What did we actually reuse?

This distinction is **extremely important**.

We did **not** create one shared:

```text
isOnline
```

state value.

Instead:

```text
Component A
    ↓
useOnlineStatus()
    ↓
its own state

Component B
    ↓
useOnlineStatus()
    ↓
its own state
```

Each call to the custom Hook has its **own independent state and Effects**. React's documentation explicitly emphasizes that custom Hooks share logic, not the actual state itself. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

Think:

```text
useOnlineStatus()
```

is like a recipe for creating the same behavior.

It isn't a global state container.

---

# 4. This is the most important custom Hook mental model

A custom Hook is essentially:

```text
reusable React logic
```

not:

```text
reusable component
```

A component usually returns JSX:

```jsx
function Button() {
  return <button>Save</button>;
}
```

A custom Hook usually returns data/functions:

```jsx
function useOnlineStatus() {
  // state/effects
  return isOnline;
}
```

So:

```text
Component
→ reusable UI

Custom Hook
→ reusable behavior/logic
```

That's a very useful distinction.

---

# 5. Why does it have to start with `use`?

You might wonder why this:

```js
function useOnlineStatus() {
```

has to be named that way.

The `use` naming convention tells React tooling and developers:

> This function is a Hook and may call other Hooks.

React requires Hooks to be called only from React function components or custom Hooks, and only at the top level. ([React](https://react.dev/reference/rules/rules-of-hooks?utm_source=chatgpt.com "Rules of Hooks – React"))

So this is valid:

```jsx
function useSomething() {
  const [value, setValue] = useState(0);
}
```

And:

```jsx
function Component() {
  const value = useSomething();
}
```

But this is not:

```js
function regularUtility() {
  const [value, setValue] = useState(0);
}
```

A normal utility function must not call Hooks. ([React](https://react.dev/reference/rules/rules-of-hooks?utm_source=chatgpt.com "Rules of Hooks – React"))

---

# 6. A custom Hook can call other Hooks

This is literally its main purpose.

For example:

```jsx
function useUser() {
  const [user, setUser] = useState(null);

  useEffect(() => {
    // fetch user
  }, []);

  return user;
}
```

This custom Hook is composed from:

```text
useState
+
useEffect
```

React itself describes custom Hooks as being built by combining built-in Hooks. ([React](https://react.dev/reference/react/hooks?utm_source=chatgpt.com "Built-in React Hooks – React"))

---

# 7. A custom Hook can return multiple things

Instead of:

```jsx
function useOnlineStatus() {
  return isOnline;
}
```

you can return an object:

```jsx
function useCounter() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount(c => c + 1);
  }

  function decrement() {
    setCount(c => c - 1);
  }

  function reset() {
    setCount(0);
  }

  return {
    count,
    increment,
    decrement,
    reset
  };
}
```

Then:

```jsx
function Counter() {
  const {
    count,
    increment,
    decrement,
    reset
  } = useCounter();

  return (
    <div>
      <p>{count}</p>

      <button onClick={increment}>
        +
      </button>

      <button onClick={decrement}>
        -
      </button>

      <button onClick={reset}>
        Reset
      </button>
    </div>
  );
}
```

This is a very common custom Hook pattern.

---

# 8. Notice what's happening

The component now doesn't care how the counter works.

It simply says:

```jsx
const {
  count,
  increment,
  decrement,
  reset
} = useCounter();
```

The custom Hook hides:

```text
state
state-update logic
```

while exposing:

```text
count
increment
decrement
reset
```

This gives you a nice separation:

```text
UI component
      ↓
"useCounter"
      ↓
stateful logic
```

---

# 9. Custom Hooks make components more declarative

Compare this:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  function increment() {
    setCount(c => c + 1);
  }

  function decrement() {
    setCount(c => c - 1);
  }

  // ...
}
```

with:

```jsx
function Counter() {
  const {
    count,
    increment,
    decrement
  } = useCounter();

  // ...
}
```

The second version says something closer to:

> "This component has a counter."

rather than:

> "Here's all the machinery required to implement a counter."

That can make large applications easier to understand.

React's docs describe good custom Hooks as making calling code more declarative by providing a focused API around a concrete use case. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 10. Custom Hook with parameters

Hooks can accept arguments just like normal JavaScript functions.

Suppose:

```jsx
function useCounter(initialValue) {
  const [count, setCount] =
    useState(initialValue);

  function increment() {
    setCount(c => c + 1);
  }

  return {
    count,
    increment
  };
}
```

Then:

```jsx
const counter = useCounter(10);
```

starts at:

```text
10
```

Another component:

```jsx
const counter = useCounter(100);
```

starts at:

```text
100
```

So:

```text
custom Hook ≈ function that encapsulates reusable React logic
```

This connects nicely to the JavaScript analogy you've been using throughout the course.

---

# 11. Custom Hooks can accept reactive values

Suppose we make:

```jsx
function useChatRoom(roomId) {
  useEffect(() => {
    const connection =
      createConnection(roomId);

    connection.connect();

    return () => {
      connection.disconnect();
    };
  }, [roomId]);
}
```

Then:

```jsx
function ChatRoom({ roomId }) {
  useChatRoom(roomId);

  return <h1>Chat room</h1>;
}
```

If:

```text
roomId = "general"
```

the Hook synchronizes with that room.

If it becomes:

```text
roomId = "travel"
```

the Hook sees the new value during the next render and its Effect handles the synchronization. React's docs explain that custom Hooks re-run along with the component and therefore receive the latest props/state. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 12. A custom Hook is not a class

Older React patterns used classes for stateful logic.

Modern React uses:

```text
function component
+
Hooks
```

A custom Hook is just another function that participates in that model.

There is no special class or object required.

---

# 13. Custom Hook vs utility function

This distinction is extremely important.

Suppose you have:

```js
function formatCurrency(amount) {
  return `₹${amount.toFixed(2)}`;
}
```

This is a normal utility function.

It does **not** need to be:

```js
function useCurrency(amount) {
  ...
}
```

Why?

Because it doesn't use Hooks.

React's docs explicitly recommend not giving regular utility functions a `use` prefix when they don't call Hooks. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

So:

```text
Uses Hooks?
    ↓
yes → custom Hook

no → normal function
```

---

# 14. Don't abuse the `use` prefix

Bad:

```js
function useSorted(items) {
  return items
    .slice()
    .sort();
}
```

if this function doesn't use any Hooks.

Better:

```js
function getSorted(items) {
  return items
    .slice()
    .sort();
}
```

The `use` prefix is meaningful because it signals Hook behavior. React's current guidance explicitly makes this distinction. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 15. Rules of Hooks apply inside custom Hooks too

For example, this is invalid:

```jsx
function useUser(isEnabled) {
  if (isEnabled) {
    const [user, setUser] =
      useState(null);
  }
}
```

Hooks cannot be called conditionally.

Instead:

```jsx
function useUser(isEnabled) {
  const [user, setUser] =
    useState(null);

  // conditionally use the value/logic
}
```

Why?

Because Hooks need to be called in a stable order across renders. React's Rules of Hooks require Hooks to be called at the top level of components or custom Hooks, not inside conditions, loops, or nested functions. ([React](https://react.dev/reference/rules/rules-of-hooks?utm_source=chatgpt.com "Rules of Hooks – React"))

---

# 16. Another common mistake

Don't do:

```jsx
function App() {
  if (loggedIn) {
    useUser();
  }
}
```

Instead:

```jsx
function App() {
  const user = useUser();

  if (!loggedIn) {
    return <Login />;
  }

  return <Dashboard user={user} />;
}
```

Or move the Hook into the component that actually needs it.

---

# 17. Custom Hooks don't need to contain Effects

This is important.

You can create:

```jsx
function useCounter() {
  const [count, setCount] =
    useState(0);

  return [count, setCount];
}
```

No `useEffect`.

Custom Hook simply means:

> A reusable function that can use React Hooks.

It doesn't mean:

> A function containing an Effect.

React's documentation gives many examples of custom Hooks composed from state, Effects, and other Hooks. ([React](https://react.dev/reference/react/hooks?utm_source=chatgpt.com "Built-in React Hooks – React"))

---

# 18. Let's extract our task logic

Remember our Todo app?

We had:

```text
tasks
add
delete
toggle
```

and eventually `useReducer`.

We could now create:

```jsx
function useTasks() {
  const [state, dispatch] =
    useReducer(tasksReducer, initialState);

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

  function toggleTask(id) {
    dispatch({
      type: "toggled",
      id
    });
  }

  return {
    tasks: state.tasks,
    addTask,
    deleteTask,
    toggleTask
  };
}
```

Then the component becomes:

```jsx
function TodoApp() {
  const {
    tasks,
    addTask,
    deleteTask,
    toggleTask
  } = useTasks();

  // render UI
}
```

This is powerful.

The component doesn't need to know whether the task state uses:

```text
useState
```

or:

```text
useReducer
```

or something more complicated.

It just interacts with:

```text
tasks
addTask
deleteTask
toggleTask
```

---

# 19. This is abstraction

You are now building an abstraction.

Inside:

```text
useTasks()
```

you have:

```text
state management
actions
reducers
update logic
```

Outside:

```text
tasks
addTask()
deleteTask()
toggleTask()
```

That's similar to using an API.

The consumer shouldn't need to know the internal implementation.

---

# 20. Why this is useful in real projects

Imagine ten components need task behavior:

```text
TaskList
TaskItem
TaskSidebar
TaskStats
MobileTasks
QuickAdd
```

Instead of duplicating task logic, you can centralize the behavior:

```text
useTasks()
```

Then different UI components can use the same logic.

But remember:

```text
shared logic ≠ automatically shared state
```

If ten components independently call:

```jsx
useTasks()
```

they generally get ten independent instances of that Hook's state.

If you actually want one shared state source, you need something such as:

```text
Context
external store
server-state cache
```

depending on the problem.

This distinction is critical. React explicitly calls this out in its custom Hook documentation. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 21. Custom Hook + Context

Now you can see why these two concepts often appear together.

You can have:

```text
Context
   ↓
shared state
   ↓
custom Hook
   ↓
easy API for consumers
```

For example:

```jsx
function useAuth() {
  const context = useContext(AuthContext);

  if (!context) {
    throw new Error(
      "useAuth must be used inside AuthProvider"
    );
  }

  return context;
}
```

Then components simply write:

```jsx
const { user, logout } = useAuth();
```

instead of repeatedly writing:

```jsx
const {
  user,
  logout
} = useContext(AuthContext);
```

The custom Hook creates a nicer abstraction.

---

# 22. Custom Hook + Effects

Another common pattern:

```jsx
function useOnlineStatus() {
  const [isOnline, setIsOnline] =
    useState(true);

  useEffect(() => {
    // subscribe
    // cleanup
  }, []);

  return isOnline;
}
```

The caller doesn't care about:

```text
window.addEventListener
window.removeEventListener
```

It simply asks:

```jsx
const isOnline = useOnlineStatus();
```

This is exactly how custom Hooks help hide synchronization details. React's docs use `useOnlineStatus` as one of their central examples. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 23. Custom Hook + `useRef`

Suppose you need a reusable "previous value" behavior:

```jsx
function usePrevious(value) {
  const ref = useRef();

  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}
```

Then:

```jsx
function Counter() {
  const [count, setCount] =
    useState(0);

  const previousCount =
    usePrevious(count);

  // ...
}
```

The implementation combines:

```text
useRef
+
useEffect
```

into a reusable behavior.

Again, the component doesn't need to know the implementation details.

---

# 24. A custom Hook can return functions

For example:

```jsx
function useCounter() {
  const [count, setCount] =
    useState(0);

  const increment = () => {
    setCount(c => c + 1);
  };

  const decrement = () => {
    setCount(c => c - 1);
  };

  return {
    count,
    increment,
    decrement
  };
}
```

The caller:

```jsx
const {
  count,
  increment,
  decrement
} = useCounter();
```

Those functions are part of the Hook's public API.

So your custom Hook is effectively exposing:

```text
data
+
operations
```

This is very similar to an object-oriented interface, except you're using functions and React's rendering model.

---

# 25. Good custom Hook design

React's current guidance is to make custom Hooks focused around **concrete high-level use cases**. Good names make their purpose obvious, such as:

```text
useChatRoom
useOnlineStatus
useMediaQuery
useIntersectionObserver
```

rather than generic wrappers like:

```text
useEffectOnce
useLifecycle
useAnything
```

The goal is a useful abstraction, not simply hiding a few lines of code. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 26. Bad custom Hook design

Suppose:

```jsx
function useEffectWrapper(callback, dependencies) {
  useEffect(callback, dependencies);
}
```

This doesn't really solve a domain problem.

It just renames an existing API.

Compare:

```jsx
useEffectWrapper(...)
```

with:

```jsx
useChatRoom({ roomId })
```

The second expresses a concrete purpose.

That's a much more valuable abstraction.

---

# 27. Another important rule: Hooks run during render

Suppose:

```jsx
function useCounter() {
  const [count, setCount] =
    useState(0);

  return count;
}
```

When your component renders:

```jsx
function App() {
  const count = useCounter();

  return <p>{count}</p>;
}
```

the custom Hook runs as part of rendering.

Conceptually:

```text
React renders App
      ↓
App calls useCounter()
      ↓
useCounter calls useState()
      ↓
returns count
      ↓
App returns JSX
```

A custom Hook isn't running in some completely separate world.

React's documentation explicitly describes custom Hook code as part of the component's body and says Hooks re-run with the component on each render. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 28. This makes the earlier callback discussion relevant again

You can now distinguish:

```text
normal function
→ reusable calculation/behavior

component
→ reusable UI

custom Hook
→ reusable stateful React logic
```

For example:

```js
formatPrice(price)
```

→ normal function.

```jsx
<ProductCard />
```

→ component.

```jsx
useShoppingCart()
```

→ custom Hook.

That's a very useful classification.

---

# 29. Let's build a real `useTasks` Hook

Here's a more complete version:

```jsx
import { useReducer } from "react";

const initialState = {
  tasks: []
};

function tasksReducer(state, action) {
  switch (action.type) {
    case "added":
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

    case "deleted":
      return {
        ...state,
        tasks: state.tasks.filter(
          task => task.id !== action.id
        )
      };

    case "toggled":
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

    default:
      throw new Error(
        `Unknown action: ${action.type}`
      );
  }
}

function useTasks() {
  const [state, dispatch] =
    useReducer(
      tasksReducer,
      initialState
    );

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

  function toggleTask(id) {
    dispatch({
      type: "toggled",
      id
    });
  }

  return {
    tasks: state.tasks,
    addTask,
    deleteTask,
    toggleTask
  };
}
```

Then:

```jsx
function TodoApp() {
  const {
    tasks,
    addTask,
    deleteTask,
    toggleTask
  } = useTasks();

  return (
    <div>
      {tasks.map(task => (
        <div key={task.id}>
          <span>{task.text}</span>

          <button
            onClick={() => toggleTask(task.id)}
          >
            Toggle
          </button>

          <button
            onClick={() => deleteTask(task.id)}
          >
            Delete
          </button>
        </div>
      ))}
    </div>
  );
}
```

Look at what's happened architecturally.

Before:

```text
component
├── reducer
├── actions
├── state logic
├── handlers
└── UI
```

After:

```text
TodoApp
└── UI

useTasks
├── state
├── reducer
├── actions
└── behavior
```

That's the kind of separation you'll see in larger applications.

---

# 30. But don't extract everything into custom Hooks

This is important.

Bad:

```jsx
function useButtonClick() {
  function handleClick() {
    console.log("clicked");
  }

  return handleClick;
}
```

That's probably unnecessary.

Just use:

```jsx
function handleClick() {
  console.log("clicked");
}
```

Custom Hooks are valuable when you're encapsulating meaningful reusable React logic.

Not every function needs to become a Hook.

---

# 31. The "should this become a Hook?" test

Ask:

> Does this logic involve React state, Effects, refs, context, or another Hook, and is the behavior useful in more than one place?

If yes:

```text
custom Hook may be appropriate
```

If it's just:

```js
calculateTotal(items)
```

use a normal function.

If it's UI:

```jsx
<ProductCard />
```

use a component.

This distinction prevents massive amounts of unnecessary abstraction.

---

# 32. Your React mental model now looks like this

```text
                 COMPONENT
                     │
          ┌──────────┼───────────┐
          ↓          ↓           ↓
        state      context      refs
          │
          ↓
        render
          │
          ↓
          UI
          │
       user event
          ↓
    event handler
          ↓
    state/dispatch
          ↓
       render
```

And custom Hooks sit on top of all of this:

```text
         Custom Hook
        /     |      \
   useState useEffect useRef
        \     |      /
         reusable logic
                ↓
            component
```

So a custom Hook is not another competing React mechanism.

It's a way of **composing the mechanisms you already learned**.

---

# 33. The major idea from this lesson

Try to remember this sentence:

> **Custom Hooks share logic, not state.**

And these three distinctions:

```text
Function
→ reusable JavaScript calculation/behavior

Component
→ reusable UI

Custom Hook
→ reusable stateful React logic
```

And:

```text
useState
→ remember UI-relevant state

useRef
→ remember mutable values without rendering

useEffect
→ synchronize with external systems

useContext
→ access contextual values

useReducer
→ organize complex state transitions

Custom Hook
→ combine these into reusable behavior
```

That is a very solid conceptual foundation.

---

## Next lesson — The React rendering model, properly

We've been using the word **"re-render"** since the beginning, but we haven't yet looked underneath it carefully enough.

Next we'll dissect:

```text
render
↓
reconciliation
↓
commit
```

and answer the questions that eventually matter in real applications:

```text
What causes a component to render?
Does a parent re-render cause all children to re-render?
What does React compare?
What does a key actually change?
When is state preserved?
When does state reset?
What is component identity?
What does "Virtual DOM" actually mean?
```

Once you understand that, things like `React.memo`, `useMemo`, `useCallback`, keys, and performance will stop being a collection of optimization tricks and start making logical sense.