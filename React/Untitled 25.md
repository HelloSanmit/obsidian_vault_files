
# Lesson 23 — Referential Equality, `React.memo`, `useMemo`, and `useCallback`

This is one of the topics that causes a huge amount of confusion in React.

You will often hear things like:

> “Objects are recreated every render.”

> “Use `useCallback` so your child doesn't rerender.”

> “Wrap it in `memo`.”

> “Use `useMemo` for performance.”

Those statements are not completely wrong, but without understanding **identity**, they become cargo-cult React.

So let's build this from JavaScript first.

---

## 1. The foundation is JavaScript identity

Consider:

```js
const a = { name: "Alice" };
const b = { name: "Alice" };

console.log(a === b);
```

What do you get?

```text
false
```

Even though they contain identical data.

Why?

Because these are two different object instances.

Think:

```text
a
 ↓
┌──────────────┐
│ name: Alice  │
└──────────────┘

b
 ↓
┌──────────────┐
│ name: Alice  │
└──────────────┘
```

Same contents.

Different identities.

So:

```js
a === b
```

asks essentially:

> “Are these the same object?”

Not:

> “Do these objects contain the same properties?”

---

# 2. Primitives behave differently

Consider:

```js
const a = 10;
const b = 10;

console.log(a === b);
```

That's:

```text
true
```

Strings:

```js
const a = "hello";
const b = "hello";

a === b
```

also give:

```text
true
```

So broadly:

```text
primitives
→ compared by value

objects/functions/arrays
→ compared by identity
```

This distinction is absolutely central to understanding React optimization.

---

# 3. Functions have identity too

Consider:

```js
const a = () => {
  console.log("hello");
};

const b = () => {
  console.log("hello");
};

console.log(a === b);
```

Again:

```text
false
```

They do the same thing.

They even have the same source code.

But they're two separate function objects.

That's important because a function can be treated as a prop:

```jsx
<Child onClick={a} />
```

and React can compare that prop with a later render.

---

# 4. Now bring React into the picture

Consider:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  const options = {
    color: "blue",
  };

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child options={options} />
    </>
  );
}
```

Every time `Parent` renders, the component function executes again.

So this line:

```js
const options = {
  color: "blue",
};
```

runs again.

That means a new object is created.

Render 1:

```text
options
   ↓
Object A
```

Render 2:

```text
options
   ↓
Object B
```

Even though both look like:

```js
{ color: "blue" }
```

we have:

```js
ObjectA === ObjectB
```

which is:

```text
false
```

This is what people mean when they say:

> “The object is recreated on every render.”

---

# 5. The same thing happens with functions

Suppose:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  function handleClick() {
    console.log("clicked");
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child onClick={handleClick} />
    </>
  );
}
```

Every render executes:

```js
function handleClick() {
  console.log("clicked");
}
```

So there is a new function object each render.

Conceptually:

```text
render 1
  ↓
handleClick → Function A

render 2
  ↓
handleClick → Function B
```

and:

```js
FunctionA === FunctionB
```

is false.

---

# 6. But is this a problem?

Usually:

**No.**

This is one of the most important things to understand.

Creating ordinary JavaScript objects and functions is normal.

You should not look at:

```js
const options = { color: "blue" };
```

and immediately think:

> “Performance disaster!”

You shouldn't.

The fact that something gets recreated does **not** automatically mean it is expensive or harmful.

The problem appears when **identity matters to something you're trying to optimize or synchronize**.

That's where memoization enters.

---

# 7. First meet `React.memo`

Suppose:

```jsx
function Child({ name }) {
  console.log("Child rendered");

  return <p>{name}</p>;
}
```

And:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child name="Alice" />
    </>
  );
}
```

When `count` changes, `Parent` renders again.

Normally, `Child` may render as part of React processing the updated tree.

But we can write:

```jsx
const Child = memo(function Child({ name }) {
  console.log("Child rendered");

  return <p>{name}</p>;
});
```

Now `Child` is memoized.

Its props haven't changed:

```text
"Alice"
   ↓
