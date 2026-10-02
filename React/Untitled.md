
## The complete React curriculum

I'd teach you React in **8 stages**, going from "I know JavaScript" to "I can design and build a serious React application."

The goal is not merely to know APIs. By the end, you should be able to look at an unfamiliar React codebase and understand **why it is structured the way it is**.

---

# Stage 0 — React mental model

Before syntax, we establish how React thinks.

### 0.1 What React is

You'll understand:

- React vs vanilla JavaScript
    
- declarative vs imperative UI
    
- components
    
- one-way data flow
    
- rendering
    
- state-driven UI
    
- why React exists
    

### 0.2 The fundamental model

We'll repeatedly come back to:

```text
UI = f(props, state)
```

And:

```text
user interaction
      ↓
event
      ↓
state update
      ↓
render
      ↓
new UI
```

### 0.3 What React does and does not do

We'll distinguish React itself from things commonly used alongside React:

```text
React
React Router
TanStack Query
Redux / Zustand
Vite
Next.js
```

This prevents the common beginner problem of thinking "React" is an entire ecosystem with one giant API.

---

# Stage 1 — React fundamentals

This is where you actually start writing React.

## 1.1 Components

You'll learn:

- function components
    
- component naming
    
- component composition
    
- parent/child relationships
    
- component boundaries
    
- importing/exporting components
    

Example:

```jsx
function Button() {
  return <button>Click</button>;
}
```

You'll also learn **when NOT to create a component**.

---

## 1.2 JSX

We'll cover JSX thoroughly:

```jsx
const element = <h1>Hello</h1>;
```

You'll understand:

- expressions inside JSX
    
- `{}` syntax
    
- attributes
    
- `className`
    
- fragments
    
- nested JSX
    
- conditional JSX
    
- JSX vs HTML
    
- JSX vs `React.createElement`
    
- what the build tool does with JSX
    

We'll also examine the surprising cases:

```jsx
{0}
{false}
{null}
{"hello"}
```

and what React actually renders.

---

## 1.3 Props

You'll learn:

```jsx
<User name="Alex" />
```

including:

- primitive props
    
- objects
    
- arrays
    
- functions
    
- JSX as props
    
- destructuring
    
- default values
    
- `children`
    
- prop conventions
    

We'll go beyond "props are arguments" and understand the **data-flow implications**.

---

## 1.4 Events

You'll learn:

```jsx
<button onClick={handleClick}>
```

including:

- event handlers
    
- event objects
    
- passing arguments
    
- inline callbacks
    
- forms/events
    
- keyboard events
    
- event propagation
    
- `preventDefault`
    
- propagation control
    

---

# Stage 2 — State and interactive UI

This is where React really starts clicking.

## 2.1 `useState`

We'll deeply understand:

```jsx
const [count, setCount] = useState(0);
```

You'll learn:

- initial state
    
- state updates
    
- functional updates
    
- multiple state variables
    
- state initialization
    
- state persistence
    
- why normal variables don't work
    
- batching
    
- asynchronous-looking state behavior
    
- stale values
    

For example, you'll learn why this can surprise people:

```jsx
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);
```

versus:

```jsx
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
```

---

## 2.2 Objects and arrays in state

You'll learn immutable updates:

```jsx
setUser({
  ...user,
  name: "Sam"
});
```

and:

```jsx
setTodos(
  todos.filter(todo => todo.id !== id)
);
```

We'll cover:

- object updates
    
- nested objects
    
- arrays
    
- add/remove/update
    
- why mutation causes problems
    
- structural sharing
    

---

## 2.3 Conditional rendering

You'll learn:

```jsx
condition && <Component />
```

```jsx
condition ? <A /> : <B />
```

and normal JavaScript conditions outside JSX.

We'll also cover practical patterns for:

```text
loading
error
empty
success
authenticated
unauthenticated
```

---

## 2.4 Lists and keys

You'll learn:

```jsx
users.map(user => ...)
```

and deeply understand:

- why `key` exists
    
- stable identity
    
- why array index can be problematic
    
- what happens when items move
    
- how keys affect state preservation
    

