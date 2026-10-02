
# Lesson 44 — Authentication and Authorization in React Applications

This is one of those topics where React itself is only a small part of the problem.

A React application can render:

```text
Login
Logout
Dashboard
Admin panel
Profile
```

but React does **not** decide whether someone is actually authenticated or authorized to perform a sensitive operation.

The real system looks more like:

```text
                 BROWSER
                    │
                    │ credentials
                    ▼
                 SERVER
                    │
             authentication
                    │
                    ▼
                identity
                    │
             authorization
                    │
                    ▼
               resource
                    │
                    ▼
                database
```

The key principle for this entire lesson is:

> **The browser can control UI, but the server must control security.**

---

# 1. Authentication vs authorization

You've already encountered this distinction with Server Functions.

Let's make it concrete.

### Authentication

Question:

```text
Who are you?
```

Example:

```text
User = Alice
```

### Authorization

Question:

```text
What is Alice allowed to do?
```

Example:

```text
Alice can:
  read project 42
  edit project 42

Alice cannot:
  delete project 99
```

So:

```text
authentication
→ identity

authorization
→ permissions
```

Never combine those ideas into one vague "logged in" boolean.

---

# 2. The dangerous beginner model

A common React implementation looks like:

```jsx
function App() {
  const [loggedIn, setLoggedIn] =
    useState(false);

  if (!loggedIn) {
    return <Login />;
  }

  return <Dashboard />;
}
```

This is perfectly valid **UI state**.

But it is not authentication.

A malicious user can modify:

```text
JavaScript
localStorage
React state
URL
DOM
```

and display:

```text
Dashboard
```

That does not magically authenticate them with the server.

The server may still say:

```text
401 Unauthorized
```

---

# 3. UI state vs security state

This distinction is critical.

Suppose:

```js
const [isAdmin, setIsAdmin] =
  useState(false);
```

Changing it to:

```js
setIsAdmin(true);
```

may cause:

```text
AdminPanel
```

to appear.

But it does not grant:

```text
database permissions
```

So:

```text
client state
→ presentation

server authorization
→ security
```

That's the rule.

---

# 4. The authentication lifecycle

A typical login flow looks like:

```text
User enters credentials
        ↓
POST /login
        ↓
server validates credentials
        ↓
server creates authenticated session
        ↓
browser stores session credential
        ↓
future requests identify user
```

Then:

```text
GET /profile
        ↓
server checks session
        ↓
user = Alice
        ↓
return Alice's profile
```

Notice something:

The browser doesn't tell the server:

```json
{
  "user": "Alice"
}
```

and expect the server to trust it.

The server derives identity from authenticated credentials.

---

# 5. Sessions

A common architecture is a **server-side session**.

Conceptually:

```text
database/session store

session_abc123
      ↓
userId = 42
```

Browser receives:

```text
session cookie
```

Then each request sends the session identifier back.

The server does:

```text
cookie
  ↓
session lookup
  ↓
user ID
  ↓
authenticated request
```

So:

```text
browser
→ "I have session abc123"

server
→ "abc123 belongs to user 42"
```

The browser does not need to know the user's internal session data.

---

# 6. Cookies

A browser cookie can be associated with the site's requests.

For an authentication session, the server may set something conceptually like:

```http
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Lax
```

The exact configuration depends on the application.

The important properties:

### `HttpOnly`

JavaScript cannot directly read the cookie.

That helps prevent some classes of credential theft through client-side script.

### `Secure`

The browser only sends the cookie over HTTPS.

### `SameSite`

Controls when the browser sends the cookie in cross-site contexts and is an important part of CSRF defenses.

These settings are security controls, not React features.

---

# 7. Why `HttpOnly` is interesting

Suppose authentication uses:

```text
session cookie
```

and the cookie is:

```text
HttpOnly
```

Your React code cannot do:

```js
document.cookie
```

to read the session credential.

That is intentional.

Instead:

```text
Browser
 ↓
automatically sends cookie
 ↓
Server
```

React doesn't need direct access to the secret.

