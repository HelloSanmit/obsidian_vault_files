
# Lesson 47 — Production React Testing

At this point, you already understand React’s rendering model, state ownership, effects, routing, server state, Actions, Server Components, SSR, performance, and production architecture.

Now we need the other half of production engineering:

> **How do you prove that all of that works?**

Testing React is not mainly about learning a testing library’s API. The deeper skill is deciding **what behavior deserves a test, at what level, and where the boundary of that test should be**.

A production application might have:

```text
Button
  ↓
Component state
  ↓
Form Action
  ↓
API / Server Function
  ↓
Database
  ↓
Cache invalidation
  ↓
Updated UI
```

A good test strategy doesn't try to test this entire chain with one gigantic test.

Instead, we test different layers at different boundaries.

---

# 1. The most important testing idea

Suppose you have this component:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>Count: {count}</p>

      <button onClick={() => setCount(c => c + 1)}>
        Increment
      </button>
    </>
  );
}
```

You could test its implementation:

```js
expect(setCount).toHaveBeenCalled();
```

But that's a poor test.

The user doesn't care whether `setCount` was called.

The user cares that:

```text
click Increment
        ↓
count changes
        ↓
"Count: 1" appears
```

So the test should describe that behavior.

```js
render(<Counter />);

await user.click(
  screen.getByRole('button', { name: /increment/i })
);

expect(
  screen.getByText('Count: 1')
).toBeInTheDocument();
```

The test is effectively saying:

> "When a user performs this action, the UI should produce this result."

That is much more resilient.

---

# 2. React testing is about observing behavior

Think of your application as a black box.

```text
             ┌─────────────────────┐
user input → │                     │ → visible UI
server data →│      React app      │ → navigation
timers      →│                     │ → accessibility state
external API→│                     │
             └─────────────────────┘
```

A test should usually interact with the box the same way a real user or real system would.

So instead of:

```js
component.state.loading
```

test:

```js
screen.getByRole('button', { name: /save/i })
```

Instead of:

```js
expect(component.props.onSubmit).toHaveBeenCalled()
```

test:

```js
await user.click(screen.getByRole('button', { name: /save/i }));

expect(screen.getByText(/saved successfully/i))
  .toBeInTheDocument();
```

This distinction will become extremely important as your application becomes more complicated.

---

# 3. Three major testing levels

A useful production model is:

```text
             E2E
              ↑
       Component / Integration
              ↑
          Unit tests
```

Each level answers a different question.

### Unit test

> Does this isolated piece of logic behave correctly?

Example:

```js
calculateCartTotal(items)
```

### Component/integration test

> Does this part of the application work when React pieces interact?

Example:

```text
form
 ↓
validation
 ↓
mutation
 ↓
loading state
 ↓
success UI
```

### End-to-end test

> Can a real browser complete an important user journey through the application?

Example:

```text
open website
 ↓
login
 ↓
open project
 ↓
create task
 ↓
task appears in project
```

The mistake is trying to use one level for everything.

---

# 4. A realistic production pyramid

Imagine a project-management application.

You may have:

```text
200 unit tests
80 component/integration tests
15 E2E tests
```

Why more lower-level tests?

Because they are usually:

```text
faster
cheaper
more isolated
easier to debug
```

while E2E tests are:

```text
slower
more infrastructure-dependent
more prone to environmental failures
```

But there is another important idea:

> **The pyramid isn't a law.**

Modern applications often need a healthy collection of integration tests because so much behavior emerges from interactions between components, routing, server state, and forms.

---

# 5. React Testing Library's philosophy

For React UI, a common approach is React Testing Library.

The important concept isn't its API.

The philosophy is:

> Test the application from the perspective of its users.

That naturally leads you toward DOM semantics.

For example:

```jsx
<button>Save changes</button>
```

should generally be found through its accessible role:

```js
screen.getByRole('button', {
  name: /save changes/i
});
```

rather than:

```js
container.querySelector('.save-button')
```

Why?

Because the first test cares about what the control **is**.

The second cares about an implementation detail.

---

# 6. Querying the UI

There are three major families of queries you'll use constantly.

## `getBy`

Use when the element should already exist.

```js
screen.getByRole('button', {
  name: /save/i
});
```

If it isn't present, the test fails immediately.

---

## `queryBy`

Useful when the element **should not exist**.

```js
expect(
  screen.queryByText(/error/i)
).not.toBeInTheDocument();
```

Why not `getByText`?

Because `getByText` throws when the element doesn't exist.

That's exactly the opposite of what you want when asserting absence.

---

## `findBy`

Used when the element appears asynchronously.

```js
const message = await screen.findByText(
  /profile saved/i
);
```

Conceptually:

```text
render
  ↓
loading
  ↓
request finishes
  ↓
React updates
  ↓
DOM changes
  ↓
findBy sees it
```

So:

```js
getBy
```

means:

> "It should be here now."

while:

```js
findBy
```

means:

> "It should appear after asynchronous work."

---

# 7. Query priority

A useful mental ordering is:

```text
accessible role
    ↓
label
    ↓
visible text
    ↓
other semantic queries
    ↓
test ID
    ↓
CSS selector
```

For example:

```js
screen.getByRole('textbox', {
  name: /email/i
});
```

is usually better than:

```js
screen.getByTestId('email-input');
```

Not because test IDs are forbidden.

They are useful when there's no meaningful semantic way to identify something.

For example:

```jsx
<div data-testid="chart-canvas" />
```

can make sense for a visualization that has no useful user-facing semantic target.

---

# 8. `userEvent` vs `fireEvent`

You will see both.

A user doesn't technically execute:

```js
fireEvent.click(button);
```

A real user interacts with the browser.

A higher-level interaction utility such as `userEvent` simulates interaction more realistically.

For example:

```js
const user = userEvent.setup();

await user.type(
  screen.getByRole('textbox', { name: /email/i }),
  'alice@example.com'
);

await user.click(
  screen.getByRole('button', { name: /submit/i })
);
```

Notice the `await`.

That's important.

Typing isn't conceptually just one synchronous assignment.

A more realistic interaction can cause a sequence like:

```text
keyboard event
 ↓
