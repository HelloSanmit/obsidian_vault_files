
# Lesson 28 — Refs Across Components: `ref` as a Prop, `useImperativeHandle`, and Imperative APIs

We learned `useRef` earlier:

```jsx
const inputRef = useRef(null);

<input ref={inputRef} />
```

and:

```js
inputRef.current.focus();
```

That gave us the basic idea:

```text
React state
→ controls what the UI should be

ref
→ gives us access to something we need to interact with imperatively
```

Now we're going to answer a more advanced question:

> **What happens when the DOM node I want isn't directly inside the component that owns the ref?**

For example:

```text
Page
 ↓
Form
 ↓
CustomInput
 ↓
<input>
```

The parent may want to focus the actual `<input>`.

That leads us into **refs across component boundaries**.

And current React has an important modern change here: starting in React 19, function components can receive `ref` as a normal prop, so new code no longer needs `forwardRef` for this purpose. `forwardRef` is planned for deprecation/removal in a future release. ([React](https://uk.react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

---

# 1. Why a normal ref doesn't automatically go through a component

Suppose:

```jsx
function MyInput() {
  return <input />;
}
```

And parent:

```jsx
function Form() {
  const inputRef = useRef(null);

  return (
    <MyInput ref={inputRef} />
  );
}
```

You might expect:

```text
inputRef.current
    ↓
<input>
```

But there is an important concept here.

A `ref` attached to a built-in DOM element:

```jsx
<input ref={inputRef} />
```

can give you the DOM node.

A `ref` attached to a function component needs that component to explicitly decide what the ref exposes.

Current React 19 makes `ref` available to function components as a prop, so you can explicitly forward/use it yourself. ([React](https://uk.react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

---

# 2. React 19's modern pattern

You can now write:

```jsx
function MyInput({ ref, placeholder }) {
  return (
    <input
      ref={ref}
      placeholder={placeholder}
    />
  );
}
```

Then:

```jsx
function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput
        ref={inputRef}
        placeholder="Enter your name"
      />

      <button onClick={handleClick}>
        Focus input
      </button>
    </>
  );
}
```

The flow is:

```text
Form
 │
 │ ref
 ↓
MyInput
 │
 │ ref
 ↓
<input>
```

So the ref effectively travels through the custom component.

---

# 3. Notice what `ref` is now

In modern React:

```jsx
function MyInput({ ref }) {
  ...
}
```

`ref` can be treated as a prop for function components. ([React](https://uk.react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

So this:

```jsx
<MyInput ref={inputRef} />
```

can conceptually be understood as:

```text
Form
  ↓
passes ref
  ↓
MyInput receives ref
  ↓
MyInput decides where/how to expose it
```

That is a useful evolution in React's API.

---

# 4. Before React 19, you will still encounter `forwardRef`

A lot of existing React code looks like:

```jsx
const MyInput = forwardRef(function MyInput(
  props,
  ref
) {
  return <input {...props} ref={ref} />;
});
```

Then:

```jsx
<MyInput ref={inputRef} />
```

This is **old/common React code that you should absolutely be able to read**.

It isn't obsolete syntax in the sense that you'll never encounter it.

The important current distinction is:

```text
React 18 and earlier style:
forwardRef

React 19+:
ref can be received as a normal prop
```

React's current documentation says `forwardRef` is no longer necessary in React 19 and is planned for deprecation in a future release. ([React](https://mn.react.dev/reference/react/forwardRef?utm_source=chatgpt.com "forwardRef – React"))

So from now on, I'll teach the modern form first, but you'll know how to read the older form.

---

# 5. The bigger question: what should the parent receive?

Suppose you have:

```jsx
function MyInput({ ref }) {
  return <input ref={ref} />;
}
```

Then the parent receives:

```text
the actual DOM node
```

So:

```js
inputRef.current
```

might be:

```text
HTMLInputElement
```

That gives the parent a lot of power.

It might do:

```js
inputRef.current.focus();
inputRef.current.select();
inputRef.current.scrollIntoView();
inputRef.current.value = "...";
inputRef.current.style...
```

And that's where we need to be careful.

A custom component should not necessarily expose its entire internal DOM implementation.

---

# 6. This is where `useImperativeHandle` comes in

Imagine:

```jsx
function MyInput({ ref }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => {
    return {
      focus() {
        inputRef.current.focus();
      },

      select() {
        inputRef.current.select();
      },
    };
  }, []);

  return <input ref={inputRef} />;
}
```

Now the parent does:

```jsx
function Form() {
  const inputRef = useRef(null);

  function handleClick() {
    inputRef.current.focus();
  }

  return (
    <>
      <MyInput ref={inputRef} />

      <button onClick={handleClick}>
        Focus
      </button>
    </>
  );
}
```

But now:

```text
inputRef.current
```

is **not the `<input>` DOM element**.

Instead it is approximately:

```js
{
  focus: function,
  select: function
}
```

That's an **imperative handle**.

Current React's `useImperativeHandle` is specifically designed for customizing what a parent receives through a ref. ([React](https://mn.react.dev/reference/react/useImperativeHandle?utm_source=chatgpt.com "useImperativeHandle – React"))

---

# 7. Why this is useful

Without `useImperativeHandle`:

```text
Parent
   ↓
entire DOM node
   ↓
knows implementation details
```

With `useImperativeHandle`:

```text
Parent
   ↓
small public interface
   ↓
focus()
select()
```

So we're essentially creating an imperative API.

Think:

```text
Component
    ↓
public API
```

much like a normal JavaScript object.

---

# 8. Think of it like a class interface

Imagine we had a class:

```js
class InputController {
  focus() {}
  select() {}
}
```

The consumer doesn't need to know how those methods internally work.

`useImperativeHandle` lets a React component expose something conceptually similar:

```text
MyInput
 ├── internal DOM
 └── public imperative API
       ├── focus()
       └── select()
```

The parent only knows:

```text
ref.current.focus()
```

It doesn't need to know:

```text
there is an HTMLInputElement inside
```

That's strong encapsulation.

---

# 9. This is very different from props

Suppose our input is:

```jsx
<MyInput
  value={name}
  onChange={setName}
/>
```

That's declarative.

You're saying:

```text
"Here is the state it should display."

"Here is what to do when it changes."
```

With:

```js
ref.current.focus()
```

you're saying:

> **"Perform this action on the component right now."**

That's imperative.

So:

```text
props
→ describe desired UI/state

ref handle
→ request an imperative action
```

This difference is extremely important.

---

# 10. Why React prefers props over imperative refs

Suppose you want a modal to be open.

You could expose:

```js
ref.current.open();
```

But it's usually better to write:

```jsx
<Modal isOpen={isOpen} />
```

Why?

Because with props:

```text
state
 ↓
render
 ↓
UI
```

everything stays in React's declarative model.

With:

```js
ref.current.open();
```

you're telling a component:

```text
"Do this action now."
```

That introduces an imperative escape hatch.

Current React's guidance explicitly recommends using props where behavior can be expressed declaratively and reserving refs for imperative behaviors that can't reasonably be expressed as props. ([React](https://mn.react.dev/reference/react/forwardRef?utm_source=chatgpt.com "forwardRef – React"))

---

# 11. A perfect example: focus

Focus is inherently imperative.

Suppose:

```text
User submits form
       ↓
validation fails
       ↓
focus email field
```

You could not naturally express:

```text
emailField should always have focus
```

as ordinary component state without interacting with the browser's focus system.

So:

```js
emailRef.current.focus();
```

is a very reasonable imperative action.

This is exactly the sort of case refs are designed for.

---

# 12. Another good use: scrolling

Imagine:

```text
Chat
────────────────
message 1
message 2
message 3
message 4
[bottom]
```

You want:

```text
Jump to latest
```

You might have:

```jsx
function Chat() {
  const bottomRef = useRef(null);

  function scrollToLatest() {
    bottomRef.current.scrollIntoView({
      behavior: "smooth",
    });
  }

  return (
    <>
      {/* messages */}

      <div ref={bottomRef} />
      
      <button onClick={scrollToLatest}>
        Latest
      </button>
    </>
  );
}
```

That is a good imperative use.

You aren't saying:

```text
scrollPosition = 1000
```

and trying to make React own every browser scroll operation.

You're saying:

> "When the user performs this action, tell the browser to scroll this element."

---

# 13. Another example: exposing a small custom API

Imagine a video player component:

```jsx
function VideoPlayer({ ref }) {
  const videoRef = useRef(null);

  useImperativeHandle(ref, () => {
    return {
      play() {
        videoRef.current.play();
      },

      pause() {
        videoRef.current.pause();
      },

      restart() {
        videoRef.current.currentTime = 0;
        videoRef.current.play();
      },
    };
  }, []);

  return (
    <video
      ref={videoRef}
      src="/movie.mp4"
    />
  );
}
```

Parent:

```jsx
function App() {
  const playerRef = useRef(null);

  return (
    <>
      <VideoPlayer ref={playerRef} />

      <button onClick={() => playerRef.current.play()}>
        Play
      </button>

      <button onClick={() => playerRef.current.pause()}>
        Pause
      </button>

      <button onClick={() => playerRef.current.restart()}>
        Restart
      </button>
    </>
  );
}
```

Now the parent knows nothing about:

```text
HTML <video>
currentTime
DOM implementation
```

It only knows:

```text
play()
pause()
restart()
```

That's a clean imperative interface.

Current React's examples of `useImperativeHandle` include exposing custom methods like these rather than exposing the entire DOM node. ([React](https://mn.react.dev/reference/react/useImperativeHandle?utm_source=chatgpt.com "useImperativeHandle – React"))

---

# 14. Notice the difference between API size and implementation size

Internally:

```text
VideoPlayer
├── videoRef
├── DOM video
├── playback state
├── effects
├── event listeners
└── other logic
```

Externally:

```text
ref.current
├── play()
├── pause()
└── restart()
```

That's excellent encapsulation.

The parent sees only what the component intentionally exposes.

---

# 15. `useImperativeHandle` doesn't create the ref

This is a common misunderstanding.

The parent creates:

```jsx
const playerRef = useRef(null);
```

The child receives that ref:

```jsx
function VideoPlayer({ ref }) {
```

Then:

```jsx
useImperativeHandle(
  ref,
  () => ({
    play() {},
    pause() {},
  }),
  []
);
```

So:

```text
Parent owns ref object
        ↓
Child controls what ref.current means
```

The child doesn't "give the parent a ref" from scratch.

It customizes what gets assigned to the parent's ref.

---

# 16. Think about the lifecycle

Initially:

```text
playerRef.current
    ↓
null
```

because the component hasn't mounted yet.

After the component is mounted:

```text
playerRef.current
    ↓
imperative handle
```

For example:

```js
{
  play,
  pause,
}
```

When the component is removed:

```text
playerRef.current
    ↓
null
```

So you should not blindly assume:

```js
playerRef.current.play();
```

is always safe.

The component must currently be mounted and the ref must be available.

---

# 17. Callback refs

There's another form of refs you should know.

Instead of:

```jsx
const inputRef = useRef(null);

<input ref={inputRef} />
```

you can give React a function:

```jsx
<input
  ref={node => {
    // node is the current DOM node
  }}
/>
```

That's a **callback ref**.

The function is called when the node becomes attached and when the ref needs to be detached.

This can be useful when you want to perform logic based on attachment itself.

For example, conceptually:

```jsx
function Example() {
  const handleRef = node => {
    if (node) {
      node.focus();
    }
  };

  return <input ref={handleRef} />;
}
```

The important difference is:

```text
object ref
→ React stores node in ref.current

callback ref
→ React calls your function with the node
```

---

# 18. Why callback refs can be surprisingly useful

Suppose you have a dynamic list and want to collect references to multiple DOM nodes.

You could maintain:

```js
const nodesRef = useRef(new Map());
```

and:

```jsx
items.map(item => (
  <div
    key={item.id}
    ref={node => {
      if (node) {
        nodesRef.current.set(item.id, node);
      } else {
        nodesRef.current.delete(item.id);
      }
    }}
  >
    {item.name}
  </div>
))
```

Now you have:

```text
id → DOM node
```

inside the ref.

This can be useful for:

```text
scrolling
measurement
focus management
virtualized lists
animations
```

Notice again:

```text
ref
→ imperative access
```

not:

```text
ref
→ UI state
```

---

# 19. Function refs and identity

Here's a subtle connection to our previous memoization lesson.

Consider:

```jsx
<div ref={node => {
  ...
}} />
```

That callback function is created during rendering.

So its identity may change between renders.

React handles ref callbacks specially, but you should be aware that changing a callback ref can result in React detaching the old ref and attaching the new one.

For many simple cases this doesn't matter.

But for advanced DOM management, callback-ref identity becomes important.

This is another place where our earlier lesson about:

```text
function identity
```

comes back.

---

# 20. `useImperativeHandle` dependencies

Look at:

```jsx
useImperativeHandle(
  ref,
  () => ({
    focus() {
      inputRef.current.focus();
    },
  }),
  []
);
```

We've seen dependency arrays before.

The handle creation function refers to:

```text
inputRef
```

which is a stable ref object.

So:

```js
[]
```

can make sense.

But suppose:

```jsx
function Editor({ ref, mode }) {
  useImperativeHandle(
    ref,
    () => ({
      submit() {
        submitEditor(mode);
      },
    }),
    [mode]
  );

  ...
}
```

Now the exposed handle depends on:

```text
mode
```

Therefore:

```text
mode changes
 ↓
new handle
 ↓
assigned to ref
```

Current React documents the dependency behavior of `useImperativeHandle` using `Object.is`, just like other dependency-based hooks. ([React](https://mn.react.dev/reference/react/useImperativeHandle?utm_source=chatgpt.com "useImperativeHandle – React"))

---

# 21. This can create stale behavior too

Suppose:

```jsx
function Editor({ ref, userId }) {
  useImperativeHandle(
    ref,
    () => ({
      save() {
        saveForUser(userId);
      },
    }),
    []
  );
}
```

Render 1:

```text
userId = 42
```

The handle's:

```js
save()
```

closes over:

```text
42
```

Later:

```text
userId = 99
```

but you've said:

```jsx
[]
```

which means:

> "This handle doesn't depend on anything that changes."

So the exposed `save()` can continue referring to the old value.

That's the same closure problem we've already studied.

Again:

```text
dependency arrays
+
closures
+
render snapshots
```

appear here too.

---

# 22. Parent and child are communicating imperatively

Let's compare normal React communication.

### Parent → child with props

```jsx
<Child
  color="red"
  disabled={true}
/>
```

Flow:

```text
parent state
 ↓
props
 ↓
child render
 ↓
UI
```

### Child exposes imperative handle

```jsx
<Child ref={childRef} />
```

Flow:

```text
parent
 ↓
ref
 ↓
child's imperative API
 ↓
method call
 ↓
imperative behavior
```

So you have two fundamentally different communication channels:

```text
declarative
props/state

imperative
refs
```

Good React code heavily favors the first and uses the second when it fits the problem.

---

# 23. A useful example: modal

Let's revisit the modal from our portal lesson.

You might be tempted to design:

```jsx
const modalRef = useRef(null);

modalRef.current.open();
modalRef.current.close();
```

You could technically expose:

```jsx
useImperativeHandle(ref, () => ({
  open() {},
  close() {},
}));
```

But a better React API is usually:

```jsx
<Modal
  isOpen={isOpen}
  onClose={setIsOpen}
/>
```

Why?

Because open/closed state is part of **UI state**.

React should own that.

The imperative API might be appropriate for:

```text
focus first input
scroll dialog content
select text
```

Those are genuinely imperative browser operations.

So:

```text
Modal open/closed
→ props/state

Focus inside modal
→ ref
```

That's a very strong design distinction.

---

# 24. Another example: rich text editor

A rich text editor is a good candidate for an imperative API because some operations are inherently commands:

```text
focus()
undo()
redo()
insertText()
selectAll()
```

You might expose:

```jsx
useImperativeHandle(ref, () => ({
  focus,
  undo,
  redo,
  selectAll,
}));
```

Then parent code can say:

```js
editorRef.current.undo();
```

without needing to understand the editor's internal DOM/state structure.

This is where imperative handles start to feel natural rather than "un-React-like."

---

# 25. Why not expose 25 methods?

You could write:

```js
{
  focus,
  blur,
  open,
  close,
  reset,
  setValue,
  clear,
  select,
  scroll,
  ...
}
```

But that may be a smell.

A component with a giant imperative API is becoming hard to reason about.

Ask:

```text
Can some of these be expressed as props?
Can some be local behavior?
Does the parent really need this method?
```

A good imperative API is usually small.

---

# 26. Refs are escape hatches

This is probably the most useful mental phrase for refs:

> **A ref is an escape hatch from React's declarative model when you need to interact directly with something imperative.**

Examples:

```text
DOM
browser focus
scroll position
media playback
animations
third-party widgets
external libraries
```

Not ideal uses:

```text
visible application state
derived values
things that could simply be props
```

Current React's own guidance emphasizes this principle. ([React](https://mn.react.dev/reference/react/forwardRef?utm_source=chatgpt.com "forwardRef – React"))

---

# 27. Refs + effects

A very common real-world combination:

```jsx
function VideoPlayer({ isPlaying }) {
  const videoRef = useRef(null);

  useEffect(() => {
    if (isPlaying) {
      videoRef.current.play();
    } else {
      videoRef.current.pause();
    }
  }, [isPlaying]);

  return <video ref={videoRef} />;
}
```

This has a beautiful architecture:

```text
React state/prop
      ↓
isPlaying
      ↓
Effect
      ↓
imperative browser API
      ↓
video.play()/pause()
```

Why use an Effect?

Because:

```text
video.play()
```

is an external side effect.

The render itself should describe the UI, while the Effect synchronizes the outside world.

This connects directly to our earlier `useEffect` lesson.

---

# 28. Refs + third-party libraries

Suppose you use a chart library:

```js
SomeChartLibrary.create(
  DOMNode,
  options
);
```

React shouldn't necessarily recreate the entire chart object during rendering.

You might do:

```jsx
function Chart({ data }) {
  const containerRef = useRef(null);
  const chartRef = useRef(null);

  useEffect(() => {
    chartRef.current =
      createChart(containerRef.current);

    return () => {
      chartRef.current.destroy();
    };
  }, []);

  useEffect(() => {
    chartRef.current.update(data);
  }, [data]);

  return <div ref={containerRef} />;
}
```

Now we have:

```text
DOM ref
+
external object ref
+
effects
```

This is one of the most realistic reasons you'll use refs in production.

---

# 29. The ref does not make the external library reactive

This is important.

Suppose:

```js
chartRef.current
```

contains a chart library instance.

Changing it:

```js
chartRef.current = newChart;
```

doesn't make React rerender.

That was already true with ordinary refs.

So:

```text
ref
→ persistent storage

not:
ref
→ reactive state
```

This distinction remains unchanged even when using `useImperativeHandle`.

---

# 30. Current React's modern ref flow

Let's build the complete modern pattern:

```jsx
function FancyInput({ ref }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current.focus();
    },

    clear() {
      inputRef.current.value = "";
    },
  }), []);

  return (
    <input ref={inputRef} />
  );
}
```

Parent:

```jsx
function Form() {
  const inputRef = useRef(null);

  return (
    <>
      <FancyInput ref={inputRef} />

      <button
        onClick={() => inputRef.current.focus()}
      >
        Focus
      </button>

      <button
        onClick={() => inputRef.current.clear()}
      >
        Clear
      </button>
    </>
  );
}
```

React tree:

```text
Form
└── FancyInput
    └── input
```

Ref relationship:

```text
Form
 ↓
inputRef
 ↓
FancyInput's exposed handle
 ↓
focus()
clear()
 ↓
internal inputRef
 ↓
<input>
```

The parent never touches the internal DOM node directly.

That's excellent encapsulation.

---

# 31. Compare direct ref vs custom handle

### Direct ref

```jsx
function Input({ ref }) {
  return <input ref={ref} />;
}
```

Parent gets:

```text
HTMLInputElement
```

### Imperative handle

```jsx
function Input({ ref }) {
  const inputRef = useRef(null);

  useImperativeHandle(ref, () => ({
    focus() {
      inputRef.current.focus();
    },
  }), []);

  return <input ref={inputRef} />;
}
```

Parent gets:

```text
{
  focus()
}
```

So:

```text
direct ref
→ expose implementation

custom handle
→ expose behavior
```

That distinction is very useful when designing reusable components.

---

# 32. This connects directly to component API design

Remember our composition/API-design lesson?

We said:

> A good component should expose **intent**, not implementation details.

That applies here too.

Bad:

```text
ref.current.inputElement
```

which exposes:

```text
"I happen to internally use an <input>."
```

Better:

```text
ref.current.focus()
```

which says:

```text
"You can ask this component to focus."
```

This is a much stronger API.

And now you can see the connection:

```text
prop API
+
imperative API
=
component interface
```

---

# 33. When `forwardRef` still appears in code

Suppose you open a codebase and see:

```jsx
const MyInput = forwardRef(
  function MyInput(props, ref) {
    return (
      <input
        {...props}
        ref={ref}
      />
    );
  }
);
```

Don't get confused.

The concept is simply:

```text
parent gives ref
 ↓
component receives ref
 ↓
component passes ref onward
```

In React 19+, new function components can instead do:

```jsx
function MyInput({ ref, ...props }) {
  return (
    <input
      {...props}
      ref={ref}
    />
  );
}
```

The underlying idea has not changed; the API has become simpler. ([React](https://mn.react.dev/reference/react/forwardRef?utm_source=chatgpt.com "forwardRef – React"))

---

# 34. One more subtle point: refs don't participate in rendering the same way props do

Suppose:

```jsx
const ref = useRef(null);
```

and later:

```js
ref.current = someValue;
```

React does not say:

```text
"Ah, ref changed. Rerender component."
```

That is deliberate.

This makes refs useful for mutable information that should survive renders but shouldn't itself drive visual output.

So think:

```text
state
→ reactive memory

ref
→ non-reactive memory
```

And with imperative handles:

```text
ref
→ externally accessible imperative interface
```

---

# 35. A real decision tree

When you're about to use a ref, ask:

### "Is this something the UI should visibly respond to?"

If yes:

```text
useState / props / derived data
```

### "Do I need to interact with an external imperative thing?"

If yes:

```text
useRef
```

### "Does a parent component need imperative access to my component?"

If yes:

```text
ref prop
+
possibly useImperativeHandle
```

### "Could this behavior simply be expressed declaratively?"

If yes:

```text
prefer props/state
```

This decision tree will prevent a lot of unnecessary imperative code.

---

# 36. The full mental model

You can now think of refs at three levels.

### Level 1 — store something

```jsx
const valueRef = useRef(null);
```

Use for:

```text
timer
DOM node
previous value
external instance
```

### Level 2 — pass a ref through a component

```jsx
function Input({ ref }) {
  return <input ref={ref} />;
}
```

Use when:

```text
parent needs access to child's underlying thing
```

### Level 3 — expose a custom imperative interface

```jsx
useImperativeHandle(ref, () => ({
  focus() {},
  reset() {},
}));
```

Use when:

```text
parent needs behavior,
but shouldn't know implementation details
```

That's the progression.

---

# 37. The broader React architecture

We've now accumulated a pretty complete set of communication mechanisms:

```text
                    COMPONENT
                       │
       ┌───────────────┼────────────────┐
       ↓               ↓                ↓
      props           context          refs
       ↓               ↓                ↓
 declarative       shared data      imperative access
```

And:

```text
state
 ↓
drives rendering

effects
 ↓
synchronize external systems

refs
 ↓
remember/interact with external or imperative things

imperative handle
 ↓
expose limited imperative behavior to parent
```

This is a much more complete picture of how a React component can interact with the world.

---

# 38. The key rules from this lesson

Keep these:

```text
1. React 19 lets function components receive ref as a normal prop.

2. forwardRef is no longer necessary for new React 19 code,
   although you'll still see it in existing code.

3. A ref can point directly to a DOM node.

4. A component can instead expose a custom imperative handle.

5. useImperativeHandle controls what ref.current exposes.

6. Prefer props/state when behavior can be expressed declaratively.

7. Use refs for genuinely imperative interactions.

8. A ref does not trigger a rerender when current changes.

9. Imperative handles should usually expose a small API.

10. Ref handles have the same closure/dependency concerns
    we've already seen with other hooks.
```

The most important distinction is:

```text
props:
"Here is what I want the component to be."

ref:
"Here is an imperative capability I need from the component."
```

And:

```text
useImperativeHandle
=
"Here are the specific capabilities I choose to expose."
```

That makes refs much more than a way to call `.focus()`.

They are a controlled bridge between React's declarative world and imperative systems such as the DOM, media APIs, animations, and third-party libraries. ([React](https://mn.react.dev/reference/react/useImperativeHandle?utm_source=chatgpt.com "useImperativeHandle – React"))