That is generally a healthier security model.

---

# 8. Why localStorage is tempting

You might see:

```js
localStorage.setItem(
  "token",
  token
);
```

and later:

```js
const token =
  localStorage.getItem("token");
```

This is convenient.

The problem is that JavaScript can read it.

If malicious script executes in your page due to an XSS vulnerability:

```text
malicious script
 ↓
localStorage.getItem("token")
 ↓
stolen credential
```

So storing long-lived authentication secrets in browser-readable storage introduces risks.

This does not mean "localStorage is always forbidden."

It means:

> Understand what becomes accessible to JavaScript before choosing where credentials live.

---

# 9. XSS

**Cross-Site Scripting (XSS)** means attacker-controlled script ends up executing in your application context.

For example, dangerously rendering HTML:

```jsx
<div
  dangerouslySetInnerHTML={{
    __html: userContent
  }}
/>
```

can be dangerous when `userContent` isn't properly sanitized.

If attacker-controlled JavaScript executes:

```js
fetch("/api/account")
```

or accesses sensitive browser-readable data, the application may be compromised.

This is one reason React's default escaping behavior is valuable.

---

# 10. React's default escaping

Suppose:

```jsx
const comment =
  "<script>...</script>";

return <p>{comment}</p>;
```

React does not treat the string as executable HTML.

It renders it as text.

Conceptually:

```text
"<script>..."
        ↓
text content
```

rather than:

```text
execute script
```

That default behavior protects against many basic injection scenarios.

But it doesn't protect you from every security problem.

For example:

```text
URL handling
third-party scripts
unsafe HTML
server vulnerabilities
authorization flaws
```

still matter.

---

# 11. The authentication boundary

Here's the architecture you should picture:

```text
                    BROWSER

React
 │
 ├── Login UI
 ├── profile UI
 ├── permission-based visibility
 └── route behavior
        │
        │ request
        ▼

                    SERVER

session validation
        │
        ▼
authenticated user
        │
        ▼
authorization
        │
        ▼
business operation
        │
        ▼
database
```

React participates in the top half.

The server owns the security-critical half.

---

# 12. Protected routes

Suppose:

```text
/dashboard
```

requires authentication.

You may create route behavior like:

```jsx
function ProtectedRoute() {
  const user = useCurrentUser();

  if (!user) {
    return <Navigate to="/login" />;
  }

  return <Dashboard />;
}
```

This is useful for UX.

For example:

```text
anonymous user
 ↓
login page
```

instead of:

```text
anonymous user
 ↓
dashboard shell
 ↓
lots of failed API calls
```

But it is **not sufficient security**.

---

# 13. Why the route guard is not security

Suppose:

```text
/dashboard
```

has:

```jsx
if (!user) {
  return <Navigate to="/login" />;
}
```

A malicious user could still directly call:

```http
GET /api/dashboard
```

without using your React route.

So the server must also enforce:

```text
GET /api/dashboard
 ↓
is authenticated?
 ↓
yes/no
```

The real security boundary is the server.

---

# 14. Better mental model for protected UI

Use route guards for:

```text
user experience
```

Use server authorization for:

```text
actual access control
```

So:

```text
Route guard
→ "Don't show this page to anonymous users."

Server auth check
→ "Don't provide protected data to anonymous users."
```

and:

```text
Client admin check
→ hide admin controls

Server authorization
→ reject unauthorized admin operation
```

---

# 15. Identity loading

A React application usually needs to know:

```text
Who is the current user?
```

One architecture is:

```text
GET /api/me
```

or equivalent server-side identity resolution.

Then:

```jsx
const user = useCurrentUser();
```

could internally use:

```text
query/cache/server-provided data
```

The important part is that `user` comes from an authenticated server source, not:

```js
localStorage.user
```

as an authoritative identity.

---

# 16. Don't store the entire user as security truth in Context

You learned Context.

You might do:

```jsx
<AuthContext value={user}>
  <App />
</AuthContext>
```

That's perfectly reasonable.

