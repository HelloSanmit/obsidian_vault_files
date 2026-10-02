
# Lesson 10 — Context: Sharing Data Without Prop Drilling

So far, our main communication mechanism has been:

```text
Parent
  ↓ props
Child
```

And when the child needs to tell the parent something:

```text
Child
  ↓ callback
Parent
```

That model is excellent and should remain your default.

But imagine this:

```text
App
├── Layout
│   ├── Sidebar
│   │   └── Navigation
│   │       └── UserMenu
│   └── MainContent
│       └── Profile
```

Suppose `UserMenu` needs the currently logged-in user.

You could pass it like this:

```jsx
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <Navigation user={user}>
        <UserMenu user={user} />
      </Navigation>
    </Sidebar>
  </Layout>
</App>
```

Notice something uncomfortable:

```text
App
 ↓ user
Layout
 ↓ user
Sidebar
 ↓ user
Navigation
 ↓ user
UserMenu
```

Most of those components don't actually care about `user`.

They're just forwarding it.

This is called **prop drilling**. React's current docs describe Context as a way for a parent to make information available to components deep in its subtree without explicitly passing it through every intermediate component. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 1. What Context actually does

Context creates a channel through which a component can provide some value to its descendants.

Think:

```text
Provider
   │
   │ value
   ↓
 ┌──────────────────────┐
 │      component tree  │
 │                      │
 │   A                  │
 │    ↓                 │
 │   B                  │
 │    ↓                 │
 │   C                  │
 │    ↓                 │
 │   D ← reads context  │
 └──────────────────────┘
```

`D` doesn't need:

```text
A → B → C → D
```

to explicitly pass the value through props.

Instead:

```text
A provides
D reads
```

React handles the connection.

---

# 2. The three pieces of Context

There are three basic operations:

```text
1. createContext()
2. provide a value
3. useContext()
```

Let's build a simple example.

---

# 3. Create a context

```jsx
import { createContext } from "react";

const ThemeContext = createContext("light");
```

This creates a context object.

The `"light"` is its default value.

Conceptually:

```text
ThemeContext
     │
     └── default = "light"
```

React's current API uses `createContext(defaultValue)` to create the context, and the default is used when there is no matching provider above the component reading it. ([React](https://react.dev/reference/react/createContext?utm_source=chatgpt.com "createContext – React"))

---

# 4. Provide the context

A component higher in the tree can provide a value:

```jsx
function App() {
  return (
    <ThemeContext value="dark">
      <Dashboard />
    </ThemeContext>
  );
}
```

Now everything underneath `Dashboard` can access `"dark"`.

Conceptually:

```text
App
 │
 │ ThemeContext = "dark"
 ↓
Dashboard
 ↓
Sidebar
 ↓
Button
```

The current React documentation supports rendering the context object itself as a provider:

```jsx
<ThemeContext value="dark">
```

Older React code commonly uses:

```jsx
<ThemeContext.Provider value="dark">
```

You'll encounter both styles in existing codebases. The underlying concept is the same; the direct provider syntax is the current React approach. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 5. Read the context

A deeply nested component can do:

```jsx
import { useContext } from "react";

function Button() {
  const theme = useContext(ThemeContext);

  return (
    <button className={theme}>
      Click
    </button>
  );
}
```

If the nearest provider gives:

```text
"dark"
```

then:

```js
theme
```

is:

```text
"dark"
```