This leads naturally into React's rendering model.

---

## 2.5 Forms

We'll build real forms.

You'll learn:

- controlled inputs
    
- text inputs
    
- checkboxes
    
- radio buttons
    
- selects
    
- textareas
    
- multiple fields
    
- form submission
    
- validation
    
- error messages
    
- resetting forms
    

Example:

```jsx
<input
  value={name}
  onChange={e => setName(e.target.value)}
/>
```

You'll understand **why** this works instead of memorizing it.

---

# Stage 3 — Thinking in React

This is one of the most important parts of the entire course.

A lot of developers can write React syntax but struggle to **design React applications**.

We'll learn:

## 3.1 State vs props

You'll be able to identify:

```text
What belongs in state?
What should be a prop?
What should be derived?
```

For example, we'll distinguish:

```text
products
searchTerm
filteredProducts
```

and understand why `filteredProducts` often shouldn't be another state variable.

---

## 3.2 Derived state

We'll examine patterns like:

```jsx
const filteredProducts =
  products.filter(...);
```

instead of unnecessarily doing:

```jsx
const [filteredProducts, setFilteredProducts] = useState(...);
```

This is a deceptively important concept.

---

## 3.3 Lifting state up

We'll understand:

```text
         Parent
        /      \
   Child A   Child B
```

and determine when shared state belongs in the parent.

---

## 3.4 State ownership

We'll develop a systematic way of asking:

> Who should own this state?

We'll cover:

- local state
    
- shared state
    
- URL state
    
- server state
    
- global state
    

That distinction becomes extremely important later.

---

## 3.5 Component composition

We'll learn patterns like:

```jsx
<Card>
  <Avatar />
  <UserInfo />
</Card>
```

and:

```jsx
<Modal>
  ...
</Modal>
```

You'll understand why composition is often preferable to creating giant configurable components.

---

# Stage 4 — Hooks deeply

This stage is where many React tutorials become confusing. We'll treat hooks as **solutions to specific problems**, not magic functions.

## 4.1 Rules of Hooks

You'll understand why this is wrong:

```jsx
if (something) {
  useState(...)
}
```

and why hook calls need stable ordering.

---

## 4.2 `useEffect`

We'll go much deeper than:

> "useEffect runs code after rendering."

We'll study:

- effects
    
- side effects
    
- synchronization
    
- dependencies
    
- cleanup
    
- effect lifecycle
    
- stale closures
    
- infinite loops
    
- race conditions
    
- fetch cancellation
    
- when an effect is appropriate
    
- when an effect is unnecessary
    

You'll learn to recognize the classic smell:

```jsx
useEffect(() => {
  setSomething(...)
}, [...])
```

and ask whether the state exists only because the code was designed incorrectly.

---

## 4.3 `useRef`

We'll use refs for:

```text
DOM access
focus
previous values
mutable values
imperative APIs
```

And we'll make the crucial distinction:

```text
useState → changing value should cause UI update
useRef   → changing value should NOT cause UI update
```

---

## 4.4 `useContext`

You'll learn:

- context
    
- provider pattern
    
- consuming context
    
- avoiding prop drilling
    
- appropriate uses
    
- common misuse
    

We'll build things such as:

```text
ThemeContext
AuthContext
```

and discuss when context is insufficient.

---

## 4.5 Custom hooks

We'll move from:

```jsx
function Component() {
  // giant pile of logic
}
```

to:

```jsx
const { data, loading, error } = useUsers();
```

You'll learn how to extract **reusable behavior**, not merely reusable UI.

---

## 4.6 `useMemo` and `useCallback`

These will be taught later than most tutorials teach them.

You'll learn:

- memoization
    
- referential equality
    
- dependency arrays
    
- `React.memo`
    
- when memoization helps
    
- when it doesn't
    
- when it makes code worse
    

The goal is to avoid the beginner habit of sprinkling `useMemo` everywhere.

---

# Stage 5 — Real applications

Now we'll move from isolated examples to applications.

## 5.1 Application architecture

We'll build applications with structures like:

```text
src/
├── components/
├── pages/
├── hooks/
├── services/
├── utils/
├── contexts/
├── features/
└── assets/
```

But we'll discuss **why** each exists and when such organization is justified.

---

## 5.2 Data fetching

We'll learn the browser side first:

```jsx
fetch(...)
```

Then build complete request states:

```text
idle
loading
success
error
empty
```

We'll handle:

- GET
    
- POST
    
- PUT/PATCH
    
- DELETE
    
- request errors
    
- loading states
    
- race conditions
    
- aborting requests
    
- optimistic UI
    

---

## 5.3 API architecture

Instead of putting this everywhere:

```jsx
fetch("/api/users")
```

we'll learn to separate:

```text
UI
 ↓
custom hook
 ↓
API/service layer
 ↓
backend
```

---

## 5.4 Routing

We'll learn React Router concepts:

- routes
    
- nested routes
    
- route parameters
    
- query parameters
    
- navigation
    
- links
    
- redirects
    
- protected routes
    
- layouts
    
- 404 pages
    

For example:

```text
/app
/app/dashboard
/app/settings
/app/users/:id
```

---

## 5.5 Authentication

We'll build an authentication flow conceptually around:

```text
login
 ↓
token/session
 ↓
authenticated app
 ↓
protected routes
 ↓
logout
```

We'll discuss the security implications rather than treating authentication as simply "put token in localStorage."

---

## 5.6 Reusable UI

We'll build:

```text
Button
Input
Modal
Dropdown
Table
Tabs
Toast
Spinner
Pagination
```

Then discuss the difference between:

```text
reusable component
```

and:

```text
over-engineered component
```

---

# Stage 6 — React internals and advanced rendering

This is where you'll stop thinking of React as magic.

## 6.1 Rendering

We'll understand:

```text
trigger
  ↓
render
  ↓
reconciliation
  ↓
commit
```

and what each stage means.

---

## 6.2 Reconciliation

We'll examine how React figures out:

```text
old UI
   ↓
new UI
```

and how it uses:

- element types
    
- keys
    
- component identity
    

---

## 6.3 State preservation

You'll understand why moving a component or changing its key can cause state to reset.

For example:

```jsx
<UserForm key={userId} />
```

is not merely a performance detail.

Keys can affect **identity and state preservation**.

---

## 6.4 Strict Mode

We'll understand why development behavior can appear unusual, including repeated development-time behavior designed to expose problematic patterns.

---

## 6.5 Performance

We'll cover:

- unnecessary re-renders
    
- expensive calculations
    
- component boundaries
    
- memoization
    
- lazy loading
    
- code splitting
    
- virtualization
    
- profiling
    

But performance will be based on **measurement**, not superstition.

---

# Stage 7 — Modern React

Here we'll move beyond the basic "React tutorial" level.

Because React evolves, we'll cover the current architecture and APIs relevant to modern React applications.

Topics will include:

## 7.1 Suspense

Understanding:

```jsx
<Suspense fallback={<Loading />}>
  ...
</Suspense>
```

and what Suspense is actually solving.

---

## 7.2 Lazy loading

We'll use:

```jsx
lazy(...)
```

and code splitting to understand bundle boundaries.

---

## 7.3 Modern async UI patterns

We'll study modern React patterns for:

```text
async rendering
loading UI
transitions
optimistic interactions
```

and understand when they are useful.

---

## 7.4 Server-oriented React

We'll cover the conceptual difference between:

```text
Client Components
Server Components
```

and how frameworks such as Next.js build on React.

We won't confuse framework behavior with React fundamentals.

---

# Stage 8 — Production-level React

This is where the curriculum becomes professional rather than tutorial-level.

## 8.1 State management

We'll compare approaches based on the problem they solve:

```text
useState
useReducer
Context
URL state
server-state libraries
external client-state libraries
```

We'll examine libraries such as:

```text
Redux Toolkit
Zustand
TanStack Query
```

without assuming that every project needs them.

---

## 8.2 `useReducer`

We'll use it for complex state transitions:

```text
state
  +
action
  ↓
reducer
  ↓
new state
```