But Context is still:

```text
client memory
```

not:

```text
security authority
```

Suppose Context says:

```js
{
  id: 42,
  role: "admin"
}
```

The server still needs to verify the user and role for:

```text
DELETE /admin/users/99
```

Context is for React consumers.

The server session/database is the security source of truth.

---

# 17. Roles

A simple authorization model is **role-based access control**.

For example:

```text
user
editor
admin
```

Then:

```text
user
→ read articles

editor
→ read + modify articles

admin
→ users + billing + system administration
```

The server can check:

```js
if (user.role !== "admin") {
  throw new Error("Forbidden");
}
```

But roles are only the beginning.

---

# 18. Permissions

A more detailed model might use explicit permissions:

```text
projects.read
projects.edit
projects.delete
billing.read
billing.manage
users.manage
```

Then:

```js
can(user, "projects.edit")
```

could determine authorization.

That gives you:

```text
identity
+
permissions
```

instead of:

```text
one giant isAdmin flag
```

---

# 19. Roles don't solve resource ownership

Suppose:

```text
Alice = normal user
```

and Alice owns:

```text
Project 42
```

Bob also has:

```text
normal user
```

but owns:

```text
Project 99
```

A rule like:

```js
if (user.role === "user") {
  allowEdit();
}
```

isn't enough.

You need:

```text
Does Alice have permission
to modify Project 42?
```

That may require checking:

```text
user.id
+
project.ownerId
+
project membership
```

This is **resource-based authorization**.

---

# 20. The classic IDOR problem

Imagine:

```http
PATCH /api/projects/42
```

Your server code does:

```js
await db.projects.update(42, body);
```

without checking ownership.

A user changes:

```text
42 → 43
```

and suddenly edits somebody else's project.

This is a classic insecure direct object reference pattern.

The URL parameter itself does not imply permission.

The server must check:

```text
resource 43
+
current user
→
allowed?
```

---

# 21. Authorization should happen near the protected operation

Good:

```js
async function updateProject(
  user,
  projectId,
  patch
) {
  const project =
    await getProject(projectId);

  assertCanEdit(
    user,
    project
  );

  return updateProjectInDb(
    projectId,
    patch
  );
}
```

Less reliable:

```text
route guard somewhere else
 ↓
assume the mutation is safe
```

Security checks should be close enough to the operation that they cannot easily be bypassed by another entry point.

---

# 22. Authentication doesn't mean authorization

Imagine:

```text
Alice logs in successfully.
```

Authentication:

```text
✅ Alice is authenticated
```

But:

```text
DELETE /users/bob
```

could still be:

```text
❌ Alice is not authorized
```

So:

```text
logged in
≠
allowed to do everything
```

This distinction is so important that you should instinctively separate the two words.

---

# 23. Login flow

Let's trace a session-based login.

User enters:

```text
email = alice@example.com
password = secret
```

Browser submits:

```http
POST /login
```

Server:

```text
validate input
 ↓
look up account
 ↓
verify password hash
 ↓
create session
 ↓
set session cookie
```

Response:

```text
200 OK
```

Browser now automatically has:

```text
session cookie
```

Then:

```http
GET /me
```

automatically includes the cookie.

Server:

```text
cookie
 ↓
session lookup
 ↓
user 42
```

Response:

```json
{
  "id": 42,
  "name": "Alice"
}
```

React can now display:

```text
Welcome, Alice
```

---

# 24. Passwords should never be stored as plaintext

The server should not store:

```text
alice@example.com
secret123
```

as-is.

It should store a strong password hash using an appropriate password-hashing algorithm.

Conceptually:

```text
password
 ↓
password hashing function
 ↓
stored password hash
```

During login:

```text
submitted password
 ↓
verify against stored hash
```

The frontend doesn't perform this as its security mechanism.

The server handles credential verification.

---

# 25. React should not implement password security

A component like:

```jsx
function Login() {
  const hash = sha256(password);
}
```

does not replace secure server-side password handling.

Even if you hash in the browser:

```text
browser hash
```

that value can itself become the credential submitted to the server.

The backend still needs proper password-storage and authentication design.

The client is not the trusted security environment.

---

# 26. Logout

Logout should invalidate the session.

Conceptually:

```text
POST /logout
 ↓
server invalidates session
 ↓
cookie cleared/expired
```

Then React can update its user representation:

```text
current user
 ↓
null
```

which causes:

```text
Dashboard
 ↓
Login
```

to render.

Again:

```text
client state change
```

is just the UI reaction.

The real logout is:

```text
server-side session invalidation
```

depending on your session architecture.

---

# 27. What happens when the session expires?

Suppose:

```text
React thinks:
user = Alice
```

but the server session expires.

Then:

```text
GET /profile
 ↓
401 Unauthorized
```

The application should recognize:

```text
authentication is no longer valid
```

and transition UI appropriately.

For example:

```text
query returns 401
 ↓
clear authenticated user
 ↓
redirect/login
```

This is why authentication is dynamic state.

Don't assume:

```text
logged in once
=
logged in forever
```

---

# 28. `401` vs `403`

This distinction is useful.

### `401 Unauthorized`

Usually means:

```text
authentication is missing or invalid
```

Conceptually:

```text
"Who are you?"
→ not successfully authenticated
```

### `403 Forbidden`

Usually means:

```text
authentication exists,
but this action/resource isn't allowed
```

Conceptually:

```text
"I know who you are.
You still cannot do this."
```

So:

```text
401
→ authentication problem

403
→ authorization problem
```

---

# 29. React handling of 401

Suppose TanStack Query sees:

```text
401
```

You may centralize handling.

For example:

```js
async function apiRequest(...) {
  const response = await fetch(...);

  if (response.status === 401) {
    throw new UnauthorizedError();
  }

  ...
}
```

Then a higher-level auth mechanism can react.

The point is to avoid having:

```text
every component
```

individually reinvent:

```text
if 401 → go login
```

---

# 30. Auth state and TanStack Query

This is a very natural use case.

You could have:

```js
useQuery({
  queryKey: ["currentUser"],
  queryFn: getCurrentUser
});
```

Then:

```text
["currentUser"]
```

becomes the client representation of current authenticated identity.

Many components can consume it:

```text
Header
Profile
Navigation
Settings
Dashboard
```

But the underlying truth remains:

```text
server session
```

not the React Query cache.

The query cache is a representation.

---

# 31. Login mutation + current-user query

Imagine:

```text
currentUser = null
```

User logs in.

Mutation:

```text
POST /login
```

succeeds.

Then:

```text
invalidate ["currentUser"]
```

and:

```text
GET /me
```

returns:

```text
Alice
```

Now the application updates:

```text
Header
Dashboard
Profile
```

This is the same server-consistency pattern you've already learned:

```text
mutation
 ↓
invalidate/read fresh server state
 ↓
UI
```

---

# 32. But don't blindly store auth in global client state

Suppose:

```js
const [user, setUser] =
  useState(...);
```

and every mutation manually updates it.

Now you have:

```text
server session
+
React state
```

that can drift.

For example:

```text
server:
role = user

client:
role = admin
```

The client must never become the authority.

This is especially important when permissions can change on the server.

---

# 33. Permission changes

Imagine Alice is an admin.

Then an administrator removes that role.

Server:

```text
Alice.role = user
```

But the browser might still have:

```text
user.role = admin
```

If the UI trusts that forever:

```text
Admin controls remain visible
```

The real server still rejects the request.

That may be okay from a security standpoint if authorization is correctly enforced, but it can produce confusing UX.

So authenticated user data should be refreshed appropriately.

---

# 34. CSRF

This is another major web security concept.

Suppose your application authenticates using cookies.

Because browsers automatically send cookies with requests, another website may potentially cause the browser to send requests to your application.

That's the basic shape of:

**Cross-Site Request Forgery (CSRF).**

Conceptually:

```text
Attacker site
   ↓
causes browser request
   ↓
your site
   ↓
browser may include session cookie
```

