
# Lesson 22 — HTTP, REST APIs, and GET / POST / PUT / PATCH / DELETE

We've been writing things like:

```js
fetch("/api/todos")
```

and:

```js
fetch("/api/todos", {
  method: "POST",
  ...
})
```

But until now, I've mostly treated the server as a black box.

Let's open that box.

The goal of this lesson is not to turn you into a backend developer. It's to make you comfortable enough with HTTP that, when you see a React application communicating with an API, you understand exactly what is happening.

---

# 1. First understand what an API actually is

Suppose your React application displays:

```text
Todos
────────────────
☐ Learn React
☐ Learn HTTP
☐ Build app
```

Your browser cannot magically know those todos.

There is usually some server somewhere that owns the data.

Conceptually:

```text
React application
       │
       │ HTTP request
       ↓
     Server
       │
       ↓
   Database
```

The server might be implemented in:

```text
Node.js
Python
Java
Go
C#
Ruby
PHP
...
```

React doesn't particularly care.

React's job is:

```text
receive data
display it
collect user actions
send requests
```

The backend's job may include:

```text
authenticate user
validate input
apply business rules
read/write database
return response
```

The communication layer between them is commonly HTTP.

---

# 2. What is HTTP?

HTTP is a protocol for communication between clients and servers.

In our situation:

```text
client = browser
server = backend API
```

A request basically says:

> "I want something from this server"  
> or  
> "I want the server to perform some operation."

The server sends a response.

So:

```text
REQUEST
   ↓
SERVER
   ↓
RESPONSE
```

That sounds simple, but a request and response contain several pieces of information.

---

# 3. An HTTP request has more than a URL

Consider:

```http
POST /api/todos
Content-Type: application/json

{
  "title": "Learn React"
}
```

There are several concepts here.

### Method

```text
POST
```

### URL/path

```text
/api/todos
```

### Headers

```text
Content-Type: application/json
```

### Body

```json
{
  "title": "Learn React"
}
```

So you can roughly think:

```text
REQUEST
├── method
├── URL
├── headers
└── body
```

Not every request has a body.

A GET request often doesn't.

---

# 4. HTTP methods describe the intent

The common methods you'll encounter in React applications are:

```text
GET
POST
PUT
PATCH
DELETE
```

A useful first approximation is:

```text
GET      → read
POST     → create / submit an operation
PUT      → replace
PATCH    → partially modify
DELETE   → delete
```

But let's go much deeper than that.

---

# 5. GET

Suppose we want all todos.

```http
GET /api/todos
```

We're essentially saying:

> "Give me the representation of the todos."

In React:

```js
const response = await fetch("/api/todos");
```

A server might respond:

```json
[
  {
    "id": 1,
    "title": "Learn React",
    "completed": false
  },
  {
    "id": 2,
    "title": "Learn HTTP",
    "completed": false
  }
]
```

Then:

```js
const todos = await response.json();
```

Now React has JavaScript data.

The complete flow is:

```text
useQuery
   ↓
fetch("/api/todos")
   ↓
GET /api/todos
   ↓
server
   ↓
JSON response
   ↓
JavaScript object/array
   ↓
React UI
```

---

# 6. GET with a specific resource

A common API structure is:

```text
GET /api/todos/42
```

Meaning:

> "Give me todo 42."

Now compare:

```text
GET /api/todos
```

with:

```text
GET /api/todos/42
```

The first usually represents a collection.

The second usually represents one resource.

Conceptually:

```text
/api/todos
      ↓
collection

/api/todos/42
      ↓
individual todo
```

That's why our React query keys looked like:

```js
["todos"]
```

and:

```js
["todo", 42]
```

There is a nice correspondence:

```text
API resource identity
        ↕
query identity
```

They're not required to be identical strings, but they often reflect the same conceptual structure.

---

# 7. Query parameters

You've already learned URL search parameters.

For example:

```text
/api/products?category=phones&page=2
```

This is still a GET request.

