
# Lesson 14 — Component Composition and Better Prop Design

You've now learned the mechanics of React pretty well.

The next step is **design**.

You can already make:

```text
Component
Props
State
Callbacks
Context
Hooks
Reducers
Effects
```

But now imagine you're building a real application with 50 components.

The question changes from:

> "How do I make this work?"

to:

> **"How should these components communicate and be structured?"**

That's what this lesson is about.

React's own documentation emphasizes composition and passing JSX through `children` as important ways to make components flexible without creating unnecessary prop plumbing. ([React](https://react.dev/learn/passing-props-to-a-component?utm_source=chatgpt.com "Passing Props to a Component – React"))

---

# 1. The problem with overly specific components

Imagine you create:

```jsx
function BlueButton() {
  return (
    <button className="blue">
      Click me
    </button>
  );
}
```

Then:

```jsx
function RedButton() {
  return (
    <button className="red">
      Delete
    </button>
  );
}
```

Then:

```jsx
function GreenButton() {
  return (
    <button className="green">
      Save
    </button>
  );
}
```

You are creating separate components for superficial variations.

A better component might be:

```jsx
function Button({ children, variant }) {
  return (
    <button className={variant}>
      {children}
    </button>
  );
}
```

Then:

```jsx
<Button variant="blue">
  Click me
</Button>

<Button variant="red">
  Delete
</Button>

<Button variant="green">
  Save
</Button>
```

Now:

```text
Button
 ├── variant → appearance/behavior
 └── children → content
```

The component provides the **structure**, while the caller provides the **content**.

---

# 2. This is composition

Composition means:

> **Build a larger UI by putting components together rather than making one component responsible for everything.**

For example:

```jsx
function Card({ children }) {
  return (
    <div className="card">
      {children}
    </div>
  );
}
```

Then:

```jsx
<Card>
  <h2>Profile</h2>
  <p>Alex Morgan</p>
</Card>
```

The Card doesn't need to know what is inside it.

It's simply providing a container.

React's documentation describes `children` as a particularly useful pattern for visual wrappers such as panels and grids. ([React](https://react.dev/learn/passing-props-to-a-component?utm_source=chatgpt.com "Passing Props to a Component – React"))

---

# 3. Think of `children` as a hole

This is one of the best mental models for `children`.

Imagine:

```text
Card
┌───────────────────────────┐
│                           │
│        children           │
│          ↓                │
│      [ whatever ]         │
│                           │
└───────────────────────────┘
```

The Card owns:

```text
border
padding
background
layout
```

The caller owns:

```text
content
```

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

is essentially:

> "Give me whatever content you want, and I'll put it inside my card."

---

# 4. Composition is more flexible than giant prop APIs

Imagine this component:

```jsx
function Card({
  title,
  subtitle,
  image,
  body,
  showButton,
  buttonText,
  buttonColor,
  showFooter,
  footerText,
  ...
}) {
  // enormous amount of logic
}
```

Eventually you get:

```text
20 props
```

and combinations like:

```text
showButton=true
showFooter=false
image=null
body=...
```

become difficult to reason about.

Instead:

```jsx
<Card>
  <CardHeader>
    ...
  </CardHeader>

  <CardBody>
    ...
  </CardBody>

  <CardFooter>
    ...
  </CardFooter>
</Card>
```

Composition lets the caller structure the content explicitly.

---

# 5. A better Card

```jsx
function Card({ children }) {
  return (
    <article className="card">
      {children}
    </article>
  );
}

function CardHeader({ children }) {
  return (
    <header className="card-header">
      {children}
    </header>
  );
}

function CardBody({ children }) {
  return (
    <div className="card-body">
      {children}
    </div>
  );
}
```

Usage:

```jsx
<Card>
  <CardHeader>
    <h2>Account</h2>
  </CardHeader>

  <CardBody>
    <p>Balance: ₹50,000</p>
  </CardBody>
</Card>
```

Notice how the components communicate primarily through composition.

---

# 6. `children` is just a prop

This is worth reinforcing because it can feel magical at first.

When you write:

```jsx
<Card>
  <h2>Hello</h2>
</Card>
```

the parent effectively supplies a prop called:

```text
children
```

So:

```jsx
function Card(props) {
  console.log(props.children);
}
```

