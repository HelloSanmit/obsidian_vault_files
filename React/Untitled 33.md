
# Lesson 30 — Testing React Properly: What to Test, How to Think About Tests, and Why Implementation Details Matter

We've covered a lot of React mechanics now.

One thing we haven't really addressed is:

> **How do you know your React code actually works?**

Not:

```text
"it looks right in my browser"
```

but:

```text
"If somebody changes this component next month,
how do we know they didn't break its behavior?"
```

That's what testing is for.

And testing React has a particularly important philosophical difference from testing ordinary JavaScript:

> **You usually want to test what the user can observe and do, rather than the internal machinery you used to produce it.**

React Testing Library is built around this user-centric approach, and current React guidance recommends it over the deprecated `react-test-renderer` approach for modern React component testing. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))

---

# 1. First: what is a test?

A test is simply a program that asks:

```text
Given this situation,
when I do this,
do I get that result?
```

For example:

```text
Given a counter starts at 0

when I click "Increment"

then I should see 1
```

That can be expressed as:

```js
expect(screen.getByText("1")).toBeInTheDocument();
```

So fundamentally:

```text
Arrange
  ↓
Act
  ↓
Assert
```

This is often called the AAA pattern:

```text
Arrange → prepare the situation

Act → perform the behavior

Assert → verify the result
```

React's `act()` helper uses this same conceptual model when ensuring updates triggered by an interaction have been processed before assertions are made. ([React](https://react.dev/reference/react/act?utm_source=chatgpt.com "act – React"))

---

# 2. What exactly are we testing?

Imagine this component:

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>Count: {count}</p>

      <button
        onClick={() => setCount(c => c + 1)}
      >
        Increment
      </button>
    </>
  );
}
```

You could test:

```text
state contains 0
setCount exists
the click handler is an arrow function
the component uses useState
```

But none of those are really the user's concern.

The user cares:

```text
I see "Count: 0"

I click "Increment"

I see "Count: 1"
```

So that's what a good component test should usually verify.

---

# 3. This is the Testing Library philosophy

React Testing Library deliberately encourages tests to resemble how people use the software, rather than inspecting component internals.

That means interacting with actual DOM nodes and finding them through things such as:

```text
role
label
text
accessible name
```

rather than:

```text
component instance
internal state
private methods
implementation-specific structure
```

Testing Library's guiding principle is essentially that the closer the test resembles real use, the more confidence it provides. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))

This connects directly to our accessibility lesson.

---

# 4. Accessibility and testing are surprisingly connected

Suppose:

```jsx
<button>Save</button>
```

Testing Library lets you write:

```js
screen.getByRole("button", {
  name: "Save",
});
```

That means your test asks:

> "Can I find the thing the user perceives as a button called Save?"

That's a stronger test than:

```js
container.querySelector(".save-button")
```

because the latter is tied to implementation details.

Testing Library explicitly prioritizes semantic queries, with `getByRole` generally recommended first, followed by queries such as `getByLabelText`. ([Testing Library](https://testing-library.com/docs/queries/about/?utm_source=chatgpt.com "About Queries | Testing Library"))

So good accessibility often makes your tests better too.

---

# 5. Your first React test

A simple test might look like:

```js
import {
  render,
  screen,
} from "@testing-library/react";

import Counter from "./Counter";

test("shows the initial count", () => {
  render(<Counter />);

  expect(
    screen.getByText("Count: 0")
  ).toBeInTheDocument();
});
```

Let's unpack it.

### `render`

```js
render(<Counter />);
```

This creates the component in a test DOM environment.

### `screen`

```js
screen.getByText(...)
```

This searches the rendered DOM.

### `expect`

```js
expect(...)
```

asks whether the result satisfies a condition.

The important mental model is:

```text
component
 ↓
render
 ↓
DOM
 ↓
query DOM
 ↓
assert behavior
```

React Testing Library's current `render` API creates the React tree using the client root and exposes DOM queries for the result. ([Testing Library](https://testing-library.com/docs/react-testing-library/api/?utm_source=chatgpt.com "API | Testing Library"))

---

# 6. Test interaction, not just initial rendering

A much more valuable test is:

```js
import {
  render,
  screen,
} from "@testing-library/react";

import userEvent from
  "@testing-library/user-event";

import Counter from "./Counter";