The path is:

```text
/api/products
```

The query string is:

```text
category=phones&page=2
```

So:

```text
GET /api/products?category=phones&page=2
```

might mean:

> "Give me page 2 of products in the phones category."

This is exactly why we previously built query keys like:

```js
[
  "products",
  {
    category,
    page,
  }
]
```

The query key represents the identity of the server data we're requesting.

---

# 8. POST

Now suppose the user fills out:

```text
Title:
Learn React

[Create]
```

We want to create a new todo.

That is commonly:

```http
POST /api/todos
```

with a body:

```json
{
  "title": "Learn React"
}
```

In JavaScript:

```js
const response = await fetch("/api/todos", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    title: "Learn React",
  }),
});
```

There are three important things here.

### `method`

```js
method: "POST"
```

### `headers`

We tell the server:

```text
"The body I'm sending is JSON."
```

### `body`

We send serialized JSON:

```js
JSON.stringify({
  title: "Learn React",
})
```

---

# 9. Why `JSON.stringify()`?

This is an important distinction.

In JavaScript:

```js
{
  title: "Learn React"
}
```

is a JavaScript object.

HTTP request bodies are ultimately bytes/text on the wire.

JSON is one format we commonly use to represent structured data.

So:

```js
JSON.stringify({
  title: "Learn React"
})
```

produces:

```json
{"title":"Learn React"}
```

Then the server can parse that JSON back into its own representation.

Conceptually:

```text
JavaScript object
      ↓
JSON.stringify
      ↓
JSON text
      ↓
HTTP request
      ↓
server
      ↓
parse JSON
      ↓
server-side object/data
```

And the reverse happens in responses:

```text
server data
      ↓
JSON response
      ↓
response.json()
      ↓
JavaScript object
```

---

# 10. `Content-Type`

We used:

```http
Content-Type: application/json
```

This tells the server what format the request body uses.

Think:

```text
Content-Type
      ↓
"What format is the body?"
```

For JSON:

```text
application/json
```

Other APIs can use other formats.

For example, file uploads commonly involve multipart form data.

The broader lesson is:

> HTTP doesn't require JSON.

JSON is just an extremely common data format used by web APIs.

---

# 11. POST does not necessarily mean only "create"

This is where simplistic explanations can become misleading.

You'll often see:

```text
POST /users
```

to create a user.

But POST can also represent an operation that doesn't map neatly to CRUD.

For example:

```text
POST /checkout
POST /login
POST /send-email
POST /password-reset
```

The server may interpret POST as:

> "Process this submitted operation."

So:

```text
POST ≈ submit/process
```

is sometimes a better mental model than:

```text
POST = create
```

Creation is one very common use.

---

# 12. PUT

Now we get to a method that beginners often misunderstand.

Suppose we have:

```http
PUT /api/users/42
```

and send:

```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"
}
```

A traditional conceptual interpretation of PUT is:

> **Replace the representation of resource 42 with the supplied representation.**

So if the existing user is:

```json
{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com",
  "role": "user"
}
```

and we PUT:

```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"
}
```

the server may treat the supplied representation as the new complete representation.

The precise semantics are defined by HTTP rather than by React.

---

# 13. PATCH

PATCH is for partial modification.

Suppose the user only wants to change:

```text
role
```

Instead of sending everything, you might send:

```http
PATCH /api/users/42
```

with:

```json
{
  "role": "admin"
}
```

The conceptual meaning is:

> "Apply this partial modification to resource 42."

So a simple distinction is:

```text
PUT
→ replace the representation

PATCH
→ partially modify it
```

But here's an important real-world caveat:

> **Actual API behavior depends on the API's contract.**

You will encounter APIs that use PUT for partial-looking updates or don't support PATCH at all.

Don't assume the backend follows your preferred design.

Read the API contract.

---

# 14. DELETE

Now:

```http
DELETE /api/todos/42
```

means:

> "Delete todo 42."

In React:

