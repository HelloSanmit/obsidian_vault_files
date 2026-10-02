# React Course — Exercise Bank · Part 1
## Lessons 1 – 20: Fundamentals → Server State

Each lesson contains:
- **C1, C2** — Conceptual questions (test the *why*, not just the *what*)
- **B1, B2** — Build challenges with complete working code answers
- **D1** — Debug challenge: find and fix the bug(s)
- **A1** — Architecture / decision question

---

## Lesson 1 — Components + JSX

### C1
**Why is `<Greeting />` in JSX fundamentally different from calling `Greeting()` directly in JavaScript, even though both run the same function?**

**Answer:** `<Greeting />` compiles to `React.createElement(Greeting, null)`, which tells React to *manage* this as a node in its component tree. React calls the function itself, tracks the output, associates state and lifecycle with this specific instance, and knows when to re-call it. Calling `Greeting()` directly is a plain JavaScript function invocation — the output is inlined wherever you called it, React has zero awareness of it as a distinct node, it cannot hold independent state, and React cannot re-render it independently. One is a *declaration* to React's runtime; the other is an immediate function call.

### C2
**JSX is often called "HTML inside JavaScript." Why is this misleading? What is JSX actually?**

**Answer:** JSX is syntactic sugar for `React.createElement()` calls — it is a JavaScript expression that compiles before the browser ever sees it. `<h1 className="title">Hello</h1>` compiles to `React.createElement("h1", { className: "title" }, "Hello")`. This explains why: (a) you write `className` instead of `class` — because this maps to the JS DOM property `element.className`, not the HTML attribute; (b) `htmlFor` instead of `for`; (c) JSX expressions can be stored in variables, passed as arguments, and returned from functions. The browser never runs JSX — Babel or SWC transforms it to plain JavaScript at build time.

### B1
**Build a `ProductCard` component that accepts `name` (string), `price` (number), `inStock` (boolean), and `imageUrl` (string). Display the price formatted as `$XX.XX`. When `inStock` is false, show "Out of Stock" instead of the price. Use it three times with different data in `App`.**

```jsx
function ProductCard({ name, price, inStock, imageUrl }) {
  return (
    <div className="product-card">
      <img src={imageUrl} alt={name} />
      <h2>{name}</h2>
      {inStock ? (
        <p className="price">${price.toFixed(2)}</p>
      ) : (
        <p className="out-of-stock">Out of Stock</p>
      )}
    </div>
  );
}

function App() {
  return (
    <div>
      <ProductCard
        name="Wireless Headphones"
        price={79.99}
        inStock={true}
        imageUrl="https://example.com/headphones.jpg"
      />
      <ProductCard
        name="Mechanical Keyboard"
        price={149.99}
        inStock={false}
        imageUrl="https://example.com/keyboard.jpg"
      />
      <ProductCard
        name="USB-C Hub"
        price={39.99}
        inStock={true}
        imageUrl="https://example.com/hub.jpg"
      />
    </div>
  );
}
```

### B2
**Create a `PageLayout` component that accepts `header`, `sidebar`, and `children` as props and assembles them into a page structure. Use it in `App` to wrap content.**

```jsx
function PageLayout({ header, sidebar, children }) {
  return (
    <div className="page-layout">
      <header className="page-header">{header}</header>
      <div className="page-body">
        <aside className="page-sidebar">{sidebar}</aside>
        <main className="page-content">{children}</main>
      </div>
    </div>
  );
}

function App() {
  return (
    <PageLayout
      header={<h1>My App</h1>}
      sidebar={<nav><a href="/home">Home</a><br /><a href="/about">About</a></nav>}
    >
      <p>This is the main content area.</p>
      <p>It is passed as children.</p>
    </PageLayout>
  );
}
```

### D1
**Find and fix every bug in this code.**

```jsx
function userProfile({ name, bio, followerCount }) {
  return (
    <div>
      <h1>name</h1>
      <p>bio</p>
      <span>{followerCount} followers<span>
    </div>
  )
}

function App() {
  return <userProfile name="Alice" bio="Engineer" followerCount=1240 />
}
```

**Bugs:**
1. `userProfile` starts with lowercase — React renders it as a native DOM element `<userprofile>`, not a component
2. `name` and `bio` are missing curly braces — they render the literal strings `"name"` and `"bio"`
3. `<span>` closing tag is missing the slash — should be `</span>`
4. `followerCount=1240` — non-string props need curly braces: `followerCount={1240}`
5. `<userProfile />` in App must also be capitalized

```jsx
// Fixed
function UserProfile({ name, bio, followerCount }) {
  return (
    <div>
      <h1>{name}</h1>
      <p>{bio}</p>
      <span>{followerCount} followers</span>
    </div>
  );
}

function App() {
  return <UserProfile name="Alice" bio="Engineer" followerCount={1240} />;
}
```

### A1
**At what point should you extract a piece of JSX into its own component? What signals tell you it's time?**

**Answer:** Extract when: (1) the same JSX structure repeats with different data — clear componentization signal; (2) a section is large enough that the parent component becomes hard to follow; (3) a piece of UI needs its own state — you can only call `useState` at the component level; (4) the section has a clear, nameable identity (`UserCard`, `PriceTag`, `NavigationBar`). Do *not* extract purely because a file is long — premature componentization adds indirection without benefit. The heuristic: if a section exceeds ~20 lines *or* repeats twice *or* needs its own state, extract it.

---

## Lesson 2 — State: Making React Interactive

### C1
**Why can't you update the UI by mutating a regular JavaScript variable? What specifically does `useState` give you that a plain variable cannot?**

**Answer:** React only re-renders a component (calls the function again) when notified through a `setState` call. Mutating a plain variable does nothing to notify React — the UI never updates. Beyond that, plain variables reset on every render: `let count = 0` starts at zero every time the component function is called. `useState` gives you two things: (1) **external persistence** — React stores the value outside the function, so it survives re-renders; (2) **re-render scheduling** — calling the setter function (`setCount`) tells React "this component's output may have changed, re-render it." Without both, interactive UI is impossible.

### C2
**What does it mean that state updates are asynchronous? Give a concrete example of the bug this causes and how to fix it.**

**Answer:** `setState` schedules a re-render; it does not immediately mutate the value. Within the same event handler, the state variable still holds the value from the current render. Classic bug: calling `setCount(count + 1)` three times in one handler only increments by 1, because all three reads see the same `count`. Fix: use the **functional update form** — `setCount(c => c + 1)`. This receives the *pending* state (result of any previous scheduled updates), not the stale rendered value. Use the functional form any time new state depends on old state.

### B1
**Build a `LikeButton` component. It starts at 0 likes. Clicking it increments the count and changes the label to "❤️ Liked". Clicking again decrements and reverts to "🤍 Like".**

```jsx
import { useState } from 'react';

function LikeButton() {
  const [liked, setLiked] = useState(false);
  const [count, setCount] = useState(0);

  function handleClick() {
    if (liked) {
      setLiked(false);
      setCount(c => c - 1);
    } else {
      setLiked(true);
      setCount(c => c + 1);
    }
  }

  return (
    <button onClick={handleClick}>
      {liked ? '❤️ Liked' : '🤍 Like'} · {count}
    </button>
  );
}
```

### B2
**Build a `TemperatureConverter`. A single number input accepts Celsius. Below it, display the Fahrenheit equivalent that updates on every keystroke. Formula: `F = C × 9/5 + 32`.**

```jsx
import { useState } from 'react';

function TemperatureConverter() {
  const [celsius, setCelsius] = useState('');

  const fahrenheit =
    celsius !== ''
      ? ((parseFloat(celsius) * 9) / 5 + 32).toFixed(1)
      : '';

  return (
    <div>
      <label>
        Celsius:
        <input
          type="number"
          value={celsius}
          onChange={e => setCelsius(e.target.value)}
        />
      </label>
      {fahrenheit !== '' && (
        <p>
          {celsius}°C = {fahrenheit}°F
        </p>
      )}
    </div>
  );
}
```

### D1
**The alert always shows the wrong count. Why? Fix it.**

```jsx
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    alert(`Count is now: ${count}`);
  }

  return <button onClick={handleClick}>Count: {count}</button>;
}
```

**Bug:** `setCount` schedules a re-render but doesn't immediately change `count`. When `alert()` runs, `count` still holds the value from the current render — so the alert always shows the value *before* the click.

```jsx
// Fixed — capture the new value explicitly
function handleClick() {
  const newCount = count + 1;
  setCount(newCount);
  alert(`Count is now: ${newCount}`);
}
```

### A1
**A form has 5 fields. Should you use 5 separate `useState` calls or one object? What are the tradeoffs and when does each make sense?**

**Answer:** Five separate calls are simpler, each setter only touches one field, and they are easy to read in isolation. One object groups related data, makes resetting to initial state trivial (`setForm(initialValues)`), and signals that the data belongs together conceptually. The tradeoff with an object: every update must spread existing state (`setForm(f => ({ ...f, email: value }))`); forgetting the spread wipes other fields. Rule: if the fields are logically unrelated (e.g., `isModalOpen` and `count`), keep them separate. If they represent one coherent piece of data (a form, a user record), use an object — and graduate to `useReducer` when the update logic has multiple interdependent transitions.

---

## Lesson 3 — Events, State, and Forms

### C1
**What is the difference between a controlled and an uncontrolled input? Why does React prefer controlled inputs?**

**Answer:** An **uncontrolled** input stores its own value in the DOM; you read it via a ref when needed. React has no visibility into the value between reads. A **controlled** input has its value driven entirely by React state: `value={email}` and `onChange={e => setEmail(e.target.value)}`. React prefers controlled because: (1) the state is always the single source of truth — you always know what's in the input; (2) you can validate, format, or conditionally show/hide fields based on the value at any point; (3) resetting the form means just resetting state. The cost: a re-render on every keystroke, which is almost always fine.

### C2
**What does `e.preventDefault()` actually prevent, and what happens if you forget it on a form submit handler?**

**Answer:** It prevents the browser's default behavior for the event. For a `<form>` submit, the default behavior is to serialize the form fields into a URL-encoded string and navigate to the `action` URL (or reload the current page if no `action` is set). Forgetting it on `onSubmit` means the page reloads, clearing all React state — your handler runs but is immediately wiped out. For `<a>` tags, it prevents navigation. For drag events, it enables custom drop behavior. The rule: any time you are handling a browser event to replace its default behavior with your own JavaScript, call `e.preventDefault()` first.

### B1
**Build a `SearchBar` with a controlled text input and a "Clear" button. Display the current search term live below the input. Clearing resets the input.**

```jsx
import { useState } from 'react';

function SearchBar() {
  const [query, setQuery] = useState('');

  return (
    <div>
      <input
        type="text"
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <button onClick={() => setQuery('')}>Clear</button>
      {query && (
        <p>
          Searching for: <strong>{query}</strong>
        </p>
      )}
    </div>
  );
}
```

### B2
**Build a `LoginForm` with email and password fields. On submit: prevent default, validate that neither field is empty, and show either a success message or a field-level error message.**

```jsx
import { useState } from 'react';

function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState({});
  const [success, setSuccess] = useState(false);

  function handleSubmit(e) {
    e.preventDefault();
    const newErrors = {};
    if (!email.trim()) newErrors.email = 'Email is required.';
    if (!password.trim()) newErrors.password = 'Password is required.';

    if (Object.keys(newErrors).length > 0) {
      setErrors(newErrors);
      return;
    }

    setErrors({});
    setSuccess(true);
  }

  if (success) return <p>Login successful!</p>;

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label>
          Email:
          <input
            type="email"
            value={email}
            onChange={e => setEmail(e.target.value)}
          />
        </label>
        {errors.email && <span className="error">{errors.email}</span>}
      </div>
      <div>
        <label>
          Password:
          <input
            type="password"
            value={password}
            onChange={e => setPassword(e.target.value)}
          />
        </label>
        {errors.password && <span className="error">{errors.password}</span>}
      </div>
      <button type="submit">Log In</button>
    </form>
  );
}
```