test("increments the count", async () => {
  const user = userEvent.setup();

  render(<Counter />);

  await user.click(
    screen.getByRole("button", {
      name: "Increment",
    })
  );

  expect(
    screen.getByText("Count: 1")
  ).toBeInTheDocument();
});
```

The sequence is:

```text
render
 ↓
find button
 ↓
click it as a user would
 ↓
React state changes
 ↓
component rerenders
 ↓
DOM updates
 ↓
assert new UI
```

This is much more valuable than testing whether:

```text
setCount was called once
```

because the user doesn't care about the setter call.

They care about the result.

---

# 7. Why `userEvent` instead of low-level `fireEvent`?

You will encounter both.

```js
fireEvent.click(button);
```

and:

```js
await user.click(button);
```

They are not identical.

`fireEvent` dispatches a lower-level DOM event.

`user-event` attempts to model fuller user interactions, potentially triggering the sequence of browser events and checks associated with those interactions. Testing Library currently recommends `user-event` for this more realistic interaction model. ([Testing Library](https://testing-library.com/docs/user-event/intro/?utm_source=chatgpt.com "Introduction | Testing Library"))

So roughly:

```text
fireEvent
→ "dispatch this event"

userEvent
→ "perform this user interaction"
```

For normal component interaction tests, `userEvent` is often the better mental model.

---

# 8. Why does `user.click()` return a Promise?

Because a realistic interaction may involve asynchronous steps.

So:

```js
await user.click(button);
```

is normal.

Don't think:

```text
"Why is clicking a button async?"
```

Think:

> The testing tool is modeling a browser interaction that can involve multiple events and React updates.

That also reduces subtle timing problems.

Testing Library's current `user-event` documentation recommends setting up a user instance and awaiting interactions. ([Testing Library](https://testing-library.com/docs/user-event/intro/?utm_source=chatgpt.com "Introduction | Testing Library"))

---

# 9. Query types: `getBy`, `queryBy`, `findBy`

This is a very important testing concept.

There are three families you should understand:

```text
getBy
queryBy
findBy
```

They answer different questions.

---

## `getBy`

Use when:

> "This element should already exist."

```js
screen.getByRole("button", {
  name: "Save",
});
```

If it isn't found, the query throws an error.

That's useful in tests because:

```text
expected
 ↓
missing
 ↓
test fails
```

---

## `queryBy`

Use when:

> "This element may not exist, and I want to check its absence."

```js
expect(
  screen.queryByText("Error")
).not.toBeInTheDocument();
```

If no matching element exists:

```text
queryBy → null
```

So it doesn't throw just because nothing was found.

Testing Library explicitly recommends this distinction for asserting absence. ([Testing Library](https://testing-library.com/docs/queries/about/?utm_source=chatgpt.com "About Queries | Testing Library"))

---

## `findBy`

Use when:

> "This element should appear asynchronously."

```js
const heading =
  await screen.findByRole("heading", {
    name: "Products",
  });
```

The query waits/retries until the element appears or the timeout is reached. ([Testing Library](https://testing-library.com/docs/queries/about/?utm_source=chatgpt.com "About Queries | Testing Library"))

So:

```text
getBy
→ must be there now

queryBy
→ might not be there

findBy
→ should appear eventually
```

This distinction is incredibly useful.

---

# 10. Example: loading → success

Suppose:

```jsx
function Products() {
  const query = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });

  if (query.isPending) {
    return <p>Loading...</p>;
  }

  if (query.isError) {
    return <p>Failed to load.</p>;
  }

  return (
    <ul>
      {query.data.map(product => (
        <li key={product.id}>
          {product.name}
        </li>
      ))}
    </ul>
  );
}
```

A test might assert:

```js
render(<Products />);

expect(
  screen.getByText("Loading...")
).toBeInTheDocument();
```

Then:

```js
expect(
  await screen.findByText("Laptop")
).toBeInTheDocument();
```

So we're testing the user-visible lifecycle:

```text
loading
 ↓
data arrives
 ↓
products visible
```

Not:

```text
useQuery called exactly once
```

---

# 11. Testing absence

Suppose a modal closes.

Before:

```text
Delete product?
```

After clicking Cancel:

```text
nothing
```

The test:

```js
await user.click(
  screen.getByRole("button", {
    name: "Cancel",
  })
);

