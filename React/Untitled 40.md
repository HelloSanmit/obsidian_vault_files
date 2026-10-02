
# Lesson 37 — Server Functions: Calling Server Code From Client React

In the last lesson, we established the boundary:

```text
SERVER
  │
  │  server/client boundary
  ▼
CLIENT
```

You learned that a Client Component can receive data from the server, but ordinary JavaScript functions cannot simply cross that boundary.

Now we solve the reverse problem:

> **How does a Client Component ask the server to execute a function?**

That is what **Server Functions** are for.

React 19 made Server Functions stable as a React feature, although the low-level framework/bundler machinery that implements them is framework-specific. React also distinguishes the general term **Server Function** from **Server Action**: a Server Function becomes a Server Action when it is used in an Action context such as a form `action` or another Action. ([React](https://react.dev/reference/rsc/server-functions?utm_source=chatgpt.com "Server Functions – React"))

---

# 1. The problem we're solving

Suppose you have this Client Component:

```jsx
"use client";

function AddToCartButton({ productId }) {
  async function handleClick() {
    await addToCart(productId);
  }

  return (
    <button onClick={handleClick}>
      Add to cart
    </button>
  );
}
```

Where does `addToCart()` live?

Suppose the actual operation must do:

```js
await db.cartItems.insert(...)
```

You absolutely do **not** want database access running in the browser.

So conceptually you need:

```text
Browser
  │
  │ "add product 42 to my cart"
  ▼
Server
  │
  ├── authenticate user
  ├── authorize operation
  ├── update database
  └── return result
```

Server Functions provide the React-level mechanism for making that call.

---

# 2. The simplest Server Function

A Server Function can be marked using:

```js
"use server";
```

For example:

```js
async function addToCart(productId) {
  "use server";

  await db.cart.insert({
    productId
  });

  return {
    success: true
  };
}
```

The `"use server"` directive tells the React Server Components system that this function is a **server-callable function**.

When client code invokes it, React/framework machinery turns the invocation into a network request to the server, serializes the arguments, runs the function on the server, and sends a serialized result back. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

That means this:

```js
await addToCart(42);
```

looks like an ordinary function call.

But architecturally it is much closer to:

```text
client function reference
        ↓
network request
        ↓
server endpoint
        ↓
server function
        ↓
database
        ↓
serialized response
        ↓
client promise resolves
```

That is the mental model you need.

---

# 3. It is not actually a local function call

This is perhaps the most important point in the lesson.

Ordinary JavaScript:

```js
const result = addToCart(42);
```

means roughly:

```text
same runtime
    ↓
call function
    ↓
return value
```

A Server Function means:

```text
browser runtime
    ↓
network boundary
    ↓
server runtime
    ↓
execute function
    ↓
network boundary
    ↓
browser runtime
```

So:

```js
await addToCart(42);
```

looks local.

But it behaves like an RPC-style call.

That means all the normal distributed-system concerns suddenly matter:

```text
latency
failure
timeouts
duplicate requests
authentication
authorization
input validation
serialization
retry behavior
```

React doesn't make those concerns disappear.

---

# 4. Why must the function be `async`?

A Server Function crosses a network boundary.

So even if the server operation is tiny, the browser cannot treat it as a synchronous local call.

That's why Server Functions marked with `"use server"` must be asynchronous. React's current model treats the call as inherently asynchronous. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

So this is valid:

```js
async function deletePost(id) {
  "use server";

  await db.posts.delete(id);
}
```

Conceptually invalid as a Server Function:

```js
function deletePost(id) {
  "use server";

  // synchronous
}
```

The important reasoning is:

```text
client
  ↓
network
  ↓
server
```

cannot give you ordinary local synchronous semantics.

---

# 5. What actually crosses the boundary?

Suppose the client calls:

```js
await addToCart(42);
```

The number:

```text
42
```

is serialized into the request.

The server receives a representation of that value.

Likewise:

```js
await addToCart({
  productId: 42,
  quantity: 2
});
```

passes a serializable object.

React supports a defined set of serializable values across this boundary, including primitives, arrays, plain objects, `Date`, `Map`, `Set`, `FormData`, Promises, and Server Function references. Ordinary arbitrary functions, React elements, class instances, and event objects are not generally supported. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

So think:

```text
SERVER
      serialization boundary
CLIENT
```

not:

```text
shared memory
```

---

# 6. Why can't we send ordinary functions?

Suppose:

```jsx
function Parent() {
  const save = () => {
    console.log("hello");
  };

  return <ClientChild save={save} />;
}
```

If `ClientChild` lives across the server/client boundary, `save` is an ordinary JavaScript function.

How would React serialize that?

You can't simply turn:

```js
() => console.log("hello")
```

into a value that magically exists in another runtime with the same closure environment.

For example, what about:

```js
const secret = "abc";

function save() {
  console.log(secret);
}
```

The function depends on:

```text
closure
   ↓
secret
```

That cannot be represented as simple JSON-like data.

A **Server Function**, however, is a special callable reference understood by the React framework/bundler machinery.

So this:

```text
ordinary function
→ code/value

Server Function
→ callable server reference
```

is a very different thing.

---

# 7. `use server` at function level

You can mark an individual function:

```js
export async function deletePost(id) {
  "use server";

  await db.posts.delete(id);
}
```

Only that function is explicitly marked.

This is convenient when you have a file containing a mixture of server-only implementation details and functions you want exposed as callable Server Functions.

---

# 8. `use server` at module level

You can also put:

```js
"use server";
```

at the top of a file.

For example:

```js
"use server";

export async function addToCart(productId) {
  ...
}

export async function removeFromCart(productId) {
  ...
}

export async function clearCart() {
  ...
}
```

The exported functions become Server Functions that can be imported from client code. React's current `"use server"` model supports this module-level form. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

This often gives you a clean structure:

```text
actions/
  cart.js
  profile.js
  comments.js
```

although the exact naming convention is an application/framework choice.

---

# 9. Client Component calling a Server Function

Imagine:

```js
// cart-actions.js

"use server";

export async function addToCart(productId) {
  await db.cart.insert({
    productId
  });

  return {
    success: true
  };
}
```

Then:

```jsx
"use client";

import { addToCart } from "./cart-actions";

function AddToCart({ productId }) {
  async function handleClick() {
    const result = await addToCart(productId);

    console.log(result);
  }

  return (
    <button onClick={handleClick}>
      Add to cart
    </button>
  );
}
```

This is the illusion:

```text
Client:
addToCart(productId)
```

But underneath:

```text
Client
  │
  │ serialized productId
  ▼
Server Function endpoint
  │
  ▼
addToCart(productId)
  │
  ▼
database
```

Then:

```text
database result
  ↓
Server Function return value
  ↓
serialization
  ↓
client promise resolves
```

React documents this client-to-server network behavior explicitly. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

---

# 10. Full execution trace

Let's say:

```jsx
await addToCart(42);
```

### Client starts

```text
productId = 42
```

The browser invokes the Server Function reference.

---

### React/framework prepares the request

Conceptually:

```text
{
  function: "some-server-reference",
  args: [42]
}
```

The exact wire format is an implementation detail. Don't build your application around an imagined HTTP payload shape.

The conceptual fact is:

```text
argument
→ serialization
→ network
```

---

### Server receives request

The server resolves the function reference:

```js
addToCart
```

and executes:

```js
await db.cart.insert({
  productId: 42
});
```

---

### Server returns

Suppose:

```js
return {
  success: true,
  cartCount: 4
};
```

---

### Browser receives result

Your code continues:

```js
const result = await addToCart(42);

console.log(result);
```

and gets:

```js
{
  success: true,
  cartCount: 4
}
```

So the lifecycle is:

```text
CALL
 ↓
serialize
 ↓
network
 ↓
execute
 ↓
return
 ↓
serialize
 ↓
network
 ↓
resolve Promise
```

---

# 11. This is why `await` matters

Because:

```js
addToCart(42)
```

doesn't immediately give you:

```js
{ success: true }
```

It gives you something promise-like representing asynchronous remote work.

So:

```js
const result = await addToCart(42);
```

means:

```text
send request
   ↓
wait
   ↓
receive response
   ↓
continue
```

Exactly like other asynchronous APIs you've already used.

---

# 12. Server Functions should usually perform mutations

This is an important architectural recommendation from React's current model.

Server Functions are primarily designed for **server-side mutations**:

```text
create
update
delete
submit
send
save
purchase
```

They are not intended to be your general-purpose data-fetching cache layer. React notes that frameworks implementing Server Functions generally process them as mutation-style operations and don't provide ordinary caching semantics for their returned values. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

So think:

```text
GET-like data retrieval
→ Server Component / query layer / loader / TanStack Query

Mutation
→ Server Function / Action
```

Not as an absolute law, but as a very useful architecture.

---

# 13. Why not use Server Functions for every GET?

Suppose:

```js
async function getProducts() {
  "use server";

  return db.products.getAll();
}
```

Then client:

```js
const products = await getProducts();
```

Technically, this is a server call.

But now every invocation is an asynchronous remote operation.

You don't automatically get:

```text
cache
staleTime
refetch
background refresh
deduplication
pagination management
```

Those are concerns handled by dedicated server-state systems or framework data loading.

So for a product list:

```text
Server Component
   ↓
fetch products
   ↓
render
```

may be a better architecture than:

```text
client
   ↓
call Server Function getProducts()
   ↓
render
```

Again, choose based on the interaction and ownership model.

---

# 14. Server Functions + forms

Now we reach the really useful part.

Suppose:

```js
"use server";

export async function updateName(formData) {
  const name = formData.get("name");

  await db.users.update({
    name
  });
}
```

And on the client/server-rendered tree:

```jsx
<form action={updateName}>
  <input name="name" />
  <button type="submit">
    Save
  </button>
</form>
```

React can use the Server Function directly as the form's `action`.

When submitted:

```text
<form>
     ↓
FormData
     ↓
Server Function
     ↓
network
     ↓
server mutation
```

React automatically treats form Server Functions as Actions/transitions. React also supports progressive enhancement here: the form can submit before the client JavaScript bundle has finished loading. ([React](https://react.dev/reference/rsc/server-functions?utm_source=chatgpt.com "Server Functions – React"))

This is a major reason the Actions APIs from Lesson 34 make sense.

---

# 15. We can combine everything we've learned

Let's create a profile form.

### Server Function

```js
"use server";

export async function updateProfile(formData) {
  const name = formData.get("name");

  if (!name?.trim()) {
    return {
      success: false,
      error: "Name is required"
    };
  }

  await db.users.update({
    name
  });

  return {
    success: true,
    name
  };
}
```

### Client

```jsx
"use client";

import {
  useActionState
} from "react";

import {
  updateProfile
} from "./actions";

function ProfileForm() {
  const [
    state,
    action,
    isPending
  ] = useActionState(
    updateProfile,
    {
      success: false,
      error: null
    }
  );

  return (
    <form action={action}>
      <input
        name="name"
        disabled={isPending}
      />

      <button disabled={isPending}>
        {isPending
          ? "Saving..."
          : "Save"}
      </button>

      {state.error && (
        <p>{state.error}</p>
      )}

      {state.success && (
        <p>
          Saved {state.name}
        </p>
      )}
    </form>
  );
}
```

Now the architecture is:

```text
User
 ↓
<form>
 ↓
useActionState
 ↓
Server Function
 ↓
HTTP/network
 ↓
database
 ↓
return result
 ↓
useActionState state
 ↓
UI
```

This is the complete Actions loop.

---

# 16. Why `useActionState` is so useful here

Before:

```js
const response =
  await updateProfile(formData);
```

you might manually manage:

```text
loading
success
error
result
```

With:

```js
const [
  state,
  action,
  isPending
] = useActionState(
  updateProfile,
  initialState
);
```

React gives you a structure around:

```text
Action
 ↓
pending
 ↓
result
```

So the division becomes:

```text
Server Function
→ perform server mutation

useActionState
→ track Action result/lifecycle

useFormStatus
→ let nested form components observe pending status
```

That separation is extremely useful.

---

# 17. FormData is a surprisingly important bridge

Recall normal browser HTML forms:

```html
<form>
  <input name="email">
  <input name="password">
</form>
```

The browser already knows how to package fields as form data.

React's Server Function integration takes advantage of that.

For:

```jsx
<form action={login}>
```

the Server Function can receive:

```js
async function login(formData) {
  "use server";

  const email =
    formData.get("email");

  const password =
    formData.get("password");
}
```

You don't necessarily have to write:

```js
const object = {
  email,
  password
};
```

on the client just to transport the fields.

The HTML form is itself the serialization structure.

This is one of the ways React 19's Action model works nicely with the existing web platform.

---

# 18. Progressive enhancement

This is a concept worth understanding deeply.

Imagine JavaScript hasn't loaded yet.

Traditional SPA form:

```text
page loaded
JavaScript not loaded
      ↓
React submit handler doesn't exist
      ↓
interaction may not work
```

A Server Function passed to a form Action has a different path.

```text
HTML form
   ↓
browser can submit
   ↓
server receives form submission
   ↓
server function executes
```

React's Server Function/form integration is designed to preserve this browser-native submission capability and progressively enhance it when JavaScript becomes available. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

That is one of the places where modern React is intentionally leaning into platform behavior rather than replacing it.

---

# 19. Why native HTML suddenly matters a lot

You've learned controlled forms:

```jsx
<input
  value={email}
  onChange={...}
/>
```

Those are still useful.

But Server Function forms remind you:

```text
HTML already has
<form>
<input name>
<button type="submit">
FormData
```

React's newer APIs don't erase those primitives.

They integrate with them.

This leads to a strong principle:

> Don't forget the browser platform just because you're using React.

Good React often means using the web platform where it already solves the problem well.

---

# 20. Server Function outside a form

Forms aren't required.

You can call a Server Function from a button:

```jsx
"use client";

import {
  startTransition,
  useState
} from "react";

import {
  incrementLike
} from "./actions";

function LikeButton() {
  const [
    isPending,
    setPending
  ] = useState(false);

  function handleClick() {
    startTransition(async () => {
      setPending(true);

      try {
        await incrementLike();
      } finally {
        setPending(false);
      }
    });
  }

  return (
    <button
      onClick={handleClick}
      disabled={isPending}
    >
      {isPending ? "Liking..." : "Like"}
    </button>
  );
}
```

The important rule is:

> When manually calling a Server Function outside a form Action, call it within a Transition.

React's current Server Function guidance explicitly recommends this because it lets React track the asynchronous work as an Action and supports pending/optimistic/error behavior. Forms using Server Functions as `action` or `formAction` get the transition behavior automatically. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

---

# 21. Better version with `useTransition`

Instead of manually storing pending state:

```jsx
const [
  isPending,
  startTransition
] = useTransition();
```

then:

```jsx
function handleClick() {
  startTransition(async () => {
    await incrementLike();
  });
}
```

and:

```jsx
<button disabled={isPending}>
  {isPending ? "Liking..." : "Like"}
</button>
```

Now React itself tracks the Transition.

This connects directly to Lesson 28.

You can visualize it:

```text
button click
   ↓
startTransition
   ↓
Server Function
   ↓
await network
   ↓
pending = true
   ↓
response
   ↓
pending = false
```

---

# 22. Server Function + optimistic UI

Now we can combine Lesson 28 and Lesson 34.

Suppose:

```js
"use server";

export async function likePost(postId) {
  await db.likes.create({
    postId
  });

  return {
    success: true
  };
}
```

Client:

```jsx
"use client";

import {
  useOptimistic,
  startTransition
} from "react";

function LikeButton({ liked, postId }) {
  const [
    optimisticLiked,
    setOptimisticLiked
  ] = useOptimistic(liked);

  function handleClick() {
    startTransition(async () => {
      setOptimisticLiked(true);

      await likePost(postId);
    });
  }

  return (
    <button onClick={handleClick}>
      {optimisticLiked
        ? "❤️ Liked"
        : "🤍 Like"}
    </button>
  );
}
```

Now:

```text
user clicks
   ↓
optimistic state immediately changes
   ↓
server function starts
   ↓
database mutation
   ↓
server response
   ↓
optimistic state settles
```

The user gets instant feedback without pretending that the optimistic state is authoritative.

---

# 23. This is where failures become important

Suppose:

```js
await likePost(postId);
```

throws.

What should happen?

The UI may temporarily have:

```text
❤️ Liked
```

but the server says:

```text
mutation failed
```

The optimistic state can disappear, revealing the underlying real state again.

And you can separately show an error.

This gives you the architecture:

```text
optimistic UI
      ↓
server mutation
      ↓
   ┌──┴──┐
success  failure
  ↓         ↓
commit    rollback
```

Notice how "optimistic state" and "server truth" remain conceptually separate.

---

# 24. Security: the most important section of this lesson

Here is the dangerous misconception:

> "Because the function lives on the server, its arguments are trustworthy."

Absolutely not.

React's current Server Function guidance says to treat arguments as **untrusted input** and authorize every mutation. ([React](https://react.dev/reference/rsc/use-server?utm_source=chatgpt.com "'use server' directive – React"))

Suppose:

```js
"use server";

export async function deletePost(postId) {
  await db.posts.delete(postId);
}
```

This is not enough.

Why?

Because the client can potentially send:

```text
postId = 123
postId = 456
postId = 999999
```

The fact that the UI shows:

```text
Delete this post
```

doesn't prove the caller is allowed to delete it.

---

# 25. Authorization must happen on the server

A safer architecture is:

```js
"use server";

export async function deletePost(postId) {
  const user = await getCurrentUser();

  if (!user) {
    throw new Error("Not authenticated");
  }

  const post = await db.posts.findById(postId);

  if (!post) {
    throw new Error("Post not found");
  }

  if (post.authorId !== user.id) {
    throw new Error("Not authorized");
  }

  await db.posts.delete(postId);
}
```

Now the server checks:

```text
Who are you?
       ↓
Are you allowed?
       ↓
Does this resource exist?
       ↓
Perform mutation
```

This is the correct security boundary.

---

# 26. Never rely on the client for authorization

Bad:

```jsx
if (user.isAdmin) {
  showDeleteButton();
}
```

This can be useful for **UI visibility**.

It is not security.

A malicious client can simply invoke the Server Function anyway.

The server must enforce:

```text
user.isAdmin === true
```

again.

The client determines what the user sees.

The server determines what the user is allowed to do.

That's a distinction worth permanently embedding in your thinking.

---

# 27. Validation and authorization are different

Suppose:

```js
postId = "hello"
```

You need validation.

```text
Is this a valid post ID?
```

Then:

```text
Does that post exist?
```

Then:

```text
Does this user have permission to modify it?
```

These are different questions.

So:

```text
Input validation
      ↓
Resource validation
      ↓
Authorization
      ↓
Mutation
```

Don't collapse them into one vague concept called "security."

---

# 28. Authentication vs authorization

You've encountered this distinction in the HTTP lesson; Server Functions make it concrete.

### Authentication

```text
"Who is this?"
```

Example:

```js
const user =
  await getCurrentUser();
```

### Authorization

```text
"Can this user do this?"
```

Example:

```js
if (post.authorId !== user.id) {
  throw new Error("Forbidden");
}
```

A logged-in user is not automatically authorized to modify everything.

---

# 29. Server Functions are exposed capabilities

This is a useful security mental model.

Imagine:

```js
export async function deleteUser(userId) {
  "use server";

  ...
}
```

Don't think:

```text
"It's just an internal helper."
```

Think:

```text
"This is a remotely callable capability."
```

That mindset encourages you to ask:

```text
What inputs are allowed?
Who may call it?
What does it mutate?
What information does it return?
Can it be abused?
```

That's much safer.

---

# 30. Don't return secrets

Suppose:

```js
"use server";

export async function getUser() {
  return db.users.find(...);
}
```

and the user record contains:

```js
{
  id,
  name,
  email,
  passwordHash,
  internalFlags
}
```

You shouldn't casually return the entire object.

Return only what's needed:

```js
return {
  id: user.id,
  name: user.name,
  email: user.email
};
```

The server/client boundary should force you to think about **data exposure**, not just code execution.

---

# 31. Server Function errors

Suppose the server throws:

```js
throw new Error("Database unavailable");
```

The client receives an asynchronous failure rather than magically sharing the server exception object as if it happened in the same JavaScript stack.

So:

```js
try {
  await saveProfile(formData);
} catch (error) {
  ...
}
```

is about the remote operation failing.

Separately, when you're using Actions, an unexpected thrown error can participate in React's Action/Error Boundary mechanisms. ([React](https://react.dev/reference/rsc/server-functions?utm_source=chatgpt.com "Server Functions – React"))

This connects back to Lesson 26:

```text
expected domain error
→ return structured result

unexpected application/runtime error
→ throw
→ error handling boundary
```

---

# 32. Don't confuse return values with cache updates

Suppose:

```js
const result = await updateProfile(...);
```

returns:

```js
{
  name: "Alex"
}
```

That doesn't automatically mean every copy of the user's profile throughout your app has magically updated.

You may have:

```text
header user
profile page
settings page
sidebar user
query cache
server-rendered page
```

The old server-consistency problem still exists.

The Server Function performs the mutation.

You still need your application's server-state architecture to converge on the new reality.

This is the same consistency problem we studied with TanStack Query.

---

# 33. Server Function + TanStack Query

Suppose:

```text
GET /posts
GET /posts/42
```

are represented in TanStack Query.

You call:

```js
await updatePost(postId, patch);
```

through a Server Function.

The database changes.

But your existing client cache may still contain:

```text
old post
```

So after the Server Function resolves, you may need:

```js
queryClient.invalidateQueries(...)
```

or:

```js
queryClient.setQueryData(...)
```

depending on your consistency strategy.

Therefore:

```text
Server Function
→ performs mutation

TanStack Query
→ manages cached server state
```

These are complementary.

---

# 34. A complete modern architecture

Now we can combine nearly everything from this course.

Imagine:

```text
                         USER
                           │
                           ▼
                     Client Component
                           │
              ┌────────────┴────────────┐
              │                         │
           local UI                Action/form
              │                         │
        useState/etc.            useActionState
                                        │
                                        ▼
                              Server Function
                                        │
                             network boundary
                                        │
                                        ▼
                                     SERVER
                                        │
                               authenticate
                                        │
                                authorize
                                        │
                                 validate
                                        │
                                        ▼
                                   database
                                        │
                                        ▼
                                  return result
                                        │
                                        ▼
                                  React Action
                                        │
                       ┌────────────────┴───────────────┐
                       │                                │
                  optimistic UI                  final state
                       │                                │
                       └────────────────┬───────────────┘
                                        ▼
                                       UI
```

This is a real architectural pattern, not just a collection of APIs.

---

# 35. The form version is even cleaner

```text
<form action={serverFunction}>
        │
        ▼
     FormData
        │
        ▼
 useActionState
        │
        ▼
 Server Function
        │
        ▼
   authorization
        │
        ▼
     database
        │
        ▼
 structured result
        │
        ▼
 useActionState state
        │
        ▼
      render
```

And nested children can observe:

```text
useFormStatus
```

while temporary UI can use:

```text
useOptimistic
```

So the three React 19 APIs from the last lesson now have an actual purpose:

```text
Server Function
       ↓
   performs work

useActionState
       ↓
 tracks result

useFormStatus
       ↓
 tracks parent form

useOptimistic
       ↓
 temporary UI
```

---

# 36. `"use client"` vs `"use server"`

This is one of the most important comparisons in modern React.

|Directive|Meaning|
|---|---|
|`"use client"`|This module is part of the client-side module graph|
|`"use server"`|This function/module exposes Server Functions callable from client-side code|

Notice the directions:

```text
"use client"
    server
      │
      ▼
    client
```

versus:

```text
"use server"
    client
      │
      ▼
    server
```

But don't interpret these as simply:

```text
"run everything here on X"
```

They define **boundaries in the application architecture**.

---

# 37. One subtle point about `"use server"`

A module-level:

```js
"use server";

export async function save() {}
```

does not mean:

```text
every arbitrary piece of code in this file is somehow callable from the browser.
```

The exported functions become the callable Server Functions.

The server-only implementation details remain server-side.

So you can have:

```js
"use server";

async function validateInput(data) {
  ...
}

async function writeDatabase(data) {
  ...
}

export async function saveProfile(data) {
  ...
}
```

Only the intended exported function is exposed as the Server Function boundary.

Still, treat the exported entry points as externally reachable capabilities.

---

# 38. Another subtle point: don't trust hidden UI state

Imagine:

```jsx
<button disabled={isAdmin !== true}>
  Delete
</button>
```

A malicious caller doesn't need your button.

They can invoke the remote endpoint directly.

Therefore:

```text
UI restriction
     ≠
authorization
```

The server must independently check.

This principle applies equally to:

```text
hidden fields
disabled buttons
route guards
client state
Context values
localStorage
```

None are security boundaries.

The server is the security boundary for server-side mutations.

---

# 39. A realistic order for a mutation

Suppose:

```js
updateComment(commentId, text)
```

A good conceptual sequence is:

```text
1. Receive request
       ↓
2. Authenticate
       ↓
3. Validate input
       ↓
4. Load resource
       ↓
5. Authorize
       ↓
6. Apply mutation
       ↓
7. Return minimal result
```

For example:

```js
"use server";

export async function updateComment(
  commentId,
  text
) {
  const user = await getCurrentUser();

  if (!user) {
    throw new Error("Unauthenticated");
  }

  if (
    typeof text !== "string" ||
    text.length > 5000
  ) {
    throw new Error("Invalid input");
  }

  const comment =
    await db.comments.findById(commentId);

  if (!comment) {
    throw new Error("Not found");
  }

  if (comment.authorId !== user.id) {
    throw new Error("Forbidden");
  }

  const updated =
    await db.comments.update(
      commentId,
      { text }
    );

  return {
    id: updated.id,
    text: updated.text
  };
}
```

That's a much healthier mental model than:

```text
client says updateComment(123)
→ blindly update row 123
```

---

# 40. Server Functions and APIs

You might wonder:

> "Isn't this just an API endpoint?"

Conceptually, it's related.

Traditional architecture:

```text
Client
  ↓
POST /api/comments
  ↓
route handler
  ↓
database
```

Server Function architecture:

```text
Client
  ↓
Server Function reference
  ↓
framework transport
  ↓
Server Function
  ↓
database
```

The key difference is that React and the framework can integrate the callable server operation with:

```text
forms
Actions
Transitions
serialization
progressive enhancement
```

rather than requiring you to manually define every HTTP interaction in your component.

But underneath, there is still network communication.

---

# 41. Server Functions aren't magic networking

Remember this when debugging.

If:

```js
await updateProfile(...)
```

takes 800 ms:

```text
your React component
```

is not "slow."

You may simply have:

```text
network = 250ms
server work = 400ms
database = 100ms
response = 50ms
```

React can't remove physics.

You still need:

```text
loading states
optimistic UI
timeouts
error handling
retry strategy
```

when appropriate.

---

# 42. This also explains why idempotence matters

Suppose your client accidentally retries:

```js
await createOrder(...)
```

twice.

You might accidentally create:

```text
Order #101
Order #102
```

That's a serious problem.

But:

```js
updateProfile(...)
```

might be safe to repeat with the same final values.

And:

```js
deletePost(...)
```

may often be designed to become effectively repeat-safe.

This is the same idempotence concept from HTTP that you learned earlier, but now it has a practical frontend consequence.

For important mutations, think about:

```text
Can this operation be repeated?
What happens if the browser retries?
What happens if the user double-clicks?
What happens if the request reaches the server twice?
```

---

# 43. Server Functions and duplicate submissions

Imagine:

```text
user double-clicks Buy
```

You might get:

```text
request #1
request #2
```

Even though your UI showed:

```text
Buying...
```

You still need server-side correctness.

Possible solutions include:

```text
idempotency keys
unique database constraints
transactional logic
server-side deduplication
```

React's Action system helps coordinate UI behavior, but it does not magically turn every backend operation into an exactly-once distributed transaction.

That distinction matters enormously in production.

---

# 44. The relationship with normal forms

There is a beautiful progression here.

### Plain HTML

```html
<form method="POST">
```

Browser owns submission.

### Traditional React

```jsx
<form onSubmit={handleSubmit}>
```

React intercepts submission and manages it manually.

### React Actions

```jsx
<form action={action}>
```

React understands the submission as an Action.

### React + Server Function

```jsx
<form action={serverFunction}>
```

The Action can directly execute server-side mutation logic.

So we're moving from:

```text
browser primitive
```

to:

```text
React-managed primitive
```

to:

```text
server-aware React primitive
```

without abandoning the browser's form model.

---

# 45. The complete React 19 mutation stack

At this point, you can understand this architecture almost completely:

```text
                 USER ACTION
                     │
             ┌───────┴────────┐
             │                │
          button             form
             │                │
      startTransition    action={...}
             │                │
             └───────┬────────┘
                     ▼
               Server Function
                     │
                network call
                     │
                     ▼
                   SERVER
                     │
          ┌──────────┼───────────┐
          │          │           │
       auth       validate    authorize
          │          │           │
          └──────────┼───────────┘
                     ▼
                 database
                     │
                     ▼
                 result/error
                     │
                     ▼
                  CLIENT
                     │
          ┌──────────┼──────────┐
          │          │          │
   useActionState  useOptimistic  Query cache
          │          │          │
          └──────────┼──────────┘
                     ▼
                     UI
```

This is the point where React's newer features start looking like a coherent system rather than isolated APIs.

---

# 46. One final conceptual distinction

We now have three very different things:

### Server Component

```text
"Run this component on the server."
```

### Server Function

```text
"Allow the client to ask the server to run this function."
```

### Server-rendered HTML

```text
"Send HTML produced by React to the browser."
```

These are three different operations.

```text
Server Component
      ↓
server execution

Server Function
      ↑
client invocation

SSR
      ↓
HTML generation
```

That distinction will save you a lot of confusion later.

---

# 47. Your mental model after Lessons 35–37

You started with:

```text
React
  ↓
browser
```

Now you should be thinking:

```text
                         APPLICATION
                              │
                  ┌───────────┴───────────┐
                  │                       │
                SERVER                  CLIENT
                  │                       │
          Server Components       Client Components
                  │                       │
            DB / filesystem        state / events
            server APIs             effects / DOM
                  │                       │
                  └───────────┬───────────┘
                              │
                       boundary protocol
                              │
                   ┌──────────┴──────────┐
                   │                     │
              server → client       client → server
                   │                     │
              data / UI            Server Functions
                   │                     │
                   ▼                     ▼
                 HTML                 mutation
                   │
                   ▼
              hydration
```

Then layer Actions on top:

```text
Server Function
      ↓
Action
      ├── pending
      ├── result
      └── optimistic UI
```

---

# 48. The five questions to ask in a real application

When you encounter a new piece of logic, ask:

```text
1. Where should this code execute?
```

Server or client?

```text
2. Who owns the data?
```

React, server, external system, cache?

```text
3. Does the client need to trigger server work?
```

If yes, a Server Function may be appropriate.

```text
4. What is the user supposed to see while it runs?
```

Pending UI? Optimistic UI?

```text
5. What security checks must happen on the server?
```

Authentication, validation, authorization.

Those five questions cover a surprising amount of real-world React architecture.

---

# 49. One warning about current React Server Components

Because this architecture introduces a real network-execution boundary, Server Components/Server Functions implementations are security-sensitive infrastructure. React disclosed and patched serious vulnerabilities in the React Server Components ecosystem during 2025–2026, so in production you should keep the React/framework/bundler stack on currently supported patched releases rather than treating the RSC transport as ordinary frontend code. ([React](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components?utm_source=chatgpt.com "Critical Security Vulnerability in React Server Components – React"))

The practical lesson isn't "RSC is unsafe."

It's:

```text
server boundary
+
remote invocation
+
deserialization
=
security-sensitive infrastructure
```

which should always be maintained deliberately.

---

# Final mental model

Remember this diagram:

```text
CLIENT
│
│  "Please update this"
│
▼
Server Function
│
│ network
▼
SERVER
│
├── authenticate
├── validate
├── authorize
├── mutate database
└── return result
│
│ network
▼
CLIENT
│
├── Action state
├── optimistic state
├── query/cache update
└── render
```

And the vocabulary:

```text
Server Component
→ server-side component execution

Server Function
→ client-callable server-side function

Server Action
→ Server Function used as an Action

"use client"
→ client module boundary

"use server"
→ Server Function boundary

useActionState
→ Action result + pending state

useFormStatus
→ status of parent form

useOptimistic
→ temporary UI during Action
```

The next natural topic is **React Router's modern data APIs**: loaders, actions, revalidation, route-level data ownership, pending navigation, and how they compare with `useEffect`, TanStack Query, and React Server Functions. That will connect your routing knowledge with everything we've just learned.