### D1
**The form reloads the page on submit. Fix it.**

```jsx
function ContactForm() {
  const [name, setName] = useState('');

  function handleSubmit() {
    console.log('Submitted:', name);
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={e => setName(e.target.value)} />
      <button type="submit">Submit</button>
    </form>
  );
}
```

**Bug:** `handleSubmit` never receives or calls `e.preventDefault()`, so the browser performs its default form submission (page reload).

```jsx
function handleSubmit(e) {
  e.preventDefault();
  console.log('Submitted:', name);
}
```

### A1
**A multi-step form has 3 steps. How do you architect the state so that navigating back to Step 1 doesn't lose Step 2 and 3 data?**

**Answer:** Hold all form data in a single object at the top-level parent, alongside a `currentStep` integer. Each step component receives its relevant slice as props and an `onChange` callback to update the parent's shared object. This means data persists across step changes because it lives in the parent, not inside each step's local state. On final submission, all data is already in one place. If the update logic grows complex (cross-field validation spanning steps, conditional branching), graduate the form object to `useReducer` with typed actions like `{ type: 'SET_ADDRESS', payload: { street, city } }`.

---

## Lesson 4 — Conditional Rendering, Lists, Keys, and Derived Data

### C1
**Why does React require a unique `key` on list items? Why is using the array index as a key a bug in most real scenarios?**

**Answer:** During reconciliation, React compares the previous and next lists to determine what changed. `key` is how React matches an item in the old list to an item in the new list. With stable unique keys, React correctly reuses DOM nodes and preserves component state. With index-as-key, removing the first item shifts all subsequent indices — React thinks item at index 0 is the same item (same key), but the data underneath it changed, causing stale state bugs (e.g., a checked checkbox appearing on the wrong item). Use index-as-key only for static, append-only lists that are never reordered or filtered.

### C2
**What is "derived data" and why is computing it during render safer than storing it in additional state?**

**Answer:** Derived data is any value computable from existing state or props — it doesn't need its own `useState`. Example: `const completedTasks = tasks.filter(t => t.completed)`. Storing this in state creates two sources of truth that can fall out of sync: if `tasks` changes and you forget to also update `completedTasks`, the UI shows stale derived data. Computing during render is always accurate (uses current state), costs nothing extra (no additional re-render), and reduces state surface area. Rule: if a value can be computed from existing state or props, compute it — don't store it.

### B1
**Build a `TaskList` component that receives `tasks: { id, title, completed }[]`. Render each task with a read-only checkbox. Above the list, show "X of Y tasks completed" using derived data — not stored state.**

```jsx
function TaskList({ tasks }) {
  const completedCount = tasks.filter(t => t.completed).length;

  return (
    <div>
      <p>
        {completedCount} of {tasks.length} tasks completed
      </p>
      <ul>
        {tasks.map(task => (
          <li key={task.id}>
            <input type="checkbox" checked={task.completed} readOnly />
            {task.title}
          </li>
        ))}
      </ul>
    </div>
  );
}

// Usage
const tasks = [
  { id: 1, title: 'Buy groceries', completed: true },
  { id: 2, title: 'Write report', completed: false },
  { id: 3, title: 'Call dentist', completed: true },
];

function App() {
  return <TaskList tasks={tasks} />;
}
```

### B2
**Build a `StatusIndicator` component that accepts `status: "online" | "offline" | "busy"` and renders a colored dot with matching text. Use a config object — not a chain of if/else or ternaries.**

```jsx
const STATUS_CONFIG = {
  online:  { color: '#22c55e', label: 'Online' },
  offline: { color: '#9ca3af', label: 'Offline' },
  busy:    { color: '#f97316', label: 'Busy' },
};

function StatusIndicator({ status }) {
  const config = STATUS_CONFIG[status];
  if (!config) return null;

  return (
    <span style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
      <span
        style={{
          width: 8,
          height: 8,
          borderRadius: '50%',
          backgroundColor: config.color,
        }}
      />
      {config.label}
    </span>
  );
}
```

### D1
**This renders but has a bug that will cause incorrect behavior if the list becomes filterable or sortable. Identify it and fix it.**

```jsx
function FruitList() {
  const fruits = ['Apple', 'Banana', 'Cherry'];
  return (
    <ul>
      {fruits.map((fruit, index) => (
        <li key={index}>{fruit}</li>
      ))}
    </ul>
  );
}
```

**Bug:** Array index as key. If the list is ever sorted, filtered, or items inserted/removed, React will mismatch items to their previous DOM nodes.

```jsx
// Fixed — unique string values are valid keys when values are unique
function FruitList() {
  const fruits = ['Apple', 'Banana', 'Cherry'];
  return (
    <ul>
      {fruits.map(fruit => (
        <li key={fruit}>{fruit}</li>
      ))}
    </ul>
  );
}
```

For real data, always use a stable database `id` field.

### A1
**When should you use `&&` vs a ternary vs an `if` statement for conditional rendering? What is the `0` trap with `&&`?**

**Answer:** Use `&&` for "render this or nothing": `{isLoading && <Spinner />}`. Critical trap: `{count && <Items />}` renders the number `0` to the DOM when `count` is 0, because `0` is falsy but still a renderable value in React. Always use `{count > 0 && <Items />}`. Use a ternary `condition ? <A /> : <B />` when choosing between two options inline in JSX. Use an `if` statement or early return in the component body when logic is complex, you have more than two branches, or nested ternaries become unreadable. A lookup object (like B2 above) is cleanest for 3+ mutually-exclusive variants.

---

## Lesson 5 — Callbacks, Closures, map / filter / find / reduce

### C1
**What is a stale closure in React, and how does it most commonly appear in event handlers or effects?**

**Answer:** A closure captures variables from its outer scope at the time of its creation. In React, every render creates new state values and new closures. If you create a function during one render and it captures `count = 5`, that closure permanently holds `count = 5` even if the component re-renders with `count = 6`. The stale closure problem: if a `setTimeout` callback or an effect cleanup captures a value and runs later, it reads the value from when it was created, not the current render's value. Fix: use the functional updater form (`setCount(c => c + 1)`) for state updates inside closures, or use a `ref` to hold a mutable "always current" reference to a value.

### C2
**Explain what `reduce` does in terms of a mental model — not just the API. When is it genuinely the right tool over `filter` + `map`?**

**Answer:** `reduce` collapses an array into a single accumulated value by running a function against a running accumulator and each item. Mental model: "fold the array from left to right, carrying a result forward." It's the right tool when the output shape is fundamentally different from an array of the same items — e.g., grouping items by category into an object, summing to a number, building a frequency map, or flattening nested arrays. When the output *is* still an array of the same items (just filtered or transformed), `filter` and `map` are clearer and more readable. Using `reduce` to do what `filter(fn).map(fn)` can do is over-engineering.

### B1
**Build a `ProductFilter` component. It receives a `products: { id, name, category, price }[]` array. Render: a dropdown of unique categories, a max-price number input, and a filtered product list. All filtering should be derived — no filtered list in state.**

```jsx
import { useState } from 'react';

function ProductFilter({ products }) {
  const [category, setCategory] = useState('All');
  const [maxPrice, setMaxPrice] = useState('');

  const categories = ['All', ...new Set(products.map(p => p.category))];

  const filtered = products.filter(p => {
    const categoryMatch = category === 'All' || p.category === category;
    const priceMatch = maxPrice === '' || p.price <= parseFloat(maxPrice);
    return categoryMatch && priceMatch;
  });

  return (
    <div>
      <select value={category} onChange={e => setCategory(e.target.value)}>
        {categories.map(c => (
          <option key={c} value={c}>{c}</option>
        ))}
      </select>
      <input
        type="number"
        placeholder="Max price"
        value={maxPrice}
        onChange={e => setMaxPrice(e.target.value)}
      />
      <ul>
        {filtered.map(p => (
          <li key={p.id}>
            {p.name} — ${p.price} ({p.category})
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### B2
**Write a function `groupByCategory(products)` that uses `reduce` to return an object where each key is a category and each value is an array of products in that category. Then render this grouped structure in a component.**

```jsx
function groupByCategory(products) {
  return products.reduce((acc, product) => {
    const { category } = product;
    if (!acc[category]) acc[category] = [];
    acc[category].push(product);
    return acc;
  }, {});
}

function GroupedProducts({ products }) {
  const grouped = groupByCategory(products);

  return (
    <div>
      {Object.entries(grouped).map(([category, items]) => (
        <section key={category}>
          <h2>{category}</h2>
          <ul>
            {items.map(item => (
              <li key={item.id}>{item.name} — ${item.price}</li>
            ))}
          </ul>
        </section>
      ))}
    </div>
  );
}
```

### D1
**The event handler has a stale closure. The alert always shows `0` no matter how many times you've clicked. Fix it.**

```jsx
import { useState, useEffect } from 'react';

function Counter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      console.log('Current count:', count);
    }, 1000);
    return () => clearInterval(timer);
  }, []); // empty deps

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

**Bug:** The effect runs once (empty deps) and its closure captures `count = 0` permanently. Even as `count` updates, the interval logs the stale value.

```jsx
// Fix: use a ref to always have the current value
import { useState, useEffect, useRef } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);
  countRef.current = count; // always up to date

  useEffect(() => {
    const timer = setInterval(() => {
      console.log('Current count:', countRef.current);
    }, 1000);
    return () => clearInterval(timer);
  }, []);

  return (
    <button onClick={() => setCount(c => c + 1)}>
      Count: {count}
    </button>
  );
}
```

### A1
**When should you use `reduce` vs a `for` loop vs a chain of `filter().map()`? Is there a performance consideration?**

**Answer:** For readability and expressiveness: `filter().map()` is clearest when you're filtering then transforming — it reads like plain English. `reduce` is right when the output shape is fundamentally different (object, number, nested structure). A `for` loop is right when you need early exits (break) or when accumulating across multiple conditions in one pass over a very large array, since it avoids creating intermediate arrays. Performance: for typical UI arrays (< 10,000 items), the difference between all three is negligible. The primary decision criterion should be readability. Only optimize with a `for` loop if a profiler actually shows the array operation as a bottleneck — which almost never happens in React UI work.

---

## Lesson 6 — State Ownership, Lifting State Up, and Component Communication

### C1
**What does "lifting state up" mean mechanically, and how do you know when you need to do it?**

**Answer:** Lifting state up means moving a `useState` call from a child component into a parent that is a common ancestor of all components that need to read or change that value. You know you need it when: two sibling components need to share the same piece of data, or a change in one component must be reflected in another. The mechanism: the parent holds the state, passes the current value down as props to children that need to read it, and passes the setter (or a callback wrapping it) down to children that need to change it. The parent becomes the single source of truth; children become controlled.

### C2
**What is the difference between passing state down as props and passing a callback down as a prop? When do you need each?**

**Answer:** Passing state down (`value={count}`) gives a child read access — it can display or use the value but cannot change it. Passing a callback down (`onChange={setCount}`) gives a child write access — it can trigger a state change in the parent without owning the state itself. You need both when a child needs to both display and modify a value (e.g., a controlled input inside a parent form). You need only the value when a child only displays (e.g., a summary that shows the total). You need only the callback when a child triggers an action but doesn't display the result (e.g., a "Reset" button).

### B1
**Build a `TemperatureDashboard` with two sibling components: `CelsiusInput` and `FahrenheitDisplay`. When the user types in `CelsiusInput`, `FahrenheitDisplay` updates in real time. State must live in `TemperatureDashboard`.**