"Alice"
```

so React can skip rendering that child when its parent rerenders. `memo` is a performance optimization based on unchanged props, not a guarantee that the component will never render. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 8. What does `memo` actually compare?

By default, React compares the component's props.

The important part is:

> **Object/function props are compared by identity, not by deep contents.**

Suppose:

```jsx
const Child = memo(function Child({ options }) {
  return <div />;
});
```

Parent:

```jsx
function Parent() {
  const options = {
    color: "blue",
  };

  return <Child options={options} />;
}
```

On the next Parent render:

```text
old options → Object A
new options → Object B
```

So React sees:

```text
old prop === new prop
        ↓
false
```

Therefore the memoized child can't treat the prop as unchanged.

Even though:

```js
{
  color: "blue"
}
```

looks the same.

This is one of the most important practical consequences of referential equality. React's `memo` documentation specifically notes that passing objects or functions created during rendering can make memoization ineffective. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 9. Primitive props are easier

Consider:

```jsx
<Child
  name="Alice"
  age={25}
/>
```

These are primitives.

If the parent rerenders and still passes:

```text
name = "Alice"
age = 25
```

the values compare equal.

So:

```text
memo
+
primitive props
```

often works very naturally.

This is one reason a component API such as:

```jsx
<Profile
  name={user.name}
  age={user.age}
/>
```

can sometimes be easier to memoize than:

```jsx
<Profile
  user={user}
/>
```

Although whether that's a better API is a separate architectural question.

---

# 10. `memo` doesn't stop a component from rendering because its own state changes

This is another common misunderstanding.

Suppose:

```jsx
const Child = memo(function Child() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(count + 1)}>
      {count}
    </button>
  );
});
```

`memo` doesn't freeze it.

Its own state changes:

```text
state changes
   ↓
Child rerenders
```

`memo` is about whether **parent-driven rendering can be skipped based on props**.

It doesn't mean:

> "Never execute this component."

---

# 11. Context can also cause a memoized component to update

Suppose:

```jsx
const Child = memo(function Child({ name }) {
  const theme = useContext(ThemeContext);

  return <div>{name}</div>;
});
```

Even if:

```text
name unchanged
```

the context value can change.

Then the component may need to update.

Again:

```text
memo ≠ force this component never to render
```

It's a performance optimization inside a larger reactive system.

---

# 12. Now the real problem: function props

Consider:

```jsx
const Child = memo(function Child({ onSave }) {
  console.log("Child rendered");

  return <button onClick={onSave}>Save</button>;
});
```

Parent:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  function handleSave() {
    console.log("saved");
  }

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <Child onSave={handleSave} />
    </>
  );
}
```

Parent rerenders.

New render creates:

```text
old handleSave → Function A
new handleSave → Function B
```

Therefore:

```text
A === B
↓
false
```

So even though `Child` is memoized:

```text
prop changed by identity
       ↓
Child rerenders
```

This is where `useCallback` enters.

---

# 13. `useCallback` does one specific thing

Consider:

```jsx
const handleSave = useCallback(() => {
  console.log("saved");
}, []);
```