```js
async function deleteTodo(id) {
  const response = await fetch(`/api/todos/${id}`, {
    method: "DELETE",
  });

  if (!response.ok) {
    throw new Error("Failed to delete todo");
  }
}
```

Usually there is no request body here.

Then, after success, we might invalidate:

```js
queryClient.invalidateQueries({
  queryKey: ["todos"],
});
```

So:

```text
DELETE
 ↓
server removes resource
 ↓
client cache may now be stale
 ↓
invalidate
 ↓
list refreshes
```

Now you can see how HTTP and TanStack Query fit together.

---

# 15. CRUD vs HTTP methods

You will often see this table:

|Operation|Typical method|
|---|---|
|Create|POST|
|Read|GET|
|Replace|PUT|
|Partial update|PATCH|
|Delete|DELETE|

That's useful, but don't turn it into a law.

The backend API determines the actual contract.

For example:

```text
POST /api/todos/42/complete
```

could represent a "complete todo" operation.

That's not conventional CRUD.

The key thing is to understand **what the endpoint is asking the server to do**.

---

# 16. The response is also structured

Suppose:

```http
GET /api/todos/42
```

The server might return:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 42,
  "title": "Learn React",
  "completed": false
}
```

There's more than just the JSON body.

Think:

```text
RESPONSE
├── status
├── headers
└── body
```

The status tells us broadly how the operation went.

---

# 17. Status codes

You don't need to memorize every HTTP status code.

You should understand the major families.

```text
1xx → informational

2xx → success

3xx → redirection

4xx → client-side/request problem

5xx → server-side failure
```

The ones you'll encounter constantly include:

```text
200 → successful request
201 → resource created
204 → successful request with no response body
400 → malformed/invalid request
401 → authentication required/failed
403 → server understood the request but refuses it
404 → resource not found
409 → conflict
422 → request understood but validation failed
500 → server error
503 → service unavailable
```

The exact meaning can vary somewhat by API, but these are common conventions.

---

# 18. `response.ok` now makes more sense

We've repeatedly written:

```js
if (!response.ok) {
  throw new Error("Request failed");
}
```

Why?

Because `fetch()` has an important behavior:

> A normal HTTP error response such as 404 or 500 does **not automatically cause the fetch promise to reject**.

For example:

```js
const response = await fetch("/api/todos/999999");
```

Suppose the server returns:

```text
404 Not Found
```

The `fetch()` call can still resolve with a `Response`.

That's why we explicitly inspect:

```js
response.ok
```

or:

```js
response.status
```

So:

```text
network failure
      ↓
fetch may reject

HTTP 404/500
      ↓
fetch can still resolve
      ↓
check response.ok/status
```

This distinction is extremely important.

---

# 19. Network error vs HTTP error

These are different.

### Network failure

For example:

```text
browser can't establish the connection
```

You might get:

```js
catch (error) {
  ...
}
```

### HTTP error

The server successfully responded:

```http
HTTP/1.1 500 Internal Server Error
```

You received a valid HTTP response.

So `fetch()` may resolve.

You need to check:

```js
if (!response.ok) {
  throw new Error(...);
}
```

This distinction becomes especially important in our TanStack Query code because a query only enters its error state if the query function actually throws/rejects.

---

# 20. Why our `fetch` functions throw

Suppose:

```js
async function fetchTodos() {
  const response = await fetch("/api/todos");

  if (!response.ok) {
    throw new Error("Failed to fetch todos");
  }

  return response.json();
}
```

Now:

```text
HTTP 500
   ↓
response.ok = false
   ↓
throw
   ↓
promise rejects
   ↓
useQuery sees an error
   ↓
isError = true
```

Without the `throw`:

```text
HTTP 500
   ↓
fetch promise resolves
   ↓
