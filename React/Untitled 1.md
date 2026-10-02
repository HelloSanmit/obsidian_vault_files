
# Lesson 1 — Components + JSX

We’ll start with the two things that make React look unusual when coming from normal JavaScript:

```jsx
function App() {
  return <h1>Hello React</h1>;
}
```

At first glance, that looks like a JavaScript function containing HTML.

It isn't quite that.

We'll unpack it from the ground up.

---

## 1. A React component is a function

Start with completely ordinary JavaScript:

```js
function greet() {
  return "Hello";
}
```

Calling it:

```js
greet();
```

gives:

```text
"Hello"
```

Now imagine the returned value describes a piece of UI:

```jsx
function Greeting() {
  return <h1>Hello</h1>;
}
```

The important idea is:

```text
Greeting()
     ↓
describes some UI
```

So yes, your earlier analogy is useful:

```text
React component ≈ JavaScript function
```

But there is an important difference:

A normal function can return anything:

```js
function add() {
  return 42;
}
```

A component returns something React can use to construct/render UI.

---

# 2. Component names start with uppercase

This:

```jsx
function UserCard() {
  return <div>User</div>;
}
```

is treated as a component.

Then:

```jsx
<UserCard />
```

means "use this component."

But:

```jsx
function userCard() {
  return <div>User</div>;
}
```

and:

```jsx
<userCard />
```

are not treated the same way.

React uses capitalization to distinguish your components from native DOM elements:

```jsx
<div />
<button />
<input />
```

versus:

```jsx
<UserCard />
<Navbar />
<ProductList />
```

A useful rule:

> **Custom React components start with an uppercase letter.**

---

# 3. What is JSX?

Consider:

```jsx
const element = <h1>Hello</h1>;
```

That isn't valid plain JavaScript syntax by itself.

JSX is a syntax extension that allows us to describe UI in a way that resembles HTML.

Conceptually, this:

```jsx
<h1>Hello</h1>
```

gets transformed by the React toolchain into JavaScript that creates a React element description.

You don't normally write that transformed code yourself.

So think:

```text
JSX
  ↓
JavaScript representation
  ↓
React
  ↓
UI
```

The important thing is not memorizing the transformation.

It's understanding that:

> JSX is syntax for describing what you want the UI tree to look like.

---

# 4. JSX is JavaScript territory

This is where JSX becomes extremely useful.

```jsx
function Greeting() {
  const name = "Alex";

  return <h1>Hello {name}</h1>;
}
```

The braces:

```jsx
{name}
```

mean:

> Evaluate this JavaScript expression here.

For example:

```jsx
<h1>{2 + 3}</h1>
```

renders:

```text
5
```

And:

```jsx
const firstName = "Alex";
const lastName = "Morgan";

return <h1>{firstName + " " + lastName}</h1>;
```

renders:

```text
Alex Morgan
```

---

# 5. What can go inside `{}`?

JavaScript expressions.

For example:

```jsx
<h1>{name}</h1>
```

```jsx
<p>{user.age}</p>
```

```jsx
<p>{price * quantity}</p>
```

```jsx
<p>{isLoggedIn ? "Logout" : "Login"}</p>
```

```jsx
<p>{items.length}</p>
```

But this is not valid:

```jsx
{
  if (isLoggedIn) {
    ...
  }
}
```

because `if` is a JavaScript statement, not an expression.

You can instead do:

```jsx
{isLoggedIn ? "Welcome" : "Please log in"}
```

or calculate something before the `return`:

```jsx
function App() {
  let content;

  if (isLoggedIn) {
    content = "Welcome";
  } else {
    content = "Please log in";
  }

  return <h1>{content}</h1>;
}
```

---

# 6. JSX can contain JavaScript, but JSX itself isn't HTML

This distinction matters.

HTML:

```html
<div class="card">
```

JSX:

```jsx
<div className="card">
```

HTML:

```html
<label for="email">
```

JSX:

```jsx
<label htmlFor="email">
```

Why?

Because JSX attributes become JavaScript-facing properties.

You'll encounter several such differences.

Another one:

```jsx
<button disabled={isLoading}>
```

Notice that the value is JavaScript:

```jsx
{isLoading}
```

So:

```jsx
disabled="true"
```

and:

```jsx
disabled={true}
```

are conceptually different.

---

