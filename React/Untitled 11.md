
# Lesson 9 — `useRef`: Values That Persist Without Re-rendering

So far, we've learned two very different ways of holding values:

```text
normal variable
state
```

Now React gives us a third tool:

```jsx
useRef()
```

The key idea is:

> **A ref lets a component remember a value between renders without causing a render when that value changes.**

That's fundamentally different from state.

---

# 1. `useState` vs `useRef`

You already know:

```jsx
const [count, setCount] = useState(0);
```

If you do:

```jsx
setCount(1);
```

React schedules a render.

A ref:

```jsx
const countRef = useRef(0);
```

has:

```js
countRef.current
```

So:

```jsx
countRef.current = 1;
```

does **not** tell React to render.

That gives us:

```text
useState
---------
change value
    ↓
React re-renders


useRef
------
change value
    ↓
React does NOT re-render
```

This is the most important distinction.

---

# 2. What exactly does `useRef()` return?

When you write:

```jsx
const ref = useRef(0);
```

you get an object conceptually like:

```js
{
  current: 0
}
```

So:

```jsx
ref.current
```

is the value.

You can mutate it:

```jsx
ref.current = 100;
```

Unlike state, that's allowed.

---

# 3. Why would we want a value that doesn't cause a render?

Because sometimes you need to remember something, but changing that thing doesn't mean the UI should change.

Examples:

```text
DOM element
timer ID
previous value
WebSocket instance
third-party library instance
mutable bookkeeping value
```

Suppose you need to remember an interval:

```jsx
const intervalRef = useRef(null);
```

You can later do:

```jsx
intervalRef.current = setInterval(...);
```

and clearing it:

```jsx
clearInterval(intervalRef.current);
```

There's no reason the screen needs to re-render just because the timer ID changed.

That's a perfect ref use case.

---

# 4. The most common use of `useRef`: DOM access

Imagine:

```jsx
function SearchBox() {
  return (
    <input />
  );
}
```

Suppose you want to focus that input programmatically.

In normal browser JavaScript, you'd do something like:

```js
document.querySelector("input").focus();
```

React gives us a more direct pattern.

```jsx
import { useRef } from "react";

function SearchBox() {
  const inputRef = useRef(null);

  function handleFocus() {
    inputRef.current.focus();
  }

  return (
    <>
      <input ref={inputRef} />

      <button onClick={handleFocus}>
        Focus input
      </button>
    </>
  );
}
```

Now the flow is:

```text
<input>
   ↓
ref={inputRef}
   ↓
React connects ref.current to DOM node
   ↓
button click
   ↓
inputRef.current.focus()
```

That's one of the primary reasons refs exist.

---

# 5. What happens to `inputRef.current`?

Initially:

```jsx
const inputRef = useRef(null);
```

so:

```text
inputRef.current = null
```

After React commits the DOM and attaches the ref:

```text
inputRef.current = actual <input> DOM node
```

So later:

```jsx
inputRef.current.focus();
```

works.

When that DOM node is removed, React will clear the ref appropriately.

---

# 6. `useRef` doesn't magically find an element

This:

```jsx
const inputRef = useRef(null);
```

doesn't itself locate anything.

The connection happens because of:

```jsx
<input ref={inputRef} />
```

Think:

```text
useRef() → creates ref object

ref={inputRef} → tells React which DOM node to attach it to
```

---

# 7. State would be the wrong tool here

You might ask:

> Why not store the input element in state?

Like:

```jsx
const [input, setInput] = useState(null);
```

That would be awkward because:

1. you don't need the DOM node displayed in the UI
    
2. storing it in state would trigger rendering when changed
    
3. refs are specifically designed for this sort of imperative access
    

So:

```text
Need value to affect UI?
→ state

Need to remember/access something without causing UI updates?
→ ref
```

That's a useful rule of thumb.

---

# 8. Refs survive re-renders

This is another key property.

Suppose:

```jsx
function Counter() {
  const countRef = useRef(0);

  // ...
}
```

On render 1:

```text
countRef.current = 0
```

Then:

```jsx
countRef.current++;
```

Now:

```text
countRef.current = 1
```

Later, some state change causes another render.

The ref still contains:

```text
1
```

It doesn't reset to `0`.

That's the important difference from an ordinary local variable.

---

# 9. Compare all three

Let's put them side by side.

### Normal variable

```js
let count = 0;
```

```text
survives?
→ no, not across component renders

causes render?
→ no
```

### State

```jsx
const [count, setCount] = useState(0);
```

```text
survives?
→ yes

causes render when changed?
→ yes
```

### Ref

```jsx
const count = useRef(0);
```

```text
survives?
→ yes

causes render when changed?
→ no
```

So:

```text
                 persists   re-renders
variable            no          no
state               yes         yes
ref                 yes         no
```

That table is worth remembering.

---

# 10. Ref as a "persistent box"

A useful mental model is:

```text
useRef(value)
     ↓
┌────────────────┐
│ current: value │
└────────────────┘
```