queryFn may appear successful
```

So our earlier TanStack Query lessons were relying on this HTTP knowledge even though we hadn't stopped to explain it.

---

# 21. Request headers vs response headers

Headers exist on both sides.

### Request

```http
Content-Type: application/json
Authorization: Bearer ...
```

### Response

```http
Content-Type: application/json
Cache-Control: ...
```

Think:

```text
request headers
    =
information about what we're sending/requesting

response headers
    =
information about what the server is returning
```

Headers are metadata accompanying the request or response.

---

# 22. Authentication

Let's say the API requires a token.

You might see:

```js
const response = await fetch("/api/todos", {
  headers: {
    Authorization: `Bearer ${token}`,
  },
});
```

Now the flow is:

```text
React
 ↓
request + authentication credentials/token
 ↓
server
 ↓
authenticate/authorize
 ↓
response
```

You need to distinguish two ideas.

### Authentication

> "Who are you?"

### Authorization

> "Are you allowed to do this?"

So:

```text
401
```

is commonly associated with missing/invalid authentication.

Whereas:

```text
403
```

commonly means the server understood who you are but refuses the action.

Exactly how an API handles these can vary, but the conceptual distinction is important.

---

# 23. Don't confuse authentication with React state

Suppose you have:

```js
const [user, setUser] = useState(null);
```

That doesn't itself authenticate anyone.

It's merely browser-side state.

The actual authentication mechanism might involve:

```text
cookie
session
access token
refresh token
OAuth flow
etc.
```

React may display the resulting user information.

But:

```text
React state
≠
authentication system
```

This is another example of our state taxonomy.

The UI can represent authentication state without being the authority that determines whether a request is authorized.

---

# 24. Cookies and credentials

A browser can also send cookies with requests.

For cross-origin requests, you may encounter:

```js
fetch("https://api.example.com/todos", {
  credentials: "include",
});
```

The exact setup depends on how the backend and browser security policies are configured.

The important mental model is:

```text
cookie/session-based auth
        ↓
browser stores credentials
        ↓
browser may include them in requests
        ↓
server identifies the session
```

You don't need to master authentication architecture yet.

You just need to recognize it when you encounter it.

---

# 25. CORS

Now something that confuses almost every beginner.

Suppose your React dev server is:

```text
http://localhost:5173
```

and your API is:

```text
http://localhost:3000
```

These are different origins.

The browser has security rules around cross-origin requests.

CORS stands for:

> Cross-Origin Resource Sharing.

It is a mechanism through which a server tells the browser which cross-origin requests it permits.

This is why you might write perfectly valid JavaScript:

```js
fetch("http://localhost:3000/api/todos");
```

yet the browser blocks access because the server isn't configured to allow that cross-origin interaction.

So CORS is not:

```text
"React can't call APIs."
```

It is:

```text
browser security
+
cross-origin request policy
```

---

# 26. Why Postman can work when the browser fails

You may eventually encounter this:

```text
Postman → API works
Browser → CORS error
```

That doesn't necessarily mean the API is broken.

Browser requests are subject to browser security policies.

Tools like Postman are not operating under exactly the same browser-origin restrictions.

So when debugging:

```text
API itself
vs
browser security layer
```

are separate questions.

---

# 27. Preflight requests

Some cross-origin requests cause the browser to perform a preliminary request:

```http
OPTIONS /api/todos
```

This is called a **preflight**.

Conceptually:

```text
Browser
   ↓
"Am I allowed to make this cross-origin request?"
   ↓
OPTIONS
   ↓
Server says what is permitted
   ↓
actual request
```

You don't normally write this manually.

The browser handles it.

But seeing an `OPTIONS` request in browser developer tools becomes much less mysterious once you know why it exists.

---

# 28. Idempotence

Now we're getting into an important HTTP concept.

A request is roughly **idempotent** when repeating the same request has the same intended effect as making it once.

For example, suppose:

```http
PUT /api/users/42

{
  "name": "Alice"
}
```

Sending that exact replacement repeatedly can conceptually leave the resource in the same state:

```text
name = Alice
```

By contrast, imagine:

```http
POST /api/orders
```

Repeating it might create multiple orders.

So:

```text
PUT
→ generally designed to be idempotent