works.

Or destructure:

```jsx
function Card({ children }) {
  ...
}
```

React's documentation explicitly describes nested JSX as being received in the `children` prop. ([React](https://react.dev/learn/passing-props-to-a-component?utm_source=chatgpt.com "Passing Props to a Component – React"))

---

# 7. `children` can be text too

For example:

```jsx
<Card>
  Hello
</Card>
```

Then:

```jsx
function Card({ children }) {
  return (
    <div>
      {children}
    </div>
  );
}
```

`children` is:

```text
"Hello"
```

It doesn't have to be another component.

---

# 8. `children` can be multiple elements

```jsx
<Card>
  <h2>Hello</h2>
  <p>Welcome</p>
  <button>Continue</button>
</Card>
```

Then the Card receives all of that as its children.

The details of the `children` data structure are intentionally opaque, so you shouldn't depend on whether React happens to represent it internally as one item or an array. React provides `Children` APIs for cases where you genuinely need to inspect/transform children. ([React](https://react.dev/reference/react/Children?utm_source=chatgpt.com "Children – React"))

For ordinary component composition, simply use:

```jsx
{children}
```

You generally don't need the `Children` API.

---

# 9. Composition with layout

This becomes extremely useful for layouts.

```jsx
function PageLayout({ children }) {
  return (
    <div className="page">
      <aside>
        Sidebar
      </aside>

      <main>
        {children}
      </main>
    </div>
  );
}
```

Then:

```jsx
<PageLayout>
  <Dashboard />
</PageLayout>
```

or:

```jsx
<PageLayout>
  <Settings />
</PageLayout>
```

Same layout.

Different page content.

That's composition.

---

# 10. Don't confuse composition with inheritance

React doesn't primarily encourage the object-oriented pattern:

```text
BaseComponent
   ↓
SpecializedComponent
   ↓
MoreSpecializedComponent
```

Instead, React tends to favor:

```text
small components
     +
composition
     +
props
     +
children
```

For example:

```jsx
<Card>
  <Profile />
</Card>
```

rather than:

```text
ProfileCard extends Card
```

Composition is often easier to reason about because you can see the structure directly in the JSX.

---

# 11. Passing a component as a prop

You aren't limited to passing rendered JSX.

You can pass a component itself:

```jsx
function Page({ Header }) {
  return (
    <div>
      <Header />
      <main>Hello</main>
    </div>
  );
}
```

Then:

```jsx
<Page Header={AdminHeader} />
```

This gives you another form of composition.

---

# 12. But understand the distinction

These are different:

### Component

```jsx
Header={AdminHeader}
```

You're passing the component type/function.

### JSX element

```jsx
header={<AdminHeader />}
```

You're passing an already-created React element.

Both can be useful, but they communicate different abstractions.

For example:

```jsx
function Layout({ header }) {
  return (
    <main>
      {header}
    </main>
  );
}
```

Then:

```jsx
<Layout
  header={<AdminHeader />}
/>
```

Here the parent decides exactly what JSX to supply.

That's often extremely flexible.

---

# 13. Multiple "slots"

Sometimes a component needs several areas of customizable content.

For example:

```jsx
function Dialog({
  title,
  actions,
  children
}) {
  return (
    <div className="dialog">
      <header>
        {title}
      </header>

      <section>
        {children}
      </section>

      <footer>
        {actions}
      </footer>
    </div>
  );
}
```

Usage:

```jsx
<Dialog
  title={<h2>Delete account?</h2>}
  actions={
    <>
      <button>Cancel</button>
      <button>Delete</button>
    </>
  }
>
  <p>
    This cannot be undone.
  </p>
</Dialog>
```

Now the component has several content "slots":

```text
title
children
actions
```

This can be cleaner than dozens of configuration props.

---

# 14. Don't overuse `children`

You might think:

> "Everything should just be children!"

Not necessarily.

Suppose:

```jsx
<UserCard user={user} />
```

That's perfectly clear.

The component has a meaningful data input:

```text
user
```

While:

```jsx
<Card>
  ...
</Card>
```

is appropriate when the caller is composing arbitrary content into a wrapper.

So:

```text
specific data
→ named prop

arbitrary nested content
→ children
```

is a useful rule.

