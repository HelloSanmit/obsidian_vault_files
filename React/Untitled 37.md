
# Lesson 34 — React 19 Actions: `useActionState`, `useFormStatus`, and `useOptimistic`

You already understand:

- controlled forms
    
- `useState`
    
- `useReducer`
    
- mutations
    
- optimistic updates conceptually
    
- `startTransition`
    
- server state and TanStack Query
    

React 19 connects several of those ideas into a unified pattern called **Actions**. React 19.3 still exposes this model, including `useActionState`, `useFormStatus`, and `useOptimistic`. ([React](https://react.dev/blog/2026/09/09/react-19-3?utm_source=chatgpt.com "React 19.3 – React"))

The important thing is not memorizing three APIs.

The important thing is understanding **what problem React is trying to solve**.

---

# 1. The old way: we manually coordinate a mutation

Suppose we have a profile form.

```jsx
function ProfileForm() {
  const [name, setName] = useState("");
  const [isPending, setIsPending] = useState(false);
  const [error, setError] = useState(null);

  async function handleSubmit(event) {
    event.preventDefault();

    setIsPending(true);
    setError(null);

    try {
      const result = await updateProfile(name);

      if (!result.ok) {
        setError(result.error);
        return;
      }

      setName(result.name);
    } catch {
      setError("Something went wrong");
    } finally {
      setIsPending(false);
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}
        onChange={e => setName(e.target.value)}
        disabled={isPending}
      />

      <button disabled={isPending}>
        {isPending ? "Saving..." : "Save"}
      </button>

      {error && <p>{error}</p>}
    </form>
  );
}
```

This works.

But notice how much **coordination code** we wrote:

```text
submit
  ↓
set pending
  ↓
clear error
  ↓
call async function
  ↓
wait
  ↓
inspect result
  ↓
set success/error state
  ↓
clear pending
```

And we had to manually maintain several pieces of state:

```text
name
isPending
error
```

For more complicated forms, this grows quickly.

React Actions are designed to make this particular workflow more explicit.

---

# 2. What exactly is an Action?

The word **Action** in React has a specific practical meaning here.

Think of it as:

> An asynchronous operation that React can track as a transition.

Conceptually:

```text
user interaction
      ↓
Action starts
      ↓
pending
      ↓
async work
      ↓
result / error
      ↓
React updates UI
```

That means React can know:

> "This update is not just an ordinary synchronous state change. Something asynchronous is happening and this update belongs to that operation."

React 19 introduced this pattern together with form Actions and optimistic state. ([React](https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

---

# 3. `useActionState`

This is the first API to understand.

```jsx
const [state, dispatchAction, isPending] =
  useActionState(action, initialState);
```

You get three things:

```text
state
dispatchAction
isPending
```

Very roughly:

```text
state        → result of the latest Action
dispatchAction → trigger the Action
isPending    → Action currently running?
```

For example:

```jsx
async function saveName(previousState, name) {
  const result = await updateName(name);

  return result;
}

function Profile() {
  const [state, saveAction, isPending] =
    useActionState(saveName, {
      success: false,
      error: null
    });

  // ...
}
```

The Action receives:

```js
previousState
```

and the value you dispatch:

```js
actionPayload
```

So mentally:

```js
async function action(previousState, actionPayload) {
  // do something
  return newState;
}
```

This looks similar to a reducer.

And that's intentional.

React itself describes it as a kind of combination of:

```text
reducer-like state transition
        +
async Action
```

Unlike `useReducer`, the function can perform asynchronous side effects. ([React](https://react.dev/reference/react/useActionState?utm_source=chatgpt.com "useActionState – React"))

---

# 4. `useActionState` vs `useReducer`

This distinction matters a lot.

### `useReducer`

```js
function reducer(state, action) {
  return nextState;
}
```

The reducer should be pure.

Example:

```js
function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return {
        count: state.count + 1
      };

    default:
      return state;
  }
}
```

There is no network request inside it.

---

### `useActionState`

```js
async function action(previousState, payload) {
  const result = await saveToServer(payload);

  return {
    ...previousState,
    result
  };
}
```

This function can perform the asynchronous operation.

So:

```text
useReducer
    ↓
pure state transition

useActionState
    ↓
async operation
    ↓
result becomes state
```

That difference is fundamental.

---

# 5. A simple `useActionState` example

Let's build one.

```jsx
import { useActionState } from "react";

async function updateName(previousState, formData) {
  const name = formData.get("name");

  if (!name.trim()) {
    return {
      success: false,
      error: "Name is required"
    };
  }

  await fakeApiCall(name);

  return {
    success: true,
    error: null,
    name
  };
}

function ProfileForm() {
  const [state, submitAction, isPending] =
    useActionState(updateName, {
      success: false,
      error: null,
      name: ""
    });

  return (
    <form action={submitAction}>
      <input name="name" />

      <button disabled={isPending}>
        {isPending ? "Saving..." : "Save"}
      </button>

      {state.error && <p>{state.error}</p>}

      {state.success && (
        <p>Saved {state.name}</p>
      )}
    </form>
  );
}
```

There is a lot happening here.

---

# 6. Why `form action={submitAction}` is important

Normally, we learned forms like this:

```jsx
<form onSubmit={handleSubmit}>
```

React 19 lets a function be supplied to the form's `action`:

```jsx
<form action={submitAction}>
```

Now React manages the Action lifecycle around the submission. ([React](https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19 – React"))

When the browser submits the form, React effectively constructs the form payload:

```text
<input name="name" value="John">
```

becomes a `FormData` object.

Then your Action gets:

```js
previousState
formData
```

So:

```js
async function updateName(previousState, formData) {
  const name = formData.get("name");
}
```

---

# 7. Trace the complete execution

Suppose the user enters:

```text
John
```

and presses Save.

### Before submission

```js
state = {
  success: false,
  error: null,
  name: ""
}

isPending = false
```

---

### Step 1 — submit form

```text
<form action={submitAction}>
```

fires.

React starts an Action.

Conceptually:

```text
isPending
   ↓
true
```

---

### Step 2 — React collects form data

The browser form fields become:

```js
FormData {
  name: "John"
}
```

---

### Step 3 — Action receives it

React effectively invokes:

```js
updateName(
  previousState,
  formData
)
```

where:

```js
previousState = {
  success: false,
  error: null,
  name: ""
}
```

and:

```js
formData.get("name")
```

returns:

```text
John
```

---

### Step 4 — async work

```js
await fakeApiCall(name);
```

Now the Action is waiting.

The UI can represent:

```text
Saving...
```

because:

```js
isPending === true
```

---

### Step 5 — Action resolves

Suppose it returns:

```js
{
  success: true,
  error: null,
  name: "John"
}
```

React stores that as the new Action state.

So:

```js
state
```

becomes:

```js
{
  success: true,
  error: null,
  name: "John"
}
```

---

### Step 6 — pending ends

```js
isPending === false
```

React renders again.

The UI becomes:

```text
Saved John
```

The entire lifecycle is now represented by one Action instead of manually maintaining several pieces of mutation state.

---

# 8. Why is the first argument called `previousState`?

This becomes especially interesting when Actions are dispatched multiple times.

Consider:

```jsx
const [count, addAction, isPending] =
  useActionState(
    async (previousCount) => {
      await saveToServer(previousCount + 1);

      return previousCount + 1;
    },
    0
  );
```

Initially:

```text
count = 0
```

Dispatch once:

```text
previousCount = 0
```

returns:

```text
1
```

Then dispatch again:

```text
previousCount = 1
```

returns:

```text
2
```

Again:

```text
previousCount = 2
```

returns:

```text
3
```

React queues multiple calls sequentially for `useActionState`, passing each result to the next call. ([React](https://react.dev/reference/react/useActionState?utm_source=chatgpt.com "useActionState – React"))

That's why the reducer-like naming makes sense:

```text
previous state
      ↓
Action
      ↓
next state
      ↓
next Action receives it
```

---

# 9. A subtle but important consequence: Actions can queue

Imagine:

```jsx
<button onClick={add}>
  Add
</button>
```

and:

```js
function add() {
  startTransition(() => {
    dispatchAction();
  });
}
```

User clicks rapidly:

```text
click
click
click
click
```

React can queue these Action calls.

Conceptually:

```text
Action #1
  previousState = 0
      ↓
    result = 1

Action #2
  previousState = 1
      ↓
    result = 2

Action #3
  previousState = 2
      ↓
    result = 3
```

This is useful when operations logically depend on the result of the previous operation.

But it also means `useActionState` is **not automatically a magical parallel request manager**. React currently executes dispatched calls sequentially for this hook. ([React](https://react.dev/reference/react/useActionState?utm_source=chatgpt.com "useActionState – React"))

That distinction becomes important in systems like:

```text
search typing
file uploads
rapid clicks
dragging
high-frequency updates
```

Those may require different architectures.

---

# 10. `isPending`

You get:

```js
const [state, submitAction, isPending] =
  useActionState(...)
```

and then:

```jsx
<button disabled={isPending}>
  {isPending ? "Saving..." : "Save"}
</button>
```

This is much cleaner than:

```js
setIsPending(true);

try {
  await ...
} finally {
  setIsPending(false);
}
```

React now knows the lifecycle of the Action itself.

---

# 11. But what about child components?

Suppose your form looks like:

```jsx
<form action={submitAction}>
  <NameInput />
  <SubmitButton />
</form>
```

You don't necessarily want to pass:

```jsx
<SubmitButton pending={isPending} />
```

through props.

That's where `useFormStatus` comes in.

---

# 12. `useFormStatus`

Import:

```jsx
import { useFormStatus } from "react-dom";
```

Inside a component rendered **inside a form**:

```jsx
function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}
```

Then:

```jsx
function Form() {
  return (
    <form action={submitAction}>
      <input name="email" />

      <SubmitButton />
    </form>
  );
}
```

No prop drilling.

`useFormStatus` reads the status of the parent `<form>`. It currently exposes:

```js
{
  pending,
  data,
  method,
  action
}
```

where `data` is the submitted `FormData`, and `action` refers to the function used as the form's Action. ([React](https://react.dev/reference/react-dom/hooks/useFormStatus?utm_source=chatgpt.com "useFormStatus – React"))

---

# 13. The most common `useFormStatus` mistake

This does **not** work the way people often expect:

```jsx
function Form() {
  const { pending } = useFormStatus();

  return (
    <form action={submitAction}>
      <button disabled={pending}>
        Submit
      </button>
    </form>
  );
}
```

Why?

Because `useFormStatus()` observes a **parent form**.

At the point where this hook executes:

```text
Form component
   ↓
useFormStatus()
```

there isn't a parent `<form>` around the component.

The form is being returned _by_ the same component.

Instead:

```jsx
function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

function Form() {
  return (
    <form action={submitAction}>
      <SubmitButton />
    </form>
  );
}
```

Now:

```text
Form
 └── <form>
      └── SubmitButton
            └── useFormStatus()
```

The hook has a parent form to observe. ([React](https://react.dev/reference/react-dom/hooks/useFormStatus?utm_source=chatgpt.com "useFormStatus – React"))

This is a very useful mental model:

> `useFormStatus` is contextual information about the nearest parent form, not a hook that searches the component's returned JSX.

---

# 14. `useFormStatus.data`

This is a less obvious feature.

Suppose:

```jsx
function Status() {
  const { pending, data } = useFormStatus();

  if (!pending || !data) {
    return null;
  }

  return (
    <p>
      Submitting {data.get("email")}...
    </p>
  );
}
```

Your form:

```jsx
<form action={submitAction}>
  <input name="email" />
  <SubmitButton />
  <Status />
</form>
```

The form submits:

```text
email = alice@example.com
```

During submission:

```js
data.get("email")
```

can be read by the nested component.

So you can build reusable components such as:

```text
SubmitButton
SubmitSpinner
SubmitStatus
UploadProgressLabel
PendingSummary
```

without manually passing form state through every level.

---

# 15. Context analogy

You can think about:

```js
useFormStatus()
```

as conceptually similar to:

```text
"give me information about the form above me"
```

It's not ordinary React Context exposed directly to you, but the mental model is similar.

Imagine:

```text
<Form>
   ↓
form context
   ↓
 ┌───────────────┐
 │ SubmitButton  │ → pending
 │ ErrorMessage  │ → submission state
 │ Status        │ → submitted data
 └───────────────┘
```

That is why form-specific information doesn't need to become a giant prop chain.

---

# 16. Now the interesting one: `useOptimistic`

We've already discussed optimistic updates when learning server mutations.

The normal problem is:

```text
User clicks Like
      ↓
POST /likes
      ↓
wait 800ms
      ↓
server says success
      ↓
UI changes
```

That feels slow.

Users generally expect:

```text
User clicks Like
      ↓
UI immediately says liked
      ↓
request runs in background
      ↓
server confirms
```

That is an **optimistic update**.

---

# 17. Traditional optimistic state

You might previously write:

```jsx
const [liked, setLiked] = useState(false);
const [pending, setPending] = useState(false);

async function handleLike() {
  setLiked(true);
  setPending(true);

  try {
    await likePost();
  } catch {
    setLiked(false);
  } finally {
    setPending(false);
  }
}
```

Notice the bookkeeping.

We're manually saying:

```text
pretend new state
      ↓
perform request
      ↓
keep it if success
      ↓
undo it if failure
```

React 19 gives us:

```js
useOptimistic
```

to represent this temporary state more directly. ([React](https://react.dev/reference/react/useOptimistic?utm_source=chatgpt.com "useOptimistic – React"))

---

# 18. Basic `useOptimistic`

```jsx
const [optimisticLiked, setOptimisticLiked] =
  useOptimistic(liked);
```

Here:

```js
liked
```

is the real value.

And:

```js
optimisticLiked
```

is what the UI should temporarily display while the Action is pending.

---

# 19. The crucial distinction

Suppose:

```js
liked = false
```

Then:

```js
setOptimisticLiked(true)
```

does **not** mean:

```js
liked = true
```

It means:

```text
real state
false

temporary displayed state
true
```

So there are conceptually two layers:

```text
REAL STATE
    ↓
false

       +
       
OPTIMISTIC OVERLAY
    ↓
true

       ↓
UI sees true
```

This is a very important architecture concept.

---

# 20. Complete optimistic example

```jsx
import {
  useOptimistic,
  startTransition
} from "react";

function LikeButton({ liked, setLiked }) {
  const [optimisticLiked, setOptimisticLiked] =
    useOptimistic(liked);

  function handleLike() {
    startTransition(async () => {
      setOptimisticLiked(true);

      const result = await likePost();

      if (result.ok) {
        setLiked(true);
      }
    });
  }

  return (
    <button onClick={handleLike}>
      {optimisticLiked ? "❤️ Liked" : "🤍 Like"}
    </button>
  );
}
```

When the user clicks:

```text
liked = false
```

Then:

```js
setOptimisticLiked(true)
```

React immediately renders:

```text
❤️ Liked
```

Meanwhile:

```js
await likePost()
```

is happening.

---

# 21. What happens when the request fails?

This is where `useOptimistic` gets interesting.

Suppose:

```text
real state = false
optimistic state = true
```

Then the request fails:

```text
POST /likes → 500
```

Once the Action is no longer pending, the optimistic overlay falls back to the real value.

So the user sees:

```text
❤️ Liked
```

temporarily, then:

```text
🤍 Like
```

again.

That gives you rollback behavior without manually maintaining:

```js
previousLiked
```

in every component.

The optimistic value is temporary by design. ([React](https://react.dev/reference/react/useOptimistic?utm_source=chatgpt.com "useOptimistic – React"))

---

# 22. Why must optimistic updates happen inside an Action?

React wants optimistic state to have a clearly defined lifetime.

Think:

```text
Action begins
   ↓
optimistic state active
   ↓
async work
   ↓
Action finishes
   ↓
optimistic state disappears
```

Therefore:

```jsx
setOptimisticLiked(true);
```

should happen inside an Action/transition.

For example:

```js
startTransition(async () => {
  setOptimisticLiked(true);

  await saveChanges();
});
```

React can then associate the temporary UI with that particular asynchronous operation. ([React](https://react.dev/reference/react/useOptimistic?utm_source=chatgpt.com "useOptimistic – React"))

---

# 23. `useActionState` + `useOptimistic`

These APIs become especially useful together.

Let's build a realistic comment form.

We have:

```js
comments
```

as real server-backed state.

User submits:

```text
"Great post!"
```

We don't want:

```text
click
 ↓
wait 1 second
 ↓
comment appears
```

We want:

```text
click
 ↓
comment appears immediately
 ↓
request runs
 ↓
server confirms
```

Conceptually:

```text
             ┌─────────────┐
             │ Real state  │
             │ comments    │
             └──────┬──────┘
                    │
                    ▼
             useOptimistic
                    │
                    ▼
             temporary list
                    │
                    ▼
                   UI
```

---

# 24. Example

```jsx
import {
  useActionState,
  useOptimistic
} from "react";

async function addComment(previousState, formData) {
  const text = formData.get("text");

  const result = await createComment(text);

  if (!result.ok) {
    return {
      ...previousState,
      error: result.error
    };
  }

  return {
    comments: [
      ...previousState.comments,
      result.comment
    ],
    error: null
  };
}

function Comments({ comments }) {
  const [state, submitAction, isPending] =
    useActionState(
      addComment,
      {
        comments,
        error: null
      }
    );

  const [
    optimisticComments,
    addOptimisticComment
  ] = useOptimistic(
    state.comments,
    (currentComments, optimisticComment) => [
      ...currentComments,
      optimisticComment
    ]
  );

  return (
    <>
      <ul>
        {optimisticComments.map(comment => (
          <li key={comment.id}>
            {comment.text}
          </li>
        ))}
      </ul>

      <form action={async formData => {
        const text = formData.get("text");

        addOptimisticComment({
          id: crypto.randomUUID(),
          text
        });

        await submitAction(formData);
      }}>
        <input name="text" />

        <button disabled={isPending}>
          Add comment
        </button>
      </form>

      {state.error && (
        <p>{state.error}</p>
      )}
    </>
  );
}
```

This is a more advanced pattern, so let's slow down and trace it.

---

# 25. Trace the optimistic comment

Initial state:

```js
state.comments = [
  { id: 1, text: "Hello" }
]
```

Therefore:

```js
optimisticComments = [
  { id: 1, text: "Hello" }
]
```

User enters:

```text
Great post!
```

and submits.

---

### Phase 1 — add optimistic item

```js
addOptimisticComment({
  id: "temporary-id",
  text: "Great post!"
});
```

The UI immediately becomes:

```text
Hello
Great post!
```

But the server may not even know this comment exists yet.

That's the essence of optimistic UI.

---

### Phase 2 — server request

Then:

```js
await submitAction(formData);
```

which eventually calls:

```js
addComment(previousState, formData)
```

and:

```js
createComment(text)
```

runs.

---

### Phase 3 — server responds

Suppose the server returns:

```js
{
  id: 9281,
  text: "Great post!",
}
```

The real state now becomes:

```js
[
  { id: 1, text: "Hello" },
  { id: 9281, text: "Great post!" }
]
```

The optimistic version is no longer needed.

React can now render using the actual server-backed state.

---

# 26. Notice something subtle

Optimistic UI is **not the same thing as truth**.

This is one of the most important ideas in modern frontend development.

You effectively have:

```text
REALITY

server state
     ↓
actual authoritative data
```

and:

```text
EXPECTATION

optimistic state
     ↓
what we expect the server to accept
```

During the request:

```text
UI = expectation
```

After success:

```text
UI = reality
```

After failure:

```text
UI = reality
```

This is why optimistic state should be treated as temporary.

---

# 27. `useOptimistic` with a reducer

The second parameter is optional.

Basic form:

```js
const [optimisticValue, setOptimisticValue] =
  useOptimistic(value);
```

You provide the temporary value directly.

But for collections, a reducer can be useful:

```js
const [
  optimisticTodos,
  updateOptimisticTodos
] = useOptimistic(
  todos,
  (currentTodos, action) => {
    switch (action.type) {
      case "add":
        return [
          ...currentTodos,
          action.todo
        ];

      case "remove":
        return currentTodos.filter(
          todo => todo.id !== action.id
        );

      default:
        return currentTodos;
    }
  }
);
```

Then:

```js
updateOptimisticTodos({
  type: "add",
  todo: {
    id: "temp-1",
    text: "Learn Actions"
  }
});
```

Conceptually:

```text
real state
   +
optimistic action
   ↓
temporary derived state
```

The reducer for `useOptimistic` must remain pure. ([React](https://react.dev/reference/react/useOptimistic?utm_source=chatgpt.com "useOptimistic – React"))

---

# 28. Action vs mutation

At this point you might ask:

> "Isn't this basically TanStack Query mutation?"

No.

They operate at different levels.

### React Action

Primarily handles:

```text
UI interaction
      ↓
async operation
      ↓
pending UI
      ↓
result / optimistic rendering
```

### TanStack Query mutation

Primarily handles:

```text
server mutation
      ↓
cache consistency
      ↓
invalidate/update queries
      ↓
server state synchronization
```

You can use both.

For example:

```text
React form Action
       ↓
update server
       ↓
TanStack Query cache invalidation
       ↓
server state refresh
```

So don't replace your entire server-state architecture just because Actions exist.

---

# 29. A useful separation

Imagine an e-commerce app.

User presses:

```text
Add to cart
```

There are really several concerns:

```text
1. Did the user submit?
2. Is something currently pending?
3. What should the UI optimistically display?
4. What did the server actually return?
5. Which cached queries are now stale?
```

Different tools can own different concerns.

```text
React Action
   → submission lifecycle

useFormStatus
   → form-level pending UI

useOptimistic
   → temporary UI

TanStack Query
   → server cache
```

This is much cleaner than putting everything into one giant `useState`.

---

# 30. Action errors: two different categories

This is another subtle distinction.

Suppose your backend returns:

```json
{
  "ok": false,
  "error": "Username is already taken"
}
```

That's an **expected application error**.

Your Action can return it as state:

```js
return {
  success: false,
  error: "Username is already taken"
};
```

Then:

```jsx
{state.error && (
  <p>{state.error}</p>
)}
```

This is appropriate because the UI knows how to display it.

---

# 31. Unexpected errors are different

Suppose your code contains:

```js
user.profile.name.toUpperCase()
```

but `profile` is unexpectedly `null`.

That's not a normal form validation result.

That's a programming/runtime failure.

You may allow that error to propagate to an Error Boundary.

Conceptually:

```text
Expected business failure
      ↓
return error as Action state
      ↓
UI displays it

Unexpected programming failure
      ↓
throw
      ↓
Error Boundary
```

This fits the Error Boundary model we learned earlier. React's current `useActionState` behavior cancels queued Actions when an Action throws and surfaces the error to the nearest Error Boundary. ([React](https://react.dev/reference/react/useActionState?utm_source=chatgpt.com "useActionState – React"))

---

# 32. `useActionState` manually

You don't have to use it only with forms.

For example:

```jsx
import {
  useActionState,
  startTransition
} from "react";

async function increment(previousCount) {
  await wait(500);

  return previousCount + 1;
}

function Counter() {
  const [
    count,
    incrementAction,
    isPending
  ] = useActionState(increment, 0);

  function handleClick() {
    startTransition(() => {
      incrementAction();
    });
  }

  return (
    <>
      <p>{count}</p>

      <button
        onClick={handleClick}
        disabled={isPending}
      >
        Increment
      </button>
    </>
  );
}
```

Notice:

```js
incrementAction();
```

is wrapped in:

```js
startTransition(...)
```

when manually dispatched.

When an Action is supplied directly to a supported Action prop such as a form's `action`, React handles the transition wrapping for you. ([React](https://react.dev/reference/react/useActionState?utm_source=chatgpt.com "useActionState – React"))

---

# 33. Why React requires the Action context

Remember our earlier lesson on transitions.

A Transition tells React:

```text
"This work is non-urgent."
```

Actions extend that idea:

```text
"This asynchronous piece of work represents one logical UI operation."
```

So React can associate:

```text
pending
optimistic state
result
error
```

with the same operation.

That relationship is what makes these APIs powerful.

---

# 34. Before Actions vs after Actions

### Before

You manually assemble:

```text
useState
useState
useState
useEffect sometimes
try/catch
finally
setPending
setError
rollback
form event handling
```

### With Actions

You can express:

```text
Action
   ↓
state result
   +
pending
   +
optimistic UI
   +
form status
```

It's less about "fewer lines" and more about **putting related asynchronous behavior under one conceptual unit**.

---

# 35. One complete architecture

Here's a pattern worth remembering:

```jsx
function ProfilePage() {
  const [state, submitAction, isPending] =
    useActionState(
      updateProfile,
      {
        success: false,
        error: null
      }
    );

  return (
    <form action={submitAction}>
      <ProfileFields />

      <SubmitButton />

      {state.error && (
        <ErrorMessage>
          {state.error}
        </ErrorMessage>
      )}
    </form>
  );
}
```

Then:

```jsx
function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button disabled={pending}>
      {pending ? "Saving..." : "Save"}
    </button>
  );
}
```

And the Action:

```js
async function updateProfile(
  previousState,
  formData
) {
  const name = formData.get("name");

  const result = await api.updateProfile({
    name
  });

  if (!result.ok) {
    return {
      success: false,
      error: result.error
    };
  }

  return {
    success: true,
    error: null
  };
}
```

Each piece has a clear responsibility.

```text
ProfilePage
    │
    ├── useActionState
    │      ├── result state
    │      ├── dispatcher
    │      └── pending
    │
    ├── form action
    │
    ├── ProfileFields
    │
    └── SubmitButton
           │
           └── useFormStatus
```

This is a much more composable design.

---

# 36. The most important conceptual distinction

You should now distinguish four kinds of state we've encountered:

|State|Question|
|---|---|
|`useState`|What interactive state does this component own?|
|`useReducer`|What are the pure state transitions?|
|`useActionState`|What state results from an asynchronous Action?|
|`useOptimistic`|What temporary result should the UI show while the Action runs?|

And then:

```text
useFormStatus
```

isn't really another place to store state.

It's more like:

```text
"Tell me what's happening with the parent form."
```

---

# 37. One mental model to keep forever

For a modern React form:

```text
                 USER
                   │
                   ▼
                <form>
                   │
                   ▼
                ACTION
                   │
          ┌────────┼────────┐
          │        │        │
          ▼        ▼        ▼
       pending  optimistic  async work
          │        │        │
          │        │        ▼
          │        │      server
          │        │        │
          │        │        ▼
          │        │      result
          │        │        │
          └────────┴────────┘
                   │
                   ▼
               final state
```

And components inside the form can observe:

```text
useFormStatus()
       ↓
pending / data / method / action
```

while the Action itself can maintain its result through:

```text
useActionState()
```

and temporary UI through:

```text
useOptimistic()
```

---

# 38. When should you actually use these?

Don't mechanically replace every form with Actions.

Use them when the UI interaction has a natural asynchronous lifecycle:

```text
submit → wait → result
```

Examples:

```text
profile update
comment submission
like/unlike
checkout
settings save
create record
delete record
send message
```

For a tiny purely local interaction:

```jsx
const [open, setOpen] = useState(false);
```

you don't need an Action.

The simplest state model that accurately represents the problem is still the best one.

---

# 39. Final mental map

At this point, the React state architecture we've built looks roughly like this:

```text
                    STATE

        ┌────────────┼──────────────┐
        │            │              │
        ▼            ▼              ▼
     Local        Shared         External
        │            │              │
 useState/Reducer  Context    useSyncExternalStore
        │
        ▼
 Async UI operations
        │
        ├── useActionState
        ├── useFormStatus
        └── useOptimistic

Server-owned data
        │
        └── TanStack Query
```

The big idea is that **state ownership and state behavior are different questions**.

For example:

```text
Server owns the actual comment.
React owns the temporary optimistic comment.
The Action coordinates the submission.
TanStack Query may own the cached server representation.
```

Once you see those as separate responsibilities, complex React applications become much easier to reason about.

---

## One final comparison

```jsx
// Normal local state
const [name, setName] = useState("");

// Pure state machine
const [state, dispatch] = useReducer(reducer, initialState);

// Async Action result
const [state, action, pending] =
  useActionState(submit, initialState);

// Temporary async UI
const [optimistic, setOptimistic] =
  useOptimistic(realValue);

// Parent form status
const { pending, data } =
  useFormStatus();
```

Don't memorize the syntax yet.

Memorize the question each API answers:

```text
useState
"What is my local state?"

useReducer
"How do my state transitions work?"

useActionState
"What is the result of my async Action?"

useOptimistic
"What should the UI temporarily believe?"

useFormStatus
"What is my parent form currently doing?"
```

That is the React 19 Actions model.