# React Course — Exercise Bank · Part 2
## Lessons 21 – 40: Advanced Patterns → Performance → Modern React

Each lesson: **C1, C2** (conceptual) · **B1, B2** (build challenges) · **D1** (debug) · **A1** (architecture)

---

## Lesson 21 — Keeping Server Data Consistent

### C1
**What is the difference between invalidating a query and directly updating the cache? What are the correctness tradeoffs?**

**Answer:** Invalidating (`queryClient.invalidateQueries`) marks the cached data as stale and triggers a background refetch — you always get the server's authoritative response. Directly updating (`queryClient.setQueryData`) immediately sets the cache to a value you compute locally, without a network round-trip. The correctness tradeoff: direct updates are instant but can diverge from server truth — if the server applied additional transformations (computed fields, server-generated timestamps, ID assignment), your locally set value won't reflect them. The safe pattern: do both — `setQueryData` for instant UI feedback, then `invalidateQueries` in `onSettled` to sync the server's actual response afterward.

### C2
**You update a user's name on a detail page. Why does the list page still show the old name? What is the query key structure that would fix this without two separate network requests?**

**Answer:** The detail page likely has queryKey `['user', userId]` and the list page has `['users']` — two separate cache entries. Updating one does not automatically update the other. Two solutions: (1) After the mutation's `onSuccess`, call `invalidateQueries({ queryKey: ['users'] })` — this refetches the list; (2) Design your query key hierarchy so that `queryClient.invalidateQueries({ queryKey: ['users'] })` matches both `['users']` (the list) and `['users', userId]` (the detail) because TanStack Query does prefix matching. The convention: `['users']` for list, `['users', id]` for detail — invalidating `['users']` invalidates all user-related cache entries including detail pages.

### B1
**Build a `UserManager` with a list of users and an inline edit form. On save, manually update the detail cache AND invalidate the list query.**

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useState } from 'react';

async function fetchUsers() {
  const res = await fetch('/api/users');
  return res.json();
}

async function updateUser(user) {
  const res = await fetch(`/api/users/${user.id}`, {
    method: 'PUT',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(user),
  });
  if (!res.ok) throw new Error('Update failed');
  return res.json();
}

function UserManager() {
  const queryClient = useQueryClient();
  const [editing, setEditing] = useState(null);
  const [draftName, setDraftName] = useState('');

  const { data: users } = useQuery({ queryKey: ['users'], queryFn: fetchUsers });

  const editUser = useMutation({
    mutationFn: updateUser,
    onSuccess: (updatedUser) => {
      // Update the detail cache immediately
      queryClient.setQueryData(['users', updatedUser.id], updatedUser);
      // Invalidate the list to sync
      queryClient.invalidateQueries({ queryKey: ['users'] });
      setEditing(null);
    },
  });

  function startEdit(user) {
    setEditing(user.id);
    setDraftName(user.name);
  }

  return (
    <ul>
      {users?.map(user => (
        <li key={user.id}>
          {editing === user.id ? (
            <>
              <input value={draftName} onChange={e => setDraftName(e.target.value)} />
              <button
                onClick={() => editUser.mutate({ ...user, name: draftName })}
                disabled={editUser.isPending}
              >
                Save
              </button>
              <button onClick={() => setEditing(null)}>Cancel</button>
            </>
          ) : (
            <>
              {user.name}
              <button onClick={() => startEdit(user)}>Edit</button>
            </>
          )}
        </li>
      ))}
    </ul>
  );
}
```

### B2
**Design and implement a query key factory pattern for a `posts` resource that cleanly handles list, detail, and filtered views — and can be invalidated at different granularities.**

```jsx
// Query key factory — the single source of truth for all post-related keys
const postKeys = {
  all: ['posts'],
  lists: () => [...postKeys.all, 'list'],
  list: (filters) => [...postKeys.lists(), { filters }],
  details: () => [...postKeys.all, 'detail'],
  detail: (id) => [...postKeys.details(), id],
};

// Usage in queries
const { data: posts } = useQuery({
  queryKey: postKeys.list({ category: 'tech', page: 1 }),
  queryFn: () => fetchPosts({ category: 'tech', page: 1 }),
});

const { data: post } = useQuery({
  queryKey: postKeys.detail(postId),
  queryFn: () => fetchPost(postId),
});

// Invalidation strategies:
// Invalidate EVERYTHING posts-related
queryClient.invalidateQueries({ queryKey: postKeys.all });

// Invalidate only lists (not detail pages)
queryClient.invalidateQueries({ queryKey: postKeys.lists() });

// Invalidate only one specific detail
queryClient.invalidateQueries({ queryKey: postKeys.detail(42) });
```

### D1
**After deleting a user, the user still appears in the list. The mutation succeeded (server confirmed 200). Why?**

```jsx
const deleteUser = useMutation({
  mutationFn: (id) => fetch(`/api/users/${id}`, { method: 'DELETE' }),
  onSuccess: () => {
    console.log('Deleted!');
    // nothing else
  },
});
```

**Bug:** The mutation succeeds on the server but the query cache is never told about it. The `['users']` list cache remains stale indefinitely.

```jsx
const deleteUser = useMutation({
  mutationFn: (id) => fetch(`/api/users/${id}`, { method: 'DELETE' }).then(r => r.json()),
  onSuccess: (_, deletedId) => {
    // Option 1: invalidate and refetch
    queryClient.invalidateQueries({ queryKey: ['users'] });

    // Option 2: remove from cache immediately (better UX)
    queryClient.setQueryData(['users'], old =>
      old?.filter(u => u.id !== deletedId)
    );
  },
});
```

### A1
**Should every component that needs the same data call `useQuery` independently, or should you fetch once and pass data down as props? What does TanStack Query's deduplication mean for this decision?**

**Answer:** Call `useQuery` independently in every component that needs the data. TanStack Query deduplicates: if ten components all call `useQuery({ queryKey: ['user', 1] })` simultaneously, only one network request fires and all ten get the cached result. This changes the architecture decision: you no longer need to thread data through props to avoid duplicate fetches — the cache handles that. Colocating data fetching with the component that needs it makes each component self-contained and independently testable. The alternative (fetching once at the top and prop-drilling) reintroduces the coupling problems Context was designed to solve, with none of the caching benefits.

---

## Lesson 22 — HTTP and REST APIs

### C1
**What is idempotency and why does it matter for which HTTP method you use?**

**Answer:** An operation is idempotent if performing it multiple times produces the same result as performing it once. GET, PUT, and DELETE are idempotent — fetching, replacing, or deleting a resource twice leaves the system in the same state as doing it once. POST is not idempotent — calling it twice creates two resources. This matters for retry logic: if a network request times out, you can safely retry GET, PUT, and DELETE automatically. Retrying a POST risks creating duplicates (two orders, two payments). TanStack Query retries failed queries (GET) automatically; mutations (POST) are not retried by default for exactly this reason.

### C2
**What is the difference between a 401, 403, and 404 response, and how should each be handled differently in a React application?**

**Answer:** **401 Unauthorized**: the client is not authenticated — no valid session or token. Handle by redirecting to the login page and clearing any stored credentials. **403 Forbidden**: the client is authenticated but does not have permission for this resource. Handle by showing an "Access Denied" error message — do not redirect to login, as the user IS logged in. **404 Not Found**: the resource doesn't exist. Handle by showing a "Not Found" state in the component or redirecting to a 404 page. These require different UX responses: 401 is an auth problem, 403 is a permission problem, 404 is a data problem.

### B1
**Build a `fetchWithErrorHandling` utility that throws typed errors for different HTTP status codes, and a `useApi` hook that uses it.**

```jsx
class HttpError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
    this.name = 'HttpError';
  }
}

async function fetchWithErrorHandling(url, options = {}) {
  const res = await fetch(url, options);

  if (!res.ok) {
    const body = await res.text();
    throw new HttpError(res.status, body || `HTTP ${res.status}`);
  }

  return res.json();
}

// Usage in TanStack Query
function UserProfile({ id }) {
  const { data, error } = useQuery({
    queryKey: ['user', id],
    queryFn: () => fetchWithErrorHandling(`/api/users/${id}`),
    retry: (failureCount, error) => {
      // Don't retry 4xx errors — they won't resolve themselves
      if (error instanceof HttpError && error.status < 500) return false;
      return failureCount < 3;
    },
  });

  if (error instanceof HttpError) {
    if (error.status === 401) return <Redirect to="/login" />;
    if (error.status === 403) return <p>You don't have permission to view this.</p>;
    if (error.status === 404) return <p>User not found.</p>;
  }
  if (error) return <p>Unexpected error: {error.message}</p>;
  if (!data) return <p>Loading...</p>;

  return <h1>{data.name}</h1>;
}
```

### B2
**Build a `CrudTable` component that displays a list of items and provides Create, Read, Update, Delete operations — each using the correct HTTP method.**

```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { useState } from 'react';

const BASE = '/api/items';

const api = {
  list:   () => fetch(BASE).then(r => r.json()),
  create: (item) => fetch(BASE, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(item) }).then(r => r.json()),
  update: (item) => fetch(`${BASE}/${item.id}`, { method: 'PUT', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(item) }).then(r => r.json()),
  delete: (id)   => fetch(`${BASE}/${id}`, { method: 'DELETE' }),
};