You'll understand when this is preferable to several `useState` calls.

---

## 8.3 Testing

We'll learn:

- unit testing
    
- component testing
    
- integration testing
    
- user-centered testing
    
- mocking
    
- testing async behavior
    

Using modern React testing practices rather than testing implementation details.

---

## 8.4 Accessibility

We'll cover:

- semantic HTML
    
- keyboard navigation
    
- labels
    
- focus management
    
- ARIA
    
- modals
    
- forms
    
- screen readers
    

This is not an optional "bonus topic."

---

## 8.5 Error handling

We'll cover:

- API errors
    
- form errors
    
- render failures
    
- error boundaries
    
- retry behavior
    
- fallback UI
    

---

## 8.6 Security

We'll discuss React-specific security concerns:

```text
XSS
dangerouslySetInnerHTML
authentication
authorization
CSRF
untrusted HTML
URL handling
```

---

## 8.7 Production deployment

We'll go through:

```text
development
     ↓
build
     ↓
environment configuration
     ↓
deployment
     ↓
monitoring
```

and understand what actually happens when a React app reaches production.

---

# The projects we'll build

I don't want this to become 100 hours of isolated examples.

We'll build progressively larger applications.

### Project 1 — Counter + UI playground

Purpose:

```text
components
JSX
state
events
conditional rendering
```

Very small, but it establishes fundamentals.

### Project 2 — Todo application

Purpose:

```text
state
arrays
forms
props
component design
derived state
```

### Project 3 — Product search/filter app

Purpose:

```text
complex state
controlled inputs
derived data
component composition
```

### Project 4 — API dashboard

Purpose:

```text
fetching
loading/error states
effects
custom hooks
tables
pagination
```

### Project 5 — Multi-page application

Purpose:

```text
routing
layouts
URL parameters
authentication
protected pages
```

### Project 6 — Production-style application

This will combine:

```text
React
routing
API layer
authentication
server data
client state
forms
validation
error handling
testing
accessibility
performance
```

The exact domain can be something realistic such as an admin dashboard, project manager, finance dashboard, inventory system, or SaaS application.

---

# How deeply we'll go

I don't want to stop at:

> "Here's `useEffect`. Copy this code."

For every significant concept, we'll work at **four levels**.

### Level 1 — Syntax

What do I write?

```jsx
const [count, setCount] = useState(0);
```

### Level 2 — Mental model

What does this actually mean?

```text
React stores state associated with this component instance
and gives the component the current value on render.
```

### Level 3 — Behavior

What happens when something goes wrong?

```text
Why is my effect running again?
Why is my state stale?
Why did my component re-render?
Why did my input lose focus?
Why did my state reset?
```

### Level 4 — Architecture

When should I use this at all?

```text
Should this be state?
Should this be derived?
Should this live in context?
Should this be URL state?
Should this be server state?
Do I need an effect?
Should this component even exist?
```

That fourth level is what turns React knowledge into actual development skill.

---

# The dependency chain

The curriculum is deliberately structured so each idea grows naturally from the previous one:

```text
JavaScript
   ↓
Components
   ↓
JSX
   ↓
Props
   ↓
Events
   ↓
State
   ↓
Rendering
   ↓
Lists / conditions / forms
   ↓
State ownership
   ↓
Effects
   ↓
Refs / Context
   ↓
Custom hooks
   ↓
Routing / APIs
   ↓
Application architecture
   ↓
Rendering internals
   ↓
Performance
   ↓
Modern React
   ↓
Production architecture
```

And there is one particularly important distinction we'll keep returning to:

```text
             CLIENT
               │
       ┌───────┴────────┐
       ↓                ↓
   UI state         Server state
       │                │
 useState/etc.     fetched data
       │                │
       └───────┬────────┘
               ↓
             React
```

Understanding the difference between **UI/client state and server state** is one of the things that later makes React architecture much less confusing.

## Where we go next

Now that you have the curriculum, I would start with **Lesson 1: Components + JSX**, but teach it properly: we'll build a component from a plain JavaScript function, see exactly what JSX is doing, then move into props and component composition with exercises along the way.