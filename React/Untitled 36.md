
# Lesson 33 — External Stores and `useSyncExternalStore`

We've spent a lot of time with state that **React owns**:

```text
useState
useReducer
Context
```

We've also learned server state:

```text
TanStack Query
```

Now there's one major category we haven't properly covered:

> **What if the state lives outside React?**

For example:

```text
Redux
Zustand
MobX
browser APIs
WebSocket connection state
third-party libraries
custom global stores
```

React still needs to know:

```text
"What value should I render?"

"When does that value change?"

"How do I subscribe to those changes?"

"How do I do this safely with concurrent rendering?"
```

That's what `useSyncExternalStore` is for.

Current React describes it as the API for subscribing a component to a store that exists outside React. It's particularly useful for third-party state libraries and browser APIs whose values change independently of React. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 1. First: what is an external store?

Imagine we have:

```js
const store = {
  count: 0,
};
```

That object isn't React state.

There's no:

```js
useState()
```

and React has no automatic knowledge that:

```js
store.count
```

has changed.

Suppose:

```js
store.count = 1;
```

What happens?

Nothing automatically.

React doesn't know.

So:

```text
external store changes
        ↓
React doesn't automatically know
        ↓
component doesn't automatically update
```

That's the fundamental problem.

---

# 2. We need two things

React needs to know:

### What is the current value?

```text
getSnapshot()
```

and:

### When should I check again?

```text
subscribe(callback)
```

That's basically the entire API.

```jsx
const value = useSyncExternalStore(
  subscribe,
  getSnapshot
);
```

Current React defines these as the two required arguments: `subscribe` registers for store changes and returns an unsubscribe function, while `getSnapshot` returns the store's current snapshot. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 3. Let's build an external store ourselves

Start simple:

```js
let count = 0;

const listeners = new Set();

export const countStore = {
  getSnapshot() {
    return count;
  },

  subscribe(listener) {
    listeners.add(listener);

    return () => {
      listeners.delete(listener);
    };
  },

  increment() {
    count += 1;

    for (const listener of listeners) {
      listener();
    }
  },
};
```

This store has:

```text
data
 ↓
count

read
 ↓
getSnapshot()

subscription
 ↓
subscribe()

change
 ↓
notify listeners
```

There is no React here.

It's just JavaScript.

---

# 4. Connecting it to React

Now:

```jsx
import { useSyncExternalStore } from "react";
import { countStore } from "./countStore";

function Counter() {
  const count =
    useSyncExternalStore(
      countStore.subscribe,
      countStore.getSnapshot
    );

  return (
    <>
      <p>Count: {count}</p>

      <button
        onClick={countStore.increment}
      >
        Increment
      </button>
    </>
  );
}
```

Now the architecture is:

```text
external store
     │
     ├── getSnapshot
     │
     └── subscribe
             │
             ↓
     useSyncExternalStore
             │
             ↓
        React component
             │
             ↓
             UI
```

---

# 5. Trace what happens when we click

Initial state:

```text
store.count = 0
```

Component renders:

```text
useSyncExternalStore(...)
       ↓
getSnapshot()
       ↓
0
```

So:

```jsx
<p>Count: 0</p>
```

Now the user clicks:

```text
button
 ↓
countStore.increment()
```

Store:

```js
count += 1;
```

Now:

```text
count = 1
```

Store notifies:

```text
listener()
```

React responds by checking:

```text
getSnapshot()
```

It now gets:

```text
1
```

The snapshot changed.

React rerenders:

```text
Count: 1
```

That's the basic mechanism.

---

# 6. Why isn't this just `useState`?

You could write:

```jsx
const [count, setCount] =
  useState(countStore.get());
```

and manually subscribe:

```jsx
useEffect(() => {
  return countStore.subscribe(() => {
    setCount(countStore.get());
  });
}, []);
```

In fact, that pattern can work for many simple cases.

So why does React have a dedicated hook?

Because external stores have subtle interactions with React's concurrent rendering model.

`useSyncExternalStore` provides React with a standardized contract for reading a snapshot and subscribing to changes, including safeguards around consistency during rendering. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 7. The problem is not ordinary rendering

Imagine React is rendering:

```text
Component A
Component B
Component C
```