input event
 ↓
React handler
 ↓
state update
 ↓
render
 ↓
DOM update
```

So:

```js
await user.type(...)
```

models the interaction more faithfully than manually assigning DOM properties.

---

# 9. Example: testing a form

Suppose:

```jsx
function LoginForm({ onLogin }) {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  return (
    <form onSubmit={e => {
      e.preventDefault();
      onLogin({ email, password });
    }}>
      <label>
        Email
        <input
          value={email}
          onChange={e => setEmail(e.target.value)}
        />
      </label>

      <label>
        Password
        <input
          type="password"
          value={password}
          onChange={e => setPassword(e.target.value)}
        />
      </label>

      <button type="submit">
        Sign in
      </button>
    </form>
  );
}
```

A behavioral test:

```js
test('submits entered credentials', async () => {
  const user = userEvent.setup();
  const onLogin = vi.fn();

  render(<LoginForm onLogin={onLogin} />);

  await user.type(
    screen.getByRole('textbox', {
      name: /email/i
    }),
    'alice@example.com'
  );

  await user.type(
    screen.getByLabelText(/password/i),
    'secret123'
  );

  await user.click(
    screen.getByRole('button', {
      name: /sign in/i
    })
  );

  expect(onLogin).toHaveBeenCalledWith({
    email: 'alice@example.com',
    password: 'secret123'
  });
});
```

Notice what we **didn't** test:

```js
expect(emailState).toBe(...)
```

There is no reason to care how the component stores the value internally.

---

# 10. Test the state machine, not the state variable

Remember our earlier React mental model:

```text
trigger
  ↓
render
  ↓
commit
```

Testing becomes even more useful if you think in terms of state transitions.

For example:

```text
          submit
idle ─────────────────→ loading
                         │
                ┌────────┴────────┐
                ↓                 ↓
             success            error
```

Then write tests for transitions:

### Test 1

```text
idle → loading
```

### Test 2

```text
loading → success
```

### Test 3

```text
loading → error
```

This is much more systematic than randomly writing tests for individual lines.

---

# 11. Async testing

Imagine:

```jsx
function Profile() {
  const [profile, setProfile] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetch('/api/profile')
      .then(r => r.json())
      .then(data => {
        setProfile(data);
        setLoading(false);
      });
  }, []);

  if (loading) {
    return <p>Loading...</p>;
  }

  return <h1>{profile.name}</h1>;
}
```

A test might look like:

```js
test('shows profile after loading', async () => {
  render(<Profile />);

  expect(
    screen.getByText(/loading/i)
  ).toBeInTheDocument();

  expect(
    await screen.findByRole('heading', {
      name: /alice/i
    })
  ).toBeInTheDocument();
});
```

Notice the test doesn't sleep:

```js
await new Promise(resolve =>
  setTimeout(resolve, 1000)
);
```

Don't write arbitrary sleeps to "wait for React."

That's a classic source of flaky tests.

Instead, wait for the observable condition.

---

# 12. What `act()` actually means

React updates can be asynchronous and scheduled.

A test can therefore accidentally ask:

```text
"What's in the DOM?"
```

before React has finished processing the update you're testing.

`act()` exists to establish an interaction boundary:

```text
perform interaction
        ↓
flush related React work
        ↓
make assertion
```

Modern React uses:

```js
import { act } from 'react';
```

rather than the old `react-dom/test-utils` location. React 19 also removed most of the old test-utils APIs, while recommending modern testing approaches such as React Testing Library. ([React](https://react.dev/blog/2024/04/25/react-19-upgrade-guide?utm_source=chatgpt.com "React 19 Upgrade Guide – React"))

In normal React Testing Library usage, you rarely call `act()` directly because the library handles the relevant wrapping for you.

That gives us an important practical rule:

> Understand `act`, but don't manually sprinkle `act()` everywhere.

---

# 13. Why `react-test-renderer` is no longer the direction to learn

Older React testing tutorials often show:

```js
import renderer from 'react-test-renderer';
```

and then inspect component trees.

That style is tightly coupled to React's implementation.

Current React marks `react-test-renderer` as deprecated and recommends moving toward DOM/user-oriented testing approaches. ([React](https://react.dev/warnings/react-test-renderer?utm_source=chatgpt.com "react-test-renderer Deprecation Warnings – React"))

This fits everything we've already learned.

You should generally care about:

```text
what rendered
what the user can do
what happened afterward
```

rather than:

```text
what internal React representation was generated
```

---

# 14. Testing context

Suppose:

```jsx
const ThemeContext = createContext('light');
```

and:

```jsx
function SaveButton() {
  const theme = useContext(ThemeContext);

  return (
    <button className={`button-${theme}`}>
      Save
    </button>
  );
}
```

Your component might need a provider in a test.

A small helper is useful:

```jsx
function renderWithTheme(ui) {
  return render(
    <ThemeContext value="dark">
      {ui}
    </ThemeContext>
  );
}
```

Then:

```js
renderWithTheme(<SaveButton />);

expect(
  screen.getByRole('button', { name: /save/i })
).toHaveClass('button-dark');
```

This helper is valuable because your test infrastructure can mirror your application's real environment.

---

# 15. Testing React Router

Suppose the application contains:

```text
/dashboard
/projects
/projects/:projectId
/settings
```

A component may depend on:

```js
useParams()
useLocation()
useNavigate()
```

Testing it in isolation without the router can produce unrealistic tests.

Instead, supply the routing environment.

Conceptually:

```jsx
render(
  <MemoryRouter initialEntries={['/projects/42']}>
    <ProjectPage />
  </MemoryRouter>
);
```

Then:

```js
expect(
  screen.getByRole('heading', {
    name: /project 42/i
  })
).toBeInTheDocument();
```

This tests the component in the environment it actually expects.

---

# 16. Test navigation behavior

Suppose:

```jsx
function ProjectCard({ id, name }) {
  return (
    <Link to={`/projects/${id}`}>
      {name}
    </Link>
  );
}
```

Don't test:

```js
expect(link.getAttribute('href'))
  .toBe('/projects/42');