expect(
  screen.queryByRole("dialog")
).not.toBeInTheDocument();
```

That's a good test because it describes behavior:

> "After Cancel, the dialog is no longer present."

---

# 12. Test the outcome, not React state

Bad:

```js
expect(result.current.isOpen).toBe(false);
```

That's testing an implementation detail.

Better:

```js
expect(
  screen.queryByRole("dialog")
).not.toBeInTheDocument();
```

Because that's what the user experiences.

Testing Library explicitly encourages avoiding internal component state and implementation details. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))

---

# 13. Testing forms

Consider:

```jsx
function LoginForm({ onSubmit }) {
  const [email, setEmail] = useState("");

  return (
    <form
      onSubmit={e => {
        e.preventDefault();
        onSubmit({ email });
      }}
    >
      <label htmlFor="email">
        Email
      </label>

      <input
        id="email"
        value={email}
        onChange={e =>
          setEmail(e.target.value)
        }
      />

      <button type="submit">
        Log in
      </button>
    </form>
  );
}
```

A good test:

```js
test("submits the entered email", async () => {
  const user = userEvent.setup();

  const handleSubmit = vi.fn();

  render(
    <LoginForm
      onSubmit={handleSubmit}
    />
  );

  const input =
    screen.getByLabelText("Email");

  await user.type(
    input,
    "alice@example.com"
  );

  await user.click(
    screen.getByRole("button", {
      name: "Log in",
    })
  );

  expect(handleSubmit).toHaveBeenCalledWith({
    email: "alice@example.com",
  });
});
```

This test actually walks through the user workflow:

```text
find field by label
 ↓
type
 ↓
submit
 ↓
callback receives expected data
```

That's excellent.

---

# 14. But is checking `handleSubmit` an implementation detail?

Not necessarily.

This is an important nuance.

Suppose `LoginForm` is explicitly designed to call:

```text
onSubmit(formData)
```

That callback is part of the component's **public contract**.

So testing:

```js
expect(handleSubmit).toHaveBeenCalledWith(...)
```

can be legitimate.

The distinction is:

```text
private internal function
→ implementation detail

public callback prop
→ component API
```

Testing the API of your component is reasonable.

---

# 15. Mock functions

You just saw:

```js
vi.fn()
```

This creates a spy/mock function in Vitest.

For example:

```js
const handleSubmit = vi.fn();
```

Then:

```js
expect(handleSubmit).toHaveBeenCalledTimes(1);
```

or:

```js
expect(handleSubmit).toHaveBeenCalledWith({
  email: "alice@example.com",
});
```

This is useful for verifying interactions with a component's public callback API.

Vitest is a current popular test runner choice for modern Vite/React projects; Testing Library itself is test-runner agnostic. Testing Library explicitly says it is not a test runner and can work with different test frameworks. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))

---

# 16. Component tests vs unit tests

This distinction causes a lot of confusion.

Imagine you have:

```js
function calculateTotal(items) {
  return items.reduce(
    (total, item) =>
      total + item.price,
    0
  );
}
```

You can test this directly:

```js
test("calculates the total", () => {
  expect(
    calculateTotal([
      { price: 10 },
      { price: 20 },
    ])
  ).toBe(30);
});
```

That's a traditional **unit test**.

You're testing a pure function.

---

Now:

```jsx
<Cart items={items} />
```

You can render it and interact with it.

That's a **component/UI test**.

So:

```text
pure function
→ direct unit test

React component
→ render + interaction + DOM assertions
```

Both are useful.

---

# 17. Test pure logic directly

This is one place where your earlier JavaScript skills help enormously.

Suppose:

```js
function getVisibleTodos(
  todos,
  filter
) {
  if (filter === "active") {
    return todos.filter(
      todo => !todo.completed
    );
  }

  if (filter === "completed") {
    return todos.filter(
      todo => todo.completed
    );
  }

  return todos;
}
```

There's no reason to render React to test that.

Just test:

```js
expect(
  getVisibleTodos(
    [
      { id: 1, completed: false },
      { id: 2, completed: true },
    ],
    "active"
  )
).toEqual([
  { id: 1, completed: false },
]);
```

This is fast and precise.

So:

> **Don't use a browser-like UI test for logic that can be tested directly as pure JavaScript.**

---

# 18. Then test the React layer separately

Suppose:

```jsx
function TodoList({
  todos,
  filter,
}) {
  const visible =
    getVisibleTodos(todos, filter);

  return (
    <ul>
      {visible.map(todo => (
        <li key={todo.id}>
          {todo.title}
        </li>
      ))}
    </ul>
  );
}
```

Your component test can simply ask:

```text
Given these todos
and this filter