DELETE
→ generally designed to be idempotent in its intended effect

POST
→ generally not assumed to be idempotent
```

This matters because networks fail.

A client might not know whether a request reached the server.

Retrying a safe-to-repeat operation is fundamentally different from retrying something that might create duplicates.

That is one reason idempotency becomes important in serious API design.

---

# 29. This connects directly to TanStack Query retries

Earlier we discussed retries.

Now you can see why blindly retrying mutations can be dangerous.

Imagine:

```text
POST /api/orders
```

The client sends it.

The server creates the order.

But the response gets lost.

The browser doesn't know whether creation succeeded.

Now automatically retrying the POST could potentially create another order.

So:

```text
query retry
```

and:

```text
mutation retry
```

have very different implications.

This is one reason retry policies need thought.

---

# 30. PUT vs PATCH in a practical example

Suppose the server stores:

```js
{
  id: 42,
  name: "Alice",
  email: "alice@example.com",
  role: "user",
  timezone: "Asia/Kolkata"
}
```

The user edits only their timezone.

### PUT-style replacement

Potentially:

```http
PUT /api/users/42
```

with a complete representation:

```json
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "user",
  "timezone": "Europe/London"
}
```

### PATCH-style partial change

```http
PATCH /api/users/42
```

with:

```json
{
  "timezone": "Europe/London"
}
```

The second expresses much more directly:

> "Change this field."

But again, the actual API contract is what matters.

---

# 31. How this maps to React

Now connect everything.

### GET

Usually appears through:

```jsx
useQuery(...)
```

Flow:

```text
component
 ↓
query key
 ↓
query function
 ↓
GET
 ↓
server
 ↓
response
 ↓
cache
 ↓
UI
```

### POST

Usually appears through:

```jsx
useMutation(...)
```

Flow:

```text
button/form
 ↓
mutation.mutate(data)
 ↓
POST
 ↓
server
 ↓
success/failure
 ↓
invalidate/update cache
 ↓
UI
```

### PUT/PATCH

Same mutation architecture:

```text
user action
 ↓
useMutation
 ↓
PUT/PATCH
 ↓
server changes resource
 ↓
cache synchronization
```

### DELETE

Again:

```text
user action
 ↓
useMutation
 ↓
DELETE
 ↓
server removes resource
 ↓
invalidate/remove/update cache
```

So React doesn't really care about HTTP methods.

The methods describe the **network operation**.

`useQuery` and `useMutation` describe the **client-side state-management role**.

---

# 32. Don't make this mistake

You might see:

```js
useQuery(...)
```

and assume:

```text
GET only
```

Usually that's how you'll use it.

But conceptually `useQuery` is about **reading/querying data**, not about enforcing an HTTP verb.

Likewise:

```js
useMutation(...)
```

doesn't literally mean:

```text
POST only
```

A mutation could use:

```text
POST
PUT
PATCH
DELETE
```

depending on the operation.

The distinction is:

```text
query
→ read/observe server state

mutation
→ cause a server-side change
```

That is much more important than memorizing verbs.

---

# 33. A complete example

Suppose we build a Todo application.

### Fetch list

```js
async function fetchTodos() {
  const response = await fetch("/api/todos");

  if (!response.ok) {
    throw new Error("Failed to fetch todos");
  }

  return response.json();
}
```

Used as:

```jsx
const todosQuery = useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
});
```

---

### Create

```js
async function createTodo(todo) {
  const response = await fetch("/api/todos", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(todo),
  });

  if (!response.ok) {
    throw new Error("Failed to create todo");
  }

  return response.json();
}
```

Used as:

```jsx
const createMutation = useMutation({
  mutationFn: createTodo,

  onSuccess: async () => {
    await queryClient.invalidateQueries({
      queryKey: ["todos"],
    });
  },
});
```

---

### Update

```js
async function updateTodo(todo) {
  const response = await fetch(`/api/todos/${todo.id}`, {
    method: "PATCH",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      completed: todo.completed,
      title: todo.title,
    }),
  });

  if (!response.ok) {
    throw new Error("Failed to update todo");
  }

  return response.json();
}
```

---

### Delete

```js
async function deleteTodo(id) {
  const response = await fetch(`/api/todos/${id}`, {
    method: "DELETE",
  });

  if (!response.ok) {
    throw new Error("Failed to delete todo");
  }
}
```

Now your React application has a complete network vocabulary.

---

# 34. A useful way to read an API endpoint

When you encounter:

```text
PATCH /api/products/42
```

ask:

```text
PATCH
  ↓