```jsx
import { useState } from 'react';

function CelsiusInput({ celsius, onChange }) {
  return (
    <label>
      Celsius:
      <input
        type="number"
        value={celsius}
        onChange={e => onChange(e.target.value)}
      />
    </label>
  );
}

function FahrenheitDisplay({ celsius }) {
  const fahrenheit =
    celsius !== '' ? ((parseFloat(celsius) * 9) / 5 + 32).toFixed(1) : '—';
  return <p>Fahrenheit: {fahrenheit}</p>;
}

function TemperatureDashboard() {
  const [celsius, setCelsius] = useState('');

  return (
    <div>
      <CelsiusInput celsius={celsius} onChange={setCelsius} />
      <FahrenheitDisplay celsius={celsius} />
    </div>
  );
}
```

### B2
**Build a `RatingSelector` with 5 stars. Clicking a star sets the rating. The parent `ReviewForm` displays the current rating value below. State lives in `ReviewForm`.**

```jsx
import { useState } from 'react';

function RatingSelector({ rating, onRate }) {
  return (
    <div>
      {[1, 2, 3, 4, 5].map(star => (
        <button
          key={star}
          onClick={() => onRate(star)}
          style={{ color: star <= rating ? 'gold' : 'gray', fontSize: 24 }}
        >
          ★
        </button>
      ))}
    </div>
  );
}

function ReviewForm() {
  const [rating, setRating] = useState(0);

  return (
    <div>
      <h2>Leave a Review</h2>
      <RatingSelector rating={rating} onRate={setRating} />
      <p>
        {rating > 0 ? `You selected ${rating} star(s).` : 'No rating yet.'}
      </p>
    </div>
  );
}
```

### D1
**Two siblings both maintain their own `count`. Clicking "Reset All" in the parent does nothing. Fix the architecture.**

```jsx
// Broken — each child owns its own state
function Counter({ label }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <span>{label}: {count}</span>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

function App() {
  function resetAll() {
    // can't reach children's state
  }
  return (
    <div>
      <Counter label="A" />
      <Counter label="B" />
      <button onClick={resetAll}>Reset All</button>
    </div>
  );
}
```

```jsx
// Fixed — lift state to App
import { useState } from 'react';

function Counter({ label, count, onIncrement }) {
  return (
    <div>
      <span>{label}: {count}</span>
      <button onClick={onIncrement}>+</button>
    </div>
  );
}

function App() {
  const [countA, setCountA] = useState(0);
  const [countB, setCountB] = useState(0);

  function resetAll() {
    setCountA(0);
    setCountB(0);
  }

  return (
    <div>
      <Counter label="A" count={countA} onIncrement={() => setCountA(c => c + 1)} />
      <Counter label="B" count={countB} onIncrement={() => setCountB(c => c + 1)} />
      <button onClick={resetAll}>Reset All</button>
    </div>
  );
}
```

### A1
**How high should you lift state? Is lifting to the top-level App component always the right answer?**

**Answer:** State should live at the *lowest common ancestor* of all components that need it — no higher. Lifting unnecessarily high means every component between the state owner and the consumer re-renders on every change, even if they don't use that state. For example, if only a `UserCard` and a `UserBio` sibling need `selectedUser`, the state should live in their shared parent — not in `App`. Colocate state as close to where it's used as possible. Only lift when sharing actually requires it. Global state (user session, theme, cart) is a legitimate exception — that state is consumed by many components across the tree, and lifting it to App or putting it in Context is appropriate.

---

## Lesson 7 — Forms in React, Properly

### C1
**What is the difference between validating `onChange` (on every keystroke) vs. `onBlur` (on field exit) vs. `onSubmit`? When should you use each?**

**Answer:** `onChange` validation gives the fastest feedback but can feel aggressive — showing "email invalid" while the user is still typing "alice@" is annoying. `onBlur` fires when a field loses focus, giving the user space to finish typing before showing an error — better UX for most fields. `onSubmit` validates everything at once when the user explicitly tries to submit, which is appropriate for short, simple forms. Best practice: validate `onBlur` to reveal errors after a field is finished, then re-validate `onChange` only *after* an error has already been shown (so the error clears as the user fixes it). Submit always re-validates everything as a final gate.

### C2
**What happens if you set `value` on an input without also setting `onChange`? What does React's warning tell you?**

**Answer:** React warns that you've created a read-only input. Without `onChange`, the input value is frozen to whatever `value` is set to — the user can type but nothing changes because the state never updates, so React re-renders with the same value each time. React's warning: "You provided a `value` prop to a form field without an `onChange` handler. This will render a read-only field." Either add `onChange` to make it fully controlled, add `readOnly` to explicitly acknowledge it's read-only, or switch to `defaultValue` for an uncontrolled input where React doesn't manage the value at all.

### B1
**Build a `RegistrationForm` with fields: name, email, password, confirmPassword. Validate on blur: name required, valid email format, password min 8 chars, confirmPassword must match password. Show inline errors. Disable the submit button until all fields are valid.**

```jsx
import { useState } from 'react';

const initialValues = { name: '', email: '', password: '', confirmPassword: '' };
const initialErrors = { name: '', email: '', password: '', confirmPassword: '' };

function validate(field, values) {
  switch (field) {
    case 'name':
      return values.name.trim() ? '' : 'Name is required.';
    case 'email':
      return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(values.email)
        ? ''
        : 'Enter a valid email.';
    case 'password':
      return values.password.length >= 8
        ? ''
        : 'Password must be at least 8 characters.';
    case 'confirmPassword':
      return values.confirmPassword === values.password
        ? ''
        : 'Passwords do not match.';
    default:
      return '';
  }
}

function RegistrationForm() {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState(initialErrors);
  const [submitted, setSubmitted] = useState(false);

  const allValid = Object.keys(values).every(
    field => validate(field, values) === ''
  );

  function handleChange(field, value) {
    const updated = { ...values, [field]: value };
    setValues(updated);
    if (errors[field]) {
      setErrors(e => ({ ...e, [field]: validate(field, updated) }));
    }
  }

  function handleBlur(field) {
    setErrors(e => ({ ...e, [field]: validate(field, values) }));
  }

  function handleSubmit(e) {
    e.preventDefault();
    const allErrors = Object.keys(values).reduce(
      (acc, field) => ({ ...acc, [field]: validate(field, values) }),
      {}
    );
    setErrors(allErrors);
    if (Object.values(allErrors).every(err => err === '')) {
      setSubmitted(true);
    }
  }

  if (submitted) return <p>Registration successful!</p>;

  return (
    <form onSubmit={handleSubmit}>
      {['name', 'email', 'password', 'confirmPassword'].map(field => (
        <div key={field}>
          <label>
            {field.charAt(0).toUpperCase() + field.slice(1)}:
            <input
              type={field.includes('assword') ? 'password' : 'text'}
              value={values[field]}
              onChange={e => handleChange(field, e.target.value)}
              onBlur={() => handleBlur(field)}
            />
          </label>
          {errors[field] && <span style={{ color: 'red' }}>{errors[field]}</span>}
        </div>
      ))}
      <button type="submit" disabled={!allValid}>Register</button>
    </form>
  );
}
```

### B2
**Build a `CheckboxGroup` that accepts `options: { id, label }[]` and a `selected: string[]` and an `onChange: (selected: string[]) => void`. Checking/unchecking an option calls `onChange` with the updated array. The parent stores the selected array in state.**

```jsx
import { useState } from 'react';

function CheckboxGroup({ options, selected, onChange }) {
  function handleToggle(id) {
    if (selected.includes(id)) {
      onChange(selected.filter(s => s !== id));
    } else {
      onChange([...selected, id]);
    }
  }

  return (
    <div>
      {options.map(option => (
        <label key={option.id} style={{ display: 'block' }}>
          <input
            type="checkbox"
            checked={selected.includes(option.id)}
            onChange={() => handleToggle(option.id)}
          />
          {option.label}
        </label>
      ))}
    </div>
  );
}

function App() {
  const [selected, setSelected] = useState([]);
  const options = [
    { id: 'react', label: 'React' },
    { id: 'vue', label: 'Vue' },
    { id: 'svelte', label: 'Svelte' },
  ];

  return (
    <div>
      <h2>Select Frameworks</h2>
      <CheckboxGroup options={options} selected={selected} onChange={setSelected} />
      <p>Selected: {selected.join(', ') || 'none'}</p>
    </div>
  );
}
```

### D1
**The user cannot type in this input. Why? Fix it.**

```jsx
import { useState } from 'react';

function NameInput() {
  const [name, setName] = useState('Alice');

  return (
    <input
      value={name}
      onchange={e => setName(e.target.value)}
    />
  );
}
```

**Bugs:**
1. `onchange` should be `onChange` (camelCase in React JSX)
2. Because `onChange` is misspelled, it never fires, making the input read-only at `"Alice"`

```jsx
// Fixed
<input
  value={name}
  onChange={e => setName(e.target.value)}
/>
```

### A1
**When should you use a form library (React Hook Form, Formik) vs. manual controlled inputs? What is the crossover point?**

**Answer:** Manual controlled inputs are perfectly appropriate for forms with ≤ 4–5 fields with simple validation. The code is transparent, no dependency, and learnable. The crossover point is when you need: (1) complex async validation (server-side checks); (2) dynamic field arrays (add/remove rows); (3) deeply nested field structures; (4) performance at scale (React Hook Form uses uncontrolled inputs under the hood, avoiding re-renders on every keystroke — significant for very large forms). React Hook Form in particular adds almost zero abstraction overhead and vastly reduces boilerplate for complex forms. Formik adds more abstraction and is more opinionated. Choose a library when manual state management of the form requires more code than the form library would.

---

## Lesson 8 — `useEffect`: What an Effect Actually Is

### C1
**What does the dependency array in `useEffect` actually tell React? What happens with no array, `[]`, and `[value]`?**

**Answer:** The dependency array tells React when to re-run the effect after renders. **No array**: the effect runs after every render — use this almost never. **Empty `[]`**: the effect runs once after the first render and never again — equivalent to "on mount." **`[value]`**: the effect runs after the first render, then after any render where `value` changed (by reference equality for objects/arrays, by value for primitives). The intent of the dependency array is not performance optimization — it's correctness. You list every reactive value the effect uses; leaving one out means the effect will run with a stale value of it, not an updated one.

### C2
**What is a cleanup function, when is it called, and why do effects that subscribe to external systems almost always need one?**

**Answer:** A cleanup function is returned from a `useEffect` callback: `return () => { ... }`. React calls it in two situations: (1) before re-running the effect (when dependencies change); (2) when the component unmounts. Without cleanup, subscriptions, timers, and event listeners accumulate on every render, causing memory leaks and multiple duplicate handlers. Example: if a component subscribes to a WebSocket on mount without cleanup, and the component mounts/unmounts 10 times, you'll have 10 active WebSocket connections. The cleanup should undo exactly what the effect set up: `clearInterval`, `removeEventListener`, `socket.close()`, `subscription.unsubscribe()`.

### B1
**Build a `UserProfile` component that fetches `https://jsonplaceholder.typicode.com/users/{id}` when `id` changes. Handle loading and error states.**

```jsx
import { useState, useEffect } from 'react';

function UserProfile({ id }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);
    setError(null);

    fetch(`https://jsonplaceholder.typicode.com/users/${id}`)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setUser(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => { cancelled = true; };
  }, [id]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  if (!user) return null;

  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
}
```

### B2
**Build a `WindowSize` component that displays the current window width and height, updating in real time as the window is resized. Make sure the event listener is cleaned up.**

```jsx
import { useState, useEffect } from 'react';