does the user see the right ones?
```

This gives you:

```text
pure logic test
+
UI behavior test
```

rather than one giant test attempting to prove every internal line.

---

# 19. Testing Context

Suppose:

```jsx
function ThemeButton() {
  const theme =
    useContext(ThemeContext);

  return (
    <button>
      Current theme: {theme}
    </button>
  );
}
```

You might need to render it under the provider:

```jsx
render(
  <ThemeContext value="dark">
    <ThemeButton />
  </ThemeContext>
);
```

Then assert:

```js
expect(
  screen.getByRole("button", {
    name: "Current theme: dark",
  })
).toBeInTheDocument();
```

Notice the test is interested in:

```text
provider
 ↓
component
 ↓
visible result
```

not:

```text
"context internally calls useContext"
```

---

# 20. Reusable test setup with a `wrapper`

Suppose your app commonly requires:

```text
QueryClientProvider
ThemeContext
Router
```

You don't want to repeat all that in every test.

Testing Library's `render` API supports a `wrapper` option, which is particularly useful for reusable provider setup. ([Testing Library](https://testing-library.com/docs/react-testing-library/api/?utm_source=chatgpt.com "API | Testing Library"))

You can create:

```jsx
function TestProviders({
  children,
}) {
  const queryClient =
    new QueryClient({
      defaultOptions: {
        queries: {
          retry: false,
        },
      },
    });

  return (
    <QueryClientProvider
      client={queryClient}
    >
      <ThemeContext value="light">
        {children}
      </ThemeContext>
    </QueryClientProvider>
  );
}
```

Then:

```js
render(<ProductPage />, {
  wrapper: TestProviders,
});
```

Now your tests can focus on behavior rather than provider boilerplate.

---

# 21. Testing router-aware components

Suppose:

```jsx
function ProductLink({ id }) {
  return (
    <Link to={`/products/${id}`}>
      View product
    </Link>
  );
}
```

You may need to render it under a router.

The conceptual test is:

```text
render in router context
 ↓
find link
 ↓
click
 ↓
navigation occurs
 ↓
user sees new route content
```

The important point is:

> **The test should reproduce enough of the environment that the component's actual contract can operate.**

Don't mock React Router's entire internal behavior unless you specifically have a reason to test that boundary.

---

# 22. Testing server state

This is where our TanStack Query lessons come back.

Suppose:

```jsx
function ProductList() {
  const query = useQuery({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });

  ...
}
```

You don't want a component test making a real request to the production API.

Instead, you provide a controlled fake server response.

Conceptually:

```text
test
 ↓
controlled API response
 ↓
React Query
 ↓
component
 ↓
assert UI
```

A modern testing architecture commonly uses an HTTP mocking layer rather than mocking `fetch()` line-by-line inside every component test.

This lets you test:

```text
"when the API returns these products,
does the UI behave correctly?"
```

rather than:

```text
"did fetch() receive this exact argument?"
```

That's a much more meaningful test boundary.

---

# 23. Example server-state scenarios

Your product test suite might cover:

```text
loading

successful response

empty response

server error

retry/recovery

pagination

filter changes

mutation success

mutation failure
```

These correspond directly to the state machine we've already learned.

For example:

```text
idle
 ↓
pending
 ↓
success
```

or:

```text
pending
 ↓
error
```

Testing is now simply verifying that those states produce the correct user experience.

---

# 24. Error testing

Suppose:

```jsx
if (query.isError) {
  return (
    <p>
      Couldn't load products.
    </p>
  );
}
```

Your test should assert:

```js
expect(
  screen.getByText(
    "Couldn't load products."
  )
).toBeInTheDocument();
```

Not:

```js
expect(query.isError).toBe(true);
```

Again:

```text
state implementation
→ not the main concern

visible failure behavior
→ yes
```

---

# 25. Testing Error Boundaries

Suppose your app has:

```jsx
<ErrorBoundary
  fallback={<p>Section failed.</p>}
>
  <BuggyComponent />