```

unless that exact URL attribute is itself important.

A stronger test is:

```text
user sees project
 ↓
user clicks project
 ↓
application navigates
 ↓
project page appears
```

For example:

```js
await user.click(
  screen.getByRole('link', {
    name: /alpha project/i
  })
);

expect(
  await screen.findByRole('heading', {
    name: /alpha project/i
  })
).toBeInTheDocument();
```

Now you're testing the behavior the user actually experiences.

---

# 17. Testing server-state applications

This is where testing gets more interesting.

Suppose your app uses TanStack Query.

Your UI isn't just:

```text
component → state
```

It is:

```text
component
   ↓
query cache
   ↓
HTTP request
   ↓
server
```

You shouldn't mock React Query's internal hooks.

Don't do:

```js
vi.mock('@tanstack/react-query');
```

just so you can force:

```js
useQuery.mockReturnValue(...)
```

That tests your mocked React Query rather than your application.

Instead, create the real query environment and control the **network boundary**.

---

# 18. Mock the network boundary, not your application

This distinction is huge.

Bad:

```text
Component
   ↓
mock useQuery()
   ↓
fake result
```

Better:

```text
Component
   ↓
real useQuery()
   ↓
fetch()
   ↓
intercepted HTTP request
   ↓
test-controlled response
```

So the actual application behavior remains alive.

Only the outside world is controlled.

This is where tools such as Mock Service Worker are useful.

---

# 19. Why network mocking is powerful

Suppose your component does:

```js
useQuery({
  queryKey: ['projects'],
  queryFn: fetchProjects
});
```

Your test can simulate:

```text
GET /api/projects
       ↓
200
[
  { id: 1, name: "Alpha" }
]
```

Then your real React Query code runs.

That means you're testing:

```text
queryKey
queryFn
loading state
success state
cache behavior
rendering
```

instead of replacing the whole data layer with a fake function.

---

# 20. Example network handler

Conceptually, a mocked endpoint might respond with:

```js
http.get('/api/projects', () => {
  return HttpResponse.json([
    {
      id: 1,
      name: 'Alpha'
    }
  ]);
});
```

Then:

```js
test('renders projects from the API', async () => {
  render(<ProjectList />);

  expect(
    screen.getByText(/loading/i)
  ).toBeInTheDocument();

  expect(
    await screen.findByText('Alpha')
  ).toBeInTheDocument();
});
```

Notice something beautiful here.

Your component still thinks:

> "I'm talking to a server."

The test server is simply deterministic.

---

# 21. Test failure responses too

A common beginner mistake:

```text
test success
test success
test success
```

Real applications fail.

So deliberately return:

```http
500
```

or:

```http
401
```

or:

```http
404
```

and test the UI.

For example:

```text
request
  ↓
500
  ↓
query error
  ↓
error UI
```

Then:

```js
expect(
  await screen.findByText(
    /unable to load projects/i
  )
).toBeInTheDocument();
```

A production-quality test suite should deliberately exercise failure paths.

---

# 22. Retry can complicate tests

Remember our TanStack Query lesson.

Queries may retry.

Suppose your test expects:

```text
GET /projects
  ↓
500
  ↓
error UI
```

but your query retries.

You might actually get:

```text
GET /projects → 500
GET /projects → 500
GET /projects → 500
                 ↓
             error UI
```

This makes tests slower.

For deterministic tests, test configuration often disables retries.

Conceptually:

```js
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false
    }
  }
});
```

This doesn't mean production should disable retries.

It means:

> production behavior and test behavior have different environmental goals.

---

# 23. Mutations need especially careful testing

Suppose:

```text
click "Delete"
        ↓
mutation starts
        ↓
optimistic UI removes item
        ↓
server request
        ↓
success
        ↓
cache remains consistent
```

There are several things to test.

### Successful mutation

```text
item exists
 ↓
delete
 ↓
item disappears
```

### Failed mutation

```text
item exists
 ↓
optimistic removal
 ↓
server failure
 ↓
rollback
 ↓
item returns
```

That second test is particularly valuable because optimistic UI introduces temporary state.

---

# 24. Testing optimistic updates

Imagine:

```js
const deleteMutation = useMutation({
  mutationFn: deleteProject,

  onMutate: async id => {
    // optimistic cache update
  },

  onError: () => {
    // rollback
  }
});
```

The test shouldn't inspect the cache implementation directly.

Instead:

```text
initial UI
  ↓
user clicks delete
  ↓
item disappears
  ↓
server responds with error
  ↓
item appears again
```

That validates the user's experience.

---

# 25. Don't mock everything

This is one of the most important lessons in testing.

Suppose you have:

```text
App
 ├── Router
 ├── Auth
 ├── QueryClient
 ├── Context
 ├── Form
 └── API client
```

You might be tempted to mock:

```js
vi.mock('./api');
vi.mock('./auth');
vi.mock('./router');
vi.mock('./query');
vi.mock('./context');
```

Eventually the test becomes:

```text
mock
mock
mock
mock
mock
assert mock
```

The test passes.

But the application could be completely broken.

Why?

Because you removed all the interesting behavior.

---

# 26. A better mocking rule

Ask:

> "What is outside the boundary I'm trying to test?"

For a component test:

```text
React/application logic
        │
        │ real
        ↓
    component
        │
        │ controlled boundary
        ↓
     network
```

For an E2E test:

```text
browser
 ↓
real application
 ↓
real backend or test environment
```

For a pure unit test:

```text
pure function
 ↓
input
 ↓
output
```

Mock only what actually needs isolation.

---

# 27. Testing custom hooks

Suppose:

```js
function useOnlineStatus() {
  return useSyncExternalStore(
    subscribe,
    getSnapshot
  );
}
```

You could test the hook's exposed behavior through a component:

```jsx
function Status() {
  const online = useOnlineStatus();

  return (
    <p>
      {online ? 'Online' : 'Offline'}
    </p>
  );
}
```

Then manipulate the external source and assert:

```text
Online
 ↓
