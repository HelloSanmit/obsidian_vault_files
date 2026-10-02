
# Lesson 8 — `useEffect`: What an Effect Actually Is

This is a particularly important lesson, because `useEffect` is probably the hook most often learned incorrectly.

The official React docs currently define `useEffect` as a hook for **synchronizing a component with an external system**. They explicitly recommend avoiding Effects for ordinary data transformation or event handling when no external system is involved. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

So let's build the idea from scratch.

---

# 1. First: what is an "external system"?

Think about the things React itself does:

```text
state
props
JSX
rendering
component tree
```

These are inside React's world.

But your component may need to interact with things outside React:

```text
browser DOM APIs
timers
network connections
WebSockets
event listeners
third-party libraries
media players
browser APIs
```

For example:

```text
React says:
"isPlaying = true"

Browser video:
"Actually, I need you to call video.play()"
```

React state and the external system need to be synchronized.

That's where an Effect comes in. ([React](https://react.dev/learn/synchronizing-with-effects?utm_source=chatgpt.com "Synchronizing with Effects – React"))

---

# 2. The simplest possible example

Imagine a component receives:

```jsx
<VideoPlayer isPlaying={isPlaying} />
```

The browser's `<video>` element has imperative methods:

```js
video.play()
video.pause()
```

Your React component can connect the two:

```jsx
import { useEffect, useRef } from "react";

function VideoPlayer({ isPlaying }) {
  const videoRef = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      videoRef.current.play();
    } else {
      videoRef.current.pause();
    }
  }, [isPlaying]);

  return (
    <video
      ref={videoRef}
      src="/video.mp4"
    />
  );
}
```

Here:

```text
React state/props
      ↓
   isPlaying
      ↓
   useEffect
      ↓
external browser API
      ↓
video.play() / video.pause()
```

This is a textbook use for `useEffect`: React state needs to stay synchronized with something that React doesn't directly control. ([React](https://react.dev/learn/synchronizing-with-effects?utm_source=chatgpt.com "Synchronizing with Effects – React"))

---

# 3. Why can't we just do this during rendering?

You might imagine:

```jsx
function VideoPlayer({ isPlaying }) {
  if (isPlaying) {
    video.play();
  } else {
    video.pause();
  }

  return <video />;
}
```

That is the wrong mental model.

Rendering should primarily calculate:

```text
"Given my current inputs, what should the UI look like?"
```

Calling an external system such as `video.play()` is a side effect.

So we separate:

```text
rendering
```

from:

```text
synchronization with external systems
```

That's the purpose of an Effect.

---

# 4. The basic syntax

```jsx
useEffect(() => {
  // effect code
}, [dependencies]);
```

There are two pieces:

```text
setup function
+
dependency list
```

The first tells React:

> "Here's the synchronization work."

The second tells React:

> "This synchronization depends on these reactive values."

That's the conceptual model.

---

# 5. What does `useEffect` return?

This is important because there are **two different "returns"** to distinguish.

Consider:

```jsx
useEffect(() => {
  console.log("Effect");
});
```

The `useEffect(...)` hook itself returns:

```text
undefined
```

But the function _inside_ the Effect can return a cleanup function:

```jsx
useEffect(() => {
  // setup

  return () => {
    // cleanup
  };
}, []);
```

So:

```text
useEffect()
    ↓
returns undefined

setup function
    ↓
may return cleanup function
```

Don't mix those two up.

---

# 6. Cleanup

Suppose you start an interval:

```jsx
useEffect(() => {
  const id = setInterval(() => {
    console.log("tick");
  }, 1000);

  return () => {
    clearInterval(id);
  };
}, []);
```

Setup:

```text
create interval
```

Cleanup:

```text
destroy interval
```

This gives you the symmetry:

```text
setup   → start something
cleanup → stop that same thing
```

React's documentation emphasizes that cleanup should "mirror" the setup. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 7. Why cleanup is necessary

Imagine:

```jsx
useEffect(() => {
  const id = setInterval(() => {
    console.log("tick");
  }, 1000);
}, []);
```

You create an interval.

But what happens when the component is removed?

The interval is still running unless you stop it.