# 7. JSX expressions don't have to be simple

Consider:

```jsx
function Product({ name, price, inStock }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>₹{price}</p>
      <p>
        {inStock ? "Available" : "Out of stock"}
      </p>
    </div>
  );
}
```

You've now got several pieces working together:

```text
props
  ↓
JavaScript
  ↓
JSX
  ↓
UI
```

That combination is central to React.

---

# 8. Components can use other components

This is where React starts becoming interesting.

```jsx
function Header() {
  return <header>My App</header>;
}

function Content() {
  return <main>Hello</main>;
}

function App() {
  return (
    <>
      <Header />
      <Content />
    </>
  );
}
```

Visually:

```text
App
├── Header
└── Content
```

This is called **component composition**.

Instead of building everything inside one enormous component, you compose a UI from smaller components.

---

# 9. Components are not HTML templates

This distinction is subtle but important.

You might initially think:

```jsx
function Header() {
  return <header>...</header>;
}
```

means:

> "Header is an HTML template."

A better model is:

> "Header is a function that describes a piece of the UI."

That means the component can contain logic too:

```jsx
function Header() {
  const isLoggedIn = true;

  return (
    <header>
      <h1>My App</h1>

      {isLoggedIn && <button>Logout</button>}
    </header>
  );
}
```

The component combines:

```text
data
logic
UI description
```

---

# 10. Props are inputs

Now let's revisit your original question.

Suppose:

```jsx
function UserCard({ name, age }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{age}</p>
    </div>
  );
}
```

And:

```jsx
<UserCard name="Alex" age={25} />
```

You can think of this approximately as:

```js
UserCard({
  name: "Alex",
  age: 25
});
```

So:

```text
Component → function
Props     → inputs/arguments
JSX       → UI description
```

That's a very useful mental model.

But here's the important React-specific part:

**Props are read-only inputs.**

The child shouldn't do:

```js
props.name = "Someone else";
```

Instead, the component that owns the data changes it and passes a new prop down.

We'll build that model carefully when we get to state.

---

# 11. Props can be anything

You're not limited to strings.

### Number

```jsx
<Product price={5000} />
```

### Boolean

```jsx
<Button disabled={true} />
```

### Object

```jsx
<User user={user} />
```

### Array

```jsx
<UserList users={users} />
```

### Function

```jsx
<Button onClick={handleClick} />
```

### JSX

```jsx
<Card icon={<Star />} />
```

That last one is particularly important.

React props aren't just "HTML attributes."

They're arbitrary data passed from one component to another.

---

# 12. `children`

There's one special-looking prop that you'll use constantly:

```jsx
<Card>
  <h2>Hello</h2>
</Card>
```

The content inside:

```jsx
<Card> ... </Card>
```

becomes the `children` prop.

So:

```jsx
function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  );
}
```

Now:

```jsx
<Card>
  <h2>Hello</h2>
  <p>Welcome.</p>
</Card>
```

conceptually means:

```text
Card({
  children: ...
})
```

This gives us a very powerful composition pattern.

---

# 13. Why `children` matters

Imagine a reusable modal:

```jsx
function Modal({ children }) {
  return (
    <div className="modal">
      {children}
    </div>
  );
}
```

Now you can put different content inside it:

```jsx
<Modal>
  <h2>Delete account?</h2>
  <button>Delete</button>
</Modal>
```

Or:

```jsx
<Modal>
  <h2>Welcome</h2>
  <p>Thanks for signing up.</p>
</Modal>
```

The modal controls the **container/layout**.

The caller controls the **content**.

That's composition.

---

# 14. Fragments

Suppose a component needs to return two elements:

```jsx
function App() {
  return (
    <h1>Hello</h1>
    <p>Welcome</p>
  );
}
```

That doesn't work.

You can wrap them:

```jsx
function App() {
  return (
    <div>
      <h1>Hello</h1>
      <p>Welcome</p>
    </div>
  );
}
```

But perhaps you don't want another DOM element.

Use a fragment:

```jsx
function App() {
  return (
    <>
      <h1>Hello</h1>
      <p>Welcome</p>
    </>
  );
}
```

The fragment groups the elements without introducing an extra DOM node.

---

# 15. Your first component hierarchy

Let's construct something realistic.