function WindowSize() {
  const [size, setSize] = useState({
    width: window.innerWidth,
    height: window.innerHeight,
  });

  useEffect(() => {
    function handleResize() {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    }

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return (
    <p>
      Window: {size.width} × {size.height}
    </p>
  );
}
```

### D1
**This effect causes an infinite loop. Why? Fix it.**

```jsx
import { useState, useEffect } from 'react';

function App() {
  const [data, setData] = useState([]);

  useEffect(() => {
    fetch('https://api.example.com/items')
      .then(res => res.json())
      .then(json => setData(json));
  }, [data]); // data in deps

  return <ul>{data.map(d => <li key={d.id}>{d.name}</li>)}</ul>;
}
```

**Bug:** `data` is listed as a dependency, but the effect sets `data`. So: effect runs → sets `data` → `data` changes → effect runs again → infinite loop.

```jsx
// Fixed — empty deps because this should only run once on mount
useEffect(() => {
  fetch('https://api.example.com/items')
    .then(res => res.json())
    .then(json => setData(json));
}, []); // no dependency on data
```

### A1
**What are the cases where you should NOT use `useEffect`, even though it seems like the right tool?**

**Answer:** Do not use `useEffect` for: (1) **Deriving data** — compute it during render instead; (2) **Resetting state when props change** — use a `key` prop to reset the component, or restructure so derived state isn't needed; (3) **Responding to user events** — that belongs in event handlers; (4) **Chaining state updates** — avoid effects that watch state and set more state (causes waterfalls of re-renders); (5) **Data fetching in general** — use TanStack Query or SWR for real applications. `useEffect` is specifically for synchronizing with external systems (timers, subscriptions, third-party DOM libraries). The React team's own documentation now explicitly warns against using effects for data transformation or event handling.

---

## Lesson 9 — `useRef`: Values That Persist Without Re-rendering

### C1
**What is the critical difference between `useRef` and `useState`? When does each apply?**

**Answer:** Both persist values across renders. The fundamental difference: changing a `ref` value does **not** trigger a re-render. Changing state **does**. This makes refs the right tool for values that: (1) need to persist but whose changes should not affect the UI; (2) are needed synchronously without waiting for a render (e.g., accessing a DOM node imperatively); (3) hold mutable values used in closures (like interval IDs, timeout IDs, or "has this effect run before" flags). State is for values whose changes should be reflected in the UI. If you catch yourself thinking "I need to remember this value but I don't want a re-render," that's a `useRef`.

### C2
**What happens if you access `ref.current` before the component has mounted? What is the safe pattern?**

**Answer:** Before mount, `ref.current` is `null` (for DOM refs) or the initial value you passed to `useRef`. Accessing `ref.current.focus()` before mount throws `Cannot read properties of null`. The safe pattern: access DOM refs only inside `useEffect` (which runs after mount) or inside event handlers (which can only fire after the UI is rendered). The ref object itself is stable (same object reference across renders), but `ref.current` is only populated after React has committed the DOM node.

### B1
**Build a `FocusableInput` component with a text input and a "Focus Input" button. Clicking the button programmatically focuses the input using a ref.**

```jsx
import { useRef } from 'react';

function FocusableInput() {
  const inputRef = useRef(null);

  function handleFocus() {
    inputRef.current.focus();
  }

  return (
    <div>
      <input ref={inputRef} type="text" placeholder="Type here..." />
      <button onClick={handleFocus}>Focus Input</button>
    </div>
  );
}
```

### B2
**Build a `PreviousValue` component that displays the current count and the previous count. Use a ref to track the previous value without causing extra re-renders.**

```jsx
import { useState, useEffect, useRef } from 'react';

function usePrevious(value) {
  const ref = useRef(undefined);
  useEffect(() => {
    ref.current = value;
  }, [value]);
  return ref.current;
}

function PreviousValue() {
  const [count, setCount] = useState(0);
  const previous = usePrevious(count);

  return (
    <div>
      <p>Current: {count}</p>
      <p>Previous: {previous ?? '—'}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
    </div>
  );
}
```

### D1
**Why does this not scroll to the bottom correctly? Fix it.**

```jsx
import { useState, useRef } from 'react';

function MessageList() {
  const [messages, setMessages] = useState(['Hello']);
  const bottomRef = useRef(null);

  function addMessage() {
    setMessages(m => [...m, `Message ${m.length + 1}`]);
    bottomRef.current.scrollIntoView(); // runs before DOM updates
  }

  return (
    <div>
      {messages.map((m, i) => <p key={i}>{m}</p>)}
      <div ref={bottomRef} />
      <button onClick={addMessage}>Add</button>
    </div>
  );
}
```

**Bug:** `scrollIntoView()` is called synchronously inside the event handler, before React has committed the new message to the DOM. The scroll happens before the new message exists.

```jsx
// Fixed — use useEffect to run after DOM updates
import { useState, useRef, useEffect } from 'react';

function MessageList() {
  const [messages, setMessages] = useState(['Hello']);
  const bottomRef = useRef(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  return (
    <div>
      {messages.map((m, i) => <p key={i}>{m}</p>)}
      <div ref={bottomRef} />
      <button onClick={() => setMessages(m => [...m, `Message ${m.length + 1}`])}>
        Add
      </button>
    </div>
  );
}
```

### A1
**A junior engineer suggests putting a loading boolean in a ref instead of state to "avoid unnecessary re-renders." Is this correct? When is this a bug?**

**Answer:** This is a bug whenever the loading state affects what the UI renders. If `isLoading` is in a ref and you set `isLoadingRef.current = true`, the component does not re-render — the UI continues showing stale content (no spinner, no disabled button). The whole point of "loading" from a UX perspective is to change what's shown. The rule: if a value change should cause a UI update, it must be in state. Use refs for values where the change itself is not the trigger for a re-render — tracking whether an effect has run before, storing interval IDs, keeping the latest version of a callback. When a junior engineer says "avoid re-renders" as a motivation for choosing a ref, that's almost always a misdiagnosis of a performance problem that doesn't exist.

---

## Lesson 10 — Context: Sharing Data Without Prop Drilling

### C1
**What problem does Context solve, and what problem does it NOT solve?**

**Answer:** Context solves prop drilling — the need to pass props through many intermediate components that don't use them just to reach a deeply nested consumer. Context makes a value available to any component in a subtree without explicit passing. What it does NOT solve: performance. When a Context value changes, every component that calls `useContext` for that context re-renders, regardless of whether the specific part of the value they use changed. Context is for infrequently-changing, globally-relevant values (theme, current user, language). For high-frequency state updates or complex state shared between many components, a dedicated state library (Zustand, Jotai) uses subscriptions to re-render only the components that use the changed slice.

### C2
**If `useContext` is called in a component but that component is not inside the matching `Provider`, what does React return?**

**Answer:** React returns the `defaultValue` you passed to `createContext()`. If you passed no default value, it returns `undefined`. This is a common source of bugs: `const { user } = useContext(UserContext)` throws `Cannot destructure property 'user' of undefined` if the consumer is accidentally rendered outside the provider tree. Best practice: always provide a meaningful default or throw an explicit error inside the context: if the component requires the context, add a guard: `if (!ctx) throw new Error("useUserContext must be used inside UserProvider")`.

### B1
**Build a `ThemeContext` that provides `theme` ("light" or "dark") and a `toggleTheme` function. A `ThemeToggle` button and a `ThemedCard` component should both consume it from different places in the tree.**

```jsx
import { createContext, useContext, useState } from 'react';

const ThemeContext = createContext(null);

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  function toggleTheme() {
    setTheme(t => (t === 'light' ? 'dark' : 'light'));
  }
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used inside ThemeProvider');
  return ctx;
}

function ThemeToggle() {
  const { theme, toggleTheme } = useTheme();
  return (
    <button onClick={toggleTheme}>
      Switch to {theme === 'light' ? 'dark' : 'light'} mode
    </button>
  );
}

function ThemedCard({ children }) {
  const { theme } = useTheme();
  return (
    <div style={{
      background: theme === 'dark' ? '#1e1e1e' : '#ffffff',
      color: theme === 'dark' ? '#ffffff' : '#000000',
      padding: 16,
      borderRadius: 8,
    }}>
      {children}
    </div>
  );
}

function App() {
  return (
    <ThemeProvider>
      <ThemeToggle />
      <ThemedCard>
        <p>This card responds to the theme.</p>
      </ThemedCard>
    </ThemeProvider>
  );
}
```

### B2
**Build a `CurrentUserContext` that provides the logged-in user object. A `NavBar` and a `ProfilePage` both consume it. Add a guard so both components throw clearly if used outside the provider.**

```jsx
import { createContext, useContext, useState } from 'react';

const CurrentUserContext = createContext(null);

function CurrentUserProvider({ children }) {
  const [user] = useState({ id: 1, name: 'Alice Chen', role: 'admin' });
  return (
    <CurrentUserContext.Provider value={user}>
      {children}
    </CurrentUserContext.Provider>
  );
}

function useCurrentUser() {
  const ctx = useContext(CurrentUserContext);
  if (!ctx) throw new Error('useCurrentUser must be used inside CurrentUserProvider');
  return ctx;
}

function NavBar() {
  const user = useCurrentUser();
  return <nav>Welcome, {user.name}</nav>;
}

function ProfilePage() {
  const user = useCurrentUser();
  return (
    <div>
      <h1>{user.name}</h1>
      <p>Role: {user.role}</p>
    </div>
  );
}

function App() {
  return (
    <CurrentUserProvider>
      <NavBar />
      <ProfilePage />
    </CurrentUserProvider>
  );
}
```

### D1
**`useContext` returns `undefined`. Why? Fix it.**

```jsx
const SettingsContext = createContext({ darkMode: false });

function App() {
  return (
    <div>
      <SettingsContext value={{ darkMode: true }}>
        <Settings />
      </SettingsContext>
    </div>
  );
}

function Settings() {
  const settings = useContext(SettingsContext);
  return <p>Dark mode: {settings.darkMode ? 'on' : 'off'}</p>;
}
```

**Bug:** `<SettingsContext value={...}>` is wrong. You must use `<SettingsContext.Provider value={...}>` to provide values.

```jsx
// Fixed
function App() {
  return (
    <SettingsContext.Provider value={{ darkMode: true }}>
      <Settings />
    </SettingsContext.Provider>
  );
}
```

### A1
**When should you choose Context over prop drilling, and when should you choose a library like Zustand over Context?**

**Answer:** Choose Context over prop drilling when a value is truly global to a subtree and changes infrequently — theme, current user, locale, feature flags. Context is built in, zero-dependency, and perfectly adequate for these. Choose Zustand (or Jotai, Recoil) over Context when: (1) the shared state changes frequently and you need components to re-render only when the specific slice they use changes (Context re-renders all consumers); (2) state transitions are complex and need to be managed outside the React tree; (3) you need time-travel debugging (Redux DevTools); (4) the state is accessed by many components with different subscriptions. The deciding question: "How often does this change, and how many components care about different parts of it?" Frequent changes + partial subscriptions = library. Infrequent changes + whole value consumption = Context.

---

## Lesson 11 — `useReducer`: Managing More Complex State

### C1
**When should you choose `useReducer` over `useState`? What are the signals in your code?**

**Answer:** Choose `useReducer` when: (1) multiple state values are updated together in coordinated ways — a reducer makes the valid transitions explicit; (2) the next state depends on the previous state in complex ways (multiple fields, conditional logic); (3) you have many different ways to update the same state (many different event handlers all calling `setState` in different ways — reducers name these as typed actions); (4) you want to test state logic independently from the component (a reducer is a pure function — trivially testable); (5) you need to share the dispatch function deeply without creating many different callbacks. `useState` is simpler and correct for independent, simple values.

### C2
**A reducer must be a pure function. What does this mean practically, and what are the two most common violations?**

**Answer:** Pure means: given the same inputs, always return the same output, with no side effects. Practically: the reducer must not mutate the current state object/array, must not make network requests or API calls, must not read from external mutable state (like `Date.now()` for a generated ID — pass it in the action payload instead). The two most common violations: (1) **Direct mutation** — `state.items.push(newItem); return state` — this mutates the existing array, which React may not detect as a change; always return a new object/array: `return { ...state, items: [...state.items, newItem] }`; (2) **Side effects in the reducer** — placing a `fetch()` call inside a reducer case, which breaks React's concurrent rendering guarantees.

### B1
**Build a shopping cart with `useReducer`. Actions: `ADD_ITEM`, `REMOVE_ITEM`, `UPDATE_QUANTITY`, `CLEAR_CART`. Render the cart with item names, quantities, subtotals, and the cart total.**

```jsx
import { useReducer } from 'react';

const initialState = { items: [] };

function cartReducer(state, action) {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(i => i.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map(i =>
            i.id === action.payload.id
              ? { ...i, quantity: i.quantity + 1 }
              : i
          ),
        };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
      };
    }
    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.payload),
      };
    case 'UPDATE_QUANTITY':
      return {
        ...state,
        items: state.items.map(i =>
          i.id === action.payload.id
            ? { ...i, quantity: action.payload.quantity }
            : i
        ),
      };
    case 'CLEAR_CART':
      return initialState;
    default:
      return state;
  }
}