That can mean:

```text
component gone
     ↓
interval still running
     ↓
unnecessary work
```

So:

```jsx
return () => clearInterval(id);
```

says:

> "When this effect needs to stop, undo what I started."

---

# 8. The Effect lifecycle

This is one of the most important things to understand.

Suppose:

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);

    connection.connect();

    return () => {
      connection.disconnect();
    };
  }, [roomId]);

  return <h1>{roomId}</h1>;
}
```

React's conceptual sequence is:

```text
Component appears
      ↓
setup runs
      ↓
connect to room A
```

Then:

```text
roomId changes A → B
      ↓
cleanup for A
      ↓
disconnect A
      ↓
setup for B
      ↓
connect B
```

Then eventually:

```text
component removed
      ↓
cleanup
      ↓
disconnect B
```

That's the actual lifecycle of this Effect. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 9. Effect lifecycle is not exactly component lifecycle

This distinction is important.

Beginners often learn:

```text
mount
update
unmount
```

and then think:

> "useEffect runs on mount."

That's incomplete.

A better model is:

```text
Effect starts
     ↓
Effect continues synchronizing
     ↓
dependency changes
     ↓
old synchronization stops
     ↓
new synchronization starts
     ↓
component removed
     ↓
synchronization stops
```

React's docs explicitly describe an Effect as having its own lifecycle of setup → cleanup → setup as dependencies change. ([React](https://react.dev/learn/lifecycle-of-reactive-effects?utm_source=chatgpt.com "Lifecycle of Reactive Effects – React"))

---

# 10. Dependency arrays

Now we reach the part everyone memorizes but few people initially understand.

Consider:

```jsx
useEffect(() => {
  console.log("Effect");
}, [roomId]);
```

The effect uses:

```text
roomId
```

So `roomId` is a dependency.

React compares the current dependency values with the previous ones using `Object.is`. When a dependency changes, React reruns the effect's synchronization. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 11. Three common forms

## No dependency array

```jsx
useEffect(() => {
  console.log("Effect");
});
```

This runs after every commit of the component.

Conceptually:

```text
render
 ↓
commit
 ↓
effect
```

Then another render:

```text
render
 ↓
commit
 ↓
effect
```

and so on. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

## Empty dependency array

```jsx
useEffect(() => {
  console.log("Effect");
}, []);
```

There are no reactive dependencies.

This is commonly used for an Effect whose setup doesn't depend on changing props/state.

But don't mentally define this as simply:

> "run once."

That framing becomes misleading when you encounter development Strict Mode and more complex component behavior.

---

## Dependencies

```jsx
useEffect(() => {
  console.log(roomId);
}, [roomId]);
```

Now the Effect synchronizes based on `roomId`.

When `roomId` changes:

```text
cleanup old
   ↓
setup new
```

---

# 12. The most important dependency rule

Suppose:

```jsx
function ChatRoom({ roomId }) {
  const [serverUrl, setServerUrl] = useState("...");
  
  useEffect(() => {
    connect(serverUrl, roomId);
  }, [serverUrl, roomId]);
}
```

The Effect reads:

```text
serverUrl
roomId
```

Therefore both belong in the dependency list.

React's current documentation describes reactive values as including props, state, and values/functions declared in the component body. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

So don't think:

> "I'll put whatever dependencies I feel like."

Think:

> **What reactive values does this Effect read?**

Those determine the dependency list.

---

# 13. The effect dependency array is not a magic scheduling menu

A common beginner mentality is:

```jsx
useEffect(..., [])
```

means:

> "I want this to run once."

and:

```jsx
useEffect(..., [x])
```

means:

> "I want this to run whenever I feel like x changes."

A better understanding is:

```text
The dependency list describes what
the Effect's synchronization depends on.
```

That's why React treats dependency analysis seriously.

It's describing the relationship between the effect and its reactive inputs.

---

# 14. Here's an important example

Suppose:

```jsx
function ChatRoom({ roomId }) {
  const [message, setMessage] = useState("");

  useEffect(() => {
    const connection = createConnection(roomId);

    connection.connect();

    return () => connection.disconnect();
  }, [roomId]);

  return (
    <input
      value={message}
      onChange={e => setMessage(e.target.value)}
    />
  );
}
```

What happens when the user types?

```text
message changes
     ↓