```jsx
function Avatar() {
  return (
    <img
      src="/avatar.jpg"
      alt="Alex"
    />
  );
}

function UserInfo() {
  return (
    <div>
      <h2>Alex</h2>
      <p>Software Developer</p>
    </div>
  );
}

function UserCard() {
  return (
    <article>
      <Avatar />
      <UserInfo />
    </article>
  );
}

function App() {
  return (
    <main>
      <UserCard />
    </main>
  );
}
```

The structure is:

```text
App
└── UserCard
    ├── Avatar
    └── UserInfo
```

This is the heart of component composition.

---

# 16. Now make it reusable

Currently `UserCard` always displays Alex.

Let's make it generic:

```jsx
function Avatar({ src, alt }) {
  return (
    <img
      src={src}
      alt={alt}
    />
  );
}

function UserInfo({ name, role }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>{role}</p>
    </div>
  );
}

function UserCard({ user }) {
  return (
    <article>
      <Avatar
        src={user.avatar}
        alt={user.name}
      />

      <UserInfo
        name={user.name}
        role={user.role}
      />
    </article>
  );
}
```

Then:

```jsx
function App() {
  const user = {
    name: "Alex",
    role: "Software Developer",
    avatar: "/alex.jpg"
  };

  return <UserCard user={user} />;
}
```

Now the component is reusable.

---

# 17. Notice the data flow

This example demonstrates an important principle.

```text
App
 │
 │ user
 ↓
UserCard
 │
 ├── user.name ──→ Avatar
 │
 └── user.role ──→ UserInfo
```

Data moves down:

```text
Parent
  ↓
Child
  ↓
Child
```

We'll eventually call this **one-way data flow**.

For now, simply get comfortable seeing props flowing downward through the component tree.

---

# 18. One component can render many components

Suppose:

```js
const users = [
  { id: 1, name: "Alex" },
  { id: 2, name: "Sarah" },
  { id: 3, name: "Mike" }
];
```

You can use JavaScript's `map()`:

```jsx
function UserList() {
  return (
    <div>
      {users.map(user => (
        <UserCard
          key={user.id}
          user={user}
        />
      ))}
    </div>
  );
}
```

We're using three things simultaneously:

```text
JavaScript
  ↓
map()
  ↓
JSX
  ↓
React components
```

You'll use this constantly in real applications.

We'll study `key` properly in the next lessons.

---

# 19. A crucial distinction: component vs component instance

Suppose:

```jsx
<UserCard user={alex} />
<UserCard user={sarah} />
<UserCard user={mike} />
```

There is one component definition:

```jsx
function UserCard(...) { ... }
```

but React has three rendered instances of it.

Conceptually:

```text
UserCard definition
       │
       ├── instance #1 → alex
       ├── instance #2 → sarah
       └── instance #3 → mike
```

This becomes extremely important when we learn state.

Each instance can have its **own state**.

---

# 20. The first big mental model

At this point, I want you to internalize this:

```text
                 props
                   ↓
Component ─────────────→ UI
                   ↑
                 state
```

A component receives data through props and may maintain state.

It uses those values to describe what the UI should look like.

Later we'll add:

```text
user interaction
      ↓
state update
      ↓
new render
      ↓
new UI
```

That is the larger React loop.

---

# Practice

Don't just read this. Write these yourself.

### Exercise A

Create:

```jsx
function Greeting(...) {
  ...
}
```

It should accept:

```text
name
```

and render:

```text
Hello, Alice!
```

So:

```jsx
<Greeting name="Alice" />
```

should produce the appropriate heading.

### Exercise B

Create:

```text
UserCard
├── Avatar
└── UserInfo
```

`UserCard` should receive:

```js
{
  name,
  role,
  avatar
}
```

and pass the appropriate props to its children.

### Exercise C

Create three different `UserCard`s from an array using `map()`.

Don't worry about `key` yet beyond putting a stable `id` in it.

---

## Next lesson

Once these pieces are comfortable, we move to the first genuinely important React concept:

**state.**

That's where we'll answer questions like:

> Why doesn't changing a normal JavaScript variable update the screen?

> What exactly happens when `setState()` is called?

> Why does React re-render?

> Why does state sometimes appear "one step behind"?

> Why is this wrong?

```jsx
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);
```

and why is this different?

```jsx
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
```

We'll use those questions to build the React rendering model rather than memorizing `useState`.