while an external store changes in the middle of that rendering work.

Potentially:

```text
A reads store = 10

store changes

B reads store = 11

C reads store = 11
```

Now one React render has observed two different versions of the same store.

That's sometimes called **tearing**.

Conceptually:

```text
one render
   ↓
A sees version 10
B sees version 11
```

You don't want a single committed UI tree representing two inconsistent snapshots of the external store.

This is one of the reasons React introduced `useSyncExternalStore`. React's documentation notes that it is designed so external stores can integrate safely with concurrent rendering; if a store changes during a non-blocking Transition, React can restart the update as blocking to ensure the screen reflects a consistent snapshot. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 8. This is why the name is meaningful

Break it apart:

```text
use
+
Sync
+
External
+
Store
```

It's a hook for synchronizing React's rendering with an external store.

The goal isn't:

```text
"make a global variable reactive"
```

It's:

> **Give React a safe, consistent way to observe state that React doesn't own.**

That's much more precise.

---

# 9. `getSnapshot` is extremely important

Suppose:

```js
const store = {
  todos: [...]
};
```

You might write:

```js
function getSnapshot() {
  return {
    todos: store.todos,
  };
}
```

This looks harmless.

But it's actually problematic.

Every call creates a new object:

```text
getSnapshot()
 ↓
Object A

getSnapshot()
 ↓
Object B

getSnapshot()
 ↓
Object C
```

Even though the underlying store hasn't changed:

```text
A !== B
```

React sees a different snapshot.

Current React explicitly requires repeated `getSnapshot()` calls to return the same value while the store hasn't changed, and warns about uncached snapshot results because they can lead to an infinite loop. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 10. The correct version

If the store already maintains immutable data:

```js
const store = {
  todos: [],
};

function getSnapshot() {
  return store.todos;
}
```

Now:

```text
store.todos
      ↓
Array A

unchanged store
      ↓
same Array A

store changes
      ↓
new Array B
```

So:

```text
unchanged
→ same snapshot identity

changed
→ new snapshot identity
```

That is exactly what React wants.