component rerenders
```

But:

```text
roomId
```

didn't change.

Therefore the Effect does not need to restart.

This is exactly why the dependency list contains `roomId`, not `message`. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 15. Why Effects can accidentally run too often

Consider:

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const options = {
      roomId
    };

    connect(options);
  }, [options]);
}
```

This is problematic because `options` is created during rendering.

Each render creates a new object:

```text
render 1 → options object A
render 2 → options object B
render 3 → options object C
```

Even though the contents may be the same, the object identity changes.

React compares dependencies with `Object.is`, so the Effect can rerun. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

One way to fix the design is to create the object inside the Effect:

```jsx
useEffect(() => {
  const options = {
    roomId
  };

  connect(options);
}, [roomId]);
```

Now the synchronization depends directly on the meaningful reactive value.

The React docs specifically recommend removing unnecessary object/function dependencies before reaching for memoization. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 16. Why functions can cause the same issue

Suppose:

```jsx
function ChatRoom({ roomId }) {
  function createOptions() {
    return {
      roomId
    };
  }

  useEffect(() => {
    const options = createOptions();
    connect(options);
  }, [createOptions]);
}
```

`createOptions` is a new function on each render.

So its identity changes.

That can make the Effect rerun more frequently than intended.

Again, first question:

> Can I structure this so the Effect depends on the actual reactive value rather than a newly-created function?

Often:

```jsx
useEffect(() => {
  const options = {
    roomId
  };

  connect(options);
}, [roomId]);
```

is cleaner. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 17. The biggest useEffect mistake: derived data

Suppose:

```jsx
const [firstName, setFirstName] = useState("");
const [lastName, setLastName] = useState("");
```

You want:

```text
fullName
```

A beginner may write:

```jsx
const [fullName, setFullName] = useState("");

useEffect(() => {
  setFullName(`${firstName} ${lastName}`);
}, [firstName, lastName]);
```

This works, but it's unnecessarily complicated.

Why?

Because `fullName` can already be calculated:

```jsx
const fullName = `${firstName} ${lastName}`;
```

No external system exists.

It's just derived data.