If the state-changing endpoint isn't protected correctly, the server might perform an unwanted action as the victim.

---

# 35. Why SameSite matters

Cookie configuration such as:

```text
SameSite=Lax
```

or stricter settings can reduce cross-site cookie sending in many cases.

But CSRF defense is an architectural topic, not merely:

```text
"set SameSite and you're done."
```

Depending on the application's cross-origin requirements, you may also use:

```text
CSRF tokens
origin checks
same-origin protections
careful CORS configuration
```

The exact design depends on the application.

---

# 36. CORS is not authentication

People often confuse:

```text
CORS
```

with:

```text
security
```

CORS controls which browser origins are allowed to read certain cross-origin responses.

It does **not** mean:

```text
"CORS allowed this request, so user is authorized."
```

You still need:

```text
authentication
authorization
```

on the server.

Think:

```text
CORS
→ browser cross-origin policy

authentication
→ identity

authorization
→ permission
```

Different problems.

---

# 37. Access tokens and refresh tokens

Another common authentication architecture uses tokens.

You may have:

```text
access token
```

short-lived:

```text
15 minutes
```

and a longer-lived:

```text
refresh credential
```

which can be used to obtain another access token.

Conceptually:

```text
login
 ↓
access token
 ↓
API requests
 ↓
expires
 ↓
refresh
 ↓
new access token
```

The exact design varies significantly by application and identity provider.

The important lesson for React is:

> **Token lifecycle is security infrastructure, not simply state management.**

---

# 38. Don't put long-lived secrets into component state

Suppose:

```js
const [refreshToken, setRefreshToken] =
  useState(...)
```

Now any code that can access the component state potentially has access to that credential.

That's usually a sign that you're mixing:

```text
security credential storage
```

with:

```text
UI state
```

Prefer architectural approaches that minimize JavaScript access to long-lived authentication secrets.

---

# 39. Route protection in server-aware frameworks

With Server Components/server-aware frameworks, authentication can often be checked before rendering protected server content.

Conceptually:

```jsx
async function DashboardPage() {
  const user = await getCurrentUser();

  if (!user) {
    redirect("/login");
  }

  return (
    <Dashboard user={user} />
  );
}
```

Now:

```text
request
 ↓
server identifies user
 ↓
authorization/authentication check
 ↓
render protected page
```

This can be preferable to:

```text
browser receives protected shell
 ↓
client discovers user isn't authenticated
 ↓
redirect
```

because the server can enforce the access decision before returning protected content.

The exact `redirect()` API depends on the framework/router you're using, but the architecture is broadly applicable.

---

# 40. Server Components make authentication interesting

Suppose:

```jsx
async function ProfilePage() {
  const user =
    await getCurrentUser();

  return (
    <Profile user={user} />
  );
}
```

The browser doesn't need to make an additional:

```text
GET /me
```

just to discover the current user if the server already has that information during rendering.

So:

```text
request
 ↓
authenticated server
 ↓
user
 ↓
Server Component
 ↓
HTML
```

This can reduce client-side bootstrap work.

---

# 41. But Client Components still need auth information sometimes

Suppose:

```jsx
"use client";

function UserMenu({ user }) {
  const [open, setOpen] =
    useState(false);

  ...
}
```

A Server Component can supply:

```jsx
<UserMenu user={user} />
```

where the user object is serialized across the boundary.

Then:

```text
server
 ↓
current user
 ↓
client component
```

The client gets enough information to render:

```text
Alice
Logout
Settings
```

without independently determining authentication.

---

# 42. But permissions still need server enforcement

Suppose:

```jsx
<UserMenu
  user={user}
/>
```

and:

```jsx
{user.role === "admin" && (
  <AdminLink />
)}
```

This is useful for the UI.

Still:

```text
/admin/users
```

must be protected server-side.

And:

```text
DELETE /api/users/42
```

must independently enforce authorization.

So again:

```text
visibility
≠
access control
```

---

# 43. Middleware