The box persists across renders.

You can change what's inside:

```jsx
ref.current = newValue;
```

without React rendering again.

So think:

> **A ref is a persistent mutable box attached to the component instance.**

That's more accurate than thinking of it as simply "a DOM selector."

---

# 11. `useRef` for previous values

Suppose you want to know what the previous count was.

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const previousCount = useRef(null);

  useEffect(() => {
    previousCount.current = count;
  }, [count]);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {previousCount.current}</p>

      <button onClick={() => setCount(c => c + 1)}>
        Increase
      </button>
    </div>
  );
}
```

There is a subtle timing issue here, though.

Because the Effect runs after rendering, the value shown as "previous" can require careful thinking.

A better example for understanding the ref itself is:

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  const previousCount = useRef();

  useEffect(() => {
    previousCount.current = count;
  }, [count]);

  // ...
}
```

The ref lets us remember something from earlier without making that remembered value itself drive rendering.

We will revisit this pattern when we study Effects and timing more deeply.

---

# 12. Refs are useful for timers

Suppose:

```jsx
function Timer() {
  const timerRef = useRef(null);

  function start() {
    timerRef.current = setInterval(() => {
      console.log("tick");
    }, 1000);
  }

  function stop() {
    clearInterval(timerRef.current);
  }

  return (
    <>
      <button onClick={start}>
        Start
      </button>

      <button onClick={stop}>
        Stop
      </button>
    </>
  );
}
```

Why use a ref?

Because:

```text
timer ID
```

doesn't need to be displayed.

It just needs to be remembered so later code can access it.

Putting that in state would cause unnecessary renders.

---

# 13. A very important rule: don't use a ref for things that should appear on screen

This is wrong conceptually:

```jsx
const countRef = useRef(0);

function increase() {
  countRef.current++;
}
```

and then expecting:

```jsx
<p>{countRef.current}</p>
```

to update automatically.

It won't.

Because:

```jsx
countRef.current++
```

does not trigger a render.

So if the UI needs to change:

```text
useState
```

is generally the right tool.

---

# 14. Ref values are mutable

State is generally treated immutably:

```jsx
setUser({
  ...user,
  name: "Sarah"
});
```

A ref is intentionally mutable:

```jsx
ref.current = "Sarah";
```

That's okay.

That's actually part of what refs are for.

But this mutation still doesn't tell React:

> "Update the UI."

That's the distinction.

---

# 15. Refs and imperative code

React is primarily declarative:

```text
state
 ↓
UI
```

But sometimes you need an imperative command:

```text
focus this input
play this video
scroll this element
open this browser-controlled widget
measure this DOM node
```

Refs give you an escape hatch into imperative APIs.

So:

```text
declarative React
       ↓
     ref
       ↓
imperative DOM/API
```

That's why refs fit nicely with the React mental model.

---

# 16. `useRef` with `scrollIntoView`

Suppose:

```jsx
function App() {
  const sectionRef = useRef(null);

  function scrollToSection() {
    sectionRef.current.scrollIntoView();
  }

  return (
    <>
      <button onClick={scrollToSection}>
        Go to section
      </button>

      <div style={{ height: "1000px" }} />

      <section ref={sectionRef}>
        Target section
      </section>
    </>
  );
}
```

The flow:

```text
button clicked
   ↓
scrollToSection()
   ↓
sectionRef.current
   ↓
DOM element
   ↓
scrollIntoView()
```

That's exactly what refs are good at.

---

# 17. A ref isn't necessarily a DOM element

This is important.

You can have:

```jsx
const timerRef = useRef(null);
```

where `current` stores:

```text
timer ID
```

Or:

```jsx
const previousValue = useRef(null);
```

where `current` stores:

```text
some previous value
```

Or:

```jsx
const socketRef = useRef(null);
```

where `current` stores:

```text
WebSocket instance
```

So:

> Ref ≠ DOM element.

DOM access is just the most visible use case.

---

# 18. Refs and effects often work together

This is extremely common.

Suppose you need a connection object:

```jsx
function Chat() {
  const connectionRef = useRef(null);

  useEffect(() => {
    const connection = createConnection();

    connection.connect();

    connectionRef.current = connection;

    return () => {
      connection.disconnect();
    };
  }, []);

  // ...
}
```

The Effect handles synchronization with the external system.

The ref remembers the connection object.

So these tools can solve different parts of the same problem:

```text
useEffect
→ synchronize with external system

useRef
→ keep a reference to the external object
```

---

# 19. Another common misconception

You may see:

```jsx
const ref = useRef(0);
```

and think:

> "That's basically state, except I access `.current`."

Not really.

The key behavioral difference is:

```text
state update → render
ref mutation → no render
```

That difference is foundational.

---

# 20. `useRef` and state together

Consider an auto-incrementing counter.

You might have:

```jsx
const [count, setCount] = useState(0);
const intervalRef = useRef(null);
```

Here:

```text
count
→ visible UI state

intervalRef
→ internal timer handle
```

Perfect separation of responsibilities.

---

# 21. A practical example: focus after opening a modal

Suppose:

```jsx
function Modal({ open }) {
  const inputRef = useRef(null);

  useEffect(() => {
    if (open) {
      inputRef.current?.focus();
    }
  }, [open]);

  if (!open) {
    return null;
  }

  return (
    <div>
      <input ref={inputRef} />
    </div>
  );
}
```

Now several React concepts work together:

```text
open prop
   ↓
render modal
   ↓
ref attaches to input
   ↓
Effect runs
   ↓
focus input
```

Notice the distinction:

- `open` determines what React renders
    
- `ref` gives access to the DOM node
    
- `useEffect` performs the imperative synchronization
    

This is a very "React-native" pattern.

---

# 22. Ref timing

One thing to remember:

During rendering:

```jsx
inputRef.current
```

may be:

```text
null
```

because React hasn't necessarily committed the DOM node yet.

So don't generally expect:

```jsx
function Component() {
  const ref = useRef(null);

  console.log(ref.current);

  return <input ref={ref} />;
}
```

to give you the input during render.

The DOM node becomes available after the relevant commit.

That's one reason Effects are often used when you need to act on the DOM after it exists.

---

# 23. `useRef` vs `useEffect`

These solve different problems.

### `useRef`

> "I need to remember/access this value across renders."

### `useEffect`

> "I need to synchronize with something external after rendering."

Together:

```text
ref
 ↓
stores external object / DOM node

effect
 ↓
interacts with external object / DOM node
```

That's an extremely useful pairing.

---

# 24. A mental decision tree

When you have a value, ask:

### Does changing it need to update the UI?

Yes:

```text
useState
```

No:

```text
Could it simply be recalculated during render?
→ ordinary variable / derived value

Does it need to persist across renders?
→ useRef
```

And separately:

### Does changing/synchronizing it involve an external system?

Yes:

```text
useEffect
```

So the tools aren't competitors.

They address different problems.

---

# 25. Let's compare four examples

### Example 1

```jsx
const fullName = firstName + " " + lastName;
```

Use:

```text
ordinary derived value
```

No state. No ref. No Effect.

---

### Example 2

```jsx
const [name, setName] = useState("");
```

Use:

```text
state
```

because UI depends on it.

---

### Example 3

```jsx
const inputRef = useRef(null);
```

Use:

```text
ref
```

because you need a persistent reference to a DOM node.

---

### Example 4

```jsx
useEffect(() => {
  connection.connect();

  return () => connection.disconnect();
}, [roomId]);
```

Use:

```text
Effect
```

because you're synchronizing an external system.

---

# 26. The bigger React picture

Your mental model can now be expanded to:

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
                user event
                     ↓
                event handler
                     ↓
                state update
                     ↓
                 re-render
```

And now:

```text
                  COMPONENT
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
       useRef                  useEffect
          ↓                       ↓
remember mutable value      external system
without render              synchronization
```

That's the architecture we're building toward.

---

# 27. One subtle point about refs and rendering

Suppose:

```jsx
function Component() {
  const ref = useRef(0);

  ref.current++;

  return <p>{ref.current}</p>;
}
```

This is a bad pattern.

Why?

Because you're mutating the ref during rendering.

Rendering should generally be kept pure: same inputs should produce the same UI description, without mutating external/persistent state as part of the render calculation.

Refs are meant for imperative code outside the render calculation, not as a way to sneak mutable state into rendering.

The React docs specifically caution against writing to refs during rendering except for tightly controlled initialization patterns. ([react.dev](https://react.dev/reference/react/useRef?utm_source=chatgpt.com))

---

# 28. The simplest rule to remember

For now, this is enough:

```text
Need the UI to react to the change?
→ useState

Need to remember something without triggering a render?
→ useRef

Need to synchronize with something outside React?
→ useEffect
```

And frequently:

```text
useRef + useEffect
```

work together.

---

# Practice

Consider this application:

```text
Search page

[ Search input ]

[ Focus Search ]

[ Start timer ]
[ Stop timer ]
```

You need:

```text
search text → displayed in UI
input DOM node → focus it
timer ID → remember it
```

What should each use?

Think:

```text
search text
→ ?

input element
→ ?

timer ID
→ ?
```

The answer:

```text
search text → useState
input element → useRef
timer ID → useRef
```

And if you need to start/stop an external timer or synchronize with an external subscription, an Effect may be involved too.

---

## Next lesson — `useContext`

We've now covered:

```text
useState
useEffect
useRef
```

The next problem is **sharing data across a component tree without manually passing it through every intermediate component**.

We'll build:

```text
App
 ↓
Layout
 ↓
Navbar
 ↓
UserMenu
```

and see what happens when `UserMenu` needs information owned much higher up.

That's where **Context** comes in, and we'll also cover an important warning: Context is useful, but it is **not simply "global state."**