---

# 15. Named content props

Sometimes you want multiple content slots.

You could write:

```jsx
<Dialog
  header={<DialogTitle />}
  body={<DialogBody />}
  footer={<DialogActions />}
/>
```

Then:

```jsx
function Dialog({
  header,
  body,
  footer
}) {
  return (
    <div>
      <header>{header}</header>
      <main>{body}</main>
      <footer>{footer}</footer>
    </div>
  );
}
```

These are just ordinary props.

There is nothing special about them.

---

# 16. Render props

Now we reach a slightly more advanced pattern.

Suppose a component knows **data**, but doesn't know how you want that data rendered.

You could pass a function:

```jsx
function List({
  items,
  renderItem
}) {
  return (
    <ul>
      {items.map(item =>
        renderItem(item)
      )}
    </ul>
  );
}
```

Then:

```jsx
<List
  items={users}
  renderItem={user => (
    <li key={user.id}>
      {user.name}
    </li>
  )}
/>
```

Here:

```text
items
 ↓
List
 ↓
renderItem(item)
 ↓
JSX
```

The function prop is often called a **render prop**.

React's documentation notes that there is nothing inherently special about render props: they're ordinary function props whose purpose is to determine how something gets rendered. ([React](https://react.dev/reference/react/Children?utm_source=chatgpt.com "Children – React"))

---

# 17. Why would you do this?

Imagine a reusable component knows:

```text
mouse position
```

but you want different UIs.

One consumer wants:

```text
x: 100
y: 300
```

Another wants:

```text
●
```

Another wants:

```text
Mouse is at 100, 300
```

Instead of forcing the reusable component to know all those designs, it can expose the data:

```jsx
<MouseTracker
  render={position => (
    <Cursor position={position} />
  )}
/>
```

The reusable logic controls the behavior.

The caller controls the presentation.

This is a powerful abstraction, although modern React code often uses custom Hooks for reusable behavior instead.

---

# 18. Render prop vs custom Hook

Suppose you need reusable mouse tracking.

Older/alternative style:

```jsx
<MouseTracker
  render={position => (
    <Cursor position={position} />
  )}
/>
```

Modern Hook-based style:

```jsx
function Cursor() {
  const position = useMousePosition();

  return (
    <div
      style={{
        left: position.x,
        top: position.y
      }}
    />
  );
}
```

The Hook version separates:

```text
logic
```

from:

```text
UI
```

without requiring a wrapper component.

This is one reason custom Hooks became such an important pattern.

---

# 19. "Wrapper hell"

Imagine:

```jsx
<Auth>
  <Theme>
    <Loading>
      <Permission>
        <Analytics>
          <Dashboard />
        </Analytics>
      </Permission>
    </Loading>
  </Theme>
</Auth>
```

Some nesting is completely normal.

But excessive wrapper components can make structure hard to understand.

Composition should simplify your architecture, not create a maze.

Context and custom Hooks can sometimes help reduce unnecessary wrappers, depending on the problem.

---

# 20. A common prop-design mistake

Suppose you create:

```jsx
<Button
  text="Save"
  color="blue"
  size="medium"
  rounded={true}
  icon="save"
  iconPosition="left"
/>
```

This isn't inherently bad.

But eventually you might end up with:

```jsx
<Button
  text="..."
  color="..."
  size="..."
  rounded={...}
  variant={...}
  outlined={...}
  loading={...}
  disabled={...}
  fullWidth={...}
  ...
/>
```

The component becomes a giant configuration engine.

Sometimes composition is better:

```jsx
<Button>
  <SaveIcon />
  Save
</Button>
```

or:

```jsx
<Button variant="primary">
  <SaveIcon />
  Save
</Button>
```

The content stays flexible without adding another prop for every possible visual arrangement.

---

# 21. Props as a component API

Start thinking of a component like a small API.

For:

```jsx
<UserCard
  user={user}
  onDelete={handleDelete}
/>
```

the component contract is:

```text
Input:
user
onDelete

Output:
UI + interaction
```

For:

```jsx
<Card>
  ...
</Card>
```

the contract is:

```text
Input:
children

Output:
a visual container
```

Good component design means making those APIs easy to understand.

---

# 22. A useful rule: expose intent, not implementation

Remember this from lifting state:

Instead of:

```jsx
<SearchBox
  setSearchTerm={setSearchTerm}
/>
```

prefer:

```jsx
<SearchBox
  onSearchChange={handleSearchChange}
/>
```

Why?

Because:

```text
setSearchTerm
```

describes the parent's implementation.

While:

```text
onSearchChange
```

describes the component's behavior.

The same idea applies everywhere.

Bad abstraction:

```jsx
<TodoItem
  setTasks={setTasks}
/>
```

Better:

```jsx
<TodoItem
  onDelete={handleDelete}
  onToggle={handleToggle}
/>
```

The child shouldn't know how the parent stores its state.

---

# 23. Another important design principle: make components ignorant

Suppose:

```jsx
function SearchBox({
  value,
  onChange
}) {
  return (
    <input
      value={value}
      onChange={e =>
        onChange(e.target.value)
      }
    />
  );
}
```

This component doesn't know:

```text
Redux?
Context?
useState?
URL?
server?
parent component?
```

It simply knows:

```text
value
onChange
```

That's excellent.

The component becomes reusable in many contexts.

---

# 24. Container vs presentation

You'll sometimes see a conceptual distinction:

```text
container
   ↓
handles data/behavior

presentational component
   ↓
renders UI
```

For example:

```jsx
function TodoPage() {
  const {
    tasks,
    deleteTask,
    toggleTask
  } = useTasks();

  return (
    <TodoList
      tasks={tasks}
      onDelete={deleteTask}
      onToggle={toggleTask}
    />
  );
}
```

And:

```jsx
function TodoList({
  tasks,
  onDelete,
  onToggle
}) {
  return (
    <ul>
      {tasks.map(task => (
        <TaskItem
          key={task.id}
          task={task}
          onDelete={onDelete}
          onToggle={onToggle}
        />
      ))}
    </ul>
  );
}
```

The first is more concerned with:

```text
data + behavior
```

The second:

```text
presentation
```

This isn't a rigid React rule, but it's a useful architectural pattern.

---

# 25. Don't force every component into that pattern

Again, avoid dogma.

Sometimes this is perfectly fine:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      {count}
    </button>
  );
}
```

There's no reason to split:

```text
CounterLogic
CounterUI
CounterContainer
CounterView
```

just because someone says components should be "separated."

Good architecture is about meaningful boundaries.

---

# 26. Compound components

Now let's look at a more advanced composition pattern.

Imagine a reusable Tabs component.

You might use:

```jsx
<Tabs>
  <Tabs.List>
    <Tabs.Tab>Overview</Tabs.Tab>
    <Tabs.Tab>Settings</Tabs.Tab>
  </Tabs.List>

  <Tabs.Panel>
    Overview content
  </Tabs.Panel>

  <Tabs.Panel>
    Settings content
  </Tabs.Panel>