offline event
 ↓
Offline
```

This is often more valuable than testing internal helper calls.

The hook's contract is:

```text
external system
     ↓
hook
     ↓
boolean exposed to component
```

Test the contract.

---

# 28. Testing reducers

Reducers are excellent candidates for pure unit tests.

Suppose:

```js
function reducer(state, action) {
  switch (action.type) {
    case 'added':
      return {
        ...state,
        count: state.count + 1
      };

    case 'removed':
      return {
        ...state,
        count: state.count - 1
      };

    default:
      return state;
  }
}
```

You can test directly:

```js
expect(
  reducer(
    { count: 2 },
    { type: 'added' }
  )
).toEqual({
  count: 3
});
```

This is appropriate because reducers are pure logic.

Here a lower-level unit test is actually the right tool.

---

# 29. Testing pure domain logic

Suppose:

```js
function calculateTotal(items, taxRate) {
  const subtotal = items.reduce(
    (sum, item) => sum + item.price * item.quantity,
    0
  );

  return subtotal * (1 + taxRate);
}
```

Don't render React to test this.

That's unnecessary.

Test:

```js
expect(
  calculateTotal(
    [
      { price: 100, quantity: 2 },
      { price: 50, quantity: 1 }
    ],
    0.1
  )
).toBe(275);
```

The architecture should help you identify where pure logic belongs.

That's one reason we discussed production architecture before testing.

---

# 30. Test component behavior, not every implementation branch

Imagine:

```jsx
function UserCard({ user }) {
  if (!user) {
    return <p>No user</p>;
  }

  if (!user.active) {
    return <p>User inactive</p>;
  }

  return (
    <article>
      <h2>{user.name}</h2>
    </article>
  );
}
```

Useful tests:

```text
no user → "No user"
inactive user → "User inactive"
active user → user name
```

You don't necessarily need a test for:

```js
if (!user)
```

as syntax.

You're testing the observable states.

---

# 31. Accessibility testing and regular UI testing overlap

A useful side effect of semantic queries is that your tests often encourage accessible UI.

For example:

```js
getByRole('button', {
  name: /save/i
})
```

implicitly requires the button to expose the expected accessible name.

That means a refactor like:

```jsx
<button>
  <Icon />
</button>
```

could break the test if the accessible name disappears.

That's useful.

The failing test says:

> "The user-visible interface lost something important."

rather than:

> "CSS selector `.button-7f8a` disappeared."

---

# 32. Forms + validation

Consider:

```jsx
<input
  aria-invalid={Boolean(error)}
  aria-describedby="email-error"
/>

<p id="email-error">
  Invalid email
</p>
```

A good test can check the actual accessibility contract:

```js
const input = screen.getByRole('textbox', {
  name: /email/i
});

await user.type(input, 'hello');

await user.click(
  screen.getByRole('button', {
    name: /submit/i
  })
);

expect(input).toHaveAttribute(
  'aria-invalid',
  'true'
);

expect(
  screen.getByText(/invalid email/i)
).toBeInTheDocument();
```

Now one test checks both behavior and accessibility semantics.

---

# 33. Testing loading states

Loading states are not decoration.

They represent a state in your state machine.

For example:

```jsx
<button disabled={isPending}>
  {isPending ? 'Saving...' : 'Save'}
</button>
```

Test:

```js
await user.click(
  screen.getByRole('button', {
    name: /save/i
  })
);

expect(
  screen.getByRole('button', {
    name: /saving/i
  })
).toBeDisabled();
```

This catches regressions such as:

```text
double submission
missing disabled state
wrong pending label
```

---

# 34. React Actions deserve state-transition tests

With React 19's Action APIs, you may have something like:

```js
const [state, submit, isPending] =
  useActionState(saveProfile, initialState);
```

Your test shouldn't care about the hook's internals.

It should care about:

```text
form visible
 ↓
user submits
 ↓
pending UI
 ↓
server action resolves
 ↓
success UI
```

And for failure:

```text
submit
 ↓
server rejects
 ↓
validation/error state
 ↓
form remains usable
```

That becomes even more important when forms contain optimistic UI.

---

# 35. `useFormStatus` testing

Suppose:

```jsx
function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? 'Saving...' : 'Save'}
    </button>
  );
}
```

Test the visible contract:

```text
idle → Save enabled
pending → Saving disabled
complete → Save enabled
```

This is preferable to testing:

```js
expect(useFormStatus).toHaveBeenCalled()
```

because the latter says almost nothing about the user experience.

---

# 36. Test Server Functions at the right boundary

A Server Function crosses an architectural boundary:

```text
browser
   ↓
network
   ↓
server function
   ↓
validation
   ↓
authorization
   ↓
database
```

You usually don't want every component test to spin up the actual database.

Instead:

### Unit test

Test validation/domain logic.

### Server-side integration test

Test the Server Function against realistic persistence or a test database.

### UI integration test

Test the form and resulting UI while controlling the server boundary.

### E2E

Test the complete user workflow in a deployed-like environment.

This gives you good coverage without making every test expensive.

---

# 37. Server Components complicate testing

Server Components aren't ordinary client-side components.

Remember the boundary:

```text
Server Component
      │
      │ server
      ↓
serialized output
      │
      ↓
Client Component
      │
      │ browser
      ↓
interactive behavior
```

So testing needs to respect execution environment.

A useful strategy is:

### Pure server logic

Test independently.

### Server-side component behavior

Test in the framework's supported server/RSC environment.

### Client interaction

Test using browser/DOM testing.

### Full user workflow

Use E2E.

Don't force every Server Component into a browser-only unit-test model.

---

# 38. End-to-end testing

Now we move upward.

Suppose you have:

```text
Login
Projects
Task list
Task editor
Notifications
```

A valuable E2E test could be:

```text
open application
 ↓
login
 ↓
projects page appears
 ↓
open project
 ↓
create task
 ↓
task appears
 ↓
log out
```

This test verifies many pieces at once:

```text
routing
authentication
server communication
form handling
cache updates
browser behavior
```

This is the type of test that can catch integration failures unit tests never see.

---

# 39. Playwright mental model

A common modern browser-testing tool is Playwright.

The basic model is:

```text
browser
  ↓