([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 11. This brings us back to immutability

We've been talking about immutable updates for a long time.

Now you can see another reason.

Imagine:

```js
store.todos.push(newTodo);
```

The array identity remains:

```text
same array
```

but its contents changed.

That makes snapshot tracking much harder.

Instead:

```js
store.todos = [
  ...store.todos,
  newTodo,
];
```

Now:

```text
old array
    ↓
new array
```

React can observe:

```text
old snapshot !== new snapshot
```

This is the same identity model we studied with `memo`, `useMemo`, and React state.

The concept is appearing again:

```text
immutability
+
identity
+
change detection
```

---

# 12. An external store is basically a small state machine

Our store has:

```text
current state
+
read operation
+
subscription mechanism
+
update operations
```

For example:

```js
const store = {
  state: {
    count: 0,
  },

  getSnapshot() {
    return store.state;
  },

  subscribe(listener) {
    ...
  },

  increment() {
    store.state = {
      ...store.state,
      count:
        store.state.count + 1,
    };

    notify();
  },
};
```

React doesn't own:

```text
state
```

It owns the subscription to it.

That's the distinction.

---

# 13. Third-party state libraries work similarly

This should make libraries like:

```text
Zustand
Redux
MobX
```

less mysterious.

Their internal implementations differ significantly, but conceptually a React integration has to solve a problem like:

```text
external state
   ↓
how does React read it?
   ↓
how does React know it changed?
```

`useSyncExternalStore` provides the standardized React mechanism for this category. React's documentation explicitly identifies third-party state libraries as one of its primary use cases. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 14. Why Context is different

Compare:

```text
Context
```

with:

```text
external store
```

Context:

```text
React owns the value
       ↓
Provider
       ↓
Consumers
```

External store:

```text
store exists outside React
       ↓
useSyncExternalStore
       ↓
React observes it
```

So:

```text
Context
→ React-native propagation mechanism

External store
→ outside-React state + subscription
```

That's an important distinction.

---

# 15. Why not put everything in an external store?

Current React explicitly recommends built-in React state (`useState`/`useReducer`) when possible and positions `useSyncExternalStore` mainly for integrating with existing non-React code or genuinely external data. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

So don't interpret this lesson as:

> "Global stores are better."

The hierarchy is more like:

```text
Can this state be local?
        ↓
useState

Complex local transitions?
        ↓
useReducer

Shared through a React subtree?
        ↓
Context

Server state?
        ↓
TanStack Query

State lives outside React?
        ↓
external store / useSyncExternalStore
```

The type of state determines the appropriate tool.

---

# 16. Browser APIs are another excellent use case

Here's one React doesn't own:

```js
navigator.onLine
```

The browser can change it when network connectivity changes.

You could do:

```jsx
const isOnline =
  useSyncExternalStore(
    subscribe,
    getSnapshot
  );
```

with:

```js
function getSnapshot() {
  return navigator.onLine;
}

function subscribe(callback) {
  window.addEventListener(
    "online",
    callback
  );

  window.addEventListener(
    "offline",
    callback
  );

  return () => {
    window.removeEventListener(
      "online",
      callback
    );

    window.removeEventListener(
      "offline",
      callback
    );
  };
}
```

Now:

```text
browser
 ↓
navigator.onLine
 ↓
external store-like value
 ↓
subscribe to browser events
 ↓
React
 ↓
UI
```

React's current documentation uses `navigator.onLine` as a canonical browser API example. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 17. Why this is better than our old `useState + useEffect` example

Earlier, we might have written:

```jsx
const [isOnline, setIsOnline] =
  useState(true);

useEffect(() => {
  function handleOnline() {
    setIsOnline(true);
  }

  function handleOffline() {
    setIsOnline(false);
  }

  window.addEventListener(
    "online",
    handleOnline
  );

  window.addEventListener(
    "offline",
    handleOffline
  );

  return () => {
    ...
  };
}, []);
```

There are several problems.

Initial state:

```text
true
```

may be incorrect.

If the device is already offline when the component mounts:

```text
React initially says:
online
```

until the browser fires a future event.

The store-style approach instead reads the actual value immediately:

```js
navigator.onLine
```

and separately subscribes to future changes.

React's documentation explicitly points out this limitation in the `useState`/`useEffect` version and presents `useSyncExternalStore` as the better abstraction for this browser API. ([React](https://react.dev/learn/reusing-logic-with-custom-hooks?utm_source=chatgpt.com "Reusing Logic with Custom Hooks – React"))

---

# 18. Extracting a custom hook

You probably don't want this everywhere:

```jsx
useSyncExternalStore(
  subscribe,
  getSnapshot
);
```

Instead:

```jsx
function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    getSnapshot
  );
}
```

Then:

```jsx
function SaveButton() {
  const isOnline =
    useOnlineStatus();

  return (
    <button
      disabled={!isOnline}
    >
      {isOnline
        ? "Save"
        : "Reconnecting..."}
    </button>
  );
}
```

Now the application code reads naturally.

This is another example of the custom-hook principle:

```text
low-level synchronization
        ↓
custom hook
        ↓
simple component API
```

React's current guidance similarly recommends extracting such subscription logic into a custom Hook. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 19. `subscribe` has an important identity requirement

Suppose you write:

```jsx
function Component() {
  function subscribe(callback) {
    ...
  }

  const value =
    useSyncExternalStore(
      subscribe,
      getSnapshot
    );

  ...
}
```

Every render creates a new:

```text
subscribe function
```

So React sees:

```text
old subscribe !== new subscribe
```

and may unsubscribe/resubscribe.

Current React explicitly recommends defining `subscribe` outside the component when it doesn't depend on component values; if it does, memoize it appropriately. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

So:

```js
function subscribe(callback) {
  ...
}
```

at module scope is often preferable.

---

# 20. `getSnapshot` has the same stability principle

Suppose:

```jsx
function getSnapshot() {
  return {
    online: navigator.onLine,
  };
}
```

Every call:

```text
new object
```

Same underlying value:

```text
online = true
```

but different identity.

That's wrong for `useSyncExternalStore`.

Instead:

```js
function getSnapshot() {
  return navigator.onLine;
}
```

Now the snapshot is simply:

```text
true
```

or:

```text
false
```

Primitives naturally give us stable snapshot values. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 21. What if the store is mutable?

Suppose the store internally uses:

```js
store.todos.push(todo);
```

but the React-facing snapshot should still obey the contract.

You could maintain a cached snapshot:

```js
let snapshot = store.todos;

function updateTodos(nextTodos) {
  store.todos = nextTodos;
  snapshot = nextTodos;
  notify();
}

function getSnapshot() {
  return snapshot;
}
```

Now:

```text
store changed
 ↓
new snapshot created
 ↓
getSnapshot returns new snapshot
```

while repeated reads before changes return:

```text
same snapshot
```

Current React specifically calls out this requirement when the underlying store uses mutable data: the store integration must provide a cached immutable snapshot rather than constructing a new result on every call. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 22. Server rendering adds another argument

The API can be:

```jsx
useSyncExternalStore(
  subscribe,
  getSnapshot,
  getServerSnapshot
);
```

What's the third function?

```text
getServerSnapshot
```

It's used when:

```text
server-rendering
```

and during:

```text
client hydration
```

React needs the server-generated UI and the initial client-rendered UI to agree.

Current React requires `getServerSnapshot` for server rendering when using an external store, and says the server snapshot must match the value used during the initial client hydration. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 23. Why does hydration care?

Imagine the server produces:

```html
Online
```

but when the client hydrates:

```text
Offline
```

The initial trees disagree.

That can create hydration problems.

So:

```text
server snapshot
      ↓
HTML
      ↓
client hydration
      ↓
same initial snapshot
```

is the desired flow.

This becomes particularly important when the external value comes from a browser-only API.

---

# 24. Browser-only APIs create a server problem

For example:

```js
navigator.onLine
```

doesn't exist in the same form on the server.

So:

```js
function getServerSnapshot() {
  return true;
}
```

could establish:

```text
server says "online"
```

and the client initially uses that same snapshot during hydration.

Then React can transition to the actual browser value afterward.

React's current docs use exactly this pattern for the online-status example. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 25. `useSyncExternalStore` and Transitions have an important interaction

This is a more advanced detail, but now that we've learned Transitions, it's worth understanding.

Suppose:

```text
Transition starts
```

while an external store changes.

Because React needs to ensure that all components observe a consistent snapshot, current React can fall back to a blocking update rather than continuing the non-blocking Transition if the store changed during the Transition. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

So:

```text
external store
+
concurrent rendering
```

is not as straightforward as ordinary React state.

This is precisely the problem `useSyncExternalStore` was designed to address.

---

# 26. There's also a current limitation with Suspense

This connects directly to our previous lesson.

React currently advises against suspending based on a snapshot returned by `useSyncExternalStore`.

Why?

Because updates from an external store can't be treated as ordinary non-blocking Transition updates in the same way, and a suspension can produce undesirable replacement by a Suspense fallback. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

So avoid building an architecture that looks like:

```text
external store
 ↓
useSyncExternalStore
 ↓
Promise-dependent Suspense
```

without understanding the implications.

This is an advanced edge of the React rendering model, but it's important because it shows that different state sources have different concurrency properties.

---

# 27. External store vs server state

This distinction is particularly important.

Suppose:

```text
products from backend
```

That's:

```text
server state
```

Use:

```text
TanStack Query
```

Suppose:

```text
shopping cart state stored in a client-side store
```

That's:

```text
client/application state
```

Potentially:

```text
external store
```

Suppose:

```text
navigator.onLine
```

That's:

```text
browser external state
```

So:

```text
server state
≠
external client store
```

Even though both can change outside an individual component.

---

# 28. External store vs Context

Imagine:

```jsx
<ThemeContext value={theme}>
```

Context is excellent when React itself owns:

```text
theme
```

But suppose some existing library owns the theme:

```text
third-party theme store
```

React could subscribe to that external store.

You could then optionally expose the result through Context to parts of the React tree.

So:

```text
external store
 ↓
useSyncExternalStore
 ↓
Context
 ↓
components
```

is possible.

But don't add layers without a reason.

---

# 29. Why libraries need this API more than ordinary app code

Suppose you're writing a state-management library.

You want consumers to write:

```jsx
const user = useStore(...);
```

The library has to integrate its external storage model safely with React.

That's a library-author problem.

Most application developers should simply use:

```text
useState
useReducer
Context
TanStack Query
```

as appropriate.

Then you may never directly write `useSyncExternalStore`.

Current React's hook documentation explicitly notes that this hook is mostly useful for library authors and integration with external systems. ([React](https://react.dev/reference/react/hooks?utm_source=chatgpt.com "Built-in React Hooks – React"))

---

# 30. A useful analogy: React is subscribing to a news source

Imagine an external store is a news agency.

It has:

```text
current headline
```

and:

```text
subscribe()
```

means:

> "Tell me when the headline changes."

and:

```text
getSnapshot()
```

means:

> "What's the headline right now?"

React doesn't own the news agency.

It only wants:

```text
consistent snapshots
+
notifications
```

That's basically `useSyncExternalStore`.

---

# 31. A complete custom external store

Let's make a slightly more realistic example.

```js
let todos = [];

const listeners = new Set();

function notify() {
  for (const listener of listeners) {
    listener();
  }
}

export const todoStore = {
  getSnapshot() {
    return todos;
  },

  subscribe(listener) {
    listeners.add(listener);

    return () => {
      listeners.delete(listener);
    };
  },

  addTodo(todo) {
    todos = [
      ...todos,
      todo,
    ];

    notify();
  },

  deleteTodo(id) {
    todos = todos.filter(
      todo => todo.id !== id
    );

    notify();
  },
};
```

React:

```jsx
function TodoList() {
  const todos =
    useSyncExternalStore(
      todoStore.subscribe,
      todoStore.getSnapshot
    );

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

Another component:

```jsx
function AddTodoButton() {
  return (
    <button
      onClick={() => {
        todoStore.addTodo({
          id: crypto.randomUUID(),
          text: "Learn React",
        });
      }}
    >
      Add
    </button>
  );
}
```

Now:

```text
AddTodoButton
      ↓
external store mutation
      ↓
notify()
      ↓
TodoList subscription
      ↓
getSnapshot()
      ↓
new array
      ↓
TodoList rerenders
```

Notice:

```text
AddTodoButton
```

doesn't need to have:

```text
TodoList state
```

The state is shared by the external store.

That's one reason external stores can be useful.

---

# 32. But there's a cost

Now compare this architecture with:

```jsx
const [todos, setTodos] =
  useState([]);
```

The external store introduces:

```text
subscription system
store module
notification mechanism
snapshot rules
external lifecycle
```

That's a lot more machinery.

So for local state:

```text
useState
```

is usually simpler.

Current React specifically recommends built-in React state when possible rather than using external-store subscriptions unnecessarily. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 33. This gives us a better state decision framework

We now have:

```text
"What kind of information is this?"
```

### Local UI state

```text
modal open
input text
selected tab
hover
```

Use:

```text
useState
```

### Complex related local transitions

```text
task state machine
multi-field workflow
```

Use:

```text
useReducer
```

### Shared React subtree

```text
theme
locale
auth context
```

Use:

```text
Context
```

### Server state

```text
products
orders
users from API
```

Use:

```text
TanStack Query
```

### External state

```text
third-party store
browser API
non-React state system
```

Use:

```text
useSyncExternalStore
```

This is an increasingly powerful way to decide which tool belongs where.

---

# 34. One thing I don't want you to do

Don't see:

```jsx
useSyncExternalStore(...)
```

and conclude:

> "This is how modern React state should be written."

No.

For ordinary React state:

```text
useState
useReducer
```

are still the natural tools.

`useSyncExternalStore` is an **integration boundary**.

That's its strength.

---

# 35. Another deep connection: snapshots

We previously learned:

```text
Every React render sees a snapshot of state.
```

Now external stores must cooperate with that idea.

React needs:

```text
one consistent external-store snapshot
```

for the render it's performing.

So `getSnapshot()` effectively participates in React's snapshot model.

That's why this API isn't simply:

```text
"call setState when something changes."
```

It gives React a way to **read and verify the version of external state it's rendering**.

That is much more sophisticated.

---

# 36. Another connection: memoization

Remember:

```text
same object
→ same identity

new object
→ new identity
```

Now `getSnapshot()` has a similar contract:

```text
store unchanged
→ return same snapshot

store changed
→ return new snapshot
```

So:

```text
memoization
+
immutable snapshots
+
identity comparison
```

all connect.

React can efficiently determine:

```text
"Has the external value actually changed?"
```

using snapshot identity.

Current React explicitly compares snapshots with `Object.is`. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))

---

# 37. Another connection: subscriptions and cleanup

The subscription function:

```js
subscribe(listener) {
  ...
  return unsubscribe;
}
```

is conceptually the same lifecycle pattern we've already seen with Effects:

```text
subscribe
   ↓
component is interested
   ↓
store changes
   ↓
listener called
   ↓
component eventually stops being interested
   ↓
unsubscribe
```

So `useSyncExternalStore` doesn't eliminate the subscription lifecycle.

It gives React a formal place to manage it safely.

---

# 38. A final browser example: `matchMedia`

Suppose you want to know whether:

```text
prefers-color-scheme: dark
```

matches.

The browser exposes a `MediaQueryList`.

You could conceptually build:

```js
const mediaQuery =
  window.matchMedia(
    "(prefers-color-scheme: dark)"
  );

function getSnapshot() {
  return mediaQuery.matches;
}

function subscribe(callback) {
  mediaQuery.addEventListener(
    "change",
    callback
  );

  return () => {
    mediaQuery.removeEventListener(
      "change",
      callback
    );
  };
}
```

Then:

```jsx
const prefersDark =
  useSyncExternalStore(
    subscribe,
    getSnapshot
  );
```

Now the browser owns:

```text
media preference
```

and React observes it.

This is exactly the kind of integration the hook was designed for.

---

# 39. The deeper lesson

We've now seen four different ways state can change:

```text
React state
     ↓
setState

Context
     ↓
provider value changes

Server state
     ↓
network/cache changes

External store
     ↓
external source notifies React
```

The important question is no longer:

> "Which hook should I memorize?"

It's:

> **"Who owns this information?"**

If React owns it:

```text
useState / useReducer
```

If the server owns it:

```text
TanStack Query
```

If some non-React system owns it:

```text
useSyncExternalStore
```

That is a very powerful architectural distinction.

---

# 40. The mental model to keep

When you see:

```jsx
useSyncExternalStore(
  subscribe,
  getSnapshot
);
```

read it as:

```text
There is a state source outside React.

             ↓

getSnapshot()
"What is its current value?"

             ↓

subscribe()
"Tell me when it may have changed."

             ↓

React checks the snapshot.

             ↓

If the snapshot changed:

             ↓

rerender component.
```

And the store must maintain:

```text
stable snapshot while unchanged
+
new snapshot when changed
+
unsubscribe mechanism
```

That's the essence.

---

# 41. Rules worth remembering

```text
1. useSyncExternalStore is for state owned outside React.

2. It is particularly useful for third-party stores and browser APIs.

3. getSnapshot() reads the current external value.

4. subscribe() tells React how to hear about changes.

5. subscribe() must return an unsubscribe function.

6. getSnapshot() must return a cached/stable value while the store
   hasn't changed.

7. Immutable store data makes snapshot handling much easier.

8. Object identity matters because React compares snapshots with Object.is.

9. A new object on every getSnapshot() call is a serious bug.

10. Define subscribe outside the component when it doesn't depend
    on component values.

11. getServerSnapshot() is needed when the external store participates
    in server rendering/hydration.

12. Don't use external stores just because they are possible.
    Prefer useState/useReducer for ordinary React-owned state.

13. Don't confuse external stores with server state.

14. useSyncExternalStore is largely an integration primitive,
    especially valuable to state-library authors.
```

The most important question from this lesson is:

```text
Who owns this state?
```

Because now our answer can be much more precise:

```text
React owns it
      → useState / useReducer

React subtree shares it
      → Context

Server owns it
      → TanStack Query

Another system owns it
      → useSyncExternalStore
```

That distinction is one of the foundations of good React architecture. Current React's own guidance similarly positions `useSyncExternalStore` primarily as the bridge to non-React state sources rather than a replacement for ordinary React state. ([React](https://react.dev/reference/react/useSyncExternalStore?utm_source=chatgpt.com "useSyncExternalStore – React"))