`useContext` looks upward through the component tree and uses the value from the **closest matching provider**. ([React](https://react.dev/reference/react/useContext?utm_source=chatgpt.com "useContext – React"))

---

# 6. Let's trace it

We have:

```jsx
function App() {
  return (
    <ThemeContext value="dark">
      <Dashboard />
    </ThemeContext>
  );
}
```

Then:

```jsx
function Dashboard() {
  return <Sidebar />;
}
```

Then:

```jsx
function Sidebar() {
  return <Navigation />;
}
```

Then:

```jsx
function Navigation() {
  return <Button />;
}
```

Then:

```jsx
function Button() {
  const theme = useContext(ThemeContext);

  return <button>{theme}</button>;
}
```

There may be several components in between:

```text
App
 ↓
Dashboard
 ↓
Sidebar
 ↓
Navigation
 ↓
Button
```

But `Button` can directly read:

```text
ThemeContext → "dark"
```

The intermediate components don't have to forward it. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 7. Context does NOT mean global variable

This is an important distinction.

You might initially think:

```text
Context = global variable
```

That's not quite right.

Context is scoped to a **subtree**.

For example:

```jsx
<ThemeContext value="dark">
  <PageA />
</ThemeContext>

<ThemeContext value="light">
  <PageB />
</ThemeContext>
```

`PageA` sees:

```text
dark
```

and `PageB` sees:

```text
light
```

So the context value depends on **where the component appears in the tree**.

That's why React's docs describe context as information available to the tree below a provider. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 8. The nearest provider wins

This is important.

Imagine:

```jsx
<ThemeContext value="dark">
  <Page>
    <ThemeContext value="light">
      <Button />
    </ThemeContext>
  </Page>
</ThemeContext>
```

`Button` sees:

```text
light
```

not:

```text
dark
```

Why?

Because the inner provider is closer.

Conceptually:

```text
dark provider
   ↓
Page
   ↓
light provider
   ↓
Button
```

`Button` asks:

> "What's the closest `ThemeContext` provider above me?"

Answer:

```text
light
```

---

# 9. Context can hold objects

You're not limited to strings.

For example:

```jsx
const AuthContext = createContext(null);
```

Then:

```jsx
<AuthContext
  value={{
    user,
    login,
    logout
  }}
>
  <App />
</AuthContext>
```

A descendant can do:

```jsx
function UserMenu() {
  const { user, logout } = useContext(AuthContext);

  return (
    <div>
      <span>{user.name}</span>

      <button onClick={logout}>
        Logout
      </button>
    </div>
  );
}
```

Now Context is carrying:

```text
user
login
logout
```

This starts becoming useful for real application architecture.

---

# 10. Context + state

This is where Context becomes much more interesting.

Context itself doesn't create changing state.

You can combine it with state:

```jsx
function App() {
  const [theme, setTheme] = useState("light");

  return (
    <ThemeContext value={theme}>
      <button
        onClick={() =>
          setTheme(theme =>
            theme === "light"
              ? "dark"
              : "light"
          )
        }
      >
        Toggle
      </button>

      <Dashboard />
    </ThemeContext>
  );
}
```

Now:

```text
state
 ↓
context value
 ↓
descendants
```

When the state changes:

```text
light → dark
```

the Context value changes, and components reading that context update accordingly. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 11. Context isn't itself state

This distinction is important.

Consider:

```jsx
const ThemeContext = createContext("light");
```

This doesn't mean React has created:

```text
theme state
```

It has created:

```text
a context definition
```

State is still something like:

```jsx
const [theme, setTheme] = useState("light");
```

Then Context is the mechanism for distributing that state:

```text
useState
   ↓
theme
   ↓
Context
   ↓
deep components
```

So:

```text
state = where the changing value lives
context = how it can be made available deeply
```

Those are separate concepts.

---

# 12. Compare props and Context

### Props

```text
Parent
  ↓
Child
  ↓
Grandchild
```

Explicit:

```jsx
<Child user={user} />
```

Advantages:

- easy to see where data comes from
    
- component dependencies are explicit
    
- great for normal parent-child communication
    

### Context

```text
Provider
   ↓
many descendants
```

The descendant can access the value without every intermediate component declaring it as a prop.

Advantages:

- avoids repetitive prop forwarding
    
- useful for data needed by many distant descendants
    

React's own documentation recommends trying props and component composition before reaching for Context, because explicit props make data dependencies easier to see. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 13. When Context makes sense

Good examples include:

```text
theme
current user/account
locale
routing-related information
application configuration
```

These tend to be values that many components within a subtree need. React's documentation specifically gives them as common Context use cases. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 14. When Context is unnecessary

Suppose:

```text
App
 ↓
Profile
```

and only `Profile` needs:

```text
user
```

Just use:

```jsx
<Profile user={user} />
```

Creating:

```text
UserContext
```

for this would make the application more complicated, not less.

Similarly, if:

```text
Parent
 ↓
Child
```

needs a single value, don't immediately reach for Context.

Props are perfectly good.

---

# 15. Context isn't a replacement for all props

This is a trap.

You might think:

> "Context eliminates props."

No.

Imagine:

```jsx
<ProductCard product={product} />
```

That's great.

`product` is specifically part of `ProductCard`'s interface.

Using:

```text
ProductContext
```

to supply every product to every component would often make the design worse.

A good distinction:

```text
Component-specific data
→ props

Widely-needed contextual information
→ potentially Context
```

---

# 16. Context and prop drilling

Let's compare.

### Without Context

```text
App
 ↓ user
Layout
 ↓ user
Sidebar
 ↓ user
Navigation
 ↓ user
UserMenu
```

### With Context

```text
             App
              │
       UserContext
              │
             ...
              │
          UserMenu
              │
       useContext()
```

The intermediate components don't need:

```jsx
user={user}
```

at all.

That's the primary benefit.

---

# 17. A real example: authentication

Let's imagine:

```text
App
├── Navbar
│   └── UserMenu
├── Dashboard
│   └── AccountSummary
└── Settings
```

Many of these components need the current user.

A Context might expose:

```jsx
const AuthContext = createContext(null);
```

Then a provider component:

```jsx
function AuthProvider({ children }) {
  const [user, setUser] = useState(null);

  function login(userData) {
    setUser(userData);
  }

  function logout() {
    setUser(null);
  }

  return (
    <AuthContext
      value={{
        user,
        login,
        logout
      }}
    >
      {children}
    </AuthContext>
  );
}
```

Then:

```jsx
function App() {
  return (
    <AuthProvider>
      <Navbar />
      <Dashboard />
      <Settings />
    </AuthProvider>
  );
}
```

And anywhere below:

```jsx
function UserMenu() {
  const { user, logout } =
    useContext(AuthContext);

  if (!user) {
    return <button>Login</button>;
  }

  return (
    <div>
      {user.name}

      <button onClick={logout}>
        Logout
      </button>
    </div>
  );
}
```

Now we have:

```text
AuthProvider
    │
    ├── Navbar
    │    └── UserMenu ── uses auth
    │
    ├── Dashboard ────── uses auth
    │
    └── Settings ─────── uses auth
```

No prop drilling.

---

# 18. Context + callbacks

Notice something interesting.

We previously learned:

```text
state
 ↓
callback props
 ↓
child
```

With Context, you can make both the state and functions available:

```text
Context
 ├── user
 ├── login
 └── logout
```

Then a deeply nested component can:

```jsx
const { logout } = useContext(AuthContext);
```

and call:

```jsx
logout();
```

So Context can distribute not only data, but also functions.

---

# 19. Creating a custom provider component

You will often see a pattern like:

```jsx
function ThemeProvider({ children }) {
  const [theme, setTheme] =
    useState("light");

  function toggleTheme() {
    setTheme(theme =>
      theme === "light"
        ? "dark"
        : "light"
    );
  }

  return (
    <ThemeContext
      value={{
        theme,
        toggleTheme
      }}
    >
      {children}
    </ThemeContext>
  );
}
```

Then:

```jsx
function App() {
  return (
    <ThemeProvider>
      <Dashboard />
    </ThemeProvider>
  );
}
```

And inside:

```jsx
function Button() {
  const {
    theme,
    toggleTheme
  } = useContext(ThemeContext);

  return (
    <button
      className={theme}
      onClick={toggleTheme}
    >
      Toggle theme
    </button>
  );
}
```

This is a common structure.

The provider owns the state and exposes a controlled interface through Context.

---

# 20. Why `children` appears here

Remember our previous lesson about `children`.

Here:

```jsx
function ThemeProvider({ children }) {
```

`children` represents whatever we put inside:

```jsx
<ThemeProvider>
  <Dashboard />
</ThemeProvider>
```

So conceptually:

```text
ThemeProvider
      │
      └── children = <Dashboard />
```

Then:

```jsx
<ThemeContext value={...}>
  {children}
</ThemeContext>
```

puts that whole subtree inside the Context provider.

This is a nice example of **composition + Context working together**.

---

# 21. A useful abstraction: custom Context hook

Instead of having every component write:

```jsx
const theme = useContext(ThemeContext);
```

you can create:

```jsx
function useTheme() {
  return useContext(ThemeContext);
}
```

Then:

```jsx
function Button() {
  const { theme, toggleTheme } = useTheme();

  // ...
}
```

This is technically a **custom hook**.

We're not going deep into custom hooks yet, but this pattern is worth recognizing.

Eventually we'll build these properly.

---

# 22. Context can have a default value

Suppose:

```jsx
const ThemeContext =
  createContext("light");
```

If a component does:

```jsx
const theme =
  useContext(ThemeContext);
```

but there is no provider above it, it gets:

```text
"light"
```

That is the default value. ([React](https://react.dev/reference/react/useContext?utm_source=chatgpt.com "useContext – React"))

But there's a subtle design question:

Should your Context have a meaningful fallback?

For example:

```jsx
const AuthContext =
  createContext(null);
```

may make more sense because:

```text
no provider → no authenticated user
```

rather than inventing some fake default user.

---

# 23. A common pattern: fail loudly

For an application-specific context, you may see:

```jsx
const AuthContext = createContext(null);

function useAuth() {
  const context =
    useContext(AuthContext);

  if (context === null) {
    throw new Error(
      "useAuth must be used within AuthProvider"
    );
  }

  return context;
}
```

Now:

```jsx
function Profile() {
  const { user } = useAuth();

  return <p>{user.name}</p>;
}
```

If `Profile` is accidentally rendered outside the provider, you get a clear error instead of silently using an incorrect fallback.

This is a common quality-of-life pattern in real applications.

---

# 24. Context updates

Suppose:

```jsx
<ThemeContext value={theme}>
  <App />
</ThemeContext>
```

and:

```text
theme = "light"
```

Then:

```js
setTheme("dark");
```

The provider gets a new value:

```text
"dark"
```

Components reading that Context are updated to reflect the new context value. React's `useContext` documentation explicitly describes components as subscribing to context. ([React](https://react.dev/reference/react/useContext?utm_source=chatgpt.com "useContext – React"))

So:

```text
state changes
 ↓
provider value changes
 ↓
consumers receive new context value
```

---

# 25. Don't put everything into one giant Context

You might be tempted to create:

```jsx
<AppContext
  value={{
    user,
    theme,
    cart,
    notifications,
    settings,
    products,
    search,
    ...
  }}
>
```

This can become difficult to reason about.

Instead, different concepts can have different contexts where appropriate:

```text
AuthContext
ThemeContext
CartContext
```

There isn't a requirement that your application have one Context.

React explicitly supports multiple independent contexts. ([React](https://react.dev/learn/passing-data-deeply-with-context?utm_source=chatgpt.com "Passing Data Deeply with Context – React"))

---

# 26. Context does not automatically mean "global state management"

This distinction becomes increasingly important as we progress.

There are different categories of state:

```text
local UI state
shared client state
server state
URL state
contextual configuration
```

Context can help distribute certain values.

But that doesn't mean:

```text
Context = Redux
Context = database
Context = server cache
Context = universal state solution
```

It solves a specific problem:

> **Making information available to a subtree without threading it through props.**

That's it.

---

# 27. Context vs lifting state

Remember our previous lesson.

Suppose:

```text
App
├── SearchBox
└── ProductList
```

and both need `searchTerm`.

Lifting state up is still appropriate:

```text
App
  searchTerm
 /        \
SearchBox  ProductList
```

You don't need Context.

Context becomes more interesting when:

```text
App
 ↓
many layers
 ↓
many distant components
```

all need the same information.

So:

```text
Two nearby components sharing state
→ lifting state

Many distant descendants needing contextual data
→ Context may help
```

That's a much better rule than:

> "Use Context whenever you have shared state."

---

# 28. Context and rendering

Here's another important detail.

If the Context value changes:

```jsx
<ThemeContext value={theme}>
```

components reading that context will respond to the new value.

That's one reason to avoid stuffing unrelated, frequently changing data into one giant Context value.

For example, if one Context contains:

```text
theme
mousePosition
```

and `mousePosition` changes constantly, you could end up making theme consumers participate in updates they don't need.

We'll study rendering and performance much more deeply later.

For now:

> Context creates a dependency between a consumer and the context value it reads.

---

# 29. The big picture so far

Our React architecture now looks like:

```text
                       App
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
          props       state      context
             │          │          │
             ↓          ↓          ↓
         components   renders   descendants
             │
             ↓
           events
             │
             ↓
        state updates
```

And the tools have distinct jobs:

```text
props
→ explicit parent → child data

state
→ data React remembers and renders from

derived values
→ calculations from existing data

callbacks
→ child can trigger parent-owned behavior

context
→ deeply available subtree data

refs
→ persistent mutable value / DOM reference

effects
→ synchronize with external systems
```

Notice how each tool has a **specific responsibility**.

That's how I want you to learn React rather than memorizing hooks independently.

---

# 30. One complete Context example

Let's build a small theme system.

```jsx
import {
  createContext,
  useContext,
  useState
} from "react";

const ThemeContext =
  createContext(null);

function ThemeProvider({ children }) {
  const [theme, setTheme] =
    useState("light");

  function toggleTheme() {
    setTheme(theme =>
      theme === "light"
        ? "dark"
        : "light"
    );
  }

  return (
    <ThemeContext
      value={{
        theme,
        toggleTheme
      }}
    >
      {children}
    </ThemeContext>
  );
}

function useTheme() {
  const context =
    useContext(ThemeContext);

  if (context === null) {
    throw new Error(
      "useTheme must be used inside ThemeProvider"
    );
  }

  return context;
}

function Toolbar() {
  return (
    <div>
      <ThemeButton />
    </div>
  );
}

function ThemeButton() {
  const {
    theme,
    toggleTheme
  } = useTheme();

  return (
    <button onClick={toggleTheme}>
      Current theme: {theme}
    </button>
  );
}

function App() {
  return (
    <ThemeProvider>
      <Toolbar />
    </ThemeProvider>
  );
}
```

Trace it:

```text
App
 ↓
ThemeProvider
 ↓
ThemeContext value
 ↓
Toolbar
 ↓
ThemeButton
 ↓
useTheme()
 ↓
useContext()
 ↓
theme
```

No:

```jsx
<ThemeButton theme={theme} />
```

was necessary.

---

# 31. But notice who owns the state

This is worth highlighting.

`ThemeContext` does **not** own the state.

This does:

```jsx
const [theme, setTheme] =
  useState("light");
```

inside `ThemeProvider`.

Then:

```jsx
<ThemeContext
  value={{
    theme,
    toggleTheme
  }}
>
```

makes it available to descendants.

So:

```text
ThemeProvider
    │
    ├── owns state
    │
    └── exposes state via Context
```

This is a much clearer mental model.

---

# 32. Context in one sentence

Try to remember this exact idea:

> **Props are explicit data passing. Context is implicit data availability within a subtree.**

And:

> **Context doesn't replace state; it provides a way for descendants to access a value without manually passing it through intermediate components.**

Those two ideas will carry you a long way.

---

# Your practice problem

Imagine this component tree:

```text
App
├── Header
│   └── UserMenu
├── Main
│   └── Dashboard
│       └── AccountSummary
└── Footer
```

The logged-in `user` is needed by:

```text
UserMenu
AccountSummary
```

but not by:

```text
Header
Main
Dashboard
Footer
```

Think about two possible designs:

### Design A

```text
App
 ↓ user
Header
 ↓ user
UserMenu
```

and separately:

```text
App
 ↓ user
Main
 ↓ user
Dashboard
 ↓ user
AccountSummary
```

### Design B

```text
App
 ↓
AuthContext
 ↓
Header → UserMenu
 ↓
Main → Dashboard → AccountSummary
```

You should now be able to explain **why Design B might be useful**, and also why Context shouldn't automatically be used for every prop.

---

## Next lesson — `useReducer`

We're now ready for another major state concept.

So far we've used:

```jsx
useState(...)
```

for state and manually written update functions.

Next we'll encounter situations where the state logic becomes complicated:

```text
loading
success
error
adding
deleting
editing
resetting
```

and several pieces of state depend on the same set of actions.

That's where `useReducer` comes in:

```text
current state
     +
action
     ↓
reducer
     ↓
new state
```

We'll learn what a reducer actually is, why this is basically **normal JavaScript state-transition logic**, and when `useReducer` is genuinely better than `useState`.