function Cart() {
  const [state, dispatch] = useReducer(cartReducer, initialState);

  const total = state.items.reduce((sum, i) => sum + i.price * i.quantity, 0);

  const sampleProducts = [
    { id: 1, name: 'Laptop', price: 999 },
    { id: 2, name: 'Mouse', price: 29 },
  ];

  return (
    <div>
      <h2>Shop</h2>
      {sampleProducts.map(p => (
        <button key={p.id} onClick={() => dispatch({ type: 'ADD_ITEM', payload: p })}>
          Add {p.name} (${p.price})
        </button>
      ))}

      <h2>Cart</h2>
      {state.items.length === 0 && <p>Cart is empty.</p>}
      {state.items.map(item => (
        <div key={item.id}>
          {item.name} × {item.quantity} = ${(item.price * item.quantity).toFixed(2)}
          <button onClick={() => dispatch({ type: 'REMOVE_ITEM', payload: item.id })}>
            Remove
          </button>
        </div>
      ))}
      {state.items.length > 0 && (
        <>
          <p>Total: ${total.toFixed(2)}</p>
          <button onClick={() => dispatch({ type: 'CLEAR_CART' })}>Clear Cart</button>
        </>
      )}
    </div>
  );
}
```

### B2
**Write unit tests (pure function tests, no React) for the `cartReducer` above. Test all four action types.**

```js
// cartReducer.test.js
import { cartReducer } from './cartReducer';

const initial = { items: [] };
const itemA = { id: 1, name: 'Laptop', price: 999 };
const itemB = { id: 2, name: 'Mouse', price: 29 };

test('ADD_ITEM adds a new item with quantity 1', () => {
  const state = cartReducer(initial, { type: 'ADD_ITEM', payload: itemA });
  expect(state.items).toHaveLength(1);
  expect(state.items[0]).toMatchObject({ ...itemA, quantity: 1 });
});

test('ADD_ITEM increments quantity for existing item', () => {
  const withOne = cartReducer(initial, { type: 'ADD_ITEM', payload: itemA });
  const withTwo = cartReducer(withOne, { type: 'ADD_ITEM', payload: itemA });
  expect(withTwo.items[0].quantity).toBe(2);
});

test('REMOVE_ITEM removes the correct item', () => {
  const withBoth = [itemA, itemB].reduce(
    (s, p) => cartReducer(s, { type: 'ADD_ITEM', payload: p }),
    initial
  );
  const afterRemove = cartReducer(withBoth, { type: 'REMOVE_ITEM', payload: 1 });
  expect(afterRemove.items).toHaveLength(1);
  expect(afterRemove.items[0].id).toBe(2);
});

test('CLEAR_CART empties the cart', () => {
  const withItem = cartReducer(initial, { type: 'ADD_ITEM', payload: itemA });
  const cleared = cartReducer(withItem, { type: 'CLEAR_CART' });
  expect(cleared.items).toHaveLength(0);
});
```

### D1
**The cart item count is always one higher than expected after removing an item. Find the mutation bug.**

```jsx
function badCartReducer(state, action) {
  switch (action.type) {
    case 'REMOVE_ITEM': {
      const index = state.items.findIndex(i => i.id === action.payload);
      state.items.splice(index, 1); // mutation!
      return { ...state };
    }
    default:
      return state;
  }
}
```

**Bug:** `state.items.splice()` mutates the original array in place. Even though you spread `state` into a new object, `items` is still the same (now mutated) array reference. React may detect this as no change in some scenarios, or it creates subtle reference bugs.

```jsx
// Fixed — never mutate, always return new arrays
case 'REMOVE_ITEM':
  return {
    ...state,
    items: state.items.filter(i => i.id !== action.payload),
  };
```

### A1
**Can you use `useReducer` as a replacement for a state management library? What are the limits?**

**Answer:** `useReducer` with Context can replace libraries for simple global state scenarios — it gives you typed actions, predictable transitions, and a dispatch function that passes through Context without prop drilling. The limits: (1) No selector subscriptions — all Context consumers re-render when any part of the state changes; (2) No middleware (no logger, no async action support out of the box); (3) No DevTools integration (Redux DevTools won't see your reducer); (4) Context + Reducer doesn't scale cleanly to many independent state slices across a large app. For small-to-medium apps with limited global state, it's entirely adequate. For large apps with complex state and performance-sensitive updates, a library like Zustand (simple) or Redux Toolkit (structured, team conventions) is the right call.

---

## Lesson 12 — Custom Hooks

### C1
**What is the rule that defines whether a function is a "hook"? What happens if you break the Rules of Hooks?**

**Answer:** A function is a hook if it calls other hooks inside it. The Rules of Hooks: (1) only call hooks at the top level — never inside conditions, loops, or nested functions; (2) only call hooks inside React function components or other custom hooks. The reason: React tracks hooks by their order of call in each render. If a hook is called conditionally, the order can change between renders, causing React to associate the wrong state to the wrong hook. Breaking this causes either a cryptic error ("Rendered more hooks than during the previous render") or, worse, silently incorrect behavior where state is mismatched.

### C2
**A custom hook returns a value. That value is computed using `useState` and `useEffect` inside the hook. Does the component that calls this hook re-render when the hook's internal state changes?**

**Answer:** Yes. Custom hooks do *not* create isolated state — they are a mechanism to share *logic*, not *state instances*. When you call `useCount()` in two different components, each component gets its own independent copy of the state — they are not shared. When the state inside a custom hook updates, the component that called the hook re-renders, because the hook's state is actually owned by that component. This is the key mental model: a custom hook is syntactic reuse of hook logic, not a separate state store.

### B1
**Build a `useLocalStorage(key, initialValue)` custom hook that reads from and writes to `localStorage`, staying in sync with React state.**

```jsx
import { useState, useEffect } from 'react';

function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const stored = window.localStorage.getItem(key);
      return stored !== null ? JSON.parse(stored) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(value));
    } catch {
      // ignore write errors
    }
  }, [key, value]);

  return [value, setValue];
}

// Usage
function Settings() {
  const [theme, setTheme] = useLocalStorage('theme', 'light');

  return (
    <div>
      <p>Current theme: {theme}</p>
      <button onClick={() => setTheme(t => (t === 'light' ? 'dark' : 'light'))}>
        Toggle
      </button>
    </div>
  );
}
```

### B2
**Build a `useDebounce(value, delay)` custom hook that returns the debounced version of a value, only updating after `delay` ms of inactivity.**

```jsx
import { useState, useEffect } from 'react';

function useDebounce(value, delay) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
function SearchWithDebounce() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 400);

  // Only fire search when debouncedQuery changes (400ms after user stops typing)
  useEffect(() => {
    if (debouncedQuery) {
      console.log('Searching for:', debouncedQuery);
    }
  }, [debouncedQuery]);

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
      placeholder="Type to search..."
    />
  );
}
```

### D1
**This custom hook violates the Rules of Hooks. Find the violation and explain why it will crash.**

```jsx
function useConditionalFetch(enabled, url) {
  if (!enabled) return null;

  const [data, setData] = useState(null);

  useEffect(() => {
    fetch(url).then(r => r.json()).then(setData);
  }, [url]);

  return data;
}
```

**Bug:** `useState` and `useEffect` are called conditionally — they only run if `enabled` is truthy. If `enabled` starts as `false`, React registers 0 hooks. If `enabled` later becomes `true`, React tries to register 2 hooks, causing "Rendered more hooks than during the previous render."

```jsx
// Fixed — hooks must always run unconditionally
function useConditionalFetch(enabled, url) {
  const [data, setData] = useState(null);

  useEffect(() => {
    if (!enabled) return;
    fetch(url).then(r => r.json()).then(setData);
  }, [enabled, url]);

  return enabled ? data : null;
}
```

### A1
**When should you extract a custom hook vs. just keeping the logic inside the component?**

**Answer:** Extract a custom hook when: (1) the same combination of hooks (`useState` + `useEffect` + `useRef`) appears in more than one component — the classic "don't repeat yourself" signal; (2) the stateful logic has a clear, nameable responsibility (`useWindowSize`, `useAuth`, `useFormField`); (3) you want to test the stateful logic independently from rendering. Do not extract prematurely — a hook that's only used in one place and has no conceptual identity beyond "logic from ComponentX" adds indirection without benefit. The name is the test: if you can give the hook a clear, domain-meaningful name that makes sense to another engineer, it's a good extraction candidate.

---

## Lesson 13 — How React Actually Renders

### C1
**What is the difference between "rendering" and "committing" in React's processing model?**

**Answer:** **Rendering** is React calling your component function(s) and computing what the UI *should* look like — it produces a description (virtual DOM / fiber tree), not actual DOM changes. **Committing** is React taking that description and applying the minimum set of DOM mutations to make the actual DOM match. Rendering can be interrupted (in concurrent mode), discarded, or re-run; the commit phase is synchronous and cannot be interrupted. This is why you must not cause side effects during rendering — they may run multiple times or be discarded. Side effects belong in `useEffect` (after commit) or in event handlers.

### C2
**Why does React sometimes preserve state when a component re-renders and sometimes reset it? What determines this?**

**Answer:** React preserves state as long as the component at a given position in the tree has the same *type*. If a component's type or its position in the JSX tree changes between renders, React unmounts the old instance (losing its state) and mounts a new one. Classic pattern: `{isLoggedIn ? <UserDashboard /> : <GuestView />}` — switching between the two discards state because the component type changes at that position. To force a reset even when the type is the same, use a `key` prop — changing the key tells React to treat the component as a different instance. `key` is the mechanism for intentionally resetting state.

### B1
**Demonstrate key-based state reset. Build two `Counter` components that share the same position in the tree. Add a "Switch User" button. When clicked, the counter should reset to 0 (not carry over the previous user's count).**

```jsx
import { useState } from 'react';