</ErrorBoundary>
```

You can intentionally create a test component that throws:

```jsx
function ThrowingComponent() {
  throw new Error("boom");
}
```

Then:

```js
render(
  <ErrorBoundary
    fallback={
      <p>Section failed.</p>
    }
  >
    <ThrowingComponent />
  </ErrorBoundary>
);
```

Then assert:

```js
expect(
  screen.getByText(
    "Section failed."
  )
).toBeInTheDocument();
```

Now you're testing the behavior:

```text
child crashes
 ↓
boundary catches
 ↓
fallback appears
```

Exactly the model we learned earlier.

---

# 26. Async testing and `act`

You may eventually see warnings like:

```text
An update to Component was not wrapped in act(...)
```

The underlying issue is:

> React updated something, but your test asserted before React had finished processing the related update.

React's `act()` helper exists to flush updates before assertions, and Testing Library's higher-level helpers are designed to handle `act()` for common interactions. ([React](https://react.dev/reference/react/act?utm_source=chatgpt.com "act – React"))

So this:

```js
await user.click(button);
```

is preferable to manually wrapping every normal interaction in:

```js
await act(async () => {
  ...
});
```

when Testing Library already handles the relevant React update coordination.

---

# 27. Why manual `act()` still matters

Even though Testing Library handles many common cases, you should understand `act`.

Suppose you're testing React directly:

```jsx
const root =
  createRoot(container);

await act(async () => {
  root.render(<Counter />);
});
```

Then:

```js
expect(...);
```

React's current guidance recommends the async form:

```js
await act(async () => {
  ...
});
```

and notes that the synchronous form will be removed in the future. ([React](https://react.dev/reference/react/act?utm_source=chatgpt.com "act – React"))

The important mental model is:

```text
perform React-related unit of work
 ↓
flush pending React updates
 ↓
make assertion
```

---

# 28. Snapshot testing

You may encounter:

```text
snapshot tests
```

A snapshot captures a representation of rendered output and compares later renders against it.

They're not inherently useless.

But blindly snapshotting large component trees often produces:

```text
huge diff
```

without making it obvious what behavior actually broke.

Compare:

```text
Expected:
button says "Save"

vs

200-line snapshot changed
```

The first is much easier to understand.

That's one reason user-oriented assertions are often more valuable than large snapshots for ordinary component behavior.

---

# 29. What not to test

Suppose your component internally does:

```jsx
const [open, setOpen] = useState(false);
```

Don't normally test:

```text
"state variable open starts false"
```

Suppose it uses:

```js
const handleClick = () => ...
```

Don't normally test:

```text
"handleClick is an arrow function"
```

Suppose it uses:

```jsx
useMemo(...)
```

Don't test:

```text
"useMemo was used"
```

Those are implementation choices.

Tomorrow you might rewrite:

```text
useState
```

into:

```text
useReducer
```

without changing the behavior.

A good user-centric test should continue passing.

That's exactly what Testing Library means by avoiding implementation details. ([Testing Library](https://testing-library.com/docs/?utm_source=chatgpt.com "Introduction | Testing Library"))

---

# 30. This gives us a powerful test-quality rule

Imagine a refactor:

```text
Version A
  useState
  map
  conditional rendering

       ↓ refactor

Version B
  reducer
  custom hook
  different internal component structure
```

If behavior stays the same:

```text
good tests should mostly continue passing
```

If your tests all break:

```text
because the component's internal structure changed
```

then your tests were probably coupled too tightly to implementation.

That's one of the strongest tests of test quality itself.

---

# 31. There is another useful layer: integration tests

Suppose you have:

```text
SearchForm
 ↓
URL state
 ↓
TanStack Query
 ↓
ProductList
```

You could test every tiny piece separately.

But you also want a test that asks:

> "Can a user search for a product and actually see the results?"

That is closer to an **integration test**.

Example:

```text
render page
 ↓
type "laptop"
 ↓
submit/search
 ↓
URL changes
 ↓
query changes
 ↓
mock server responds
 ↓
results appear
```

This can catch bugs that unit tests of each isolated component would miss.

---

# 32. Unit vs integration vs end-to-end

A useful mental model:

### Unit test

```text
one small piece
```

Example:

### Component/integration test

```text
several React pieces working together
```

Example:

```text
 + Router
 + Query
 + results