</Tabs>
```

This style is called a **compound component pattern**.

The components cooperate around a shared conceptual system.

Typically, Context is involved behind the scenes so the pieces can share state.

Conceptually:

```text
Tabs
├── Tabs.List
├── Tabs.Tab
└── Tabs.Panel
```

This is more advanced and you don't need to implement it from scratch yet.

But you'll see this pattern in UI libraries.

---

# 27. Why compound components are useful

Compare:

```jsx
<Tabs
  activeTab={1}
  tabs={...}
  renderTab={...}
  renderPanel={...}
  ...
/>
```

with:

```jsx
<Tabs>
  <Tabs.List>
    ...
  </Tabs.List>

  <Tabs.Panel>
    ...
  </Tabs.Panel>
</Tabs>
```

The second can read more like the actual UI structure.

The abstraction is expressed through composition rather than a huge configuration object.

---

# 28. But there's a tradeoff

Composition gives flexibility.

Too much composition can give you:

```text
15 tiny components
```

that are difficult to navigate.

Similarly, a component with 25 props can be difficult to use.

You're balancing:

```text
flexibility
     vs
simplicity
```

There's no universal number of props or components that is "correct."

Good React architecture is largely about choosing useful boundaries.

---

# 29. Your component design toolbox

At this point, you have several ways to design communication:

### Ordinary props

```jsx
<Product product={product} />
```

Use when the child needs specific data.

### Callback props

```jsx
<Product onDelete={handleDelete} />
```

Use when the child needs to report/trigger something.

### `children`

```jsx
<Card>
  <Profile />