function Counter({ user }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <p>{user}'s count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

function App() {
  const [userId, setUserId] = useState(1);

  return (
    <div>
      {/* key={userId} forces a new Counter instance for each user */}
      <Counter key={userId} user={`User ${userId}`} />
      <button onClick={() => setUserId(id => (id === 1 ? 2 : 1))}>
        Switch User
      </button>
    </div>
  );
}
```

Without `key={userId}`, React reuses the same Counter instance and state is preserved across users.

### B2
**Show that a parent re-render causes all children to re-render. Build a component tree with a parent counter and two children. Add `console.log` to each. Then use `React.memo` to stop one child from re-rendering unnecessarily.**

```jsx
import { useState, memo } from 'react';

function RegularChild({ label }) {
  console.log(`${label} rendered`);
  return <p>{label}</p>;
}

const MemoizedChild = memo(function MemoizedChild({ label }) {
  console.log(`${label} rendered`);
  return <p>{label} (memoized)</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  console.log('Parent rendered');

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <RegularChild label="Regular Child" />      {/* re-renders every time */}
      <MemoizedChild label="Memoized Child" />    {/* re-renders only if props change */}
    </div>
  );
}
```

### D1
**A form input loses focus every time the user types a character. Why? Fix it.**

```jsx
import { useState } from 'react';

function App() {
  const [value, setValue] = useState('');

  function Input() {
    return (
      <input value={value} onChange={e => setValue(e.target.value)} />
    );
  }

  return (
    <div>
      <Input />
    </div>
  );
}
```

**Bug:** `Input` is defined *inside* `App`. On every render of `App`, `Input` is a new function — a new component type. React sees a new type at the same position and unmounts/remounts it, which clears focus.

```jsx
// Fixed — define Input outside App so its type is stable
function Input({ value, onChange }) {
  return <input value={value} onChange={onChange} />;
}

function App() {
  const [value, setValue] = useState('');
  return <Input value={value} onChange={e => setValue(e.target.value)} />;
}
```

### A1
**What does the Virtual DOM actually mean, and is it the reason React is "fast"?**

**Answer:** The "Virtual DOM" is React's in-memory representation of the UI (the fiber tree). On each render, React produces a new representation and diffs it against the previous one (reconciliation) to find the minimum set of real DOM mutations. React is not inherently faster than vanilla DOM manipulation — in fact, direct DOM manipulation can be faster for very specific targeted updates. React's value is in *correctness and developer productivity* — it guarantees that your UI is always consistent with your state without you manually tracking which DOM nodes to update. The Virtual DOM is a technique for making the *declarative programming model* efficient enough for practical use, not a magic performance booster.

---

## Lesson 14 — Component Composition and Better Prop Design

### C1
**What is the difference between component composition and component inheritance? Why does React favor composition?**

**Answer:** Inheritance means a component extends another to gain its behavior (as you might in class-based OOP). Composition means building complex components from simpler ones by nesting and combining them. React favors composition because: (1) it is more explicit — you can see exactly what is being combined; (2) it avoids tight coupling between components; (3) it naturally supports the `children` prop and "slot" patterns that let the parent control layout while children control content; (4) it maps directly to how HTML elements compose (a `<div>` doesn't inherit from `<button>`, it wraps it). The React team explicitly states: if you're considering inheritance to share UI, use composition instead.

### C2
**What is "prop drilling" and what are the two solutions? When is each appropriate?**

**Answer:** Prop drilling is passing props through intermediate components that don't need the props themselves — they just forward them to a child deeper in the tree. Solution 1: **Component composition / children prop** — instead of drilling `user` through three components, restructure so the component that *needs* `user` is composed closer to where `user` lives, avoiding the intermediate components entirely. Solution 2: **Context** — make `user` available to any component in the tree without passing it explicitly. Composition is the first choice: it's simpler, more explicit, and doesn't add a new abstraction. Context is appropriate when restructuring the component tree to avoid drilling would require significant refactoring or when many unrelated components at different tree levels need the same value.

### B1
**Build a `Tabs` compound component with `Tabs`, `TabList`, `Tab`, and `TabPanel` sub-components. The selected tab state should live in `Tabs` and be shared with children via Context (not props).**

```jsx
import { createContext, useContext, useState } from 'react';

const TabsContext = createContext(null);

function Tabs({ children, defaultTab = 0 }) {
  const [activeTab, setActiveTab] = useState(defaultTab);
  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  );
}

function TabList({ children }) {
  return <div role="tablist" className="tab-list">{children}</div>;
}

function Tab({ index, children }) {
  const { activeTab, setActiveTab } = useContext(TabsContext);
  return (
    <button
      role="tab"
      aria-selected={activeTab === index}
      onClick={() => setActiveTab(index)}
      style={{ fontWeight: activeTab === index ? 'bold' : 'normal' }}
    >
      {children}
    </button>
  );
}

function TabPanel({ index, children }) {
  const { activeTab } = useContext(TabsContext);
  if (activeTab !== index) return null;
  return <div role="tabpanel">{children}</div>;
}

// Usage
function App() {
  return (
    <Tabs defaultTab={0}>
      <TabList>
        <Tab index={0}>Overview</Tab>
        <Tab index={1}>Details</Tab>
        <Tab index={2}>Settings</Tab>
      </TabList>
      <TabPanel index={0}><p>Overview content</p></TabPanel>
      <TabPanel index={1}><p>Details content</p></TabPanel>
      <TabPanel index={2}><p>Settings content</p></TabPanel>
    </Tabs>
  );
}
```

### B2
**Refactor this prop-drilled component tree to use composition instead. `user` should not pass through `Dashboard`.**

```jsx
// Before — prop drilling
function App() {
  const user = { name: 'Alice' };
  return <Dashboard user={user} />;
}

function Dashboard({ user }) {
  return <Sidebar user={user} />;
}

function Sidebar({ user }) {
  return <UserBadge user={user} />;
}

function UserBadge({ user }) {
  return <p>Hello, {user.name}</p>;
}
```

```jsx
// After — composition
function App() {
  const user = { name: 'Alice' };
  return (
    <Dashboard>
      <Sidebar>
        <UserBadge user={user} />
      </Sidebar>
    </Dashboard>
  );
}

function Dashboard({ children }) {
  return <div className="dashboard">{children}</div>;
}

function Sidebar({ children }) {
  return <aside className="sidebar">{children}</aside>;
}

function UserBadge({ user }) {
  return <p>Hello, {user.name}</p>;
}
```

`Dashboard` and `Sidebar` no longer know about `user` — they just pass children through.

### D1
**This component API is difficult to use correctly and will cause problems. Redesign the props.**

```jsx
// Hard to use
function Alert({ type1, type2, message, showClose, closeFn, isVisible, style1 }) {
  if (!isVisible) return null;
  return (
    <div className={`alert alert-${type1 || type2}`} style={style1}>
      {message}
      {showClose && <button onClick={closeFn}>×</button>}
    </div>
  );
}
```

**Problems:** `type1`/`type2` are redundant; `style1` is an odd name; `showClose`/`closeFn` should be derived from one prop.

```jsx
// Redesigned — clean, consistent API
function Alert({ variant = 'info', message, onClose, style }) {
  return (
    <div className={`alert alert--${variant}`} style={style}>
      {message}
      {onClose && (
        <button onClick={onClose} aria-label="Close alert">×</button>
      )}
    </div>
  );
}

// Usage — if onClose is provided, the close button appears
<Alert variant="success" message="Saved!" onClose={() => setAlertVisible(false)} />
<Alert variant="error" message="Something went wrong." />
```

### A1
**What is the "render prop" pattern and when would you still use it in modern React?**

**Answer:** A render prop is a prop whose value is a function that returns JSX — the component calls it to render part of its output, passing data the component controls. Example: `<DataFetcher url="/api" render={data => <UserList users={data} />} />`. The component manages fetching; the render prop controls the display. In modern React, custom hooks have largely replaced render props for logic reuse — `useDataFetcher(url)` is cleaner than a `<DataFetcher render={...} />` wrapper. Where render props still appear: component libraries that need to give consumers control over rendering a specific slot (virtualized list rows, dropdown item rendering) where the library owns the container logic but the consumer controls the item appearance. It remains a valid pattern for "inversion of control" scenarios.

---

## Lesson 15 — Routing with React Router

### C1
**What is client-side routing and how is it fundamentally different from traditional server-based routing?**

**Answer:** Traditional (server) routing: every URL change sends a request to the server, which returns a complete new HTML page. The browser tears down and rebuilds the entire page, losing all JavaScript state. Client-side routing: a JavaScript library (React Router) intercepts navigation, updates the URL using the History API (`pushState`), and swaps the rendered components — without any server round-trip or full page reload. The server only serves the initial HTML/JS bundle once. The result: faster page transitions (no full reload), preserved application state between routes, and the ability to animate route transitions. The tradeoff: the server must be configured to serve the same `index.html` for all routes, or deep-links will 404 on refresh.

### C2
**What does `<Outlet />` do and why is it essential for nested routes?**

**Answer:** `<Outlet />` is a placeholder that renders the matched child route's component inside a parent layout component. Without it, parent layout components couldn't embed child content — they'd either render nothing or have to know about every possible child. With nested routes, the parent (`/dashboard`) renders a layout (sidebar, navbar) and uses `<Outlet />` where the child content should appear — React Router replaces `<Outlet />` with whichever child route is active (`/dashboard/settings`, `/dashboard/profile`, etc.). This is what enables layouts that persist across route changes: the layout component stays mounted; only the `<Outlet />` content changes.

### B1
**Build a 3-page app with routes `/`, `/about`, and `/contact`. Include a persistent navigation bar on every page with `<Link>` tags. Each page shows unique content.**

```jsx
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';

function NavBar() {
  return (
    <nav>
      <Link to="/">Home</Link> |{' '}
      <Link to="/about">About</Link> |{' '}
      <Link to="/contact">Contact</Link>
    </nav>
  );
}

function Home() {
  return <><NavBar /><h1>Home</h1><p>Welcome to the homepage.</p></>;
}

function About() {
  return <><NavBar /><h1>About</h1><p>About this application.</p></>;
}

function Contact() {
  return <><NavBar /><h1>Contact</h1><p>Get in touch with us.</p></>;
}

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/contact" element={<Contact />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### B2
**Build a nested route structure with a `DashboardLayout` (always visible sidebar) and child routes `/dashboard/overview` and `/dashboard/settings`. Use `<Outlet />`.**

```jsx
import { BrowserRouter, Routes, Route, Link, Outlet } from 'react-router-dom';

function DashboardLayout() {
  return (
    <div style={{ display: 'flex' }}>
      <aside style={{ width: 200 }}>
        <nav>
          <Link to="/dashboard/overview">Overview</Link><br />
          <Link to="/dashboard/settings">Settings</Link>
        </nav>
      </aside>
      <main style={{ flex: 1, padding: 16 }}>
        <Outlet /> {/* child route renders here */}
      </main>
    </div>
  );
}

function Overview() {
  return <h2>Dashboard Overview</h2>;
}

function Settings() {
  return <h2>Dashboard Settings</h2>;
}

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/dashboard" element={<DashboardLayout />}>
          <Route path="overview" element={<Overview />} />
          <Route path="settings" element={<Settings />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}
```

### D1
**Navigation doesn't work and the page reloads on every link click. Why?**

```jsx
function NavBar() {
  return (
    <nav>
      <a href="/home">Home</a>
      <a href="/about">About</a>
    </nav>
  );
}
```

**Bug:** Native `<a href>` tags trigger full browser navigation (page reload), bypassing React Router's client-side routing entirely.

```jsx
// Fixed — use React Router's Link component
import { Link } from 'react-router-dom';

function NavBar() {
  return (
    <nav>
      <Link to="/home">Home</Link>
      <Link to="/about">About</Link>
    </nav>
  );
}
```

### A1
**When should you use flat routes vs. nested routes? What is the cost of deep nesting?**