```

### End-to-end test

```text
actual application in a browser
+
real-ish infrastructure
```

Example:

```text
open app
 ↓
login
 ↓
search
 ↓
add product
 ↓
checkout
```

The more realistic the test, generally the more expensive it is.

So you don't replace every unit test with an end-to-end test.

You use the different layers intentionally.

---

# 33. A practical testing pyramid

A rough mental model:

```text
             E2E
           /      \
        integration
       /            \
    component      component
      \              /
          unit tests
```

Usually:

```text
many small fast tests
+
fewer integration tests
+
fewer expensive end-to-end tests
```

You want confidence without making the test suite painfully slow.

The exact balance depends on the application.

---

# 34. Testing a component with a callback prop

Let's build a good example.

```jsx
function DeleteButton({ onDelete }) {
  return (
    <button onClick={onDelete}>
      Delete
    </button>
  );
}
```

Test:

```js
test("calls onDelete when clicked", async () => {
  const user = userEvent.setup();
  const onDelete = vi.fn();

  render(
    <DeleteButton
      onDelete={onDelete}
    />
  );

  await user.click(
    screen.getByRole("button", {
      name: "Delete",
    })
  );

  expect(onDelete)
    .toHaveBeenCalledTimes(1);
});
```

That's a good test because `onDelete` is part of the component's external behavior.

The test says:

> "When the user clicks Delete, this component notifies its owner."

That's an actual contract.

---

# 35. Testing controlled inputs

Suppose:

```jsx
function SearchBox({
  value,
  onChange,
}) {
  return (
    <input
      aria-label="Search"
      value={value}
      onChange={e =>
        onChange(e.target.value)
      }
    />
  );
}
```

Test:

```js
test("reports the user's search text", async () => {
  const user = userEvent.setup();
  const onChange = vi.fn();

  render(
    <SearchBox
      value=""
      onChange={onChange}
    />
  );

  await user.type(
    screen.getByRole("textbox", {
      name: "Search",
    }),
    "react"
  );

  expect(onChange)
    .toHaveBeenLastCalledWith("react");
});
```

Again:

```text
user types
 ↓
event
 ↓
public callback
 ↓
assert
```

That's exactly the behavior we learned when we studied controlled forms.

---

# 36. Testing a custom hook

You may eventually see:

```js
function useCounter() {
  const [count, setCount] =
    useState(0);

  return {
    count,
    increment: () =>
      setCount(c => c + 1),
  };
}
```

There are utilities for testing hooks directly, and Testing Library currently provides `renderHook`. But its current guidance says ordinary component tests are generally preferable when a hook can be tested naturally through a component, because that keeps the test closer to actual use. ([Testing Library](https://testing-library.com/docs/react-testing-library/api/?utm_source=chatgpt.com "API | Testing Library"))

So:

```text
custom hook has meaningful component behavior
→ test through component

library-level hook is itself the thing being published/tested
→ direct hook testing can make sense
```

That's another example of choosing the right test boundary.

---

# 37. Testing refs

Suppose:

```jsx
function SearchInput({ ref }) {
  return <input ref={ref} />;
}
```

You usually don't need a test that says:

```text
"ref is attached"
```

unless the ref attachment itself is part of the public behavior.

Instead, test the user-level effect:

```text
click "Focus"
 ↓
input receives focus
```

For example:

```js
const input =
  screen.getByRole("textbox");

await user.click(
  screen.getByRole("button", {
    name: "Focus",
  })
);

expect(input).toHaveFocus();
```

That is much better.

You aren't checking the implementation:

```text
"useRef was used"
```

You're checking:

```text
the field became focused
```

---

# 38. Testing accessibility itself

This is one of the nicest benefits of user-centric testing.

Suppose your component has:

```jsx
<label htmlFor={id}>
  Email
</label>

<input id={id} />
```

Your test can do:

```js
screen.getByLabelText("Email");
```

If somebody later breaks the label association, the test fails.

So your test suite isn't only checking behavior.

It's helping protect accessibility structure too.

Testing Library explicitly prioritizes semantic and accessible queries for this reason. ([Testing Library](https://testing-library.com/docs/queries/about/?utm_source=chatgpt.com "About Queries | Testing Library"))

---

# 39. Test IDs are the escape hatch

Sometimes there genuinely isn't a useful semantic query.

Then:

```jsx
<div data-testid="chart-canvas" />
```

and:

```js
screen.getByTestId("chart-canvas");
```

can be reasonable.

But Testing Library explicitly recommends test IDs lower in the query priority because users don't interact with them. ([Testing Library](https://testing-library.com/docs/queries/about/?utm_source=chatgpt.com "About Queries | Testing Library"))

So think:

```text
role/label/text
      ↓