Some frameworks provide middleware/proxy layers that can inspect authentication before the request reaches a route.

Conceptually:

```text
request
 ↓
middleware
 ↓
auth check
 ↓
route
```

Useful for:

```text
redirect anonymous users
attach request context
block obviously unauthorized requests
```

But don't assume middleware replaces authorization inside business operations.

A mutation still needs to check:

```text
who is this?
what are they allowed to do?
```

at the point where the sensitive action occurs.

---

# 44. Defense in depth

A healthy system may have several layers:

```text
Layer 1
Client route guard
 ↓
UX

Layer 2
Server route/auth check
 ↓
authentication

Layer 3
Server resource authorization
 ↓
permissions

Layer 4
database constraints
 ↓
data integrity
```

Each layer has a different purpose.

This is **defense in depth**.

---

# 45. Database permissions can be another boundary

Imagine:

```text
application server
 ↓
database
```

The application enforces:

```text
user can only update their own records
```

but you might also use database mechanisms such as:

```text
foreign keys
unique constraints
row-level security
transactions
```

depending on your database.

Then even if an application bug slips through, lower-level integrity constraints can prevent some categories of invalid data.

Security and correctness are layered.

---

# 46. Multi-tenant applications

This gets even more interesting.

Suppose your app supports:

```text
Company A
Company B
```

Alice belongs to Company A.

A request:

```text
GET /projects/42
```

must not merely check:

```text
Alice is logged in
```

It must check:

```text
Project 42 belongs to
Alice's tenant
```

The authorization relationship may be:

```text
user
 ↓
organization
 ↓
project
```

So authorization often becomes contextual:

```text
Can this user
perform this action
on this resource
in this tenant?
```

This is much richer than:

```text
isAdmin
```

---

# 47. Permission checking should be composable

A useful design:

```js
function canEditProject(user, project) {
  return (
    project.organizationId ===
      user.organizationId &&
    (
      project.ownerId === user.id ||
      user.permissions.includes(
        "projects.edit"
      )
    )
  );
}
```

Then your server mutation can do:

```js
if (!canEditProject(user, project)) {
  throw new ForbiddenError();
}
```

This puts the business rule in a testable function.

You've now combined:

```text
TypeScript/domain logic
+
Server Functions
+
authentication
+
authorization
```

---

# 48. Don't make permission checks in JSX your source of truth

This:

```jsx
{canEdit && (
  <button>Edit</button>
)}
```

is good UX.

But:

```text
Can the user see Edit?
```

and:

```text
Can the user actually perform Edit?
```

are separate.

The server must still validate.

---

# 49. Authentication failures vs business failures

Imagine:

```text
POST /projects/42/archive
```

returns:

```text
401
```

That's:

```text
not authenticated
```

Returns:

```text
403
```

That's:

```text
authenticated but forbidden
```

Returns:

```text
409
```

That may indicate a domain conflict:

```text
project is already archived
```

Returns:

```text
422
```

depending on API conventions, might indicate validation failure.

These distinctions help your React UI show the right response.

---

# 50. Don't send unnecessary authorization details to the client

Suppose server knows:

```js
{
  canEditBilling: true,
  canDeleteUsers: false,
  internalSecurityLevel: 7
}
```

You usually don't need to expose every internal security detail.

Return only what the UI genuinely needs.

For example:

```js
{
  id: user.id,
  name: user.name,
  permissions: [
    "billing.read",
    "projects.edit"
  ]
}
```

or an even smaller representation.

The principle:

> **Expose the minimum information necessary to the client.**

This is data minimization.

---

# 51. Don't trust client-supplied role information

Suppose a request contains:

```json
{
  "role": "admin"
}
```

and the server trusts it.

That's broken.

Role and permission information should come from:

```text
authenticated server-side identity
+
trusted authorization data
```

not from arbitrary client input.

---

# 52. A complete login architecture

Let's design one.