page
  ↓
locate element
  ↓
perform user action
  ↓
assert UI
```

Example:

```js
test('user can create a task', async ({ page }) => {
  await page.goto('/projects/42');

  await page.getByRole('button', {
    name: /new task/i
  }).click();

  await page.getByRole('textbox', {
    name: /task title/i
  }).fill('Fix dashboard bug');

  await page.getByRole('button', {
    name: /create task/i
  }).click();

  await expect(
    page.getByText('Fix dashboard bug')
  ).toBeVisible();
});
```

The important thing isn't the syntax.

It's that a real browser is exercising your application.

Playwright's current tooling also supports component-testing workflows in addition to browser end-to-end testing, though these serve different purposes. ([Playwright](https://playwright.dev/docs/testing-library?utm_source=chatgpt.com "Migrating from Testing Library | Playwright"))

---

# 40. Why E2E tests are powerful

Imagine your router has accidentally changed:

```text
/projects/42
```

to:

```text
/project/42
```

You might have:

```text
component tests → all pass
unit tests → all pass
```

But E2E:

```text
page.goto('/projects/42')
       ↓
404
       ↓
FAIL
```

Exactly what we want.

E2E catches problems at the system boundary.

---

# 41. But E2E shouldn't test every detail

Suppose you have:

```text
20 different button variants
```

You don't need:

```text
20 browser journeys
```

for every trivial variation.

A stronger strategy is:

```text
component tests
    ↓
many local behaviors

E2E
    ↓
a smaller number of critical user journeys
```

For example:

```text
login
checkout
create record
edit record
delete record
permission restriction
logout
```

These are high-value flows.

---

# 42. E2E and authentication

Authentication frequently makes E2E tests painful.

Imagine every test does:

```text
launch browser
 ↓
open login
 ↓
type email
 ↓
type password
 ↓
submit
 ↓
wait
```

for 100 tests.

That's wasteful.

A common pattern is to authenticate once in a controlled setup and reuse the authenticated browser state.

Conceptually:

```text
setup
 ↓
login
 ↓
save authenticated session
 ↓
tests reuse session
```

Now:

```text
test A ─┐
test B ─┼→ authenticated state
test C ─┤
test D ─┘
```

This dramatically reduces duplication.

---

# 43. But don't remove the login journey entirely

You still want at least some test covering:

```text
login page
 ↓
credentials
 ↓
successful authentication
 ↓
redirect
```

Because that flow itself is important.

So use:

```text
one or few login E2E tests
+
many tests using reusable authenticated state
```

That's a good example of balancing realism and speed.

---

# 44. Flaky tests

One of the most dangerous testing problems is:

> A test sometimes passes and sometimes fails.

Suppose:

```js
await page.click('#save');

await page.waitForTimeout(1000);

expect(...);
```

This may pass on your machine.

Then fail in CI.

Why?

Because you've encoded an assumption:

> "The application will finish within exactly 1000ms."

But actual timing is:

```text
network
browser
React scheduling
CPU
server load
CI environment
```

all variable.

---

# 45. Synchronize with meaning, not time

Bad:

```js
await sleep(1000);
```

Better:

```js
await expect(
  page.getByText(/saved successfully/i)
).toBeVisible();
```

Or:

```js
await screen.findByText(/saved successfully/i);
```

The difference is profound:

### Time synchronization

```text
wait 1000ms
```

### State synchronization

```text
wait until observable condition becomes true
```

Always prefer the second.

---

# 46. Race conditions in tests

Suppose:

```text
click Save
 ↓
request starts
 ↓
test immediately asserts
```

The test may be racing the application.

That's essentially the same concurrency problem we studied with React effects and server requests.

A test should establish a clear synchronization point:

```text
action
 ↓
expected transition
 ↓
assertion
```

For example:

```js
await user.click(saveButton);

expect(
  await screen.findByText(/saved/i)
).toBeInTheDocument();
```

---

# 47. Testing timers

Sometimes time itself is part of the application.

Example:

```text
search input
 ↓
debounce 300ms
 ↓
request
```

Or:

```text
toast
 ↓
hide after 5 seconds
```

Now fake timers can be appropriate.

For example conceptually:

```js
vi.useFakeTimers();

render(<Toast />);

act(() => {
  vi.advanceTimersByTime(5000);
});

expect(
  screen.queryByText(/saved/i)
).not.toBeInTheDocument();
```

Notice the distinction:

Using a fake clock is fine when **time is actually the thing you're testing**.

Using a timer to wait for arbitrary React work is not.

---

# 48. Component tests should still compose components

Suppose:

```jsx
function Checkout() {
  return (
    <Form>
      <AddressFields />
      <PaymentFields />
      <SubmitButton />
    </Form>
  );
}
```

Testing `Checkout` with every child mocked away is often counterproductive.

You lose important interactions:

```text
form state
 ↓
validation
 ↓
field updates
 ↓
submit state
```

A component/integration test should often allow several real components to participate.

This is why the term **integration test** is valuable.

You're testing a small working subsystem.

---

# 49. Test setup helpers are part of architecture

Large React applications quickly accumulate setup requirements:

```text
Router
QueryClient
Context providers
i18n
theme
feature flags
```

Instead of repeating:

```jsx
<QueryClientProvider client={queryClient}>
  <ThemeContext value="dark">
    <MemoryRouter>
      <App />
    </MemoryRouter>
  </ThemeContext>