semantic attributes
      ↓
test id
```

Test IDs are not evil.

They're just less user-centric.

---

# 40. Testing a modal from our earlier lessons

Imagine:

```text
ProductPage
 ↓
Delete button
 ↓
Modal
 ↓
Delete / Cancel
```

A good integration-style test:

```text
render product page

find Delete button

click Delete

assert dialog appears

find dialog heading:
"Delete product?"

click Cancel

assert dialog disappears
```

Notice we aren't testing:

```text
setOpen(true)
setOpen(false)
portal implementation
DOM placement in body
useState calls
```

We're testing the experience.

---

# 41. Testing a portal

This is interesting.

Because the modal is portaled into `document.body`, a good test should still generally query the user-visible result:

```js
screen.getByRole("dialog");
```

not:

```js
container.querySelector(...)
```

This is another place where `screen` and Testing Library's document-level queries are especially useful.

The current `render` API uses the document as the normal query context, and the library's guidance generally prefers semantic queries over digging into `container`. ([Testing Library](https://testing-library.com/docs/react-testing-library/api/?utm_source=chatgpt.com "API | Testing Library"))

So our portal lesson and testing lesson fit together nicely:

```text
logical React tree
+
physical DOM location
```

and tests simply ask:

> "Can the user find and interact with the dialog?"

---

# 42. Testing transitions and async UI

Suppose:

```jsx
startTransition(() => {
  setTab("settings");
});
```

You generally don't test:

```text
"startTransition was called"
```

Instead test:

```text
click Settings

eventually Settings content appears
```

Similarly, for Suspense:

```text
content not ready
 ↓
fallback visible
 ↓
content ready
 ↓
actual content visible
```

Your test should focus on that user-observable lifecycle.

---

# 43. A note about `react-test-renderer`

If you encounter older tutorials that say:

```js
import renderer from "react-test-renderer";
```

don't copy that blindly into a new React project.

Current React marks `react-test-renderer` as deprecated and recommends Testing Library for modern React component testing. ([React](https://react.dev/warnings/react-test-renderer?utm_source=chatgpt.com "react-test-renderer Deprecation Warnings – React"))

This is a good example of why I don't want you memorizing old React patterns from random tutorials.

The underlying testing concepts matter more than one library's historical API.

---

# 44. What a strong React test reads like

Good:

```js
test("shows an error when saving fails", ...)
```

Good:

```js
test("opens the delete dialog", ...)
```

Good:

```js
test("adds the new todo after submission", ...)
```

Good:

```js
test("keeps the previous results visible while the next page loads", ...)
```

These names describe **behavior**.

Less useful:

```text
test("useState works", ...)
test("memo is called", ...)
test("handleClick function exists", ...)
test("div is rendered", ...)
```

Those tell you how the code is implemented rather than what the application promises.

---

# 45. Testing is another form of API thinking

We've repeatedly talked about component APIs.

A component has inputs:

```text
props
context
environment
user actions
```

and outputs:

```text
DOM
callbacks
visible behavior
side effects
```

Testing is essentially checking:

```text
INPUT
 ↓
behavior
 ↓
OUTPUT
```

That means a useful test boundary often matches a useful component boundary.

This is one reason good component design and good testing tend to reinforce each other.

---

# 46. The most important testing heuristic

When writing a test, ask:

> **"Could I explain this test without mentioning React implementation details?"**

For example:

```text
"When the user types an invalid email,
the error message appears."
```

Excellent.

But:

```text
"When `setEmail` updates state,
the component calls `setError` in the effect..."
```

Now we're testing implementation mechanics.

That may be necessary in some low-level tests, but it shouldn't be the default.

---

# 47. A practical testing stack

For a modern React application, you may encounter something like:

```text
Vitest
   ↓
test runner + assertions/mocking

React Testing Library
   ↓
render React components + query DOM

user-event
   ↓
simulate user interactions