function CrudTable() {
  const qc = useQueryClient();
  const invalidate = () => qc.invalidateQueries({ queryKey: ['items'] });

  const { data: items = [] } = useQuery({ queryKey: ['items'], queryFn: api.list });
  const create = useMutation({ mutationFn: api.create, onSuccess: invalidate });
  const update = useMutation({ mutationFn: api.update, onSuccess: invalidate });
  const remove = useMutation({ mutationFn: api.delete, onSuccess: invalidate });

  const [newName, setNewName] = useState('');

  return (
    <div>
      <h2>Items</h2>
      <form onSubmit={e => { e.preventDefault(); create.mutate({ name: newName }); setNewName(''); }}>
        <input value={newName} onChange={e => setNewName(e.target.value)} placeholder="New item name" />
        <button type="submit" disabled={create.isPending}>Add</button>
      </form>
      <ul>
        {items.map(item => (
          <li key={item.id}>
            {item.name}
            <button onClick={() => update.mutate({ ...item, name: item.name + ' (edited)' })}>Edit</button>
            <button onClick={() => remove.mutate(item.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### D1
**The app crashes with "Unexpected end of JSON input" after a successful DELETE. Why?**

```jsx
const deleteItem = useMutation({
  mutationFn: (id) =>
    fetch(`/api/items/${id}`, { method: 'DELETE' }).then(r => r.json()), // problem here
});
```

**Bug:** Many REST APIs return a `204 No Content` response for successful DELETE — an empty body. Calling `.json()` on an empty body throws "Unexpected end of JSON input."

```jsx
// Fixed — check for empty body before parsing
mutationFn: async (id) => {
  const res = await fetch(`/api/items/${id}`, { method: 'DELETE' });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  if (res.status === 204) return null; // no content — don't parse
  return res.json();
},
```

### A1
**What should live in a shared API layer vs. inline in `queryFn`? How do you structure API functions for a large React application?**

**Answer:** Create a dedicated `api/` folder with modules per domain: `api/users.js`, `api/posts.js`. Each module exports typed async functions: `fetchUser(id)`, `createUser(data)`, `updateUser(id, data)`. The `queryFn` in your TanStack Query config simply calls these: `queryFn: () => fetchUser(id)`. Benefits: (1) the API functions are independently testable without React; (2) if the base URL or auth header logic changes, you update one place; (3) the component stays clean — it doesn't know about HTTP method, Content-Type, or URL construction; (4) TypeScript types for request/response live in the API layer and propagate to components automatically. Keep `queryFn` as a thin adapter: `() => api.users.fetch(id)`.

---

## Lesson 23 — `React.memo`, `useMemo`, `useCallback`

### C1
**What is referential equality and why does it matter for `React.memo`?**

**Answer:** JavaScript uses referential equality for objects and arrays: `{} === {}` is `false` because they are different objects in memory, even with identical contents. `React.memo` shallowly compares each prop between renders using `===`. If a parent re-renders and passes `style={{ color: 'red' }}` as a prop, memo sees a new object reference every render — the prop fails the equality check and the memoized child re-renders anyway. This is why `React.memo` often appears to not work: the component is memoized, but a non-primitive prop is created inline on every parent render. Fix: stabilize the prop with `useMemo` or define it outside the component.

### C2
**What is the difference between `useMemo` and `useCallback`? Are they different hooks or the same concept?**

**Answer:** Same concept, different return value. `useMemo` memoizes the *result* of a function: `const sortedList = useMemo(() => [...list].sort(), [list])`. `useCallback` memoizes the *function itself*: `const handleClick = useCallback(() => doSomething(id), [id])`. `useCallback(fn, deps)` is literally equivalent to `useMemo(() => fn, deps)`. Use `useMemo` when you want to cache an expensive computed value. Use `useCallback` when you want a stable function reference to pass as a prop to a memoized child (so the child doesn't re-render every time the parent does). If the child is not memoized, `useCallback` achieves nothing.

### B1
**Build a `ParentWithMemo` component that renders an expensive `DataTable` child. Use `React.memo` on `DataTable` and `useCallback` on the sort handler so the table only re-renders when its data actually changes.**

```jsx
import { useState, useCallback, memo } from 'react';

const DataTable = memo(function DataTable({ data, onSort }) {
  console.log('DataTable rendered'); // should only log when data or onSort changes
  return (
    <table>
      <thead>
        <tr>
          <th onClick={() => onSort('name')}>Name</th>
          <th onClick={() => onSort('age')}>Age</th>
        </tr>
      </thead>
      <tbody>
        {data.map(row => (
          <tr key={row.id}>
            <td>{row.name}</td>
            <td>{row.age}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
});

function ParentWithMemo() {
  const [data] = useState([
    { id: 1, name: 'Alice', age: 30 },
    { id: 2, name: 'Bob', age: 25 },
  ]);
  const [unrelated, setUnrelated] = useState(0);

  // Stable reference — DataTable won't re-render when unrelated changes
  const handleSort = useCallback((column) => {
    console.log('Sorting by', column);
  }, []); // no dependencies — function never needs to change

  return (
    <div>
      <button onClick={() => setUnrelated(n => n + 1)}>
        Unrelated counter: {unrelated}
      </button>
      <DataTable data={data} onSort={handleSort} />
    </div>
  );
}
```

### B2
**Build a `FilteredList` component with an expensive filtering operation on a large array. Use `useMemo` to ensure the filter only re-runs when the array or filter term changes — not on every render.**

```jsx
import { useState, useMemo } from 'react';

// Simulate an expensive filter
function expensiveFilter(items, term) {
  console.log('Running expensive filter...');
  return items.filter(item =>
    item.name.toLowerCase().includes(term.toLowerCase())
  );
}

function FilteredList({ items }) {
  const [filter, setFilter] = useState('');
  const [unrelated, setUnrelated] = useState(0);

  // Only recalculates when items or filter changes
  const filtered = useMemo(
    () => expensiveFilter(items, filter),
    [items, filter]
  );

  return (
    <div>
      <input
        value={filter}
        onChange={e => setFilter(e.target.value)}
        placeholder="Filter..."
      />
      <button onClick={() => setUnrelated(n => n + 1)}>
        Unrelated: {unrelated}
      </button>
      <ul>
        {filtered.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### D1
**`React.memo` is not preventing the `Child` from re-rendering. Why?**

```jsx
const Child = memo(function Child({ config }) {
  console.log('Child rendered');
  return <div style={config.style}>{config.label}</div>;
});

function Parent() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child config={{ style: { color: 'red' }, label: 'Hello' }} />
    </div>
  );
}
```

**Bug:** `config` is an inline object literal — a new object reference on every Parent render. `memo`'s shallow comparison sees a new reference for `config` and re-renders Child every time.

```jsx
// Fix — stabilize with useMemo
function Parent() {
  const [count, setCount] = useState(0);

  const config = useMemo(
    () => ({ style: { color: 'red' }, label: 'Hello' }),
    [] // never changes
  );

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child config={config} />
    </div>
  );
}
```

### A1
**When should you NOT memoize? What is the cost of premature memoization?**

**Answer:** Do not memoize by default. Memoization has a real cost: React must store the previous value, compare dependencies on every render, and manage the memo entry in memory — for cheap operations, this overhead can exceed the cost of just recomputing. The correct process: (1) build the feature without any memoization; (2) measure with the React Profiler if you observe performance issues; (3) identify the specific expensive computation or unnecessary re-render; (4) apply the minimum memoization that fixes the measured problem. Premature memoization adds cognitive overhead (the reader must reason about dependency arrays), can introduce subtle bugs when dependencies are wrong, and often achieves nothing for operations that take microseconds. Only memoize when a profiler confirms the cost.

---

## Lesson 24 — Understanding Re-renders

### C1
**What are the four causes of a React component re-rendering? Which one is most commonly surprising to engineers?**

**Answer:** (1) **State changes** — calling `setState` in the component; (2) **Parent re-render** — when a parent re-renders, all its children re-render by default (unless memoized); (3) **Context changes** — any component that calls `useContext` for a context re-renders when that context's value changes; (4) **Hook changes** — when internal state in a hook the component uses changes. The most surprising: parent re-renders. Many engineers assume a child only re-renders if its props change, but without `React.memo`, a child re-renders whenever the parent does — regardless of whether any prop changed.

### C2
**What is the difference between a render "bailout" and a skipped render? How does React.memo achieve a bailout?**

**Answer:** A **render** means React calls the component function and reconciles the output. A **bailout** means React starts to re-render a component, does a quick props comparison, and decides to skip calling the function — the previous output is reused. `React.memo` achieves a bailout: before calling the component function, React compares each prop using shallow equality. If all props are the same as the previous render, React reuses the previous render output without calling the function at all. This is more efficient than calling the function and finding nothing changed, but the comparison itself costs time — which is why memoization only pays off when the function call cost exceeds the comparison cost.

### B1
**Build a component tree where you can observe parent re-renders triggering child re-renders. Use `console.log` labels and verify the behavior before and after adding `React.memo`.**

```jsx
import { useState, memo } from 'react';

// Without memo — re-renders every time Parent does
function RegularChild({ name }) {
  console.log(`RegularChild (${name}) rendered`);
  return <p>Regular: {name}</p>;
}

// With memo — only re-renders when name prop changes
const MemoChild = memo(function MemoChild({ name }) {
  console.log(`MemoChild (${name}) rendered`);
  return <p>Memo: {name}</p>;
});

function Parent() {
  const [count, setCount] = useState(0);
  console.log('Parent rendered');

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>
        Parent count: {count}
      </button>
      {/* Observe: RegularChild logs on every button click */}
      <RegularChild name="Alice" />
      {/* Observe: MemoChild only logs on first render */}
      <MemoChild name="Alice" />
    </div>
  );
}
```

### B2
**Restructure a "chatty" component to minimize re-renders without using `memo`, `useMemo`, or `useCallback` — by colocating state closer to where it's used.**

```jsx
// Before — counter state in App causes entire tree to re-render on every click
function App() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <ExpensiveTree />
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
    </div>
  );
}

// After — extract the counter into its own component
// ExpensiveTree is no longer a child of anything that re-renders on click
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}

function App() {
  return (
    <div>
      <ExpensiveTree />
      <Counter />  {/* re-renders only affect Counter's subtree */}
    </div>
  );
}
```

### D1
**`Child` re-renders on every click of the parent button even though it's wrapped in `memo`. Find all reasons why.**

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const theme = { color: 'red', fontSize: 14 };  // new object every render
  const handleClick = () => console.log('clicked'); // new function every render

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoizedChild theme={theme} onClick={handleClick} />
    </>
  );
}

const MemoizedChild = memo(function MemoizedChild({ theme, onClick }) {
  console.log('Child rendered');
  return <button onClick={onClick} style={theme}>Click me</button>;
});
```

**Two problems:** `theme` is a new object literal each render; `handleClick` is a new function each render. Both fail the shallow equality check.

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  const theme = useMemo(() => ({ color: 'red', fontSize: 14 }), []);
  const handleClick = useCallback(() => console.log('clicked'), []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <MemoizedChild theme={theme} onClick={handleClick} />
    </>
  );
}
```

### A1
**Is it always better to have fewer re-renders? What is the actual metric you should optimize for?**

**Answer:** No. The goal is not to minimize re-renders — it is to minimize *perceptible jank* and *wasted work*. A component that re-renders 60 times per second but completes each render in 0.1ms causes no visible problem. A component that renders once but takes 200ms causes dropped frames and janky UI. The metric is: is any frame taking longer than 16ms (60fps budget) or 10ms (90fps)? Measure with the React Profiler before optimizing. Over-optimization with memo/useMemo/useCallback adds cognitive complexity and maintenance burden, and can actually slow down simple operations by adding comparison overhead that exceeds the render cost. Profile first, then optimize specifically.

---

## Lesson 25 — Portals, Event Propagation, and Modals

### C1
**What is a React portal and why is it specifically needed for modals, tooltips, and dropdowns?**

**Answer:** A portal renders a component's output into a different DOM node than its position in the React tree. `createPortal(children, domNode)` renders `children` into `domNode` (typically `document.body`). This is needed for modals because: CSS properties on parent elements (`overflow: hidden`, `z-index` stacking contexts, `transform`) can clip or obscure a modal that is rendered inside them. By portaling to `document.body`, the modal is outside any clipping or z-index constraint and appears over everything. Crucially, the portal component remains in the React tree at its original position — events and Context still flow through the React tree, not the DOM tree.

### C2
**What is event bubbling and when should you use `e.stopPropagation()` vs. checking `e.target` in a parent handler?**

**Answer:** Event bubbling: a click on a child element fires handlers on the child, then the parent, then grandparent — all the way to `document`. `stopPropagation()` prevents the event from bubbling beyond the element that called it. Use `stopPropagation` when: a click inside a modal should not close the modal (the modal's backdrop has an `onClick` close handler, and clicking inside must not bubble to the backdrop). Use `e.target` checking when: you want one handler on a parent to handle events from multiple children differently — this is "event delegation" and is more efficient than attaching many individual handlers.

### B1
**Build a `Modal` component using `createPortal`. It renders into `document.body`. Clicking the backdrop closes it. Clicking inside the modal content does not close it.**

```jsx
import { createPortal } from 'react-dom';
import { useEffect } from 'react';

function Modal({ isOpen, onClose, children }) {
  // Close on Escape key
  useEffect(() => {
    if (!isOpen) return;
    function handleKey(e) {
      if (e.key === 'Escape') onClose();
    }
    document.addEventListener('keydown', handleKey);
    return () => document.removeEventListener('keydown', handleKey);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="modal-backdrop"
      onClick={onClose}
      style={{
        position: 'fixed', inset: 0,
        background: 'rgba(0,0,0,0.5)',
        display: 'flex', alignItems: 'center', justifyContent: 'center',
        zIndex: 1000,
      }}
    >
      <div
        className="modal-content"
        onClick={e => e.stopPropagation()} // prevent closing when clicking inside
        style={{
          background: '#fff', borderRadius: 8,
          padding: 24, minWidth: 300,
        }}
      >
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.body
  );
}

// Usage
function App() {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Modal</button>
      <Modal isOpen={isOpen} onClose={() => setIsOpen(false)}>
        <h2>Modal Title</h2>
        <p>Modal content here.</p>
      </Modal>
    </>
  );
}
```

### B2
**Build a `Dropdown` that closes when clicking outside of it. Use a `useClickOutside` custom hook.**

```jsx
import { useRef, useEffect, useState } from 'react';

function useClickOutside(callback) {
  const ref = useRef(null);

  useEffect(() => {
    function handleClick(e) {
      if (ref.current && !ref.current.contains(e.target)) {
        callback();
      }
    }
    document.addEventListener('mousedown', handleClick);
    return () => document.removeEventListener('mousedown', handleClick);
  }, [callback]);

  return ref;
}