</QueryClientProvider>
```

everywhere, create a test harness:

```jsx
function renderApp(ui, {
  route = '/',
  user = null
} = {}) {
  return render(
    <QueryClientProvider client={createTestQueryClient()}>
      <AuthContext value={user}>
        <MemoryRouter initialEntries={[route]}>
          {ui}
        </MemoryRouter>
      </AuthContext>
    </QueryClientProvider>
  );
}
```

Now:

```js
renderApp(<ProjectPage />, {
  route: '/projects/42',
  user: { id: 7, name: 'Alice' }
});
```

This makes your tests shorter without hiding important behavior.

---

# 50. Test helpers should not become giant magic machines

There is a trap here.

You can create:

```js
renderEverythingForMe(...)
```

with 30 options.

Eventually nobody understands what the test environment actually contains.

So test helpers should expose the important dimensions:

```text
route
authenticated user
server response
feature flags
```

but avoid becoming an entire second application framework.

---

# 51. Test isolation

Suppose test A creates:

```text
project #42
```

Then test B assumes:

```text
project #42 exists
```

That's dangerous.

Test B now depends on test A.

Correct principle:

> Each test should establish the state it requires.

Conceptually:

```text
Test A
  setup → run → cleanup

Test B
  setup → run → cleanup
```

not:

```text
Test A
  ↓
Test B
  ↓
Test C
```

This makes failures reproducible.

---

# 52. Database testing

For applications with a backend, you'll eventually need database strategy.

There are several approaches.

### Mock repository

```text
UI → fake repository
```

Fast, but limited realism.

### Test database

```text
UI
 ↓
API
 ↓
test DB
```

More realistic.

### Ephemeral database

Create a fresh database/container/environment for tests.

Very realistic but more expensive.

The right choice depends on your application.

The important architecture lesson is to separate:

```text
UI
 ↓
application/service layer
 ↓
repository/database
```

so that different tests can choose the appropriate boundary.

---

# 53. Contract testing

Imagine frontend expects:

```json
{
  "id": 42,
  "name": "Alice"
}
```

but backend changes it to:

```json
{
  "userId": 42,
  "displayName": "Alice"
}
```

Your UI breaks.

Contract testing asks:

> Do the systems still agree on the shape and meaning of the API?

This is particularly useful when:

```text
frontend team
        ↕
independent backend
```

or multiple services communicate.

It sits somewhere between traditional unit testing and full E2E testing.

---

# 54. Don't make tests depend on exact text unnecessarily

Suppose:

```jsx
<button>Save</button>
```

and product changes it to:

```jsx
<button>Save changes</button>
```

Some test failures are desirable.

But if your test is checking an unrelated behavior, unnecessarily coupling it to exact copy creates noise.

Prefer:

```js
getByRole('button', {
  name: /save/i
});
```

rather than:

```js
getByText('Save changes now!');
```

when exact wording isn't important.

The test should encode the requirement, not arbitrary details.

---

# 55. Snapshot testing

You may encounter:

```js
expect(tree).toMatchSnapshot();
```

Snapshots can be useful for some kinds of output.

But giant snapshots of entire component trees can become:

```text
large
noisy
hard to review
easy to blindly update
```

Then someone changes:

```diff
- old UI
+ new UI
```

and clicks:

```text
"Update snapshot"
```

without checking what changed.

A smaller explicit assertion is often more informative:

```js
expect(
  screen.getByRole('heading', {
    name: /account settings/i
  })
).toBeInTheDocument();
```

Use snapshots where they answer a meaningful question, not as a substitute for understanding the UI.

---

# 56. Coverage is not quality

Imagine:

```text
100% line coverage
```

and tests like:

```js
render(<App />);
```

Congratulations.

Almost every line executed.

But perhaps you never tested:

```text
failure
permissions
validation
race conditions
optimistic rollback
navigation
```

So:

```text
coverage ≠ correctness
```

Coverage is a measurement tool.

It can help identify untested areas.

It is not a proof that the application works.

---

# 57. What should you prioritize?

Think in terms of risk.

Suppose:

```text
Marketing header
Settings page
Payment flow
Permission system
```

A bug in:

```text
marketing header
```

may be inconvenient.

A bug in:

```text
payment flow
```

may be catastrophic.

So test density should reflect business and technical risk.

A useful question is:

> "What would hurt the product most if this silently broke tomorrow?"

Those behaviors deserve stronger tests.

---

# 58. Testing permissions

Suppose:

```js
const canDelete = user.permissions.includes(
  'project.delete'
);
```

Then:

```jsx
{canDelete && (
  <button>Delete project</button>
)}
```

Test both:

```text
authorized → delete control available
unauthorized → delete control unavailable
```

But remember an important security principle from our authentication lesson:

```text
UI hides button
        ≠
server protects resource
```

So frontend tests verify the UI behavior.

Backend authorization tests verify actual security.

You need both.

---

# 59. A permission test should cover behavior, not CSS

Don't write:

```js
expect(deleteButton).not.toHaveStyle({
  display: 'none'
});
```

Prefer:

```js
expect(
  screen.queryByRole('button', {
    name: /delete project/i
  })
).not.toBeInTheDocument();
```

The requirement is:

> User without permission cannot access this action through the UI.

---

# 60. Error boundaries

We previously learned that Error Boundaries catch render-time failures.

Now we can test them.

Suppose:

```jsx
function BrokenWidget() {
  throw new Error('Boom');
}
```

and:

```jsx
function App() {
  return (
    <ErrorBoundary>
      <BrokenWidget />
    </ErrorBoundary>
  );
}
```

The test should verify the user-visible recovery interface:

```js
expect(
  screen.getByText(/something went wrong/i)
).toBeInTheDocument();
```

You don't need to prove every lifecycle method ran.

The behavior is what matters.

---

# 61. Testing Suspense

Suppose:

```jsx
<Suspense fallback={<p>Loading dashboard...</p>}>
  <Dashboard />
</Suspense>
```

A useful test verifies the state sequence:

```text
not ready
   ↓
"Loading dashboard..."
   ↓
dashboard
```

Test:

```js
expect(
  screen.getByText(/loading dashboard/i)
).toBeInTheDocument();

expect(
  await screen.findByRole('heading', {
    name: /dashboard/i
  })
).toBeInTheDocument();
```

Again:

```text
observable state transitions
```

are more important than internal Suspense mechanics.

---

# 62. Testing a real application architecture

Imagine our production application has:

```text
React Router
      ↓
Route
      ↓
ProjectPage
      ↓
TanStack Query
      ↓
API
      ↓
Server
```

One integration test can cover:

```text
open project page
  ↓