jest-dom
   ↓
DOM-specific assertions
```

Testing Library itself is deliberately not tied to one test runner. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))

The exact tool choices can vary, but the conceptual roles are what matter.

---

# 48. A complete example

Suppose:

```jsx
function TodoForm({ onAdd }) {
  const [text, setText] =
    useState("");

  function handleSubmit(e) {
    e.preventDefault();

    if (!text.trim()) {
      return;
    }

    onAdd(text.trim());
    setText("");
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="todo">
        Todo
      </label>

      <input
        id="todo"
        value={text}
        onChange={e =>
          setText(e.target.value)
        }
      />

      <button type="submit">
        Add
      </button>
    </form>
  );
}
```

A strong test suite might include:

```text
renders the form

types text

does not submit blank input

submits trimmed text

clears input after successful submission
```

Notice how those are behavior statements.

---

# 49. The blank-input test

```js
test("does not add an empty todo", async () => {
  const user = userEvent.setup();
  const onAdd = vi.fn();

  render(
    <TodoForm onAdd={onAdd} />
  );

  await user.click(
    screen.getByRole("button", {
      name: "Add",
    })
  );

  expect(onAdd)
    .not.toHaveBeenCalled();
});
```

This is a very nice test because it protects a real requirement.

Now suppose six months later somebody accidentally changes:

```js
if (!text.trim()) {
  return;
}
```

to:

```js
if (!text) {
  return;
}
```

The test might catch behavior around whitespace inputs.

Again:

```text
test
→ behavior contract
```

not:

```text
specific implementation
```

---

# 50. What should you test heavily?

Prioritize things that matter to users:

```text
important workflows
forms
authentication flows
navigation
loading/error states
mutations
permissions
data transformation that affects UI
accessibility-critical behavior
```

Don't spend 20 tests proving trivial markup:

```text
<div exists>
<p exists>
wrapper has class
```

unless those details are genuinely part of the contract.

---

# 51. One final connection: testing and refactoring

A good test suite should give you freedom.

Suppose you refactor:

```text
5 components
 ↓
2 components
```

or:

```text
useState
 ↓
useReducer
```

or:

```text
manual fetch
 ↓
TanStack Query
```

If user behavior remains unchanged, a well-designed test suite should mostly stay green.

That's extremely valuable.

A test suite should not punish you merely for improving your implementation.

Testing Library explicitly emphasizes this maintainability goal: tests focused on user behavior are less likely to break during implementation-only refactors. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))

---

# 52. The mental model I want you to keep

When you write a React test, think:

```text
WHAT DOES THE USER SEE?
        ↓
WHAT CAN THE USER DO?
        ↓
WHAT SHOULD HAPPEN?
        ↓
CAN I ASSERT THAT?
```

Then choose the appropriate layer:

```text
pure calculation
→ unit test

component interaction
→ React Testing Library

several app layers working together
→ integration test

whole business workflow
→ end-to-end test
```

And for queries:

```text
getBy
→ should exist now

queryBy
→ may not exist

findBy
→ should appear eventually
```

And for interactions:

```text
userEvent
→ realistic interaction
```

And for assertions:

```text
assert the result
not the implementation
```

---

# 53. The deeper lesson

Testing is not really about:

```text
"How can I verify my React code?"
```

It's about:

> **"What does this component promise, and how can I automatically protect that promise?"**

For example:

```text
Button
→ clicking it invokes the requested action

Form
→ valid submission produces expected data

Modal
→ opens, closes, and manages interaction correctly

Product page
→ loading/error/success states behave correctly

Search
→ user can type and see matching results

Mutation
→ successful changes appear in the UI

Error Boundary
→ failed region gets contained
```

Once you think in **behavioral contracts**, testing becomes much more intuitive.

And this lesson ties together a surprising amount of what we've already learned:

```text
React rendering
     +
state
     +
events
     +
forms
     +
routing
     +
server state
     +
mutations
     +
Suspense
     +
Error Boundaries
     +
accessibility
     +
component API design
     ↓
testable behavior
```

That's why testing is worth learning now: you're no longer testing tiny isolated React examples. You're learning how to protect the behavior of the larger systems we've been building. ([Testing Library](https://testing-library.com/docs/react-testing-library/intro/?utm_source=chatgpt.com "React Testing Library | Testing Library"))