function Dropdown({ options }) {
  const [isOpen, setIsOpen] = useState(false);
  const [selected, setSelected] = useState(null);

  const ref = useClickOutside(() => setIsOpen(false));

  return (
    <div ref={ref} style={{ position: 'relative', display: 'inline-block' }}>
      <button onClick={() => setIsOpen(o => !o)}>
        {selected ?? 'Select...'} ▾
      </button>
      {isOpen && (
        <ul style={{
          position: 'absolute', top: '100%', left: 0,
          background: '#fff', border: '1px solid #ccc',
          listStyle: 'none', margin: 0, padding: 0, minWidth: 150,
        }}>
          {options.map(opt => (
            <li
              key={opt}
              onClick={() => { setSelected(opt); setIsOpen(false); }}
              style={{ padding: '8px 12px', cursor: 'pointer' }}
            >
              {opt}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### D1
**Clicking the "Delete" button inside the modal immediately closes the modal. Why?**

```jsx
function ConfirmModal({ onClose, onConfirm }) {
  return createPortal(
    <div onClick={onClose} style={{ /* backdrop styles */ }}>
      <div style={{ /* modal styles */ }}>
        <p>Are you sure?</p>
        <button onClick={onConfirm}>Delete</button>
        <button onClick={onClose}>Cancel</button>
      </div>
    </div>,
    document.body
  );
}
```

**Bug:** The "Delete" button click bubbles up through the inner `<div>` to the backdrop `<div>`, which has `onClick={onClose}`. There is no `stopPropagation` on the inner content div.

```jsx
<div style={{ /* modal styles */ }} onClick={e => e.stopPropagation()}>
  <p>Are you sure?</p>
  <button onClick={onConfirm}>Delete</button>
  <button onClick={onClose}>Cancel</button>
</div>
```

### A1
**Should modals be managed with a portal or with conditional rendering in-place? When does the choice matter?**

**Answer:** For most modals, always use portals — the risk of CSS stacking contexts and `overflow: hidden` clipping a modal that is rendered inside a deeply nested component tree is real and hard to debug. The failure mode (modal cut off or hidden behind other elements) is invisible during development but breaks in production layouts. Portal is the correct default for anything that should appear above all other content. The only case where in-place rendering is acceptable: a modal-like overlay that is specifically scoped to a parent container (an inline popover within a bounded region) where you intentionally want it to respect the parent's clipping. For global overlays — always portal to `document.body`.

---

## Lesson 26 — Error Boundaries

### C1
**Why can't you use `try/catch` to handle React rendering errors? What is structurally different about render errors?**

**Answer:** `try/catch` works with imperative, synchronous code — you call a function, it throws, you catch. React's rendering is driven by React itself: it calls your component function, and if that function throws, React is the one handling it — your code has already returned control to React's render loop. You can't wrap `<MyComponent />` in a `try/catch` because `try/catch` only catches synchronous throws in *your* call stack, not exceptions thrown inside React's reconciliation. Error Boundaries work because they are class components with the `componentDidCatch` lifecycle, which React itself calls when it catches a render error in a descendant.

### C2
**Error boundaries do NOT catch errors from these four sources. Name them and explain why.**

**Answer:** (1) **Event handlers** — these are not part of React's render cycle; wrap them in regular `try/catch`; (2) **Asynchronous code** — `setTimeout`, `fetch` callbacks, promises — these throw in a separate call stack frame after React's render phase; (3) **Server-side rendering** — Error Boundaries are a client-side mechanism; (4) **The Error Boundary component itself** — a boundary can't catch its own errors. For async errors (e.g., a failed fetch), throw the error into React's render cycle by setting it in state (`setError(err)`) and throwing from render: `if (error) throw error`.

### B1
**Build a complete `ErrorBoundary` class component with a `fallback` prop and a `resetKeys` prop that resets the boundary when specified values change.**

```jsx
import { Component } from 'react';

class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    // Report to your error tracking service
    console.error('Error caught by boundary:', error, info);
  }

  componentDidUpdate(prevProps) {
    if (this.state.hasError) {
      const { resetKeys = [] } = this.props;
      const { resetKeys: prevKeys = [] } = prevProps;
      if (resetKeys.some((key, i) => key !== prevKeys[i])) {
        this.setState({ hasError: false, error: null });
      }
    }
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div>
          <h2>Something went wrong.</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try Again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage
function App() {
  const [userId, setUserId] = useState(1);
  return (
    <ErrorBoundary
      resetKeys={[userId]}
      fallback={<p>Failed to load user.</p>}
    >
      <UserProfile userId={userId} />
    </ErrorBoundary>
  );
}
```

### B2
**Build a `withErrorBoundary` HOC and an `useErrorBoundary` hook that lets you throw async errors into the nearest boundary.**

```jsx
// HOC wrapper
function withErrorBoundary(Component, fallback) {
  return function Wrapped(props) {
    return (
      <ErrorBoundary fallback={fallback}>
        <Component {...props} />
      </ErrorBoundary>
    );
  };
}

// Hook to throw errors into the nearest boundary
import { useState, useCallback } from 'react';

function useErrorBoundary() {
  const [, setState] = useState();

  const throwError = useCallback((error) => {
    setState(() => { throw error; });
  }, []);

  return throwError;
}

// Usage — throw async errors into the boundary
function DataLoader() {
  const throwError = useErrorBoundary();

  async function loadData() {
    try {
      const data = await fetchSomething();
      setData(data);
    } catch (err) {
      throwError(err); // surfaces in the nearest ErrorBoundary
    }
  }

  return <button onClick={loadData}>Load</button>;
}
```

### D1
**The Error Boundary is not catching errors from this component. Why?**

```jsx
function FetchComponent() {
  const [data, setData] = useState(null);

  async function load() {
    const res = await fetch('/api/data');
    if (!res.ok) throw new Error('Failed'); // this throw is not caught by the boundary
    setData(await res.json());
  }

  useEffect(() => { load(); }, []);

  return <div>{data?.name}</div>;
}
```

**Bug:** The error is thrown inside an `async` function (`load`) — an async operation. Error Boundaries only catch synchronous render errors and lifecycle errors. The thrown error here happens in a Promise rejection, outside React's control.

```jsx
// Fix — bring the error into React's world via state
function FetchComponent() {
  const [data, setData] = useState(null);
  const throwError = useErrorBoundary(); // from the hook in B2

  async function load() {
    try {
      const res = await fetch('/api/data');
      if (!res.ok) throw new Error('Failed');
      setData(await res.json());
    } catch (err) {
      throwError(err); // surfaces in the Error Boundary
    }
  }

  useEffect(() => { load(); }, []);
  return <div>{data?.name}</div>;
}
```

### A1
**Where in a component tree should you place Error Boundaries? What is the tradeoff between too few and too many?**

**Answer:** Too few (only one at the root): any render error in any component kills the entire application UI and shows a blank page or a single root error message — catastrophic for users, especially if the error is in a minor widget. Too many: every component wrapped in its own boundary adds performance overhead and makes the codebase noisy. The right strategy: (1) one root boundary as a last resort that shows a recovery UI; (2) boundaries at each major "section" of the UI — sidebar, main content, header — so a failure in one section doesn't kill the others; (3) boundaries around third-party components or dynamic imports that you can't fully control; (4) boundaries around specific components that are known to fail on bad data. The principle: isolate failures to the smallest possible impactful unit.

---

## Lesson 27A — Suspense

### C1
**What does Suspense do and what does a component need to do to "suspend"?**

**Answer:** Suspense catches components that "suspend" — that throw a Promise during rendering — and renders a fallback UI (`<Suspense fallback={<Spinner />}>`) while that Promise is pending. Once the Promise resolves, React re-renders the suspended component. For a component to suspend, it must throw a Promise during render. In practice, you don't do this manually — React's `lazy()` and data-fetching libraries that integrate with Suspense (TanStack Query with `suspense: true`, Next.js's data fetching, React Server Components) handle the throwing for you. Suspense is the declarative way to show loading states for asynchronous component or data loading.

### C2
**What is the relationship between Suspense and `React.lazy`? How does lazy loading work mechanically?**

**Answer:** `React.lazy(() => import('./Component'))` returns a component that throws a Promise the first time it renders — the Promise resolves when the dynamic import completes and the component's module is loaded. Suspense catches this Promise and shows the fallback. When the import resolves, React re-renders and the component renders normally. This means: the component's JavaScript code is not included in the initial bundle — it's fetched in a separate network request only when the component is first about to render. This is "code splitting" at the component level, reducing the initial bundle size.

### B1
**Set up lazy loading for a `HeavyAnalyticsDashboard` component. Wrap it in Suspense with a loading skeleton. Handle the error case with an Error Boundary.**

```jsx
import { lazy, Suspense } from 'react';

const HeavyAnalyticsDashboard = lazy(
  () => import('./HeavyAnalyticsDashboard')
);

function LoadingSkeleton() {
  return (
    <div style={{ padding: 20 }}>
      <div style={{ height: 24, background: '#e5e7eb', borderRadius: 4, marginBottom: 12 }} />
      <div style={{ height: 200, background: '#e5e7eb', borderRadius: 4 }} />
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary fallback={<p>Failed to load dashboard.</p>}>
      <Suspense fallback={<LoadingSkeleton />}>
        <HeavyAnalyticsDashboard />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### B2
**Build a nested Suspense architecture where different parts of the UI have independent loading states.**

```jsx
import { lazy, Suspense } from 'react';

const UserProfile = lazy(() => import('./UserProfile'));
const ActivityFeed  = lazy(() => import('./ActivityFeed'));
const Recommendations = lazy(() => import('./Recommendations'));

function Dashboard() {
  return (
    <div style={{ display: 'grid', gridTemplateColumns: '1fr 2fr 1fr', gap: 16 }}>
      {/* Each section has its own loading state — they don't block each other */}
      <Suspense fallback={<p>Loading profile...</p>}>
        <UserProfile />
      </Suspense>

      <Suspense fallback={<p>Loading feed...</p>}>
        <ActivityFeed />
      </Suspense>

      <Suspense fallback={<p>Loading recommendations...</p>}>
        <Recommendations />
      </Suspense>
    </div>
  );
}
```

### D1
**The Suspense fallback shows forever and the component never renders. Why?**

```jsx
const MyComponent = lazy(() => {
  import('./MyComponent'); // missing return
});
```

**Bug:** The dynamic import is not returned from the lazy callback — so `lazy` receives `undefined` as the Promise. The import fires but the result is never connected.

```jsx
// Fixed — always return the import() Promise
const MyComponent = lazy(() => import('./MyComponent'));
```

### A1
**Where should Suspense boundaries be placed? What is the granularity tradeoff?**

**Answer:** Place Suspense boundaries at the level where you want the fallback to be scoped. One root Suspense means the entire page shows a spinner if any part is loading — acceptable for initial page load, but jarring if a sidebar widget loads late and blacks out the whole page. Fine-grained Suspense (per section, per widget) means each part independently shows its own loading state — better UX, especially for nested data loading. The tradeoff: too many Suspense boundaries creates "loading waterfall" spinners that pop in sequentially. The correct approach: coarse boundaries for initial page load (route-level Suspense), fine-grained for secondary content (sidebar, recommendations, related items). Pair Suspense with `startTransition` to avoid showing the fallback for fast loads.

---

## Lesson 27B — `startTransition`, `useTransition`, `useDeferredValue`

### C1
**What does React's Concurrent Mode mean for rendering, and what problem do `useTransition` and `useDeferredValue` solve?**

**Answer:** In Concurrent Mode, React can interrupt and deprioritize non-urgent renders to keep the UI responsive. Without this, a heavy state update (filtering 10,000 items) blocks the thread and makes the input feel frozen. `useTransition` and `useDeferredValue` let you mark some updates as "non-urgent" — React prioritizes keeping the UI responsive (typing in an input) and defers the expensive work (filtering the list) to background renders that can be interrupted. The user sees instant input response; the list update trails slightly behind, appearing when the work is done.

### C2
**What is the difference between `useTransition` and `useDeferredValue`? When do you control the state vs. receive the value?**

**Answer:** Use `useTransition` when you *control the state update* — you have a `setState` call and you want to mark it as non-urgent: `startTransition(() => setFilter(value))`. You get an `isPending` boolean to show a "results updating..." indicator. Use `useDeferredValue` when you receive a value as a prop or from a context that you *don't control* — you can't wrap the setState in a transition: `const deferredFilter = useDeferredValue(filter)`. The deferred value lags behind the live value during transitions. `useTransition` is the proactive approach; `useDeferredValue` is the reactive approach for values you receive from outside.

### B1
**Build a searchable list of 5,000 items. Without transitions, typing feels sluggish. Fix it with `useTransition`.**

```jsx
import { useState, useTransition, useMemo } from 'react';

const ALL_ITEMS = Array.from({ length: 5000 }, (_, i) => ({
  id: i,
  name: `Item ${i + 1} — ${Math.random().toString(36).slice(2)}`,
}));

function HeavyList({ filter }) {
  // Expensive computation
  const filtered = useMemo(
    () => ALL_ITEMS.filter(item =>
      item.name.toLowerCase().includes(filter.toLowerCase())
    ),
    [filter]
  );

  return (
    <ul>
      {filtered.slice(0, 100).map(item => (
        <li key={item.id}>{item.name}</li>
      ))}
      {filtered.length > 100 && <li>...and {filtered.length - 100} more</li>}
    </ul>
  );
}

function SearchPage() {
  const [inputValue, setInputValue] = useState('');
  const [filter, setFilter] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    const value = e.target.value;
    setInputValue(value); // urgent — input must update instantly
    startTransition(() => {
      setFilter(value); // non-urgent — list update can be deferred
    });
  }

  return (
    <div>
      <input
        value={inputValue}
        onChange={handleChange}
        placeholder="Search 5000 items..."
      />
      {isPending && <span style={{ color: 'gray' }}> Updating results...</span>}
      <HeavyList filter={filter} />
    </div>
  );
}
```

### B2
**Use `useDeferredValue` to achieve the same effect when you receive the filter value as a prop.**

```jsx
import { useDeferredValue, useMemo } from 'react';

function DeferredList({ filter }) {
  // Defer the filter value — the list uses the lagging version
  const deferredFilter = useDeferredValue(filter);

  const isStale = deferredFilter !== filter;

  const filtered = useMemo(
    () => ALL_ITEMS.filter(item =>
      item.name.toLowerCase().includes(deferredFilter.toLowerCase())
    ),
    [deferredFilter]
  );

  return (
    <div style={{ opacity: isStale ? 0.7 : 1, transition: 'opacity 0.2s' }}>
      <ul>
        {filtered.slice(0, 100).map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </div>
  );
}
```

### D1
**The transition doesn't make the input feel responsive. Why?**

```jsx
function SearchPage() {
  const [filter, setFilter] = useState('');
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    startTransition(() => {
      setFilter(e.target.value); // both state updates are deferred
    });
  }

  return (
    <input value={filter} onChange={handleChange} />
  );
}
```

**Bug:** The input value itself is controlled by `filter`, which is inside `startTransition`. The input's displayed value is now non-urgent too — it lags behind typing. There must be a separate, urgent state for the input display value.

```jsx
function SearchPage() {
  const [inputValue, setInputValue] = useState(''); // urgent — controls the input
  const [filter, setFilter] = useState('');          // non-urgent — controls the list
  const [isPending, startTransition] = useTransition();

  function handleChange(e) {
    setInputValue(e.target.value);               // instant
    startTransition(() => setFilter(e.target.value)); // deferred
  }

  return <input value={inputValue} onChange={handleChange} />;
}
```

### A1
**When should you use transitions vs. `useDeferredValue` vs. neither?**

**Answer:** Use neither as the first approach — most React renders are fast enough that transitions are unnecessary. Profile first. Use `useTransition` when: you control a state update that triggers an expensive re-render (filtering a large list, navigating to a heavy route) and you want to keep the triggering interaction (input, button) immediately responsive. Use `useDeferredValue` when: the expensive value comes from a prop or context you don't control and you want the component that uses it to defer. Don't use either for: network requests (use TanStack Query's background refetch), animations (use CSS or a library), or performance issues that are actually caused by unnecessary re-renders of memoizable components (fix with `memo` first).

---

## Lesson 28 — Refs Across Components: `forwardRef` and `useImperativeHandle`

### C1
**Why can you not pass a `ref` to a custom component as a regular prop? What does `forwardRef` do differently?**

**Answer:** `ref` is a special attribute in React, like `key` — it is handled by React itself and is not included in the `props` object passed to the component function. If you write `<MyInput ref={inputRef} />`, the `ref` is intercepted by React and is not available as `props.ref` inside `MyInput`. `forwardRef` explicitly opts a component into ref forwarding — it receives `(props, ref)` as its two arguments, where `ref` is the ref passed by the parent. The component then attaches this ref to a DOM node (`<input ref={ref} />`) or uses `useImperativeHandle` to customize what the ref exposes.

### C2
**What is `useImperativeHandle` for and when would you use it over simply forwarding the ref to a DOM node?**

**Answer:** `useImperativeHandle` lets you customize the value exposed by a ref — instead of exposing the raw DOM node, you expose a specific object with selected methods. Use it when: (1) you want to expose a limited, intentional API (`{ focus, clear }`) rather than the full DOM node (which would expose everything, breaking encapsulation); (2) the "imperative action" you want to expose (`scrollToBottom`, `shake`, `reset`) is composed of multiple DOM operations and isn't a single method on a DOM element. The rule: if you're forwarding a ref just so the parent can call `.focus()` on an input, simply forwarding the ref to the DOM node is fine. If you want to hide the DOM node and expose a higher-level API, use `useImperativeHandle`.

### B1
**Build a `FancyInput` component that forwards a ref to its underlying `<input>` so the parent can programmatically focus it.**

```jsx
import { forwardRef, useRef } from 'react';

const FancyInput = forwardRef(function FancyInput({ label, ...props }, ref) {
  return (
    <label>
      {label}
      <input
        ref={ref}
        className="fancy-input"
        {...props}
      />
    </label>
  );
});

function Form() {
  const inputRef = useRef(null);

  return (
    <div>
      <FancyInput ref={inputRef} label="Name:" placeholder="Enter your name" />
      <button onClick={() => inputRef.current?.focus()}>
        Focus the input
      </button>
    </div>
  );
}
```

### B2
**Build a `VideoPlayer` component that uses `useImperativeHandle` to expose `play`, `pause`, and `seek` methods — without exposing the raw HTMLVideoElement.**

```jsx
import { forwardRef, useRef, useImperativeHandle } from 'react';

const VideoPlayer = forwardRef(function VideoPlayer({ src }, ref) {
  const videoRef = useRef(null);

  useImperativeHandle(ref, () => ({
    play: () => videoRef.current?.play(),
    pause: () => videoRef.current?.pause(),
    seek: (time) => { if (videoRef.current) videoRef.current.currentTime = time; },
  }), []);

  return <video ref={videoRef} src={src} controls style={{ width: '100%' }} />;
});

function PlayerController() {
  const playerRef = useRef(null);

  return (
    <div>
      <VideoPlayer ref={playerRef} src="/sample.mp4" />
      <button onClick={() => playerRef.current?.play()}>Play</button>
      <button onClick={() => playerRef.current?.pause()}>Pause</button>
      <button onClick={() => playerRef.current?.seek(30)}>Jump to 0:30</button>
    </div>
  );
}
```

### D1
**`ref.current` is always null even though the component is mounted. Why?**

```jsx
// Parent
function Form() {
  const ref = useRef(null);
  return <CustomInput ref={ref} />;
}

// Child — NOT using forwardRef
function CustomInput({ ref }) {
  return <input ref={ref} />;
}
```

**Bug:** `ref` is a special React attribute and is not passed as a regular prop. `props.ref` is undefined inside `CustomInput` — so the `ref` passed to `<input>` is undefined, and `ref.current` stays null.

```jsx
// Fixed — use forwardRef
import { forwardRef } from 'react';

const CustomInput = forwardRef(function CustomInput(props, ref) {
  return <input ref={ref} {...props} />;
});
```

### A1
**Should you prefer imperative refs or declarative props for controlling child behavior? What signals tell you to reach for a ref?**

**Answer:** Always prefer declarative props as the first approach — pass `isPlaying={true}` to a video player, not `videoRef.current.play()`. Declarative props keep the parent in control, the behavior is visible in JSX, and it's easier to test and reason about. Reach for a ref only when: (1) the behavior is inherently imperative and stateless from the parent's perspective — "focus this input right now" is an action, not a persistent state; (2) you need to synchronize with a third-party library that exposes an imperative API (`mapRef.current.flyTo(coords)`); (3) you need precise timing control (scroll to position exactly when a transition ends). The signal: if the parent is trying to *tell* a child to *do something once*, refs are appropriate. If the parent is *describing what the child should be*, props are appropriate.

---

## Lesson 29 — Accessibility in React

### C1
**What is semantic HTML and why does it matter more in React SPAs than in traditional multi-page sites?**

**Answer:** Semantic HTML means using the correct element for the correct purpose: `<button>` for clickable actions, `<nav>` for navigation, `<h1>` for the main heading, `<label>` for form labels. Screen readers and assistive technologies use these semantics to build an accessible "model" of the page. In traditional multi-page sites, navigating between pages reloads the document and screen readers re-announce context. In SPAs, navigation is silent at the DOM level — screen readers don't know the page changed. React SPAs must work harder: announcing route changes, managing focus after navigation, using ARIA live regions for dynamic updates, and using semantic HTML so the accessibility model is correct to begin with.

### C2
**What is `useId` and why was it introduced in React 18? What problem does it solve that a simple static string doesn't?**

**Answer:** `useId` generates a unique, stable ID that is consistent between server and client rendering. The problem with static strings: if you have multiple instances of a `FormField` component and you hardcode `id="email"`, every instance has the same ID — which breaks both accessibility (`<label htmlFor="email">` is ambiguous) and HTML validity (IDs must be unique in a document). Random IDs (Math.random()) cause hydration mismatches between SSR and client. `useId` solves both: it generates a deterministic ID based on the component's position in the React tree, which is the same on server and client, and unique per component instance.

### B1
**Build a fully accessible `FormField` component with proper label association, error message announcement, and `useId` for unique IDs.**

```jsx
import { useId } from 'react';

function FormField({ label, error, ...inputProps }) {
  const id = useId();
  const errorId = `${id}-error`;

  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        aria-describedby={error ? errorId : undefined}
        aria-invalid={error ? 'true' : undefined}
        {...inputProps}
      />
      {error && (
        <span
          id={errorId}
          role="alert"
          style={{ color: 'red', display: 'block' }}
        >
          {error}
        </span>
      )}
    </div>
  );
}

// Usage
function SignupForm() {
  return (
    <form>
      <FormField
        label="Email"
        type="email"
        error="Please enter a valid email address."
      />
      <FormField
        label="Password"
        type="password"
      />
    </form>
  );
}
```

### B2
**Build an accessible `TabPanel` component. Tabs must be keyboard navigable (arrow keys), have correct ARIA roles, and focus management.**

```jsx
import { useState, useRef } from 'react';

function AccessibleTabs({ tabs }) {
  const [activeIndex, setActiveIndex] = useState(0);
  const tabRefs = useRef([]);

  function handleKeyDown(e, index) {
    let nextIndex = index;
    if (e.key === 'ArrowRight') nextIndex = (index + 1) % tabs.length;
    if (e.key === 'ArrowLeft')  nextIndex = (index - 1 + tabs.length) % tabs.length;
    if (e.key === 'Home')       nextIndex = 0;
    if (e.key === 'End')        nextIndex = tabs.length - 1;

    if (nextIndex !== index) {
      setActiveIndex(nextIndex);
      tabRefs.current[nextIndex]?.focus();
      e.preventDefault();
    }
  }

  return (
    <div>
      <div role="tablist" aria-label="Content tabs">
        {tabs.map((tab, i) => (
          <button
            key={tab.id}
            role="tab"
            id={`tab-${tab.id}`}
            aria-selected={activeIndex === i}
            aria-controls={`panel-${tab.id}`}
            tabIndex={activeIndex === i ? 0 : -1}
            ref={el => (tabRefs.current[i] = el)}
            onClick={() => setActiveIndex(i)}
            onKeyDown={e => handleKeyDown(e, i)}
          >
            {tab.label}
          </button>
        ))}
      </div>
      {tabs.map((tab, i) => (
        <div
          key={tab.id}
          role="tabpanel"
          id={`panel-${tab.id}`}
          aria-labelledby={`tab-${tab.id}`}
          hidden={activeIndex !== i}
        >
          {tab.content}
        </div>
      ))}
    </div>
  );
}
```

### D1
**This button is inaccessible. Identify all accessibility violations and fix them.**

```jsx
function LikeButton({ onClick }) {
  return (
    <div onClick={onClick} style={{ cursor: 'pointer' }}>
      ❤️
    </div>
  );
}
```

**Violations:**
1. `<div>` is not keyboard focusable — keyboard users can't reach it
2. `<div>` has no ARIA role — screen readers don't know it's a button
3. No accessible name — screen reader users hear only "❤️" with no context
4. No keyboard handler — pressing Enter or Space does nothing

```jsx
// Fixed — use a semantic button
function LikeButton({ onClick, liked }) {
  return (
    <button
      onClick={onClick}
      aria-label={liked ? 'Unlike this post' : 'Like this post'}
      aria-pressed={liked}
    >
      {liked ? '❤️' : '🤍'}
    </button>
  );
}
```

### A1
**When should you add ARIA attributes vs. fix the HTML structure? What is the "ARIA first" anti-pattern?**

**Answer:** The correct priority is the ARIA specification's own rule: "No ARIA is better than bad ARIA." The decision order: (1) use the correct semantic HTML element — a `<button>` already has the right role, keyboard behavior, and accessibility semantics built in; (2) if no native element fits, use an appropriate ARIA role on a generic element (`role="dialog"`, `role="status"`); (3) use ARIA attributes to supplement semantics that HTML alone cannot express (`aria-expanded`, `aria-live`, `aria-describedby`). The "ARIA first" anti-pattern: using `<div role="button">` instead of `<button>` — now you must manually add tabIndex, keyboard handlers, and focus management that the native element provides for free. Fix the element; add ARIA only for what the element cannot express natively.

---

## Lesson 30 — Testing React

### C1
**What is the philosophical difference between testing "implementation" and testing "behavior"? Why does testing-library.com enforce this?**

**Answer:** Testing implementation: asserting on internal details — the component called `setState`, a specific internal function was called, the component's internal `count` variable is 5. Testing behavior: asserting on what the user actually sees and does — the button shows "Submit", clicking it shows a success message, the form is disabled while loading. Implementation tests are fragile — any refactor breaks them even if the user experience is unchanged. Behavior tests survive refactors: if you rewrite the form from class to hook to useReducer but the UX is identical, behavior tests still pass. Testing Library enforces this by only exposing queries that users would use to find elements: `getByRole`, `getByLabelText`, `getByText` — not `getByClassName` or `getByTestId` as a first resort.

### C2
**What is `userEvent` (from `@testing-library/user-event`) vs. `fireEvent`? When should each be used?**

**Answer:** `fireEvent` dispatches individual DOM events — `fireEvent.click(button)` fires a click event and nothing else. `userEvent` simulates real user interactions including all the events a real browser fires: `userEvent.click` fires `pointerover`, `pointerenter`, `mouseover`, `mouseenter`, `pointermove`, `mousemove`, `pointerdown`, `mousedown`, `pointerup`, `mouseup`, `click`. `userEvent.type` fires individual keydown/keypress/keyup/input events per character. Use `userEvent` for all tests involving user interactions — it catches bugs that only appear with the full event sequence (e.g., a component that checks `onMouseDown` before `onClick`). `fireEvent` is appropriate only for events that have no user-event equivalent or for testing specific low-level event behavior.

### B1
**Write complete tests for a `Counter` component: test initial render, increment, decrement, and that count cannot go below 0.**

```jsx
// Counter.jsx
import { useState } from 'react';

function Counter({ min = 0 }) {
  const [count, setCount] = useState(0);
  return (
    <div>
      <span aria-live="polite">Count: {count}</span>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button
        onClick={() => setCount(c => Math.max(min, c - 1))}
        disabled={count <= min}
      >
        Decrement
      </button>
    </div>
  );
}

// Counter.test.jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Counter } from './Counter';

describe('Counter', () => {
  it('renders with initial count of 0', () => {
    render(<Counter />);
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  it('increments when the increment button is clicked', async () => {
    const user = userEvent.setup();
    render(<Counter />);
    await user.click(screen.getByRole('button', { name: 'Increment' }));
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });

  it('decrements when the decrement button is clicked', async () => {
    const user = userEvent.setup();
    render(<Counter />);
    await user.click(screen.getByRole('button', { name: 'Increment' }));
    await user.click(screen.getByRole('button', { name: 'Decrement' }));
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });

  it('disables decrement button when count is at minimum', () => {
    render(<Counter min={0} />);
    expect(screen.getByRole('button', { name: 'Decrement' })).toBeDisabled();
  });

  it('does not go below the minimum', async () => {
    const user = userEvent.setup();
    render(<Counter min={0} />);
    // Try clicking decrement when already at 0
    await user.click(screen.getByRole('button', { name: 'Decrement' })).catch(() => {});
    expect(screen.getByText('Count: 0')).toBeInTheDocument();
  });
});
```

### B2
**Write a test for an async data-fetching component using Mock Service Worker (msw) to intercept the network request.**

```jsx
// UserCard.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import { http, HttpResponse } from 'msw';
import { setupServer } from 'msw/node';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { UserCard } from './UserCard';

const server = setupServer(
  http.get('/api/users/:id', ({ params }) => {
    return HttpResponse.json({
      id: Number(params.id),
      name: 'Alice Chen',
      email: 'alice@example.com',
    });
  })
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

function renderWithQuery(ui) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  );
}

describe('UserCard', () => {
  it('shows loading state initially', () => {
    renderWithQuery(<UserCard userId={1} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('displays user data after loading', async () => {
    renderWithQuery(<UserCard userId={1} />);
    await waitFor(() => {
      expect(screen.getByText('Alice Chen')).toBeInTheDocument();
      expect(screen.getByText('alice@example.com')).toBeInTheDocument();
    });
  });

  it('shows error state when request fails', async () => {
    server.use(
      http.get('/api/users/:id', () => HttpResponse.error())
    );
    renderWithQuery(<UserCard userId={1} />);
    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

### D1
**This test passes but tests the wrong thing. What is the issue?**

```jsx
it('submits the form', () => {
  const handleSubmit = jest.fn();
  render(<LoginForm onSubmit={handleSubmit} />);
  
  fireEvent.click(screen.getByRole('button', { name: 'Submit' }));
  
  expect(handleSubmit).toHaveBeenCalled();
});
```

**Problem:** This tests that `onSubmit` is called when the button is clicked — but it doesn't fill in any form fields. In most real forms, the submit handler isn't called (or shouldn't be called) when the form is empty — validation should prevent it. This test would pass even if validation is completely broken.

```jsx
// Fixed — test realistic user behavior
it('submits the form with valid data', async () => {
  const user = userEvent.setup();
  const handleSubmit = jest.fn();
  render(<LoginForm onSubmit={handleSubmit} />);

  await user.type(screen.getByLabelText('Email'), 'alice@example.com');
  await user.type(screen.getByLabelText('Password'), 'password123');
  await user.click(screen.getByRole('button', { name: 'Log In' }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'alice@example.com',
    password: 'password123',
  });
});

it('does not submit with empty fields', async () => {
  const user = userEvent.setup();
  const handleSubmit = jest.fn();
  render(<LoginForm onSubmit={handleSubmit} />);
  await user.click(screen.getByRole('button', { name: 'Log In' }));
  expect(handleSubmit).not.toHaveBeenCalled();
  expect(screen.getByText(/required/i)).toBeInTheDocument();
});
```

### A1
**What is the testing pyramid for a React application and how much of each layer is appropriate?**

**Answer:** The pyramid, bottom to top: (1) **Unit tests** (most numerous, fastest) — pure functions, reducers, utility functions. These have zero React overhead and run in milliseconds. Cover exhaustively. (2) **Integration tests** (middle) — full component trees, real user interactions, mocked network (msw). This is where most React testing value lives — tests that simulate what a user actually does. Aim for broad coverage here. (3) **E2E tests** (fewest, slowest) — Playwright or Cypress running against a real browser. Cover the most critical user journeys (checkout, login, signup) — 10-20 tests, not 200. The common mistake: too many unit tests on implementation details (testing individual functions that will be refactored), not enough integration tests on user-visible behavior.

---

## Lesson 31 — TypeScript with React

### C1
**What does TypeScript add beyond just catching type errors? What are the 3 most valuable benefits in a React codebase?**

**Answer:** (1) **Self-documenting component APIs** — a component's props interface is its contract; new engineers immediately see what a component accepts without reading its source or documentation; (2) **Refactor safety** — rename a prop in the interface and TypeScript points to every usage that needs updating; without it, renaming a prop silently breaks consumers; (3) **IDE intelligence** — autocomplete for props, Go-to-definition, Find-all-references — these multiply developer speed on large codebases. Beyond these: discriminated unions for state machines, generics for reusable components, and mapped types for deriving types from API contracts.

### C2
**What is the difference between `React.FC<Props>` and just `(props: Props) => JSX.Element`? Why do many modern TS-React codebases avoid `React.FC`?**

**Answer:** `React.FC<Props>` (formerly `React.FunctionComponent`) used to implicitly include `children` in the props type, making every component accept children whether it should or not — this was a safety hole removed in React 18's type update. It also has a slightly more verbose generic syntax. The modern approach is simpler: `function MyComponent({ name }: { name: string }) { ... }` or with a named interface: `function MyComponent({ name }: Props) { ... }`. This is explicit about whether the component accepts children (`children: React.ReactNode`), and TypeScript infers the return type correctly. `React.FC` is not wrong, but the plain function approach is preferred in most modern codebases.

### B1
**Add complete TypeScript types to a `ProductCard` component including props, event handlers, and a union type for the `status` field.**

```tsx
type ProductStatus = 'in-stock' | 'out-of-stock' | 'discontinued';

interface ProductCardProps {
  id: number;
  name: string;
  price: number;
  status: ProductStatus;
  imageUrl?: string;               // optional
  onAddToCart: (id: number) => void;
  onViewDetails?: (id: number) => void; // optional
}

function ProductCard({
  id,
  name,
  price,
  status,
  imageUrl,
  onAddToCart,
  onViewDetails,
}: ProductCardProps) {
  const STATUS_LABELS: Record<ProductStatus, string> = {
    'in-stock': 'In Stock',
    'out-of-stock': 'Out of Stock',
    'discontinued': 'Discontinued',
  };

  return (
    <div>
      {imageUrl && <img src={imageUrl} alt={name} />}
      <h2>{name}</h2>
      <p>${price.toFixed(2)}</p>
      <p>{STATUS_LABELS[status]}</p>
      <button
        onClick={() => onAddToCart(id)}
        disabled={status !== 'in-stock'}
      >
        Add to Cart
      </button>
      {onViewDetails && (
        <button onClick={() => onViewDetails(id)}>View Details</button>
      )}
    </div>
  );
}
```

### B2
**Type a `useLocalStorage` custom hook with generics so it can store any type-safe value.**

```tsx
import { useState, useEffect } from 'react';

function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T | ((prev: T) => T)) => void] {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item !== null ? (JSON.parse(item) as T) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    try {
      window.localStorage.setItem(key, JSON.stringify(storedValue));
    } catch {
      // ignore storage errors
    }
  }, [key, storedValue]);

  return [storedValue, setStoredValue];
}

// Usage — TypeScript infers the type from initialValue
const [theme, setTheme] = useLocalStorage<'light' | 'dark'>('theme', 'light');
const [cart, setCart] = useLocalStorage<CartItem[]>('cart', []);
```

### D1
**Fix the TypeScript error in this event handler.**

```tsx
function SearchInput() {
  const [query, setQuery] = useState('');

  function handleChange(e) { // Error: 'e' implicitly has type 'any'
    setQuery(e.target.value);
  }

  return <input value={query} onChange={handleChange} />;
}
```

```tsx
// Fixed — explicitly type the event
function handleChange(e: React.ChangeEvent<HTMLInputElement>) {
  setQuery(e.target.value);
}

// Alternatively, let TypeScript infer it from the JSX
// by inlining the handler:
<input value={query} onChange={e => setQuery(e.target.value)} />
// TypeScript infers e: React.ChangeEvent<HTMLInputElement> from the JSX context
```

### A1
**How strict should TypeScript be in a React project? What is the tradeoff between strict mode and development speed?**

**Answer:** Always enable `"strict": true` in `tsconfig.json` from the start of a project — retrofitting strict mode onto an existing codebase is painful and disruptive. Strict mode enables: `strictNullChecks` (most valuable — prevents `undefined is not an object`), `noImplicitAny`, `strictFunctionTypes`. Additional strict options worth enabling: `noUncheckedIndexedAccess` (array access returns `T | undefined`, not just `T`) and `exactOptionalPropertyTypes`. The development speed tradeoff is real but front-loaded: writing types carefully at the start costs 10-20% more time per component. The savings are back-loaded: fewer runtime bugs in production, faster and safer refactoring, and better IDE support that accelerates all future work. The ROI is strongly positive for any project that will last more than a few weeks.

---

## Lesson 32 — Advanced TypeScript with React

### C1
**What is a discriminated union and why is it specifically valuable for typing React component state?**

**Answer:** A discriminated union is a union type where each member has a shared "discriminant" literal property that TypeScript uses to narrow the type. Example for async state:
```ts
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };
```
When you check `if (state.status === 'success')`, TypeScript knows `state.data` exists and is of type `T`. When `status === 'error'`, TypeScript knows `state.error` exists. This eliminates impossible states (you can't have both `data` and `error`) and forces exhaustive handling — if you add a new status, TypeScript flags every switch statement that doesn't handle it.

### C2
**What are generic components and when do you need them?**

**Answer:** A generic component is one that can work with different data types while maintaining type safety. Without generics, a `<List items={...} />` component might type `items` as `any[]`. With generics:
```tsx
function List<T extends { id: string | number }>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}) { ... }
```
Now TypeScript infers `T` from the `items` array and ensures `renderItem` receives the correct type. Need generics when: your component is a structural abstraction over typed data (tables, lists, select dropdowns, virtualized lists) where the component provides layout and behavior but the data type varies per usage.

### B1
**Type a `useReducer`-based state machine using discriminated unions so each action only accepts valid payloads.**

```tsx
type CartItem = { id: number; name: string; price: number; quantity: number };

type CartState = {
  items: CartItem[];
  total: number;
};

type CartAction =
  | { type: 'ADD_ITEM'; payload: Omit<CartItem, 'quantity'> }
  | { type: 'REMOVE_ITEM'; payload: { id: number } }
  | { type: 'UPDATE_QUANTITY'; payload: { id: number; quantity: number } }
  | { type: 'CLEAR' };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(i => i.id === action.payload.id);
      const items = existing
        ? state.items.map(i =>
            i.id === action.payload.id ? { ...i, quantity: i.quantity + 1 } : i
          )
        : [...state.items, { ...action.payload, quantity: 1 }];
      const total = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
      return { items, total };
    }
    case 'REMOVE_ITEM':
      const items = state.items.filter(i => i.id !== action.payload.id);
      return { items, total: items.reduce((s, i) => s + i.price * i.quantity, 0) };
    case 'UPDATE_QUANTITY':
      // TypeScript knows action.payload has { id, quantity } here
      const updated = state.items.map(i =>
        i.id === action.payload.id ? { ...i, quantity: action.payload.quantity } : i
      );
      return { items: updated, total: updated.reduce((s, i) => s + i.price * i.quantity, 0) };
    case 'CLEAR':
      return { items: [], total: 0 };
  }
}
```

### B2
**Build a generic `Select` component that is type-safe — the `onChange` callback receives the same type as the items, not just a string.**

```tsx
interface SelectProps<T> {
  options: T[];
  value: T | null;
  onChange: (value: T) => void;
  getLabel: (item: T) => string;
  getValue: (item: T) => string | number;
}

function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  function handleChange(e: React.ChangeEvent<HTMLSelectElement>) {
    const selectedValue = e.target.value;
    const selected = options.find(o => String(getValue(o)) === selectedValue);
    if (selected !== undefined) onChange(selected);
  }

  return (
    <select
      value={value !== null ? String(getValue(value)) : ''}
      onChange={handleChange}
    >
      <option value="">Select...</option>
      {options.map(option => (
        <option key={String(getValue(option))} value={String(getValue(option))}>
          {getLabel(option)}
        </option>
      ))}
    </select>
  );
}

// Usage — fully type-safe
type Country = { code: string; name: string };
const [country, setCountry] = useState<Country | null>(null);

<Select<Country>
  options={countries}
  value={country}
  onChange={setCountry}           // TypeScript knows setCountry receives Country
  getLabel={c => c.name}
  getValue={c => c.code}
/>
```

### D1
**Fix the TypeScript narrowing issue.**

```tsx
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; side: number };

function getArea(shape: Shape): number {
  if (shape.kind === 'circle') {
    return Math.PI * shape.radius ** 2;
  }
  return shape.side ** 2; // TypeScript error: Property 'side' does not exist on type 'Shape'
}
```

**The issue is subtle — this actually WORKS in TypeScript!** The common mistake is forgetting to add the discriminant property, which breaks narrowing:

```tsx
// This breaks narrowing — no discriminant
type BadShape =
  | { radius: number }   // no 'kind'
  | { side: number };

function getArea(shape: BadShape): number {
  if ('radius' in shape) { // must use 'in' instead of discriminant
    return Math.PI * shape.radius ** 2;
  }
  return shape.side ** 2;
}

// Correct pattern — always include a discriminant
type GoodShape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; side: number };
```

### A1
**When should you use `type` vs. `interface` in a React TypeScript project? Is there a meaningful difference?**

**Answer:** Functionally, they are nearly identical for most React use cases. The meaningful differences: (1) `interface` supports declaration merging — you can re-open an interface to add properties across files; `type` cannot be re-opened; (2) `type` supports unions, intersections, mapped types, and conditional types — things `interface` cannot do; (3) error messages for `interface` are sometimes cleaner in IDE tooling. Pragmatic convention: use `interface` for component props and class contracts (the things you design as explicit APIs); use `type` for unions, intersections, computed types, and utility types. Consistency within a codebase matters more than the choice itself — pick one convention for props and stick to it.

---

## Lesson 33 — `useSyncExternalStore`

### C1
**What problem does `useSyncExternalStore` solve that `useState` + `useEffect` cannot reliably solve in React 18's Concurrent Mode?**

**Answer:** In Concurrent Mode, React can pause a render and resume it later, or render a component multiple times before committing. If you subscribe to an external store inside a `useEffect` and read its value in render, there is a window between the render (which reads the store value) and the effect (which subscribes to updates) where the store can update — causing the component to display a stale value that's inconsistent with what the store actually holds. This is called "tearing" — different parts of the UI showing different versions of the same data. `useSyncExternalStore` guarantees that the subscribed value is consistent throughout a single render by reading and subscribing synchronously.

### C2
**What are the three arguments to `useSyncExternalStore` and what does each do?**

**Answer:** `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)`: (1) **`subscribe(callback)`**: a function that registers the `callback` to be called whenever the store changes — must return an unsubscribe function; (2) **`getSnapshot()`**: a function React calls synchronously to read the current value from the store — must return the same value if the store hasn't changed (by reference equality for objects); (3) **`getServerSnapshot()`** (optional): same as `getSnapshot` but called during SSR — typically returns the initial/default value since the browser store isn't available on the server.

### B1
**Build a `useOnlineStatus` hook using `useSyncExternalStore` that subscribes to the browser's `navigator.onLine` and updates components when the network status changes.**

```jsx
import { useSyncExternalStore } from 'react';

function useOnlineStatus() {
  return useSyncExternalStore(
    (callback) => {
      window.addEventListener('online', callback);
      window.addEventListener('offline', callback);
      return () => {
        window.removeEventListener('online', callback);
        window.removeEventListener('offline', callback);
      };
    },
    () => navigator.onLine,
    () => true  // server snapshot — assume online during SSR
  );
}

function NetworkIndicator() {
  const isOnline = useOnlineStatus();
  return (
    <span style={{ color: isOnline ? 'green' : 'red' }}>
      {isOnline ? '● Online' : '○ Offline'}
    </span>
  );
}
```

### B2
**Build a simple observable store (publish/subscribe) and subscribe to it with `useSyncExternalStore`.**

```jsx
import { useSyncExternalStore } from 'react';

// Observable store — can live outside React
function createStore(initialState) {
  let state = initialState;
  const subscribers = new Set();

  return {
    getState: () => state,
    setState: (updater) => {
      state = typeof updater === 'function' ? updater(state) : updater;
      subscribers.forEach(fn => fn());
    },
    subscribe: (fn) => {
      subscribers.add(fn);
      return () => subscribers.delete(fn);
    },
  };
}

const themeStore = createStore({ theme: 'light' });

function useThemeStore() {
  return useSyncExternalStore(
    themeStore.subscribe,
    themeStore.getState,
    themeStore.getState
  );
}

function ThemeConsumer() {
  const { theme } = useThemeStore();
  return (
    <div>
      <p>Current theme: {theme}</p>
      <button onClick={() =>
        themeStore.setState(s => ({
          theme: s.theme === 'light' ? 'dark' : 'light',
        }))
      }>
        Toggle Theme
      </button>
    </div>
  );
}
```

### D1
**The component throws "The result of getSnapshot should be cached" warning. Why?**

```jsx
function useWindowSize() {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener('resize', cb);
      return () => window.removeEventListener('resize', cb);
    },
    () => ({ width: window.innerWidth, height: window.innerHeight }) // new object every call!
  );
}
```

**Bug:** `getSnapshot` returns a new object literal on every call. React checks if the snapshot changed by comparing the previous and new return values with `Object.is`. New objects fail this check even if the dimensions are the same — causing infinite re-renders.

```jsx
// Fix — return primitives, or cache the object
let cachedSize = { width: window.innerWidth, height: window.innerHeight };

function getSnapshot() {
  const w = window.innerWidth;
  const h = window.innerHeight;
  if (cachedSize.width !== w || cachedSize.height !== h) {
    cachedSize = { width: w, height: h };
  }
  return cachedSize;
}
```

### A1
**When should you use `useSyncExternalStore` vs `useState`/`useEffect`? What types of external systems benefit most?**

**Answer:** Use `useSyncExternalStore` for subscriptions to external stores that exist outside React's state system and update independently: browser APIs (`localStorage`, `navigator.onLine`, `matchMedia`, `history`), third-party state managers that don't natively integrate with React, WebSocket message streams, and any shared mutable state that multiple components subscribe to. The key signal: if you're writing `useEffect(() => { const unsub = store.subscribe(() => setState(store.getState())); return unsub; }, [])` — that pattern is exactly what `useSyncExternalStore` is designed to replace, more safely. Don't use it for data fetched from APIs (use TanStack Query), for local component state (use `useState`), or for state shared via React (use Context + `useReducer`).

---

## Lesson 34 — React 19 Actions

### C1
**What is the mental model shift that React 19 Actions introduce? How is `useActionState` different from managing loading and error state manually?**

**Answer:** The shift: React 19 treats form submissions as first-class operations with built-in state management. Previously: `const [isPending, setIsPending] = useState(false); const [error, setError] = useState(null);` — manually tracking mutation state. With Actions, you pass an async function as a form's `action` attribute. `useActionState(action, initialState)` wraps it: it returns `[state, formAction, isPending]` — where `state` is whatever your action function returned (data or error), `formAction` is the action to pass to the form, and `isPending` is true while the action is running. React handles the pending/success/error lifecycle automatically — you just describe what the action does.

### C2
**What does `useOptimistic` do and when is it safe to use it?**

**Answer:** `useOptimistic(state, updateFn)` returns `[optimisticState, addOptimistic]`. While an action is pending, `optimisticState` shows the optimistically updated value (produced by `updateFn`); once the action settles, `optimisticState` reverts to the actual `state`. It is safe when: the optimistic update is visually non-destructive and quick to apply — adding a like, sending a message, toggling a setting. It requires a rollback strategy: if the server rejects the action, the optimistic value is automatically discarded and the real state is shown. Do not use for operations where the user would be confused by a brief incorrect value (financial transactions, permission changes).

### B1
**Build a comment submission form using `useActionState`. The form should show a pending state while submitting and display the server's response.**

```jsx
import { useActionState } from 'react';

async function submitComment(prevState, formData) {
  const comment = formData.get('comment');

  try {
    const res = await fetch('/api/comments', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: comment }),
    });
    if (!res.ok) throw new Error('Submission failed');
    const data = await res.json();
    return { success: true, data, error: null };
  } catch (err) {
    return { success: false, data: null, error: err.message };
  }
}

function CommentForm() {
  const [state, formAction, isPending] = useActionState(submitComment, {
    success: false,
    data: null,
    error: null,
  });

  return (
    <form action={formAction}>
      <textarea name="comment" required rows={4} placeholder="Write a comment..." />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Submitting...' : 'Submit Comment'}
      </button>
      {state.error && <p style={{ color: 'red' }}>{state.error}</p>}
      {state.success && <p style={{ color: 'green' }}>Comment submitted!</p>}
    </form>
  );
}
```

### B2
**Build a like button with `useOptimistic` that instantly toggles the UI while the server request is in flight.**

```jsx
import { useState, useOptimistic, useTransition } from 'react';

async function toggleLike(postId, currentLiked) {
  const res = await fetch(`/api/posts/${postId}/like`, {
    method: currentLiked ? 'DELETE' : 'POST',
  });
  if (!res.ok) throw new Error('Failed to toggle like');
  return res.json();
}

function LikeButton({ postId, initialLiked, initialCount }) {
  const [liked, setLiked] = useState(initialLiked);
  const [count, setCount] = useState(initialCount);
  const [isPending, startTransition] = useTransition();

  const [optimisticLiked, setOptimisticLiked] = useOptimistic(liked);
  const [optimisticCount, setOptimisticCount] = useOptimistic(count);

  function handleClick() {
    startTransition(async () => {
      setOptimisticLiked(!liked);
      setOptimisticCount(liked ? count - 1 : count + 1);

      try {
        const result = await toggleLike(postId, liked);
        setLiked(result.liked);
        setCount(result.count);
      } catch {
        // optimistic state reverts automatically on error
      }
    });
  }

  return (
    <button onClick={handleClick} disabled={isPending}>
      {optimisticLiked ? '❤️' : '🤍'} {optimisticCount}
    </button>
  );
}
```

### D1
**The form doesn't reset after submission. Why, and how do you fix it with React 19 Actions?**

```jsx
const [state, formAction, isPending] = useActionState(submitForm, null);

return (
  <form action={formAction}>
    <input name="title" />
    <button type="submit">Submit</button>
  </form>
);
```

**Issue:** HTML forms with `action` as a function *do* reset automatically in React 19 when the action completes. If the form is NOT resetting, it is likely because the `action` is not being called as a React 19 form action — perhaps the form is using `onSubmit` instead.

```jsx
// Correct — use 'action' not 'onSubmit' for React 19 Actions
<form action={formAction}>  {/* resets automatically */}
  <input name="title" />
  <button type="submit">Submit</button>
</form>

// If you need custom reset behavior:
const formRef = useRef(null);
// In the action:
async function myAction(prevState, formData) {
  await submitToServer(formData);
  // form resets automatically; if you need imperatively:
  // formRef.current.reset();
  return { success: true };
}
```

### A1
**When should you use React 19 Actions vs TanStack Query mutations? Can they work together?**

**Answer:** React 19 Actions and TanStack Query mutations solve overlapping but distinct problems. Actions are ideal for: form submissions where the browser's native form reset behavior is valuable, Server Actions in framework contexts (Next.js), and simple mutations where cache invalidation is not needed. TanStack Query mutations are better for: mutations that must invalidate or update a query cache (the vast majority of data mutations in a real app), optimistic updates tied to cached data, complex mutation sequences with rollback, and background retry logic. They work together: inside a TanStack Query `useMutation.mutationFn`, you can call a React 19 Server Action. Or: use `useActionState` for the form UI, and inside the action function, call `queryClient.invalidateQueries` to sync the cache. The combination gives you native form behavior AND cache consistency.

---

## Lesson 35 — SSR and Hydration

### C1
**What is hydration and what exactly happens during it?**

**Answer:** Hydration is the process of React attaching event listeners and state to server-rendered HTML. The server sends a fully rendered HTML page — the browser displays it immediately (fast First Contentful Paint). React then loads, runs, and "hydrates" the existing HTML by: traversing the DOM, finding elements that match the component tree it expects, and attaching event handlers and React state to them — without re-creating or replacing the DOM nodes. The result: a static HTML page becomes a fully interactive React application. Hydration requires that the server-rendered HTML matches exactly what React would render on the client — any mismatch causes a hydration error and React falls back to re-rendering, losing the SSR performance benefit.

### C2
**What causes a hydration mismatch and what are the most common sources?**

**Answer:** A mismatch occurs when the HTML React received from the server differs from what React would render on the client. Common sources: (1) **Date/time rendering** — `new Date().toLocaleDateString()` returns different values in different timezones — the server and client may be in different zones; (2) **`Math.random()` or UUID generation** in render — different values each call; (3) **`typeof window !== 'undefined'` checks** — rendering different content when `window` is available; (4) **Browser extensions** that inject HTML; (5) **Incorrect HTML nesting** (e.g., a `<p>` inside another `<p>`) — the browser silently corrects it, creating a DOM that React didn't expect. Fix: ensure any client-only rendering happens in a `useEffect` (after hydration), or use `suppressHydrationWarning` for intentionally different content.

### B1
**Build a `ClientOnly` wrapper component that renders its children only on the client, after hydration — preventing hydration mismatches for browser-only content.**

```jsx
import { useState, useEffect } from 'react';

function ClientOnly({ children, fallback = null }) {
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  if (!mounted) return fallback;
  return children;
}

// Usage — anything that reads browser APIs at render time
function App() {
  return (
    <div>
      <h1>My App</h1>
      <ClientOnly fallback={<p>Loading...</p>}>
        <p>Window width: {window.innerWidth}px</p>
        <p>User timezone: {Intl.DateTimeFormat().resolvedOptions().timeZone}</p>
      </ClientOnly>
    </div>
  );
}
```

### B2
**Fix a hydration mismatch caused by a random ID being generated in render.**

```jsx
// Broken — different IDs on server and client
function CommentSection() {
  const sessionId = Math.random().toString(36); // new value every render!
  return <div data-session={sessionId}>Comments</div>;
}
```

```jsx
// Fixed — generate the ID only on the client, after mount
import { useState, useEffect, useId } from 'react';

function CommentSection() {
  const stableId = useId(); // same on server and client — no mismatch

  // For truly random values needed only client-side:
  const [sessionId, setSessionId] = useState(null);
  useEffect(() => {
    setSessionId(Math.random().toString(36)); // runs only after hydration
  }, []);

  return (
    <div data-id={stableId} data-session={sessionId ?? ''}>
      Comments
    </div>
  );
}
```

### D1
**React is throwing a hydration error. Diagnose and fix this component.**

```jsx
function Greeting() {
  return (
    <p>
      Current time: {new Date().toLocaleTimeString()}
    </p>
  );
}
```

**Bug:** `new Date().toLocaleTimeString()` produces a different string between server render and client hydration (even milliseconds apart). React's SSR output shows one time; the client renders a different time.

```jsx
// Fixed — render static content on the server, dynamic content after hydration
import { useState, useEffect } from 'react';

function Greeting() {
  const [time, setTime] = useState(null);

  useEffect(() => {
    setTime(new Date().toLocaleTimeString());
    const timer = setInterval(
      () => setTime(new Date().toLocaleTimeString()),
      1000
    );
    return () => clearInterval(timer);
  }, []);

  return <p>Current time: {time ?? 'Loading...'}</p>;
}
```

### A1
**What should be server-rendered vs. client-only rendered? What is the decision framework?**

**Answer:** Server-render: (1) content critical for SEO (page title, meta, main copy); (2) content that is meaningful without JavaScript (the page should be readable before JS loads); (3) content that is the same for all users and doesn't depend on browser state. Client-only render (with `useEffect` or a `ClientOnly` wrapper): (1) content that depends on browser APIs (`window`, `navigator`, `localStorage`); (2) personalized content (user's name, avatar) that would cause a mismatch since the server doesn't know the logged-in user at render time (unless using cookie-based SSR with auth); (3) real-time content (timestamps, live counts) that changes continuously; (4) heavy interactive widgets where the SSR HTML is a placeholder. The rule: if it can be meaningful without JavaScript, SSR it. If it requires the browser to exist or the user to be identified, client-render it.

---

## Lesson 36 — React Server Components

### C1
**What is the fundamental difference between a React Server Component and a Client Component?**

**Answer:** A **Server Component** runs *only* on the server (or at build time). It can be `async`, it can directly `await` database queries or API calls, it has access to server-side secrets and the filesystem, and it sends its output (serialized React elements, not JavaScript) to the client. It cannot use `useState`, `useEffect`, event handlers, browser APIs, or any client-side interactivity. A **Client Component** (marked with `'use client'`) runs on the client (and also on the server during SSR/hydration). It can use all hooks, handle events, access browser APIs, but cannot `await` on the server or access server-side resources. The key insight: RSCs reduce JavaScript sent to the client — a component that fetches and renders a list never ships its fetch logic to the browser.

### C2
**What does the Server/Client boundary mean for the component tree? Can a Server Component render a Client Component? Can a Client Component render a Server Component?**

**Answer:** A Server Component *can* render a Client Component — it passes it as a child or as a prop. The Client Component boundary is one-directional: once you cross into a Client Component (`'use client'`), everything it imports is also client-side. A Client Component *cannot* directly import and render a Server Component — once in client-land, you stay there. However, a Client Component *can* receive a Server Component's output as `children` — because `children` is just React elements that were already rendered on the server. Pattern: `<ClientLayout>{/* Server Component output */}</ClientLayout>`. This "passing RSC as children" is how you embed server-rendered content inside an interactive client shell.

### B1
**Build a `ProductPage` with a Server Component that fetches product data directly (no API route needed) and renders it, with a Client Component `AddToCartButton` island for interactivity.**

```tsx
// ProductPage.tsx — Server Component (no 'use client')
import { AddToCartButton } from './AddToCartButton';

async function getProduct(id: string) {
  // Direct database access — never shipped to client
  const product = await db.products.findUnique({ where: { id } });
  return product;
}

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);

  if (!product) return <p>Product not found.</p>;

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <p>${product.price.toFixed(2)}</p>
      {/* Client Component island — receives serializable props only */}
      <AddToCartButton productId={product.id} productName={product.name} />
    </div>
  );
}

// AddToCartButton.tsx — Client Component
'use client';
import { useState } from 'react';

export function AddToCartButton({ productId, productName }: {
  productId: string;
  productName: string;
}) {
  const [added, setAdded] = useState(false);

  return (
    <button onClick={() => setAdded(true)} disabled={added}>
      {added ? `${productName} added!` : 'Add to Cart'}
    </button>
  );
}
```

### B2
**Demonstrate the "RSC as children" pattern — a Client Component shell that receives Server Component output as children.**

```tsx
// Shell.tsx — Client Component
'use client';
import { useState } from 'react';

export function CollapsibleSection({
  title,
  children,
}: {
  title: string;
  children: React.ReactNode;
}) {
  const [open, setOpen] = useState(true);
  return (
    <section>
      <button onClick={() => setOpen(o => !o)}>{title} {open ? '▲' : '▼'}</button>
      {open && <div>{children}</div>}
    </section>
  );
}

// Page.tsx — Server Component
import { CollapsibleSection } from './Shell';

async function getAnalytics() {
  return await db.analytics.getLatest(); // server-only
}

export default async function Page() {
  const analytics = await getAnalytics();
  return (
    // Client Component wraps Server-rendered content as children
    <CollapsibleSection title="Analytics">
      <p>Page views: {analytics.views}</p>
      <p>Unique visitors: {analytics.uniqueVisitors}</p>
    </CollapsibleSection>
  );
}
```

### D1
**This Server Component is throwing "useState is not defined". Fix it.**

```tsx
// Dashboard.tsx
async function Dashboard() {
  const [isOpen, setIsOpen] = useState(false); // Error!
  const data = await fetchDashboardData();
  return <div>{/* ... */}</div>;
}
```

**Bug:** `useState` cannot be used in a Server Component — server components are async functions with no client-side lifecycle.

```tsx
// Fix — extract interactive part to a Client Component
'use client';
// Sidebar.tsx
import { useState } from 'react';
export function CollapsibleSidebar({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <aside>
      <button onClick={() => setIsOpen(o => !o)}>Toggle</button>
      {isOpen && children}
    </aside>
  );
}

// Dashboard.tsx — Server Component
import { CollapsibleSidebar } from './Sidebar';
async function Dashboard() {
  const data = await fetchDashboardData();
  return (
    <CollapsibleSidebar>
      <p>Server-rendered content: {data.title}</p>
    </CollapsibleSidebar>
  );
}
```

### A1
**What is the decision framework for choosing Server Component vs. Client Component for a new component?**

**Answer:** Start with Server Component as the default — it produces less client JavaScript and can fetch data directly. Make it a Client Component only when it needs: (1) `useState`, `useReducer`, `useContext` — any React state; (2) `useEffect` — any lifecycle side effect; (3) event handlers (`onClick`, `onChange`); (4) browser APIs (`window`, `localStorage`, `navigator`); (5) third-party libraries that use client-only APIs. The signal: if you write `'use client'`, ask "does this component *need* to be interactive, or can I extract the interactive part to a smaller child and keep the data-fetching parent as a server component?" The goal is to push `'use client'` as far down the tree as possible, keeping only truly interactive "leaf" components as client components.

---

## Lesson 37 — Server Functions (Server Actions)

### C1
**What is a Server Function and how is it architecturally different from a REST API endpoint?**

**Answer:** A Server Function (declared with `'use server'`) is an async function defined on the server that can be called from the client. The framework (Next.js, Remix) automatically creates an HTTP endpoint behind the scenes — the client calls the function like a normal JavaScript function, but the call serializes to an HTTP POST under the hood. The architectural difference from REST API: (1) no manually written endpoint code — the framework handles routing, serialization, and security; (2) TypeScript types flow directly from server to client — no manual interface duplication; (3) can be co-located with the component that uses them; (4) no need to think about URL design. The tradeoff: less control over the HTTP contract — you can't set custom status codes or headers as easily as with a REST endpoint.

### C2
**What security considerations are unique to Server Functions that don't apply to client-side code?**

**Answer:** (1) **Any client can call them** — Server Functions are HTTP endpoints; a malicious actor can call your Server Function with crafted data even if your UI never presents that option. Always validate and authorize server-side — never trust client input; (2) **Authorization must be checked inside the function** — don't rely on the calling component having checked permissions; (3) **Sensitive data exposure** — Server Functions can access database connections, API keys, and secrets. Never return more data than the client needs — a leaked field in the return value is as bad as a REST API that over-fetches; (4) **CSRF** — framework implementations typically include CSRF tokens automatically, but verify for your specific setup; (5) **Input sanitization** — all `FormData` values are strings; parse and validate before using.

### B1
**Build a Server Function for creating a new task with validation and error handling. Call it from a Client Component form.**

```tsx
// actions/tasks.ts
'use server';
import { z } from 'zod'; // validation
import { revalidatePath } from 'next/cache';
import { auth } from '@/lib/auth';

const CreateTaskSchema = z.object({
  title: z.string().min(1, 'Title is required').max(200),
  priority: z.enum(['low', 'medium', 'high']),
});

export async function createTask(formData: FormData) {
  // Authorization
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  // Validation
  const parsed = CreateTaskSchema.safeParse({
    title: formData.get('title'),
    priority: formData.get('priority') ?? 'medium',
  });

  if (!parsed.success) {
    return { success: false, errors: parsed.error.flatten().fieldErrors };
  }

  // Database
  await db.tasks.create({
    data: { ...parsed.data, userId: session.user.id },
  });

  revalidatePath('/tasks'); // invalidate Next.js cache
  return { success: true, errors: null };
}

// components/CreateTaskForm.tsx
'use client';
import { useActionState } from 'react';
import { createTask } from '@/actions/tasks';

export function CreateTaskForm() {
  const [state, action, isPending] = useActionState(createTask, null);

  return (
    <form action={action}>
      <input name="title" placeholder="Task title" />
      {state?.errors?.title && <p style={{ color: 'red' }}>{state.errors.title[0]}</p>}
      <select name="priority">
        <option value="low">Low</option>
        <option value="medium">Medium</option>
        <option value="high">High</option>
      </select>
      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create Task'}
      </button>
      {state?.success && <p>Task created!</p>}
    </form>
  );
}
```

### B2
**Write a Server Function that deletes a resource with proper authorization — ensuring the user owns the resource before deleting.**

```tsx
'use server';
import { auth } from '@/lib/auth';
import { revalidatePath } from 'next/cache';

export async function deletePost(postId: string) {
  const session = await auth();
  if (!session) throw new Error('Unauthorized');

  // Fetch the post to verify ownership — never trust client-provided userId
  const post = await db.posts.findUnique({ where: { id: postId } });

  if (!post) throw new Error('Post not found');
  if (post.userId !== session.user.id) throw new Error('Forbidden');

  await db.posts.delete({ where: { id: postId } });
  revalidatePath('/posts');

  return { success: true };
}
```

### D1
**This Server Function has a critical security vulnerability. Find it.**

```tsx
'use server';

export async function updateUserRole(userId: string, newRole: string) {
  // Missing: auth check!
  await db.users.update({
    where: { id: userId },
    data: { role: newRole },
  });
}
```

**Vulnerability:** No authentication or authorization check. Any unauthenticated client can call this Server Function (it is an HTTP endpoint) and promote any user to any role, including admin.

```tsx
// Fixed
'use server';
import { auth } from '@/lib/auth';

export async function updateUserRole(userId: string, newRole: string) {
  const session = await auth();

  // Only authenticated admins can change roles
  if (!session || session.user.role !== 'admin') {
    throw new Error('Forbidden');
  }

  // Validate the new role against allowed values
  const ALLOWED_ROLES = ['user', 'moderator', 'admin'] as const;
  if (!ALLOWED_ROLES.includes(newRole as any)) {
    throw new Error('Invalid role');
  }

  await db.users.update({
    where: { id: userId },
    data: { role: newRole },
  });
}
```

### A1
**When should you use a Server Function vs. a traditional REST API endpoint? What are the tradeoffs of each?**

**Answer:** Server Functions shine for: form submissions in Next.js App Router, mutations co-located with the UI they modify, full-stack TypeScript with end-to-end type safety without manual API interfaces, and operations that pair with framework caching (`revalidatePath`). REST API endpoints are better for: public APIs consumed by third parties or mobile apps; operations requiring custom HTTP semantics (streaming responses, specific status codes, custom headers, file uploads with `multipart/form-data`); APIs shared between multiple frontends; and when you need versioning (`/api/v1/`). REST also has better tooling for API documentation (OpenAPI), rate limiting at the HTTP layer, and integration testing. The pragmatic rule: use Server Functions for internal form mutations in a Next.js app; use REST API routes for any endpoint consumed outside the Next.js frontend.

---

## Lesson 38 — React Router Data APIs

### C1
**What is a `loader` in React Router v6.4+ and how does it change the data fetching mental model?**

**Answer:** A `loader` is an async function associated with a route that runs *before* the component renders. When a user navigates to `/posts/42`, React Router calls the loader, waits for it to complete, and only then renders the component — with the loaded data available via `useLoaderData()`. This eliminates the loading-state-inside-component pattern entirely for initial data: no `isLoading`, no `useEffect` fetch, no empty state. The data is ready when the component mounts. This is "fetch as you navigate" — the loading happens during the navigation transition, not after the component renders. The result: no loading spinner inside the component for its primary data; instead, the previous route stays visible (or a route-level pending indicator shows) while loading.

### C2
**What is a route `action` in React Router v6.4+ and how does it differ from a Server Function or `useMutation`?**

**Answer:** A route `action` is a function that handles non-GET form submissions for a route. When a `<Form method="post">` submits, React Router calls the action with the request. After the action completes, React Router automatically re-runs all active loaders (to reflect the data change) and re-renders the components. The difference from `useMutation`: TanStack Query mutations are client-side; you call `queryClient.invalidateQueries` manually. React Router actions are route-level; React Router revalidates all loaders automatically. The difference from Server Functions: actions run on the client (in CSR apps) — they're not server-only; they are route handlers for form submissions. In Remix, actions *can* be server-side. In browser React Router, they run in the browser.

### B1
**Build a route with a loader that fetches a user, and render the data with `useLoaderData`. No loading state needed in the component.**

```jsx
// router.jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

async function userLoader({ params }) {
  const res = await fetch(`/api/users/${params.userId}`);
  if (!res.ok) throw new Response('Not Found', { status: 404 });
  return res.json();
}

const router = createBrowserRouter([
  {
    path: '/users/:userId',
    loader: userLoader,
    element: <UserPage />,
    errorElement: <ErrorPage />,
  },
]);

// UserPage.jsx — no loading state, data is always ready
import { useLoaderData } from 'react-router-dom';

function UserPage() {
  const user = useLoaderData(); // always defined — loader ran before render

  return (
    <div>
      <h1>{user.name}</h1>
      <p>{user.email}</p>
    </div>
  );
}

function App() {
  return <RouterProvider router={router} />;
}
```

### B2
**Add a route `action` that handles a form submission to create a post, then reloads the post list automatically.**

```jsx
import { Form, redirect, useLoaderData } from 'react-router-dom';

async function postsLoader() {
  const res = await fetch('/api/posts');
  return res.json();
}

async function createPostAction({ request }) {
  const formData = await request.formData();
  const title = formData.get('title');

  const res = await fetch('/api/posts', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title }),
  });

  if (!res.ok) throw new Response('Failed to create post', { status: 500 });

  // redirect triggers loader revalidation automatically
  return redirect('/posts');
}

function PostsPage() {
  const posts = useLoaderData();

  return (
    <div>
      {/* React Router Form — triggers the action */}
      <Form method="post">
        <input name="title" placeholder="Post title" required />
        <button type="submit">Create Post</button>
      </Form>
      <ul>
        {posts.map(post => (
          <li key={post.id}>{post.title}</li>
        ))}
      </ul>
    </div>
  );
}

const router = createBrowserRouter([
  {
    path: '/posts',
    loader: postsLoader,
    action: createPostAction,
    element: <PostsPage />,
  },
]);
```

### D1
**The loader data is `undefined` in the component. Why?**

```jsx
// Route config
{ path: '/items', loader: itemsLoader, element: <ItemsPage /> }

function itemsLoader() {
  fetch('/api/items').then(r => r.json()); // missing return!
}

function ItemsPage() {
  const items = useLoaderData(); // undefined
}
```

**Bug:** The loader function doesn't `return` the Promise — it fires the fetch and returns `undefined`.

```jsx
// Fixed
async function itemsLoader() {
  const res = await fetch('/api/items');
  if (!res.ok) throw new Response('Server error', { status: 500 });
  return res.json(); // must return the data
}
```

### A1
**React Router loaders vs. TanStack Query — can they coexist? When would you use each in the same application?**

**Answer:** They coexist well and serve different purposes. React Router loaders are ideal for: initial route data that must be ready before the component renders, data tightly coupled to the route (the user's profile page always needs that user), and leveraging React Router's automatic revalidation after actions. TanStack Query is ideal for: data that updates in the background after initial load, data shared across multiple routes (shopping cart, user session), paginated or filtered data driven by client state, and any data that needs background refresh or cache management. A common pattern: use a loader to fetch the initial data, populate the TanStack Query cache in the loader, then use `useQuery` in the component — getting the "fetch before render" benefit of loaders with the "background refresh" benefit of TanStack Query.

---

## Lesson 39 — Code Splitting with `React.lazy` and Suspense

### C1
**What is code splitting and why does it improve the user experience for large applications?**

**Answer:** Code splitting divides the JavaScript bundle into smaller chunks that are loaded on demand instead of upfront. Without it, users download the entire application's JavaScript before any page renders — for large apps this can be megabytes, causing slow Time to Interactive even for pages that need only a fraction of that code. With code splitting: the initial bundle contains only what's needed for the first page; additional chunks are downloaded lazily when needed (when a route is visited, when a modal is opened). The benefit: faster initial load (smaller first bundle), faster Time to Interactive, and users only download code for features they actually use.

### C2
**What happens at the network level when `React.lazy(() => import('./HeavyChart'))` is used and the user navigates to a route that uses it?**

**Answer:** At build time, the bundler (Vite/webpack) splits `HeavyChart` and its dependencies into a separate chunk file (e.g., `HeavyChart-abc123.js`). At runtime: when the user navigates to the route, React renders the Suspense fallback, and simultaneously fires a network request for the chunk file. When the chunk loads (the import Promise resolves), React re-renders and shows the actual component. The loading time is the network round-trip for the chunk — which can be parallelized with data fetching if designed well. Prefetching (`<link rel="prefetch">` or `import()` during idle time) can eliminate this delay for anticipated navigations.

### B1
**Set up route-level code splitting for a 3-route application where each route's component is lazily loaded.**

```jsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

// Lazy load each route component
const Home      = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings  = lazy(() => import('./pages/Settings'));

function PageLoader() {
  return (
    <div style={{ display: 'flex', justifyContent: 'center', padding: 40 }}>
      Loading page...
    </div>
  );
}

const router = createBrowserRouter([
  { path: '/',          element: <Suspense fallback={<PageLoader />}><Home /></Suspense> },
  { path: '/dashboard', element: <Suspense fallback={<PageLoader />}><Dashboard /></Suspense> },
  { path: '/settings',  element: <Suspense fallback={<PageLoader />}><Settings /></Suspense> },
]);

// Or wrap the entire router in one Suspense:
function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <RouterProvider router={router} />
    </Suspense>
  );
}
```

### B2
**Implement prefetching — start loading a component's chunk on hover/focus of a link, before the user clicks.**

```jsx
import { lazy, Suspense } from 'react';

// Expose the import promise for prefetching
const importHeavyDashboard = () => import('./pages/HeavyDashboard');
const HeavyDashboard = lazy(importHeavyDashboard);

function NavLink({ to, prefetchImport, children }) {
  function handlePrefetch() {
    prefetchImport(); // starts loading the chunk on hover
  }

  return (
    <a
      href={to}
      onMouseEnter={handlePrefetch}
      onFocus={handlePrefetch}
    >
      {children}
    </a>
  );
}

function App() {
  return (
    <nav>
      <NavLink
        to="/dashboard"
        prefetchImport={importHeavyDashboard}
      >
        Dashboard
      </NavLink>
    </nav>
  );
}
```

### D1
**The lazy component causes a jarring flash of the fallback even for fast connections. Why and how do you fix it?**

```jsx
const Chart = lazy(() => import('./Chart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<p>Loading chart...</p>}>
          <Chart />
        </Suspense>
      )}
    </div>
  );
}
```

**The issue:** For fast connections, the fallback still flashes for a frame or two — perceptible and jarring. Fix: wrap the state update in `startTransition` so React waits for the lazy component to load before committing the transition, keeping the button visible instead of showing the fallback for fast loads.

```jsx
import { lazy, Suspense, useTransition, useState } from 'react';

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  const [isPending, startTransition] = useTransition();

  return (
    <div>
      <button
        onClick={() => startTransition(() => setShowChart(true))}
        disabled={isPending}
      >
        {isPending ? 'Loading...' : 'Show Chart'}
      </button>
      {showChart && (
        <Suspense fallback={<p>Loading chart...</p>}>
          <Chart />
        </Suspense>
      )}
    </div>
  );
}
```

### A1
**What is the right granularity for code splitting? Should you split every component or only routes?**

**Answer:** Route-level splitting is the default and usually sufficient — each page's code loads only when that page is visited. Below route-level, split when: (1) a component is large and used on a page that is frequently visited without needing the component (a modal that is rarely opened, an analytics dashboard embedded in a simpler page, a rich text editor for a "create" path that most users don't reach); (2) a third-party library (a chart library, a PDF renderer) is large and used by only a subset of the app. Do not split: small components (the network overhead of a separate chunk exceeds the bundle size savings), components that are always needed on first load, or shared utilities (splitting utility functions creates duplication). The test: does the Webpack/Vite bundle analysis show a component or library that is large and not needed on the initial route? Split it.

---

## Lesson 40 — React Performance Deep Dive

### C1
**What is the correct first step when a React application feels slow? Why is "add `memo` everywhere" the wrong first step?**

**Answer:** The correct first step is to **measure with the React DevTools Profiler**. Open DevTools, click "Profiler", record while performing the slow interaction, and examine which components rendered, how long each took, and why they rendered. "Add `memo` everywhere" is wrong because: (1) memo adds comparison overhead that can exceed the render cost for simple components; (2) it often doesn't work anyway — because inline objects/functions as props defeat shallow comparison; (3) it treats symptoms without diagnosing the cause — the slow component might be slow because of an expensive calculation inside it, not because of re-rendering frequency; (4) it adds maintenance burden and misleads future engineers about which components are genuinely expensive. Measure, find the actual bottleneck, fix specifically.

### C2
**What is `useLayoutEffect` and when must you use it instead of `useEffect`?**

**Answer:** `useLayoutEffect` fires synchronously *after* React updates the DOM but *before* the browser paints. `useEffect` fires asynchronously *after* the browser paints. Use `useLayoutEffect` when: you need to read DOM measurements (element dimensions, scroll position, bounding rect) and then immediately synchronize state or apply DOM mutations based on those measurements — if you use `useEffect` for this, the user sees a flash of the incorrect layout before your effect runs. Classic use cases: tooltip positioning (measure the trigger element, compute the tooltip position, set it before paint), synchronizing scroll positions, and measuring DOM nodes to apply animations. For everything else — data fetching, subscriptions, logging — use `useEffect`, which doesn't block the browser paint.

### B1
**Profile a component with a slow render. Use the React DevTools Profiler (describe the process) and then apply the correct optimization.**

```jsx
// This component is slow — find out why and fix it

import { useState, useMemo } from 'react';

// Expensive computation
function processData(data) {
  console.time('processData');
  const result = data.reduce((acc, item) => {
    // simulate expensive work
    for (let i = 0; i < 10000; i++) acc += item.value;
    return acc;
  }, 0);
  console.timeEnd('processData');
  return result;
}

function DataDisplay({ data }) {
  const [count, setCount] = useState(0);

  // BUG: runs on every render, even when count changes
  const processed = processData(data);

  return (
    <div>
      <p>Processed: {processed}</p>
      <button onClick={() => setCount(c => c + 1)}>Unrelated: {count}</button>
    </div>
  );
}

// Fixed — memoize the expensive computation
function DataDisplayFixed({ data }) {
  const [count, setCount] = useState(0);

  // Only re-runs when `data` changes
  const processed = useMemo(() => processData(data), [data]);

  return (
    <div>
      <p>Processed: {processed}</p>
      <button onClick={() => setCount(c => c + 1)}>Unrelated: {count}</button>
    </div>
  );
}
```

**Profiling process:** (1) Open React DevTools → Profiler tab; (2) Click Record; (3) Click the unrelated button a few times; (4) Stop recording; (5) Inspect — you'll see `DataDisplay` takes e.g., 85ms per render, with "processData" visible in the flame chart. The fix is `useMemo`. After fix: the button renders in <1ms; processData only shows in renders where `data` changed.

### B2
**Implement virtualization for a list of 10,000 items using `react-window` so only the visible items render.**

```jsx
import { FixedSizeList } from 'react-window';

const ITEM_COUNT = 10000;
const ITEM_HEIGHT = 35;
const LIST_HEIGHT = 600;

const items = Array.from({ length: ITEM_COUNT }, (_, i) => ({
  id: i,
  name: `Item ${i + 1}`,
  value: Math.round(Math.random() * 1000),
}));

function Row({ index, style }) {
  const item = items[index];
  return (
    <div
      style={{
        ...style, // required — contains the position/height from react-window
        display: 'flex',
        alignItems: 'center',
        padding: '0 16px',
        borderBottom: '1px solid #eee',
      }}
    >
      <span style={{ flex: 1 }}>{item.name}</span>
      <span>{item.value}</span>
    </div>
  );
}

function VirtualizedList() {
  return (
    <FixedSizeList
      height={LIST_HEIGHT}
      itemCount={ITEM_COUNT}
      itemSize={ITEM_HEIGHT}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

### D1
**The Profiler shows a component re-rendering 60 times per second with no user interaction. Find the cause.**

```jsx
function LiveClock() {
  const [time, setTime] = useState(new Date());

  useEffect(() => {
    const interval = setInterval(() => {
      setTime(new Date());
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  return <p>{time.toLocaleTimeString()}</p>;
}

function Dashboard() {
  return (
    <div>
      <LiveClock />
      <ExpensiveDataTable data={largeDataset} />
    </div>
  );
}
```

**Problem:** `LiveClock` updates every second (fine), but it causes `Dashboard` to re-render, which causes `ExpensiveDataTable` to re-render — even though `largeDataset` hasn't changed.

```jsx
// Fix 1 — memo on ExpensiveDataTable
const ExpensiveDataTable = memo(function ExpensiveDataTable({ data }) { ... });

// Fix 2 (better) — colocate the clock state, don't let it propagate up
function Dashboard() {
  return (
    <div>
      <LiveClock />  {/* LiveClock's re-renders stay in LiveClock's subtree */}
      <ExpensiveDataTable data={largeDataset} />  {/* never re-renders */}
    </div>
  );
}
// This already works IF LiveClock's state doesn't live in Dashboard.
// If Dashboard has shared state with LiveClock, extract LiveClock's state inside LiveClock.
```

### A1
**What is the correct order of performance optimizations to try? When does each apply?**

**Answer:** In order: (1) **Fix the architecture** — unnecessary re-renders from state in the wrong place or too-broad Context. Restructure components to colocate state. Cost: refactoring time. Benefit: permanent, no maintenance overhead. (2) **Memoize expensive calculations** — `useMemo` for computations that are measurably slow (confirmed by profiling). (3) **Stabilize references** — `useCallback` for functions passed to memoized children; `useMemo` for objects passed as props. Only needed if `memo` is in place and the reference instability defeats it. (4) **`React.memo`** — for components that are genuinely re-rendering unnecessarily and have measurably slow renders. (5) **Virtualization** — for lists of >100-500 items where DOM node count is the bottleneck. (6) **Code splitting** — for initial load time. (7) **Concurrent features** (`useTransition`, `useDeferredValue`) — for interactions that block input during heavy re-renders. Each step requires a profiler measurement to confirm the problem and verify the fix.