```text
                         BROWSER
                            │
                      Login Form
                            │
                            ▼
                       POST /login
                            │
                            ▼
                         SERVER
                            │
                  validate credentials
                            │
                    verify password
                            │
                    create session
                            │
                            ▼
                     session cookie
                            │
                            ▼
                         BROWSER
                            │
                    React user state
                            │
                            ▼
                      Dashboard UI
```

Then:

```text
Dashboard request
      ↓
session cookie
      ↓
server validates session
      ↓
current user
      ↓
authorization
      ↓
data
```

That's the actual security lifecycle.

---

# 53. A protected mutation

Suppose:

```js
async function archiveProject(
  projectId
) {
  const user =
    await getCurrentUser();

  if (!user) {
    throw new UnauthorizedError();
  }

  const project =
    await getProject(projectId);

  if (
    !canArchiveProject(
      user,
      project
    )
  ) {
    throw new ForbiddenError();
  }

  await db.projects.archive(
    projectId
  );
}
```

The React client might simply do:

```js
await archiveProject(projectId);
```

But the security work happens on the server:

```text
identity
 ↓
resource
 ↓
permission
 ↓
mutation
```

That's exactly how Server Functions should be thought about.

---

# 54. Then React manages the UX

The client can show:

```text
Archiving...
```

through:

```text
useActionState
useFormStatus
useTransition
```

and use:

```text
useOptimistic
```

for immediate UI.

So:

```text
security
→ server

UX
→ React
```

This separation is one of the most important architectural boundaries in full-stack React.

---

# 55. Authentication + Router + Server Functions

Now combine several lessons:

```text
Router
  ↓
protected route

Client Component
  ↓
form/button

Server Function
  ↓
authenticate

Server Function
  ↓
authorize

database mutation
  ↓
result

Action state
  ↓
UI
```

That's a complete modern workflow.

---

# 56. Authentication + TanStack Query

Another architecture:

```text
current user
 ↓
TanStack Query

project data
 ↓
TanStack Query

mutation
 ↓
Server Function/API

success
 ↓
invalidate relevant queries
```

For example:

```text
login
 ↓
current-user query invalidated
 ↓
user = Alice
 ↓
dashboard queries become enabled
```

This ties authentication into server-state management.

---

# 57. Dependent queries

Suppose:

```js
const user = useQuery({
  queryKey: ["me"],
  queryFn: getCurrentUser
});
```

Then:

```js
const projects = useQuery({
  queryKey: ["projects"],
  queryFn: getProjects,
  enabled: !!user.data
});
```

The relationship is:

```text
current user
 ↓
authenticated
 ↓
load protected resources
```

But remember:

```text
enabled: !!user.data
```

is a client optimization/UX behavior.

The server still checks authorization for the project request.

---

# 58. Don't treat 404 as authorization automatically

Suppose:

```text
GET /projects/42
```

returns:

```text
404
```

Possible meanings include:

```text
resource truly doesn't exist
```

or, in some privacy-sensitive systems:

```text
user is not allowed to know whether it exists
```

The exact decision is an API/security design choice.

The important point is that status-code behavior can be intentionally designed rather than blindly mapped to UI messages.

---

# 59. Session expiration + mutations

Imagine:

```text
User opens page
 ↓
session valid
```

Twenty minutes later:

```text
user clicks Save
 ↓
POST
 ↓
401
```

Your UI might currently show:

```text
Saving...
```

Then:

```text
401
 ↓
authentication expired
 ↓
show login / renew session
```

This is why loading/error/authentication states should be considered together.

A mutation isn't only:

```text
success/failure
```

It can fail because:

```text
network
validation
authorization
authentication
conflict
server failure
```

---

# 60. Security should survive direct requests

Here's the test I recommend for every protected feature.

Ignore your React application completely.

Pretend someone sends:

```http
POST /api/projects/42/archive
```

directly with a custom script.

Ask:

```text
Does the server still correctly:
  identify the caller?
  validate input?
  check permissions?
  perform the correct mutation?
```

If yes, the security boundary is in the right place.

If the request only works safely because:

```text
React button is hidden
```

the system is insecure.

---