**Answer:** Use nested routes when routes share a persistent layout — a sidebar, header, or tab bar that should remain mounted across child route changes. The parent route provides the layout; child routes provide the content. Use flat routes when each route has a completely independent layout with no shared persistent UI elements. The cost of deep nesting: (1) URL path construction becomes more complex to reason about; (2) loader data access in v6.4+ requires understanding which loader owns which data at which nesting level; (3) `Outlet` chains can become hard to trace. Rule: only nest routes when they genuinely share a layout component that you want to stay mounted. Never nest routes just because the URL segments nest — that's not a sufficient reason.

---

## Lesson 16 — URL State, Search Params, and Route Params

### C1
**What is the difference between a route param (`:id`) and a search param (`?filter=active`)? When should each be used?**

**Answer:** A **route param** (`/users/:id`) identifies a specific resource — it is part of the resource's identity. `/users/42` and `/users/43` are fundamentally different resources. Route params should be used when you're navigating to a specific entity. A **search param** (`/users?filter=active&page=2`) represents optional view state on top of the same resource or page — filters, sorting, pagination, view modes. Search params are easy to add/remove without restructuring the route hierarchy, they can be combined freely, and they are preserved across navigation as part of the URL. The test: "Would different values here represent different resources, or different views of the same resource?" Resource identity → route param. View state → search param.

### C2
**What is the advantage of storing filter/sort/pagination state in the URL rather than in `useState`?**

**Answer:** URL state is inherently shareable, bookmarkable, and survives page refresh. If pagination is in `useState`, refreshing the page resets to page 1 — a poor experience if the user was on page 7. With URL state (`?page=7`), the page state survives. Additionally, the browser's back/forward buttons work correctly — pressing back returns to the previous filter state. Sharing a link gives the recipient the exact same view. The tradeoff: URL state is a string, so you must serialize/deserialize (numbers become strings, arrays need a convention). Not all state belongs in the URL — transient UI state (is a dropdown open) should stay in `useState`.

### B1
**Build a `ProductList` with URL-driven filter and sort. Use `useSearchParams` to read and write the `category` and `sort` params. The list should filter without any `useState` — all view state lives in the URL.**

```jsx
import { useSearchParams } from 'react-router-dom';

const products = [
  { id: 1, name: 'Laptop', category: 'Electronics', price: 999 },
  { id: 2, name: 'Shirt', category: 'Clothing', price: 29 },
  { id: 3, name: 'Phone', category: 'Electronics', price: 699 },
  { id: 4, name: 'Pants', category: 'Clothing', price: 59 },
];

function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get('category') || 'All';
  const sort = searchParams.get('sort') || 'name';

  const filtered = products
    .filter(p => category === 'All' || p.category === category)
    .sort((a, b) =>
      sort === 'price' ? a.price - b.price : a.name.localeCompare(b.name)
    );

  function updateParam(key, value) {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      next.set(key, value);
      return next;
    });
  }

  return (
    <div>
      <select
        value={category}
        onChange={e => updateParam('category', e.target.value)}
      >
        {['All', 'Electronics', 'Clothing'].map(c => (
          <option key={c} value={c}>{c}</option>
        ))}
      </select>

      <select value={sort} onChange={e => updateParam('sort', e.target.value)}>
        <option value="name">Sort by Name</option>
        <option value="price">Sort by Price</option>
      </select>

      <ul>
        {filtered.map(p => (
          <li key={p.id}>{p.name} — ${p.price}</li>
        ))}
      </ul>
    </div>
  );
}
```

### B2
**Build a `UserDetailPage` that reads a `:userId` route param and displays "Viewing user #X". Include a link from a `UserListPage`.**

```jsx
import { BrowserRouter, Routes, Route, Link, useParams } from 'react-router-dom';

const users = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
  { id: 3, name: 'Carol' },
];

function UserListPage() {
  return (
    <div>
      <h1>Users</h1>
      <ul>
        {users.map(u => (
          <li key={u.id}>
            <Link to={`/users/${u.id}`}>{u.name}</Link>
          </li>
        ))}
      </ul>
    </div>
  );
}

function UserDetailPage() {
  const { userId } = useParams();
  const user = users.find(u => u.id === Number(userId));

  if (!user) return <p>User not found.</p>;

  return (
    <div>
      <h1>Viewing user #{userId}</h1>
      <p>Name: {user.name}</p>
      <Link to="/users">Back to list</Link>
    </div>
  );
}

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/users" element={<UserListPage />} />
        <Route path="/users/:userId" element={<UserDetailPage />} />
      </Routes>
    </BrowserRouter>
  );
}
```

### D1
**Search params are lost when navigating to the detail page and back. Fix it.**

```jsx
function updateFilter(value) {
  setSearchParams({ filter: value }); // replaces ALL params
}
```

**Bug:** Passing a plain object to `setSearchParams` replaces all existing search params, not just the one you're updating.

```jsx
// Fixed — update one param while preserving others
function updateFilter(value) {
  setSearchParams(prev => {
    const next = new URLSearchParams(prev);
    next.set('filter', value);
    return next;
  });
}
```

### A1
**What state should live in the URL vs. in `useState` vs. in a Context or global store?**

**Answer:** URL: anything the user should be able to bookmark, share, or return to after refresh — filters, pagination, selected tab, search query, view mode. `useState` (local): transient UI state that matters only for the current session and has no meaning outside the component — is a dropdown open, which item is being hovered, animation state. Context/global store: cross-cutting state that many components need simultaneously but that doesn't belong in the URL — authentication session, shopping cart contents, theme. The URL is underused in most React apps. If a user would be frustrated by losing the state on refresh, put it in the URL.

---

## Lesson 17 — Data Fetching in React

### C1
**Why is fetching data directly inside `useEffect` considered a problematic pattern in production applications, even if it works?**

**Answer:** It works but creates a need to manually rebuild the same infrastructure every time: loading state, error state, race condition handling, cancellation, caching, deduplication (multiple components requesting the same data triggering multiple fetches), refetching on window focus, stale data management, background refresh, and pagination. Each of these requires additional code. `useEffect` + `useState` for fetching is a low-level primitive that must be wrapped in significant machinery before it's production-ready. Libraries like TanStack Query solve all of these in a tested, optimized way. For learning and simple cases, it's appropriate — for real applications, it's a maintenance liability.

### C2
**What is a race condition in data fetching, and how does the cleanup function solve it?**

**Answer:** A race condition occurs when two requests are in flight simultaneously and the responses arrive out of order. Example: user selects "US", request A fires. User quickly selects "UK", request B fires. If B resolves before A, the UI briefly shows UK data — then A resolves and overwrites with stale US data. The cleanup function solves this with a cancelled flag: when the effect re-runs (user changes selection), the cleanup sets `cancelled = true`. The previous request's `.then()` checks this flag before calling `setState` and does nothing if cancelled. The current request's flag starts as `false`, so it sets state normally. This pattern ensures only the most recent request can update state.

### B1
**Build a `PostList` component that fetches from `https://jsonplaceholder.typicode.com/posts?userId={userId}` whenever `userId` changes. Show a loading state, an error state, and the list of post titles. Handle race conditions.**

```jsx
import { useState, useEffect } from 'react';

function PostList({ userId }) {
  const [posts, setPosts] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;

    async function fetchPosts() {
      setLoading(true);
      setError(null);
      try {
        const res = await fetch(
          `https://jsonplaceholder.typicode.com/posts?userId=${userId}`
        );
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        if (!cancelled) {
          setPosts(data);
          setLoading(false);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      }
    }

    fetchPosts();
    return () => { cancelled = true; };
  }, [userId]);

  if (loading) return <p>Loading posts...</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}
```

### B2
**Build a `useFetch(url)` custom hook that abstracts the loading/error/data pattern and handles cancellation. Any component should be able to use it.**

```jsx
import { useState, useEffect } from 'react';

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!url) return;
    let cancelled = false;
    setLoading(true);

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(json => {
        if (!cancelled) { setData(json); setLoading(false); }
      })
      .catch(err => {
        if (!cancelled) { setError(err.message); setLoading(false); }
      });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage
function TodoItem({ id }) {
  const { data: todo, loading, error } = useFetch(
    `https://jsonplaceholder.typicode.com/todos/${id}`
  );

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <p>{todo?.title}</p>;
}
```

### D1
**This component makes two network requests when it should make one. Why? Fix it.**

```jsx
function UserPosts({ userId }) {
  const [posts, setPosts] = useState([]);

  useEffect(() => {
    fetch(`/api/posts?user=${userId}`)
      .then(r => r.json())
      .then(setPosts);
  });

  // ...
}
```

**Bug:** The `useEffect` has no dependency array — it runs after *every* render. The fetch sets `posts`, which triggers a re-render, which runs the effect again — infinite fetch loop.

```jsx
// Fixed — add userId as the dependency
useEffect(() => {
  fetch(`/api/posts?user=${userId}`)
    .then(r => r.json())
    .then(setPosts);
}, [userId]); // only re-fetch when userId changes
```

### A1
**Is it acceptable to fetch data in a `useEffect` directly in a production React application, or should you always use a library?**

**Answer:** For simple cases — a component that fetches once on mount with no caching needs, deduplication, or background refresh — `useEffect` is acceptable and keeps the dependency footprint small. For anything more complex — paginated lists, data that multiple components share, data that should refresh on window focus, optimistic mutations, cache invalidation — the manual `useEffect` approach requires you to rebuild TanStack Query's feature set yourself, with bugs. The pragmatic line: if the same data is fetched in more than one component, or if you need any of caching/refetching/background refresh/deduplication, reach for TanStack Query. Use `useEffect` only for truly isolated, one-off fetches with no cross-component sharing requirements.

---

## Lesson 18 — Server State and TanStack Query

### C1
**What is the fundamental difference between UI (client) state and server state? Why does treating server state like UI state cause problems?**

**Answer:** UI state is data your application owns and controls entirely — `isModalOpen`, `selectedTab`, form values. It lives on the client, you decide when it changes, and it has no external authoritative version. Server state is data that lives on a backend and your application only has a cached copy of — `users`, `products`, `orders`. The authoritative version is on the server, it can change at any time without your knowledge, and multiple clients may be modifying it simultaneously. Treating server state like UI state (just `useState`) fails because: the data goes stale immediately; you have no built-in mechanism to know when to refresh; multiple components that need the same data each fetch it independently; and there's no deduplication.

### C2
**What are the four pieces of complexity TanStack Query handles that you would have to build yourself with `useEffect`?**

**Answer:** (1) **Caching** — if two components request the same data (`queryKey`), only one network request fires; both get the same cached result; (2) **Background refetching** — on window focus or network reconnect, stale data is silently refreshed without showing a loading state; (3) **Stale-while-revalidate** — show the cached (possibly stale) data immediately while revalidating in the background, giving users instant perceived performance; (4) **Deduplication** — multiple simultaneous calls to `useQuery` with the same key result in a single request. Additionally: retry logic, garbage collection of unused cache entries, optimistic updates, and pagination utilities — all built in.

### B1
**Convert this manual fetch pattern to use TanStack Query's `useQuery`.**

```jsx
// Before
function UserList() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/users')
      .then(r => r.json())
      .then(data => { setUsers(data); setLoading(false); })
      .catch(err => { setError(err.message); setLoading(false); });
  }, []);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

```jsx
// After — TanStack Query
import { useQuery } from '@tanstack/react-query';

async function fetchUsers() {
  const res = await fetch('/api/users');
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

function UserList() {
  const { data: users, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: fetchUsers,
  });

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <ul>
      {users.map(u => <li key={u.id}>{u.name}</li>)}
    </ul>
  );
}

// App must be wrapped in QueryClientProvider
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <UserList />
    </QueryClientProvider>
  );
}
```

### B2
**Build a `UserSearch` component with a text input. Use `useQuery` with the `enabled` option to only fetch when the user has typed at least 2 characters. Use the search term as part of the query key.**

```jsx
import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';

async function searchUsers(query) {
  const res = await fetch(`/api/users?search=${query}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}