What kind of operation?

/api/products
  ↓
What resource?

/42
  ↓
Which resource?

request body
  ↓
What properties should change?

response status
  ↓
Did the server accept it?

response body
  ↓
What does the server say the resulting state is?
```

This is much better than thinking:

> "Oh, that's some endpoint I'm supposed to call."

---

# 35. Resource-oriented thinking

A lot of APIs are designed around resources.

For example:

```text
/users
/products
/orders
/todos
```

Then individual resources:

```text
/users/42
/products/123
/orders/9001
/todos/7
```

And common operations:

```text
GET    /todos
GET    /todos/7
POST   /todos
PATCH  /todos/7
DELETE /todos/7
```

This is commonly called a REST-style design.

But here's an important point:

> **Not every API is REST.**

You may also encounter:

```text
GraphQL
RPC
gRPC
WebSockets
custom action-based APIs
```

We'll eventually touch those concepts too.

For now, REST-style HTTP APIs are the right foundation.

---

# 36. What happens when a form submits?

Let's trace a real React interaction.

```text
User types:
"Learn React"
       ↓
controlled input
       ↓
local React state
       ↓
User clicks Submit
       ↓
event handler
       ↓
mutation.mutate({
  title: "Learn React"
})
       ↓
mutation function
       ↓
POST /api/todos
       ↓
JSON body
       ↓
server
       ↓
database
       ↓
response
       ↓
mutation success
       ↓
invalidate ["todos"]
       ↓
GET /api/todos
       ↓
new list
       ↓
cache
       ↓
React renders
```

Look at how many layers you can now understand individually.

That's the real payoff of this lesson.

---

# 37. The complete vocabulary

At this point, you should be comfortable with:

```text
HTTP
request
response
method
URL
path
query parameter
header
body
JSON
status code
Content-Type
Authorization
GET
POST
PUT
PATCH
DELETE
CORS
authentication
authorization
idempotence
API endpoint
resource
```

And importantly, you should understand how these connect to React:

```text
GET
 ↓
useQuery

POST / PUT / PATCH / DELETE
 ↓
useMutation

mutation succeeds
 ↓
invalidate/update cache

server response
 ↓
server state

URL params
 ↓
query key
```

---

# 38. One subtle but very important final distinction

There are **three separate things** that beginners often collapse into one.

### HTTP

How the browser and server communicate.

```text
GET
POST
status codes
headers
body
```

### API

The contract the backend exposes.

```text
GET /api/todos
POST /api/todos
PATCH /api/todos/:id
```

### TanStack Query

How the React application manages the server data resulting from those API interactions.

```text
cache
query keys
freshness
refetching
mutations
invalidation
optimistic updates
```

So:

```text
HTTP
  ↓
transport/protocol

API
  ↓
server contract

TanStack Query
  ↓
client-side server-state management
```

They are related, but they are not the same thing.

That distinction will save you a lot of confusion when we eventually encounter GraphQL, WebSockets, server actions, or other ways of communicating with backends.

---

And now you have the missing foundation behind all the network examples we've been using. The next topic can move back upward into React itself and tackle **render optimization and referential equality**—why objects/functions get “new identities” on each render, what `React.memo` actually does, and why `useMemo`/`useCallback` exist in the first place.