# 61. React is not a security boundary

This sentence is worth remembering exactly:

```text
React is not a security boundary.
```

Neither are:

```text
Context
useState
Redux
TanStack Query
router guards
disabled buttons
hidden inputs
localStorage
```

They can all help the UX.

Security decisions happen in trusted server-side infrastructure.

---

# 62. The complete security model

```text
                        CLIENT
                           │
                   ┌───────┴────────┐
                   │                │
                 React           Browser
                   │
             UI visibility
             loading state
             navigation
             optimistic UI
                   │
                   ▼
              HTTP request
                   │
                   ▼
                       SERVER
                         │
                  ┌──────┴──────┐
                  │             │
             authenticate   validate
                  │             │
                  └──────┬──────┘
                         │
                   authorize
                         │
                         ▼
                      DOMAIN
                         │
                         ▼
                      DATABASE
```

This is the architecture to internalize.

---

# 63. The five security questions

Whenever you build a protected action, ask:

### 1. Who is calling?

```text
authentication
```

### 2. Is the input valid?

```text
validation
```

### 3. Is this user allowed?

```text
authorization
```

### 4. Is the resource actually theirs/accessible?

```text
resource-level authorization
```

### 5. What happens if this request is repeated?

```text
idempotence / transaction safety
```

That last question matters because authentication and authorization don't protect against duplicate execution.

---

# 64. The full application flow

Let's combine everything we've learned.

```text
                    USER
                     │
                     ▼
               React UI
                     │
              protected route
                     │
                     ▼
                form/button
                     │
                     ▼
             Action / mutation
                     │
                     ▼
              Server Function
                     │
                network
                     │
                     ▼
                  SERVER
                     │
              ┌──────┼──────┐
              │      │      │
             auth  validate authorize
              │      │      │
              └──────┼──────┘
                     │
                     ▼
                  domain
                     │
                     ▼
                database
                     │
                     ▼
                  result
                     │
              ┌──────┴──────┐
              │             │
          success         failure
              │             │
              ▼             ▼
       query/revalidation  UI error
              │
              ▼
             React
```

This is a realistic full-stack React mental model.

---

# 65. What to remember

```text
Authentication
→ who are you?

Authorization
→ what are you allowed to do?

Session
→ server-side representation of authenticated identity

Cookie
→ browser-managed credential transport

HttpOnly
→ JavaScript cannot directly read the cookie

Secure
→ cookie sent over HTTPS

SameSite
→ controls cross-site cookie behavior

401
→ authentication problem

403
→ authorization problem

Client route guard
→ UX

Server authorization
→ security

Role
→ broad permission category

Permission
→ specific capability

Resource authorization
→ "Can this user do this to this resource?"

Context
→ shares user information with React

Query cache
→ represents server state

Neither Context nor query cache
→ security authority

Server Function
→ must authenticate/validate/authorize before mutation

XSS
→ attacker-controlled script execution

CSRF
→ unwanted authenticated request induced across sites

CORS
→ browser cross-origin policy, not authorization
```

And the principle to keep permanently:

> **The client can suggest, display, and optimize. The server must decide.**

---

# Where this fits into everything we've learned

Your React architecture now has a much more complete shape:

```text
                         REACT APPLICATION
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
         UI                   ROUTER               STATE
          │                     │                     │
      components            loaders/actions      local/context
      forms                fetchers              query cache
      accessibility        navigation            external store
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
                         SERVER BOUNDARY
                                │
                   ┌────────────┴────────────┐
                   │                         │
             Server Components        Server Functions
                   │                         │
               read/render               mutate
                   │                         │
                   └────────────┬────────────┘
                                │
                         authentication
                                │
                         authorization
                                │
                              domain
                                │
                            database
```

At this point, you've covered most of the important **conceptual React ecosystem**.

The remaining lessons can focus increasingly on putting everything together: **SSR/hydration production patterns, deployment, environment variables, security hardening, testing real applications, observability/error reporting, architecture tradeoffs, and finally a complete production-style React project built from these principles.**