function UserSearch() {
  const [query, setQuery] = useState('');

  const { data: results, isLoading, isFetching } = useQuery({
    queryKey: ['users', 'search', query],
    queryFn: () => searchUsers(query),
    enabled: query.length >= 2,
  });

  return (
    <div>
      <input
        value={query}
        onChange={e => setQuery(e.target.value)}
        placeholder="Search users (min 2 chars)..."
      />
      {isFetching && <p>Searching...</p>}
      {results && (
        <ul>
          {results.map(u => <li key={u.id}>{u.name}</li>)}
        </ul>
      )}
      {results?.length === 0 && <p>No results found.</p>}
    </div>
  );
}
```

### D1
**The query keeps refetching every second even when nothing changed. Why? Fix it.**

```jsx
function PostList() {
  const { data } = useQuery({
    queryKey: ['posts', { page: 1, filter: {} }], // object literal in key
    queryFn: fetchPosts,
  });
  return <div>{/* ... */}</div>;
}
```

**Bug:** `{ page: 1, filter: {} }` is a new object literal on every render — even though its contents are the same, it is a different reference each time. TanStack Query compares query keys by deep equality, so this should actually be fine. However, if a component re-renders and rebuilds the filter object with new variable references (from props or state), the key changes and triggers a refetch.

```jsx
// Correct approach — keep the key stable using primitives or stable references
const { data } = useQuery({
  queryKey: ['posts', 1, 'all'], // primitives — always stable
  queryFn: fetchPosts,
});

// Or if dynamic:
const { data } = useQuery({
  queryKey: ['posts', page, filter], // where page and filter are state values, not inline objects
  queryFn: () => fetchPosts({ page, filter }),
});
```

### A1
**`staleTime` vs. `gcTime` (formerly `cacheTime`) — what does each control and what are good values for different use cases?**

**Answer:** `staleTime` controls how long data is considered fresh — during this window, TanStack Query will not refetch in the background. Default is 0 (immediately stale). `gcTime` controls how long unused cache entries are kept in memory before being garbage collected. For user profile data that rarely changes: `staleTime: 5 * 60 * 1000` (5 min) is appropriate — no background refresh for 5 minutes. For real-time price data: `staleTime: 0` with `refetchInterval: 30000`. For static reference data (countries list, config): `staleTime: Infinity` — never refetch, just use what's cached. `gcTime` is usually fine at default (5 min) unless you want to keep data available longer after a component unmounts.

---

## Lesson 19 — Advanced Server Data: Parallel, Dependent, Pagination

### C1
**What is the difference between parallel queries and dependent (sequential) queries? Give a use case for each.**

**Answer:** **Parallel queries** fire simultaneously and independently. Use when: fetching user profile AND their notifications AND their recent activity — none depends on the others; firing them in parallel reduces total wait time to the slowest individual request. **Dependent (sequential) queries** require data from a previous query. Use when: first fetch the user to get their `organizationId`, then fetch the organization's details using that ID. The second query cannot start until the first completes. TanStack Query handles dependent queries via the `enabled` option: `enabled: !!userId` — the query only runs when `userId` is truthy (i.e., after the first query returns it).

### C2
**What is the `keepPreviousData` option (now `placeholderData: keepPreviousData`) in TanStack Query and why is it essential for a good pagination UX?**

**Answer:** Without it, navigating to the next page causes `isLoading: true` and the previous page's results disappear, replaced by a loading spinner — jarring UX. With `keepPreviousData`, the previous page's results remain visible while the next page is being fetched. `isFetching` is `true` (you can show a subtle loading indicator), but `data` still holds the previous page — the UI doesn't flash. This is the "stale-while-revalidate" pattern applied to pagination. After the new page resolves, data updates. Essential for any paginated table or list.

### B1
**Build a `UserDashboard` that fetches both user info and user posts in parallel using multiple `useQuery` calls.**

```jsx
import { useQuery } from '@tanstack/react-query';

async function fetchUser(id) {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error('Failed to fetch user');
  return res.json();
}

async function fetchUserPosts(userId) {
  const res = await fetch(`/api/posts?userId=${userId}`);
  if (!res.ok) throw new Error('Failed to fetch posts');
  return res.json();
}

function UserDashboard({ userId }) {
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
  });

  const postsQuery = useQuery({
    queryKey: ['posts', 'user', userId],
    queryFn: () => fetchUserPosts(userId),
  });

  if (userQuery.isLoading || postsQuery.isLoading) return <p>Loading...</p>;
  if (userQuery.error) return <p>Error loading user: {userQuery.error.message}</p>;
  if (postsQuery.error) return <p>Error loading posts: {postsQuery.error.message}</p>;

  return (
    <div>
      <h1>{userQuery.data.name}</h1>
      <p>{userQuery.data.email}</p>
      <h2>Posts ({postsQuery.data.length})</h2>
      <ul>
        {postsQuery.data.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### B2
**Build a dependent query: first fetch a user, then use the user's `teamId` to fetch team details. The team query must not run until the user query succeeds.**

```jsx
import { useQuery } from '@tanstack/react-query';

function TeamDetails({ userId }) {
  const userQuery = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
  });

  const teamQuery = useQuery({
    queryKey: ['team', userQuery.data?.teamId],
    queryFn: () =>
      fetch(`/api/teams/${userQuery.data.teamId}`).then(r => r.json()),
    enabled: !!userQuery.data?.teamId, // only runs when teamId is available
  });

  if (userQuery.isLoading) return <p>Loading user...</p>;
  if (userQuery.error) return <p>Error: {userQuery.error.message}</p>;

  return (
    <div>
      <h2>User: {userQuery.data.name}</h2>
      {teamQuery.isLoading && <p>Loading team...</p>}
      {teamQuery.data && (
        <p>Team: {teamQuery.data.name}</p>
      )}
    </div>
  );
}
```

### D1
**The dependent query runs immediately and returns an error because `teamId` is undefined at first render. Fix it.**

```jsx
const teamQuery = useQuery({
  queryKey: ['team', user?.teamId],
  queryFn: () => fetch(`/api/teams/${user.teamId}`).then(r => r.json()),
  // missing enabled
});
```

```jsx
// Fixed — guard with enabled
const teamQuery = useQuery({
  queryKey: ['team', user?.teamId],
  queryFn: () => fetch(`/api/teams/${user.teamId}`).then(r => r.json()),
  enabled: !!user?.teamId,
});
```

### A1
**What is the tradeoff between using `useQueries` (parallel array of queries) vs. calling `useQuery` multiple times?**

**Answer:** Calling `useQuery` multiple times is simpler and appropriate when the number of queries is fixed and known at compile time — you always need user + posts + notifications. `useQueries` is necessary when the number of queries is dynamic — you need to fetch details for an unknown number of items (e.g., N selected products whose count varies). `useQueries` accepts an array of query configs and returns an array of query results, which can be mapped over. The limitation of calling hooks in a loop ("you can't call hooks conditionally") is exactly why `useQueries` exists — it lets you declare a variable number of queries without violating the Rules of Hooks. Use multiple `useQuery` for fixed parallel queries; use `useQueries` for dynamic ones.

---

## Lesson 20 — Mutations: Changing Server State

### C1
**What does `useMutation` do that a plain `async function` called in an event handler doesn't?**

**Answer:** `useMutation` gives you a stateful wrapper around an async operation: it tracks `isPending` (request in flight), `isError`, `isSuccess`, `error`, and `data` — and re-renders the component when these states change. Without it, you'd maintain these as separate `useState` calls. More importantly, `useMutation` provides lifecycle callbacks (`onSuccess`, `onError`, `onSettled`) that integrate with TanStack Query's cache — you call `queryClient.invalidateQueries(['posts'])` in `onSuccess` to tell TanStack Query that the cached posts data is now stale and needs refetching. A plain `async function` in an event handler has no connection to the query cache at all.

### C2
**What is an optimistic update and what are the two things that must happen for it to be safe?**

**Answer:** An optimistic update immediately applies the expected result of a mutation to the UI before the server responds — making the interface feel instant. The two requirements for safety: (1) **Rollback on error** — if the server returns an error, the UI must revert to the previous state. TanStack Query's `onMutate` / `onError` pattern handles this: save the previous cache value in `onMutate`, restore it in `onError`; (2) **Final sync on settlement** — call `invalidateQueries` in `onSettled` to replace the optimistic value with the server's actual response, catching any discrepancy (e.g., the server assigned a different ID than what you assumed).

### B1
**Build a `TodoApp` that uses `useMutation` to add new todos. Show a loading state on the button while the mutation is in flight. Invalidate the todos query on success.**

```jsx
import { useState } from 'react';
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

async function fetchTodos() {
  const res = await fetch('/api/todos');
  return res.json();
}

async function createTodo(title) {
  const res = await fetch('/api/todos', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title, completed: false }),
  });
  if (!res.ok) throw new Error('Failed to create todo');
  return res.json();
}

function TodoApp() {
  const queryClient = useQueryClient();
  const [title, setTitle] = useState('');

  const { data: todos, isLoading } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos,
  });

  const addTodo = useMutation({
    mutationFn: createTodo,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
      setTitle('');
    },
  });

  function handleSubmit(e) {
    e.preventDefault();
    if (!title.trim()) return;
    addTodo.mutate(title);
  }

  if (isLoading) return <p>Loading...</p>;

  return (
    <div>
      <form onSubmit={handleSubmit}>
        <input
          value={title}
          onChange={e => setTitle(e.target.value)}
          placeholder="New todo..."
        />
        <button type="submit" disabled={addTodo.isPending}>
          {addTodo.isPending ? 'Adding...' : 'Add Todo'}
        </button>
      </form>
      {addTodo.isError && <p style={{ color: 'red' }}>{addTodo.error.message}</p>}
      <ul>
        {todos?.map(todo => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### B2
**Add optimistic updates to the `TodoApp` above: the new todo appears immediately in the list, and is removed if the mutation fails.**

```jsx
const addTodo = useMutation({
  mutationFn: createTodo,
  onMutate: async (title) => {
    // Cancel any outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos'] });

    // Save current state for rollback
    const previousTodos = queryClient.getQueryData(['todos']);

    // Optimistically add the todo
    queryClient.setQueryData(['todos'], old => [
      ...(old || []),
      { id: Date.now(), title, completed: false }, // temporary ID
    ]);

    return { previousTodos };
  },
  onError: (err, title, context) => {
    // Roll back to previous state
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    // Always sync with server truth
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

### D1
**The list doesn't update after adding a new item. Why?**

```jsx
const addItem = useMutation({
  mutationFn: postNewItem,
  onSuccess: (data) => {
    console.log('Created:', data);
    // nothing else
  },
});
```

**Bug:** `onSuccess` logs but never invalidates the query or updates the cache. The query cache still holds the old list and won't refetch.

```jsx
const queryClient = useQueryClient();

const addItem = useMutation({
  mutationFn: postNewItem,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['items'] });
  },
});
```

### A1
**When should you manually update the cache with `setQueryData` vs. invalidating and refetching?**

**Answer:** **Invalidate and refetch** when you don't know exactly what the server returned and want the authoritative version. This is always safe but costs a network round-trip. Use it as the default for most mutations. **Manually update the cache** (`setQueryData`) when: (1) the mutation returns the updated object and you can update the cache without an extra fetch (e.g., a `PUT /users/1` that returns the updated user — you can set the `['user', 1]` cache directly); (2) you are doing optimistic updates and need instant UI response; (3) the list is large and refetching the entire list for a single update would be wasteful. Always call `invalidateQueries` in `onSettled` even when you manually set data — to catch server-side discrepancies (timestamps, computed fields, ID generation) that your optimistic value didn't account for.