React's current guidance explicitly says you generally don't need an Effect for transforming data used for rendering. ([React](https://react.dev/learn/you-might-not-need-an-effect?utm_source=chatgpt.com "You Might Not Need an Effect – React"))

---

# 18. This is exactly what you did earlier with `visibleTasks`

You had:

```jsx
const visibleTasks = tasks.filter(...);
```

Good.

You should **not** do:

```jsx
const [visibleTasks, setVisibleTasks] = useState([]);

useEffect(() => {
  setVisibleTasks(
    tasks.filter(...)
  );
}, [tasks, filter]);
```

Why?

Because:

```text
tasks + filter
      ↓
visibleTasks
```

is ordinary calculation.

An Effect would introduce an extra state update and another render cycle.

React's docs specifically call out this exact category of mistake. ([React](https://react.dev/learn/you-might-not-need-an-effect?utm_source=chatgpt.com "You Might Not Need an Effect – React"))

---

# 19. Another big mistake: user events

Suppose a user clicks Buy.

You need to send:

```text
POST /api/buy
```

A beginner might create:

```jsx
const [shouldBuy, setShouldBuy] = useState(false);

useEffect(() => {
  if (shouldBuy) {
    buyProduct();
  }
}, [shouldBuy]);
```

This is usually the wrong abstraction.

You already know exactly what caused the action:

```text
the user clicked Buy
```

So put the action in the event handler:

```jsx
function handleBuy() {
  buyProduct();
}
```

React's current docs explicitly recommend event handlers for logic caused by specific user interactions. ([React](https://react.dev/learn/you-might-not-need-an-effect?utm_source=chatgpt.com "You Might Not Need an Effect – React"))

---

# 20. Event vs Effect

This distinction is worth memorizing.

### Event handler

Use when:

```text
the user did something
```

Examples:

```text
click
submit
typing
drag
select
```

Mental model:

```text
USER ACTION
    ↓
EVENT HANDLER
    ↓
ACTION
```

### Effect

Use when:

```text
the component needs to synchronize
with an external system
```

Mental model:

```text
RENDERED REACT STATE
       ↓
     EFFECT
       ↓
EXTERNAL SYSTEM
```

This distinction will save you from a tremendous amount of unnecessary `useEffect`.

---

# 21. Example: browser event listener

Suppose you need to listen to window resize.

```jsx
useEffect(() => {
  function handleResize() {
    console.log(window.innerWidth);
  }

  window.addEventListener(
    "resize",
    handleResize
  );

  return () => {
    window.removeEventListener(
      "resize",
      handleResize
    );
  };
}, []);
```

Why is this an Effect?

Because you're synchronizing React with a browser event system that exists outside React.

Setup:

```text
addEventListener
```

Cleanup:

```text
removeEventListener
```

Perfect setup/cleanup symmetry.

---

# 22. Timer example

```jsx
useEffect(() => {
  const id = setInterval(() => {
    console.log("tick");
  }, 1000);

  return () => {
    clearInterval(id);
  };
}, []);
```

Again:

```text
Effect starts
    ↓
external timer starts

cleanup
    ↓
external timer stops
```

That's what an Effect is good at.

---

# 23. Chat connection example

This is one of the clearest examples.

```jsx
function ChatRoom({ roomId }) {
  useEffect(() => {
    const connection = createConnection(roomId);

    connection.connect();

    return () => {
      connection.disconnect();
    };
  }, [roomId]);

  return <h1>Room: {roomId}</h1>;
}
```

Think:

```text
roomId
  ↓
React UI
  ↓
Effect
  ↓
chat connection
```

If roomId changes:

```text
general
  ↓
cleanup general
  ↓
setup travel
```

The effect is maintaining synchronization between:

```text
React's current roomId
```

and:

```text
the external chat connection
```

That's the cleanest way to understand `useEffect`. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 24. Strict Mode and "Why did this run twice?"

You may encounter this while developing.

With Strict Mode enabled, React may perform an extra development-only:

```text
setup
cleanup
setup
```

cycle before the first real setup.

This is intended to expose Effects whose setup and cleanup aren't properly symmetrical. It is development behavior, not the same as the production sequence. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

So don't "fix" a correct Effect by trying to prevent that behavior with hacks.

Instead ask:

> If setup runs, cleanup runs, and setup runs again, does my code still behave correctly?

A properly written Effect should.

---

# 25. An infinite loop

Consider:

```jsx
function App() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    setCount(count + 1);
  }, [count]);

  return <p>{count}</p>;
}
```

Trace it:

```text
render
 ↓
Effect runs
 ↓
setCount(count + 1)
 ↓
render
 ↓
count changed
 ↓
Effect runs
 ↓
setCount(...)
 ↓
render
...
```

Infinite loop.

React's documentation describes this as requiring two conditions:

```text
1. The Effect updates state.
2. That state change causes one of its dependencies to change.
```

([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

The solution isn't:

> "Find a dependency array trick."

The first question should be:

> **Why am I using an Effect for this at all?**

If there is no external system being synchronized, the Effect may simply be the wrong design. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

---

# 26. The classic "useEffect for everything" trap

Bad mental model:

```text
Something changed
   ↓
useEffect
```

Better:

```text
Something changed
   ↓
What caused it?

User event?
   → event handler

Derived calculation?
   → calculate during render

External system needs synchronization?
   → useEffect

Need mutable value without rerender?
   → later: useRef
```

This is a much better decision tree.

---

# 27. Fetching data

This is an interesting case.

You can fetch data inside an Effect:

```jsx
useEffect(() => {
  fetch("/api/users")
    .then(res => res.json())
    .then(data => setUsers(data));
}, []);
```

This is a legitimate use of an Effect in many client-side applications because you're synchronizing with a network resource. React's docs still describe fetching in Effects as possible, while noting that modern frameworks can provide more efficient built-in data-fetching approaches. ([React](https://react.dev/reference/react/useEffect?utm_source=chatgpt.com "useEffect – React"))

But fetching introduces additional problems:

```text
loading
error
cleanup
race conditions
cancellation
stale responses
```

We'll tackle those in the data-fetching section rather than throwing all of that into today's lesson.

---

# 28. The mental model I want you to use

Don't think:

```text
useEffect = run some code after render
```

That's technically useful but too shallow.

Think:

> **`useEffect` lets your component synchronize something outside React with the current React state/props.**

For example:

```text
React state
    ↓
isPlaying = true
    ↓
Effect
    ↓
video.play()
```

or:

```text
React prop
    ↓
roomId = "travel"
    ↓
Effect
    ↓
connect to travel room
```

or:

```text
component exists
    ↓
Effect
    ↓
add window event listener
```

or:

```text
component exists
    ↓
Effect
    ↓
start timer
```

The word **synchronize** is the key.

---

# 29. Compare these four examples

### Example A

```jsx
const fullName = `${firstName} ${lastName}`;
```

**No Effect.**

It's derived data.

---

### Example B

```jsx
function handleSubmit() {
  sendOrder();
}
```

**No Effect.**

It's a user event.

---

### Example C

```jsx
useEffect(() => {
  window.addEventListener(...);

  return () => {
    window.removeEventListener(...);
  };
}, []);
```

**Effect.**

External browser system.

---

### Example D

```jsx
useEffect(() => {
  connection.connect();

  return () => {
    connection.disconnect();
  };
}, [roomId]);
```

**Effect.**

External network connection.

---

# 30. A very useful test

Before writing:

```jsx
useEffect(() => {
  ...
}, [...]);
```

ask:

> **What external system am I synchronizing with?**

If your answer is:

```text
"None."
```

stop and reconsider.

You may actually need:

```text
ordinary calculation
event handler
state
derived data
```

instead.

That is directly aligned with React's current guidance. ([React](https://react.dev/learn/you-might-not-need-an-effect?utm_source=chatgpt.com "You Might Not Need an Effect – React"))

---

# 31. One final example tying everything together

Consider:

```jsx
function SearchResults({ products, query }) {
  const filteredProducts = products.filter(product =>
    product.name
      .toLowerCase()
      .includes(query.toLowerCase())
  );

  return (
    <ul>
      {filteredProducts.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

No Effect.

Why?

```text
products + query
      ↓
filteredProducts
      ↓
UI
```

That's pure rendering logic.

Now imagine we need to connect the UI to a browser subscription:

```jsx
useEffect(() => {
  const handler = () => {
    // synchronize with external system
  };

  window.addEventListener("resize", handler);

  return () => {
    window.removeEventListener("resize", handler);
  };
}, []);
```

That's an Effect.

Different problem → different tool.

---

# 32. Your current React mental model

You've now accumulated something pretty substantial:

```text
                PROPS
                  ↓
               COMPONENT
                  ↑
                STATE
                  │
                  ↓
                 JSX
                  ↓
                  UI
                  │
             user events
                  ↓
             event handler
                  ↓
             state update
                  ↓
               render
```

And now there's another branch:

```text
                 RENDER
                   ↓
               useEffect
                   ↓
          external system
                   ↓
        browser / network /
        timer / subscription /
        third-party library
```

That's the distinction I want you to internalize.

---

# A small exercise

For each of these, decide whether you'd use an Effect:

### A

```js
const fullName = firstName + " " + lastName;
```

### B

```text
When user clicks Buy → send POST request
```

### C

```text
When roomId changes → reconnect WebSocket
```

### D

```text
Display products filtered by searchTerm
```

### E

```text
Listen to browser window resize events
```

The answers are:

```text
A → no Effect
B → event handler
C → Effect
D → no Effect
E → Effect
```

And that's the basic judgment you need to develop.

---

## Next lesson — `useRef`

Now that you understand the distinction between React's rendering world and external systems, `useRef` will make much more sense.

We'll cover why:

```jsx
const ref = useRef(null);
```

is **not another kind of state**, why changing `ref.current` doesn't trigger a render, how refs access DOM elements, and why refs are also useful for storing mutable values that should survive renders without causing the UI to update.