loading state
  ↓
API response
  ↓
project appears
```

Another:

```text
click archive
  ↓
mutation begins
  ↓
pending state
  ↓
server succeeds
  ↓
query invalidation
  ↓
project disappears
```

This is much more valuable than writing isolated tests for every hook involved.

---

# 63. A complete example

Let's build a realistic miniature example.

```jsx
function TodoList() {
  const {
    data = [],
    isPending,
    isError
  } = useQuery({
    queryKey: ['todos'],
    queryFn: fetchTodos
  });

  const addTodo = useMutation({
    mutationFn: createTodo
  });

  const [text, setText] = useState('');

  if (isPending) {
    return <p>Loading todos...</p>;
  }

  if (isError) {
    return <p>Unable to load todos.</p>;
  }

  async function handleSubmit(e) {
    e.preventDefault();

    await addTodo.mutateAsync({
      title: text
    });

    setText('');
  }

  return (
    <>
      <form onSubmit={handleSubmit}>
        <label>
          New todo
          <input
            value={text}
            onChange={e => setText(e.target.value)}
          />
        </label>

        <button
          type="submit"
          disabled={addTodo.isPending}
        >
          {addTodo.isPending
            ? 'Adding...'
            : 'Add todo'}
        </button>
      </form>

      <ul>
        {data.map(todo => (
          <li key={todo.id}>
            {todo.title}
          </li>
        ))}
      </ul>
    </>
  );
}
```

A meaningful test suite might look like this.

---

## Test A — Loading

```text
query starts
 ↓
loading UI
```

```js
test('shows loading state', () => {
  renderApp(<TodoList />);

  expect(
    screen.getByText(/loading todos/i)
  ).toBeInTheDocument();
});
```

---

## Test B — Successful load

```text
request
 ↓
200
 ↓
todos rendered
```

```js
test('renders todos', async () => {
  renderApp(<TodoList />);

  expect(
    await screen.findByText('Learn React')
  ).toBeInTheDocument();
});
```

---

## Test C — Server failure

```text
request
 ↓
500
 ↓
error UI
```

```js
test('shows load error', async () => {
  // network handler returns 500

  renderApp(<TodoList />);

  expect(
    await screen.findByText(
      /unable to load todos/i
    )
  ).toBeInTheDocument();
});
```

---

## Test D — User adds todo

```text
type
 ↓
submit
 ↓
mutation
 ↓
server
 ↓
todo appears
```

```js
test('user can add a todo', async () => {
  const user = userEvent.setup();

  renderApp(<TodoList />);

  const input = screen.getByRole('textbox', {
    name: /new todo/i
  });

  await user.type(
    input,
    'Study React testing'
  );

  await user.click(
    screen.getByRole('button', {
      name: /add todo/i
    })
  );

  expect(
    await screen.findByText(
      'Study React testing'
    )
  ).toBeInTheDocument();
});
```

That one test covers a surprisingly large chain.

---

# 64. The testing boundary matrix

Here's a useful way to think about where something belongs.

|Thing|Typical test|
|---|---|
|Pure calculation|Unit|
|Reducer|Unit|
|Formatter/parser|Unit|
|Custom hook behavior|Component/integration|
|Form behavior|Component/integration|
|Context behavior|Component/integration|
|Query loading/error UI|Component/integration|
|Mutation behavior|Component/integration|
|Router flow|Integration/E2E|
|Authentication journey|E2E + server tests|
|Authorization|Server/integration + UI|
|Database repository|Integration|
|Critical business journey|E2E|
|Visual layout|Visual regression|
|API contract|Contract/integration|

The exact split varies by project, but the mental model is powerful.

---

# 65. Visual regression testing

There is another category:

> Does the UI still look correct?

A behavior test may pass while the layout is broken.

For example:

```text
button exists ✅
button is clickable ✅
but button is completely off-screen ❌
```

Visual regression tools can compare screenshots across builds.

This is useful for:

```text
design systems
complex dashboards
marketing pages
responsive layouts
critical visual components
```

But don't turn every component into a screenshot test.

Pixel-perfect comparison can be noisy because rendering can vary across:

```text
browser
OS
font rendering
device
```

Use it selectively.

---

# 66. Testing responsive behavior

A responsive E2E suite might test:

```text
desktop
tablet
mobile
```

But don't test every viewport imaginable.

Pick representative breakpoints based on your design system.

For example:

```text
mobile: 390
desktop: 1440
```

Then test meaningful behavior:

```text
mobile nav opens
desktop sidebar visible
table becomes scrollable
dialog fits viewport
```

Again, test behavior rather than arbitrary dimensions.

---

# 67. CI pipeline

Your tests need to run automatically.

A healthy CI pipeline might look like:

```text
commit
  ↓
install dependencies
  ↓
lint
  ↓
typecheck
  ↓
unit tests
  ↓
integration tests
  ↓
build
  ↓
E2E
  ↓
deploy
```

Sometimes E2E runs against a preview deployment or dedicated environment.

The exact ordering depends on cost and project needs.

---

# 68. Why run type checking separately?

Because:

```text
tests
```

and:

```text
TypeScript
```

answer different questions.

A test can prove:

```text
this runtime behavior happened
```

while TypeScript can prove:

```text
these values satisfy the declared static types
```

Neither replaces the other.

---

# 69. Why tests should run against production-like builds

Development environments can behave differently from production.

Remember our previous deployment lesson:

```text
development
 ↓
bundler dev server
 ↓
hot reload
```

versus:

```text
production
 ↓
built assets
 ↓
optimized bundles
 ↓