</Card>
```

Use when the parent controls the container and caller controls the content.

### JSX props / slots

```jsx
<Dialog
  header={<Title />}
  footer={<Actions />}
/>
```

Use when you want multiple customizable content regions.

### Function/render props

```jsx
<List
  renderItem={item => <Item item={item} />}
/>
```

Use when the reusable component owns behavior/data and the caller controls rendering.

### Context

```text
Provider
  ↓
deep descendants
```

Use when contextual information needs to be available deep in a subtree.

These aren't competing "React tricks."

They're different ways of expressing component relationships.

---

# 30. How to decide which one to use

Ask:

### Does the child need a specific piece of data?

Use:

```text
props
```

### Does the child need to report an event?

Use:

```text
callback prop
```

### Does the parent want to inject arbitrary JSX into a wrapper?

Use:

```text
children
```

### Does the component need multiple customizable areas?

Use:

```text
named JSX props / composition
```

### Does the component own reusable behavior but let the caller decide how to display it?

Possibly:

```text
render prop
```

or, often in modern React:

```text
custom Hook
```

### Do many distant descendants need contextual information?

Possibly:

```text
Context
```

This decision tree will become very useful.

---

# 31. A complete example

Let's build a reusable modal.

```jsx
function Modal({
  title,
  children,
  actions,
  onClose
}) {
  return (
    <div className="modal">
      <div className="modal-header">
        <h2>{title}</h2>

        <button onClick={onClose}>
          ×
        </button>
      </div>

      <div className="modal-body">
        {children}
      </div>

      <div className="modal-footer">
        {actions}
      </div>
    </div>
  );
}
```

Usage:

```jsx
<Modal
  title="Delete account?"
  onClose={handleClose}
  actions={
    <>
      <button onClick={handleClose}>
        Cancel
      </button>

      <button onClick={handleDelete}>
        Delete
      </button>
    </>
  }
>
  <p>
    This action cannot be undone.
  </p>
</Modal>
```

The responsibilities are nicely separated:

```text
Modal
 ├── layout
 ├── title location
 ├── close behavior
 ├── content location
 └── actions location

caller
 ├── title content
 ├── body content
 └── action buttons
```

That's composition working properly.

---

# 32. The deeper idea

We're moving from:

```text
"How do I pass data?"
```

to:

```text
"How do I design component boundaries?"
```

That's a major step in becoming good at React.

A component should ideally expose a **small, understandable API**.

For example:

```jsx
<UserCard
  user={user}
  onDelete={handleDelete}
/>
```

is easy to understand.

Whereas:

```jsx
<UserCard
  setUsers={setUsers}
  setSelectedUser={setSelectedUser}
  setLoading={setLoading}
  setError={setError}
  ...
/>
```

is a sign that the component may be coupled too tightly to the parent's implementation.

---

# 33. The composition mental model

Keep this picture:

```text
                  Parent
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
        props    children   callbacks
          ↓         ↓         ↑
        Child    Child UI    Child event
```

And:

```text
props
→ "Here's some information."

children
→ "Here's some UI to put inside you."

callback
→ "Call me when this thing happens."

context
→ "This information is available throughout this subtree."
```

Those four phrases are enough to decode a huge amount of React code.

---

# One final thing

You've now covered most of the **core React conceptual toolkit**:

```text
✓ Components
✓ JSX
✓ Props
✓ Events
✓ State
✓ Lists
✓ Keys
✓ Derived data
✓ State ownership
✓ Lifting state
✓ Forms
✓ useEffect
✓ useRef
✓ useContext
✓ useReducer
✓ Custom Hooks
✓ Rendering model
✓ Component composition
```

That means we're ready to move from **React concepts** to **building applications**.

## Next lesson — Routing

We'll take the single-page React application:

```text
App
```

and turn it into an application with:

```text
/
 /dashboard
 /products
 /products/:id
 /settings
```

We'll learn what routing actually means, client-side navigation, route parameters, nested routes, layouts, links, navigation, and protected routes.

That will be the beginning of the **real application-building phase** of the curriculum.