Now React can give you the same cached function between renders as long as the dependencies remain unchanged. ([React](https://react.dev/reference/react/useCallback?utm_source=chatgpt.com "useCallback – React"))

Conceptually:

```text
render 1
   ↓
Function A
   ↓
React stores it

render 2
   ↓
dependencies unchanged
   ↓
return Function A again

render 3
   ↓
dependencies unchanged
   ↓
return Function A again
```

So:

```text
old handleSave === new handleSave
```

can now be true.

---

# 14. `useCallback` doesn't prevent creating the function

This is subtle.

You write:

```jsx
useCallback(() => {
  console.log("saved");
}, []);
```

It might sound like React somehow prevents the function from being created.

Not exactly.

The function expression is still produced during the render.

Conceptually:

```text
render
 ↓
new function expression exists
 ↓
useCallback checks dependencies
 ↓
if unchanged:
    return previously stored function
```

So the main thing you're caching is the **function reference you receive**, not eliminating JavaScript function creation itself. React explicitly describes this behavior in its current `useCallback` reference. ([React](https://react.dev/reference/react/useCallback?utm_source=chatgpt.com "useCallback – React"))

---

# 15. Why `useCallback` matters mostly with something that cares about identity

Suppose:

```jsx
function Parent() {
  const handleSave = useCallback(() => {
    console.log("saved");
  }, []);

  return (
    <Child onSave={handleSave} />
  );
}
```

If `Child` is not memoized:

```jsx
function Child({ onSave }) {
  ...
}
```

the stable function identity may provide little or no rendering benefit.

The interesting combination is:

```text
useCallback
+
memoized child
```

because now:

```text
stable function identity
        ↓
prop remains equal
        ↓
memo can potentially skip child render
```

That's the actual chain.

---

# 16. `useMemo` is the same basic idea, but for values

Suppose:

```jsx
const visibleTodos = todos.filter(
  todo => todo.completed === false
);
```

Every render runs the filter.

Usually:

**that's completely fine.**

But suppose the calculation is expensive.

You could write:

```jsx
const visibleTodos = useMemo(() => {
  return todos.filter(
    todo => todo.completed === false
  );
}, [todos]);
```

Now React can reuse the previous result while `todos` remains referentially unchanged. `useMemo` caches the result of a calculation between renders based on its dependencies. ([React](https://react.dev/reference/react/useMemo?utm_source=chatgpt.com "useMemo – React"))

---

# 17. `useMemo` caches a value, `useCallback` caches a function

This is the simplest useful comparison:

```text
useMemo
   ↓
cache the RESULT of a calculation

useCallback
   ↓
cache the FUNCTION itself
```

For example:

```jsx
const value = useMemo(
  () => expensiveCalculation(data),
  [data]
);
```

versus:

```jsx
const fn = useCallback(
  () => doSomething(data),
  [data]
);
```

There's a direct relationship between the two.

In fact, conceptually:

```jsx
useCallback(fn, deps)
```

is closely related to:

```jsx
useMemo(() => fn, deps)
```

The dedicated hook just makes the intention clearer. React's current reference describes them this way as essentially equivalent in purpose for memoizing a function. ([React](https://react.dev/reference/react/useCallback?utm_source=chatgpt.com "useCallback – React"))

---

# 18. But this does NOT mean `useMemo` should be everywhere

Suppose:

```jsx
const fullName = useMemo(() => {
  return firstName + " " + lastName;
}, [firstName, lastName]);
```

That's probably absurd.

The calculation:

```js
firstName + " " + lastName
```

is trivial.

You've introduced:

```text
more code
more dependencies
more cognitive load
```

to avoid a tiny calculation.

Memoization has a cost too.

So don't think:

> "Caching is always faster."

Instead:

> **Caching is useful when the cost of recalculation or downstream identity changes is worth avoiding.**

React's current guidance is to treat `useMemo` and `useCallback` primarily as performance optimizations, not correctness mechanisms. ([React](https://react.dev/reference/react/useCallback?utm_source=chatgpt.com "useCallback – React"))

---

# 19. There are actually two different jobs for `useMemo`

This is important.

### Job A: expensive calculation

```jsx
const result = useMemo(() => {
  return expensiveCalculation(data);
}, [data]);
```

Here the goal is:

```text
avoid recalculating
```

### Job B: stable object identity

Suppose:

```jsx
const options = {
  color,
  size,
};
```

and a memoized child receives it:

```jsx
<Chart options={options} />
```

You could do:

```jsx
const options = useMemo(() => {
  return {
    color,
    size,
  };
}, [color, size]);
```

Now when `color` and `size` haven't changed:

```text
old options === new options
```

can remain true.

So the reason for `useMemo` isn't always:

> "This calculation is computationally expensive."

Sometimes it's:

> **"I need this value to maintain stable identity."**

---

# 20. Stable identity can also matter to Effects

Suppose:

```jsx
function Component({ userId }) {
  const options = {
    userId,
  };

  useEffect(() => {
    connect(options);
  }, [options]);

  ...
}
```

What's wrong?

Every render creates a new object:

```text
options A
options B
options C
```

Even if:

```text
userId = 42
```

hasn't changed.

So the effect sees:

```text
options changed
```

on every render.

That can cause repeated synchronization work.

You _could_ stabilize the object:

```jsx
const options = useMemo(() => {
  return { userId };
}, [userId]);
```

Then the object identity only changes when `userId` changes.

Current React guidance explicitly identifies memoization of a value as one way to prevent an effect from firing too often when the value's identity changes unnecessarily. ([React](https://react.dev/learn/react-compiler/introduction?utm_source=chatgpt.com "Introduction – React"))

But there is often an even simpler solution:

```jsx
useEffect(() => {
  connect({ userId });
}, [userId]);
```

Now the object exists inside the effect and doesn't need to be part of the dependency list.

This is why:

> **Don't automatically reach for `useMemo`. First see whether the code structure itself can remove the unstable dependency.**

---

# 21. `useCallback` has the same kind of issue with Effects

Suppose:

```jsx
function Component({ roomId }) {
  function createOptions() {
    return {
      roomId,
    };
  }

  useEffect(() => {
    const options = createOptions();
    connect(options);
  }, [createOptions]);
}
```

`createOptions` gets recreated every render.

Therefore:

```text
createOptions identity changed
       ↓
effect dependency changed
       ↓
effect reruns
```

You could write:

```jsx
const createOptions = useCallback(() => {
  return { roomId };
}, [roomId]);
```

But again, often you can avoid the function dependency altogether:

```jsx
useEffect(() => {
  const options = {
    roomId,
  };

  connect(options);
}, [roomId]);
```

That's simpler.

The broader rule is:

> **Stabilize identity when it solves a real problem. Don't stabilize everything by reflex.**

---

# 22. Let's see the classic `memo + useCallback` pattern

```jsx
const ShippingForm = memo(function ShippingForm({
  onSubmit,
}) {
  console.log("ShippingForm rendered");

  return (
    <form onSubmit={onSubmit}>
      ...
    </form>
  );
});
```

Parent:

```jsx
function ProductPage({ productId, referrer }) {
  const [theme, setTheme] = useState("light");

  const handleSubmit = useCallback(
    orderDetails => {
      postOrder(productId, referrer, orderDetails);
    },
    [productId, referrer]
  );

  return (
    <>
      <button
        onClick={() =>
          setTheme(theme === "light" ? "dark" : "light")
        }
      >
        Change theme
      </button>

      <ShippingForm onSubmit={handleSubmit} />
    </>
  );
}
```

Now:

```text
theme changes
 ↓
ProductPage renders
 ↓
handleSubmit dependencies unchanged
 ↓
same function reference
 ↓
ShippingForm props unchanged
 ↓
memo can skip ShippingForm
```

That's a meaningful optimization.

But notice the dependency array:

```js
[productId, referrer]
```

Why not:

```js
[]
```

Because `handleSubmit` reads:

```text
productId
referrer
```

If either changes, the callback needs to represent those new values.

Otherwise you create a stale closure.

---

# 23. This brings us back to closures

Remember our earlier discussion?

Each render has its own snapshot.

Suppose:

```jsx
function ProductPage({ productId }) {
  const handleSubmit = useCallback(() => {
    console.log(productId);
  }, []);
}
```

Imagine:

```text
render 1
productId = 42
```

The callback closes over:

```text
42
```

Then later:

```text
render 2
productId = 99
```

But your dependency list is:

```js
[]
```

so React is told:

> "Reuse the old function."

That function still closes over the old render's:

```text
productId = 42
```

Now you have:

```text
UI says 99
callback thinks 42
```

That's a stale closure.

So dependency arrays are not arbitrary performance settings.

They describe **which reactive values the cached function/value depends on**. React compares those dependencies with `Object.is`. ([React](https://react.dev/reference/react/useCallback?utm_source=chatgpt.com "useCallback – React"))

---

# 24. This is why lying about dependencies is dangerous

Don't do:

```jsx
const handleSubmit = useCallback(() => {
  save(userId);
}, []);
```

just because:

```text
"it stops rerendering"
```

You've potentially broken correctness.

The callback depends on:

```text
userId
```

so if the callback needs to reflect a new `userId`, that dependency must be represented.

Performance optimization should never be:

```text
make dependencies smaller
```

just for the sake of fewer updates.

---

# 25. Referential equality and `useEffect`

This deserves a compact example.

```jsx
function Search({ query }) {
  const options = {
    query,
    limit: 20,
  };

  useEffect(() => {
    search(options);
  }, [options]);

  return ...
}
```

Suppose:

```text
query = "react"
```

and nothing changes visually.

Parent causes a rerender.

New render:

```text
options A
```

becomes:

```text
options B
```

Even though:

```js
A.query === B.query
A.limit === B.limit
```

the objects themselves are different:

```js
A !== B
```

Therefore:

```text
effect sees changed dependency
 ↓
cleanup
 ↓
setup again
```

That's not React being irrational.

You told React:

> "The effect depends on this object."

and the object changed identity.

React is doing exactly what the dependency model describes.

---

# 26. This is also why inline arrays can matter

Suppose:

```jsx
const tabs = ["Home", "Products", "Settings"];
```

Every render creates a new array.

Then:

```jsx
<Child tabs={tabs} />
```

If Child is memoized:

```text
old tabs !== new tabs
```

so the prop changed by identity.

Again, this isn't inherently wrong.

Sometimes you should just accept it.

Sometimes you should restructure the props.

Sometimes you might use `useMemo`.

The correct answer depends on whether the identity difference actually matters.

---

# 27. Often the better fix is changing the prop

Instead of:

```jsx
const Child = memo(function Child({ user }) {
  ...
});
```

you might pass:

```jsx
<Child
  name={user.name}
  hasAvatar={user.avatar !== null}
/>
```

Now you're passing smaller primitive values.

This can sometimes make memoization more effective because the child's props change only when the things it actually needs change.

Current React guidance explicitly points out that reducing props to values that change less frequently can make memoization more effective. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

This is not just a performance trick.

It can improve component API design too.

---

# 28. A very common mistake: `useMemo` around everything

You might see code like:

```jsx
const data = useMemo(() => dataFromProps, [dataFromProps]);

const options = useMemo(() => ({
  enabled: true,
}), []);

const handleClick = useCallback(() => {
  ...
}, []);

const label = useMemo(() => {
  return firstName + " " + lastName;
}, [firstName, lastName]);
```

And suddenly a tiny component has seven memoization hooks.

That can be worse for humans reading the code.

Remember:

```text
memoization has a mental cost
```

You now have to understand:

```text
dependencies
identity
cache lifetime
why it exists
whether it is still necessary
```

The code became more complicated.

So don't start with:

> "What should I memoize?"

Start with:

> **"What is actually slow or causing an unnecessary identity-sensitive update?"**

---

# 29. React Compiler changes the practical landscape

There is a modern React development detail worth knowing.

Current React includes **React Compiler**, which can automatically memoize components, values, and functions in supported setups. The current React guidance recommends relying on the compiler for automatic memoization in new code, while retaining `useMemo`, `useCallback`, and `memo` when you need precise manual control or are working in codebases that use them already. ([React](https://react.dev/learn/react-compiler/introduction?utm_source=chatgpt.com "Introduction – React"))

This does **not** make today's lesson obsolete.

Quite the opposite.

You still need to understand:

```text
identity
closures
dependencies
rendering
memoization
```

because otherwise you won't understand what the compiler is optimizing or why an optimization matters.

Also, existing applications and libraries won't all have the same compiler configuration.

So manual memoization remains important knowledge.

---

# 30. A better mental model for `memo`

Don't think:

```text
memo
=
"don't render this component"
```

Think:

```text
memo
=
"before rerendering this child because its parent rendered,
check whether its props are effectively unchanged"
```

Conceptually:

```text
Parent rerenders
      ↓
Should Child rerender?
      ↓
Are props unchanged?
   ↙          ↘
 yes          no
 ↓             ↓
skip         render
```

That's a much better understanding.

---

# 31. A better mental model for `useMemo`

Don't think:

```text
useMemo
=
"make code faster"
```

Think:

```text
useMemo
=
"reuse the previous calculated value
when its dependencies haven't changed"
```

Then ask:

```text
Why do I care about reusing this value?
```

Possible answers:

```text
expensive calculation
stable object identity
stable dependency for another hook
stable prop for memoized child
```

---

# 32. A better mental model for `useCallback`

Don't think:

```text
useCallback
=
"prevent function recreation"
```

Think:

```text
useCallback
=
"keep the same function reference
between renders while dependencies stay the same"
```

Then ask:

```text
Who cares about that function's identity?
```

Usually:

```text
memoized child
effect dependency
custom hook dependency
some identity-sensitive consumer
```

If nobody cares:

```text
you may not need useCallback
```

---

# 33. Here's the entire chain in one example

```jsx
function Parent({ items }) {
  const [count, setCount] = useState(0);

  const visibleItems = useMemo(() => {
    return items.filter(item => item.visible);
  }, [items]);

  const handleSelect = useCallback(
    id => {
      console.log(id);
    },
    []
  );

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      <List
        items={visibleItems}
        onSelect={handleSelect}
      />
    </>
  );
}
```

Suppose:

```jsx
const List = memo(function List({
  items,
  onSelect,
}) {
  ...
});
```

Now:

```text
count changes
 ↓
Parent rerenders
```

`items` prop has not changed.

Therefore:

```text
useMemo
 ↓
same visibleItems reference
```

`handleSelect` dependencies haven't changed.

Therefore:

```text
useCallback
 ↓
same function reference
```

So `List` receives:

```text
same items reference
same onSelect reference
```

Therefore:

```text
memo
 ↓
List can skip rerendering
```

That is the whole optimization pipeline.

---

# 34. What happens if you remove `useMemo`?

Then:

```jsx
const visibleItems = items.filter(
  item => item.visible
);
```

Every Parent render creates a new array.

So:

```text
old visibleItems → Array A
new visibleItems → Array B
```

Even if the actual contents haven't changed:

```text
A !== B
```

Then `List` sees a changed `items` prop.

So:

```text
Parent rerender
 ↓
new array
 ↓
List prop changed
 ↓
memo cannot skip
```

Notice that `useMemo` isn't making the `filter` magically faster.

It is preserving **the same resulting array identity** when its dependencies haven't changed.

---

# 35. What happens if you remove `useCallback`?

Suppose:

```jsx
function Parent() {
  function handleSelect(id) {
    console.log(id);
  }

  return <List onSelect={handleSelect} />;
}
```

Every Parent render:

```text
new function
```

So even with:

```jsx
memo(List)
```

the child sees:

```text
old function !== new function
```

and renders.

Again:

```text
useCallback
```

isn't primarily about the speed of creating a function.

It is about preserving the function's identity so something identity-sensitive can reuse it.

---

# 36. The optimization equation

You can think of a common pattern as:

```text
useMemo
   ↓
stable value identity
   ↓
memo
   ↓
child can skip render
```

and:

```text
useCallback
   ↓
stable function identity
   ↓
memo
   ↓
child can skip render
```

That's why you'll often see:

```text
memo + useMemo
```

and:

```text
memo + useCallback
```

together.

One creates the stable prop.

The other consumes that stability.

---

# 37. When memoization does absolutely nothing

Consider:

```jsx
const Child = memo(function Child({ config }) {
  return ...
});

function Parent() {
  const config = {
    dark: true,
  };

  return <Child config={config} />;
}
```

You have:

```text
memo ✅
```

but:

```text
config identity changes every render ❌
```

So the optimization doesn't help.

Likewise:

```jsx
const Child = memo(...);

function Parent() {
  const items = useMemo(...);
  const callback = () => {};

  return (
    <Child
      items={items}
      onClick={callback}
    />
  );
}
```

The stable `items` may help, but the unstable `callback` can still invalidate the memoization.

A single always-changing prop can be enough to make the child rerender. React's current documentation calls out this exact problem. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 38. But don't interpret that as "stabilize every prop"

This is the trap.

Suppose a component takes:

```jsx
<Child
  title="Hello"
  count={5}
  user={user}
  onClick={handleClick}
/>
```

Maybe `user` legitimately changes often.

Maybe `Child` is extremely cheap to render.

Maybe the application isn't experiencing any performance problem.

Then:

```text
memo + useMemo + useCallback + custom comparison
```

could simply make the code harder to understand.

Remember:

> **Optimization should follow a measured or well-understood cost.**

Current React guidance likewise recommends using memoization when it addresses an actual performance need rather than treating it as mandatory boilerplate. ([React](https://react.dev/reference/react/memo?utm_source=chatgpt.com "memo – React"))

---

# 39. Another subtle point: memoization doesn't make data immutable

Suppose:

```js
const user = {
  name: "Alice",
};
```

and you mutate it:

```js
user.name = "Bob";
```

The identity remains:

```text
same object
```

So a memoized child receiving that object may see the same prop identity even though its contents changed.

That is one more reason React's immutable-data patterns matter.

We previously learned:

```text
old object
   ↓
new object
```

rather than:

```text
mutate old object
```

Because identity is part of how React can efficiently reason about changes.

This is a deep connection between:

```text
immutability
+
referential equality
+
memoization
```

---

# 40. The big picture

You now have another important part of the React rendering model:

```text
                         RENDER
                           │
                           ↓
                 component executes
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
          objects       arrays        functions
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                     new identities
                           │
               ┌───────────┴───────────┐
               ↓                       ↓
            ordinary                identity-sensitive
             props                    consumers
                                       │
                    ┌──────────────────┼────────────────┐
                    ↓                  ↓                ↓
                  memo              Effect          other hooks
                    │
             compares props
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
      unchanged             changed
          ↓                   ↓
        skip                 render
```

And then:

```text
useMemo
 ↓
stable value

useCallback
 ↓
stable function
```

These tools exist to influence **identity**, not to magically make React faster in every situation.

---

# 41. The most important rules from this lesson

Keep these mental rules:

```text
1. Two objects with identical contents can still be different.

2. Two separately created functions can still be different.

3. React rerenders components by executing them again.

4. Therefore object/array/function literals created inside rendering
   usually get new identities.

5. memo can skip a child when its props remain unchanged.

6. Object/function props can defeat memo because identity changes.

7. useMemo caches a calculated value.

8. useCallback caches a function reference.

9. Dependency arrays determine when those cached identities/results
   need to be replaced.

10. Never sacrifice correctness just to make a dependency array smaller.

11. Memoization is an optimization, not a requirement.

12. Stable identity matters only when something actually cares about identity.
```

And one especially important rule:

> **Do not reach for `useMemo` or `useCallback just because you saw a new object/function being created. First determine whether that new identity is causing a real problem.**

Modern React's compiler can automate much of this optimization in supported setups, but understanding the underlying mechanics remains essential for reasoning about existing code, dependencies, and cases where manual control is still useful. ([React](https://react.dev/learn/react-compiler/introduction?utm_source=chatgpt.com "Introduction – React"))

The next layer after this is particularly interesting: **why components sometimes rerender even when props appear unchanged, how state/Context/Effects interact with memoization, and how to actually diagnose unnecessary renders instead of guessing**.