real deployment environment
```

Some bugs only appear after:

```text
code splitting
minification
asset hashing
SSR
routing
environment substitution
```

So critical systems should test production-like builds too.

---

# 70. A useful CI layering strategy

For example:

### Every commit

```text
lint
typecheck
fast unit tests
```

### Pull request

```text
component/integration tests
build
selected E2E
```

### Main/release

```text
full E2E
visual regression
deployment checks
```

This reduces feedback time while still giving the final pipeline strong coverage.

---

# 71. Test failures should be diagnosable

A bad test failure:

```text
Expected false to be true.
```

A useful test failure:

```text
Expected button "Save changes"
to be disabled
but it was enabled.
```

So tests should use descriptive assertions and meaningful names.

For example:

```js
test(
  'disables Save while profile submission is pending',
  async () => {
    ...
  }
);
```

The test name itself becomes part of your engineering feedback system.

---

# 72. Avoid testing private implementation details

These are fragile:

```js
component.instance()
component.state
internal callback
private helper
specific hook call count
specific DOM class
```

unless the implementation itself is genuinely part of a public contract.

The more your tests depend on internals, the less freedom you have to refactor.

And React applications are constantly refactored:

```text
useState
 ↓
useReducer
 ↓
useActionState
```

or:

```text
useEffect fetch
 ↓
TanStack Query
```

or:

```text
client fetching
 ↓
route loader
```

A good behavior test can survive all those architectural changes.

---

# 73. Tests as executable specifications

This is a deeper idea.

A test can serve as a miniature statement of the product requirement.

For example:

```js
test(
  'users without project.delete permission cannot delete projects',
  ...
);
```

That communicates more than code comments.

It describes:

```text
actor
+
permission
+
behavior
+
expected outcome
```

Good tests are therefore part of system documentation.

---

# 74. Testing strategy for the architecture we built

Let's combine everything we've studied.

Our application may have:

```text
                Browser
                   │
             React UI
                   │
        ┌──────────┴──────────┐
        │                     │
     Router              UI state
        │                     │
   route data             Context
        │
   server state
        │
   TanStack Query
        │
   HTTP / Server Functions
        │
      Server
        │
      Database
```

Testing can mirror this architecture:

```text
Pure logic
    ↓
Unit tests

UI + state + routing + query
    ↓
Integration tests

Server functions / repositories / DB
    ↓
Server integration tests

Critical complete workflows
    ↓
E2E tests
```

That's a coherent testing architecture.

---

# 75. A real failure scenario

Imagine production code:

```text
Create Task
```

works like this:

```text
form
 ↓
Action
 ↓
server function
 ↓
database INSERT
 ↓
query invalidation
 ↓
task list refresh
```

A developer changes:

```js
invalidateQueries({
  queryKey: ['tasks']
});
```

to:

```js
invalidateQueries({
  queryKey: ['task']
});
```

What happens?

The server correctly creates the task.

The mutation succeeds.

The UI doesn't refresh.

A unit test may not detect it.

An integration test that checks the mutation/UI interaction should detect it.

That's why testing **behavior across boundaries** matters.

---

# 76. The same bug through the testing hierarchy

Here's the fascinating part.

Suppose the bug is:

```text
wrong query key
```

Then:

### Unit test

```text
PASS
```

because the mutation function itself works.

### Server test

```text
PASS
```

because the database insert works.

### E2E

```text
PASS or FAIL depending on whether page refetches naturally
```

### Integration test

```text
FAIL
```

because:

```text
create task
 ↓
UI does not update
```

That tells you something important:

> **Different tests detect different classes of architectural failure.**

---

# 77. Testing isn't "more tests"

A common beginner mindset is:

```text
More tests = better software
```

Not necessarily.

Suppose you have:

```text
500 trivial tests
```

and no tests for:

```text
payments
permissions
data consistency
failure recovery
```

You may have poor coverage of actual risk.

A better goal is:

> **Enough tests at the right boundaries to detect meaningful regressions.**

---

# 78. A practical React testing checklist

For an important component, mentally walk through:

```text
Initial state
    ↓
Loading
    ↓
Success
    ↓
Empty
    ↓
Error
    ↓
User interaction
    ↓
Pending
    ↓
Success after mutation
    ↓
Failure after mutation
    ↓
Accessibility
    ↓
Permissions
    ↓
Navigation
```

You don't necessarily need one test for every line.

But thinking through these states exposes missing scenarios quickly.

---

# 79. The most important mental model from this lesson

When you're unsure what test to write, ask four questions:

### 1. What can the user do?

```text
click
type
submit
navigate
select
drag
```

### 2. What can the application observe?

```text
server response
timer
browser event
route change
external store
```

### 3. What state transitions should happen?

```text
idle → pending
pending → success
pending → error
```

### 4. What should the user see afterward?

```text
message
list update
navigation
disabled control
error
```

Then build the test around that sequence.

---

# 80. The complete mental model

You've now learned a progression that should feel very natural:

```text
React rendering
      ↓
state ownership
      ↓
effects
      ↓
context/reducer/hooks
      ↓
routing
      ↓
server state
      ↓
mutations
      ↓
server/client boundaries
      ↓
production architecture
      ↓
testing
```

Testing isn't a separate topic bolted onto React.

It's a way of validating the architecture we've been building.

And the deepest testing skill is not:

> "How do I mock this hook?"

It's:

> **"Where is the correct boundary for proving this behavior?"**

That question will save you an enormous amount of time on real projects.

### One current React detail worth remembering

With current React 19.3, `act` is part of the `react` package, while older `react-dom/test-utils` APIs have been removed/deprecated; the React team also marks `react-test-renderer` as deprecated. Modern React testing therefore strongly favors testing rendered behavior rather than relying on React's internal tree representation. ([React](https://react.dev/reference/react/act?utm_source=chatgpt.com "act – React"))

---

## Where we are now

You've covered nearly the full React engineering stack:

```text
React fundamentals
      ↓
Hooks
      ↓
Rendering model
      ↓
Composition
      ↓
Routing
      ↓
Server state
      ↓
Concurrency
      ↓
Accessibility
      ↓
TypeScript
      ↓
External stores
      ↓
Actions
      ↓
SSR
      ↓
Server Components
      ↓
Server Functions
      ↓
Production architecture
      ↓
Deployment
      ↓
Testing
```

The next major topic is **production security for React applications**: XSS, safe HTML, CSP, Trusted Types, CSRF, cookies, browser security boundaries, dependency/supply-chain risks, Server Functions security, and the difference between a UI restriction and an actual security control.