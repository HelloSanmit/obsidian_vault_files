
# Lesson 52 — React Master Review

You've now gone from:

```jsx
function App() {
  return <h1>Hello</h1>;
}
```

to understanding:

```text
React rendering
state
effects
refs
context
reducers
custom hooks
routing
URL state
server state
mutations
concurrency
Suspense
Actions
SSR
Server Components
Server Functions
external stores
real-time systems
security
testing
performance
deployment
observability
architecture
```

The goal of this final lesson is **not** to repeat every API.

It is to compress everything into a few mental models that remain useful even when React itself evolves.

---

# 1. The first mental model: React is a rendering system

At the center of everything:

```text
inputs
  ↓
React render
  ↓
output
```

Inputs include:

```text
props
state
context
external-store snapshots
route state
```

Conceptually:

```text
UI = f(state, props, context, external data)
```

This is why render should be pure.

Given the same relevant inputs:

```text
same inputs
   ↓
same intended UI
```

The render function should not secretly perform:

```text
network requests
subscriptions
random mutations
DOM manipulation
database writes
```

Those are side effects and belong elsewhere.

---

# 2. Render is not the same as DOM mutation

This distinction should be permanently installed in your brain.

```text
trigger
  ↓
render
  ↓
React decides what changed
  ↓
commit
  ↓
DOM changes
```

So:

```text
render ≠ commit
```

A component function running does not mean the browser DOM has already changed.

This explains a huge number of React behaviors.

---

# 3. State is a snapshot

Suppose:

```jsx
const [count, setCount] = useState(0);
```

A render sees:

```text
count = 0
```

Calling:

```js
setCount(1);
```

does not mutate the `count` variable inside that already-running render.

Instead:

```text
request update
 ↓
new render
 ↓
count = 1
```

That gives you the essential model:

> **Each render sees its own snapshot of state.**

---

# 4. Why functional updates exist

Suppose three updates all depend on previous state:

```js
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);
```

Think:

```text
0
 ↓ +1
1
 ↓ +1
2
 ↓ +1
3
```

Whereas:

```js
setCount(count + 1);
setCount(count + 1);
setCount(count + 1);
```

may be repeatedly based on the same render snapshot.

Whenever the next value depends on the previous value:

```text
previous state
    ↓
functional update
```

is the right mental model.

---

# 5. Identity is one of React's hidden foundations

Remember:

```js
{}
[] 
() => {}
```

all create new identities.

So:

```js
{ name: 'Alice' } === { name: 'Alice' }
```

is:

```text
false
```

even though the contents look identical.

This matters for:

```text
props
memo
dependencies
context values
keys
state updates
```

A huge amount of advanced React behavior is really about **identity**.

---

# 6. Keys are identity for list elements

Consider:

```jsx
items.map(item => (
  <Row key={item.id} item={item} />
))
```

The key tells React:

> "This logical child is the same entity across renders."

That's why:

```text
stable key
```

helps React preserve the right component/state identity.

Using:

```js
key={Math.random()}
```

effectively tells React:

> "This is a new thing every time."

That's disastrous for state preservation.

---

# 7. State preservation is identity-based

Imagine:

```text
Form
```

changes from:

```text
Step A
```

to:

```text
Step B
```

Whether state survives depends on whether React considers the relevant component identity the same.

To reset state deliberately, change identity:

```jsx
<Form key={userId} />
```

Now:

```text
user A
 ↓
Form identity A

user B
 ↓
Form identity B
```

and state resets.

This is much cleaner than manually resetting every field.

---

# 8. State should live where its meaning lives

This is perhaps the most important architecture lesson.

Ask:

> "Who actually owns this information?"

Examples:

```text
one component
→ useState

complex local workflow
→ useReducer

subtree-wide value
→ Context

URL-visible state
→ router/search params

server-owned data
→ server-state cache

external mutable source
→ useSyncExternalStore

temporary optimistic result
→ optimistic state
```

Don't choose a state mechanism because it is fashionable.

Choose it because its **ownership model** fits.

---

# 9. Derived data should usually remain derived

Suppose:

```js
const completed =
  tasks.filter(t => t.status === 'done');
```

You generally shouldn't create:

```js
const [completed, setCompleted] = useState([]);
```

because now you must keep:

```text
tasks
+
completed
```

synchronized.

Better:

```text
canonical tasks
 ↓
derived completed
```

Fewer sources of truth mean fewer synchronization bugs.

---

# 10. `useEffect` is not "code that runs after render"

This is one of the biggest React misconceptions.

The better definition is:

> **An Effect synchronizes React with an external system.**

External systems include:

```text
network
DOM APIs
browser events
timers
subscriptions
third-party widgets
WebSocket connections
```

For example:

```text
React state
   ↓
Effect
   ↓
external system
```

or:

```text
external system
   ↓
Effect subscription
   ↓
React state/store
```

If nothing external is involved, ask whether you need an Effect at all.

---

# 11. Cleanup is part of synchronization

Suppose:

```jsx
useEffect(() => {
  const socket = new WebSocket(url);

  return () => {
    socket.close();
  };
}, [url]);
```

The full lifecycle is:

```text
render
 ↓
commit
 ↓
setup connection
 ↓
dependency changes
 ↓
cleanup old connection
 ↓
setup new connection
```

Cleanup isn't an optional extra.

It's half of the synchronization contract.

---

# 12. Ref vs state

State:

```text
changing it
 ↓
request a render
```

Ref:

```text
changing .current
 ↓
no render by itself
```

So use a ref when you need:

```text
DOM node
mutable instance-like value
latest value for imperative code
```

not when the screen needs to update immediately.

A good question is:

> **Should changing this value cause UI to recalculate?**

Yes → state.

No → a ref may be appropriate.

---

# 13. Context is dependency distribution, not automatically state management

Context answers:

> "How can descendants access this value without passing it through every intermediate component?"

Typical examples:

```text
theme
current locale
authenticated user information
configuration
```

Context itself does not solve:

```text
server cache
complex async workflows
database synchronization
```

The mistake is using Context as a universal global-state bucket.

---

# 14. Reducers are state machines

A reducer:

```js
(state, action) => nextState
```

means:

```text
current state
   +
event
   ↓
next state
```

Example:

```text
idle
 ↓ SUBMIT
pending
 ↓ SUCCESS
success
```

Reducers become valuable when many transitions interact.

They make state transitions explicit instead of scattering them across handlers.

---

# 15. Custom hooks are logic boundaries

A custom hook should usually answer:

> "What reusable stateful behavior can I extract?"

For example:

```js
useOnlineStatus()
useDebouncedValue()
useProjectFilters()
useCreateTask()
```

A custom hook shares:

```text
logic
```

not automatically:

```text
the same state instance
```

This distinction is critical.

---

# 16. Router is about navigation and URL state

React Router gives you concepts such as:

```text
route
params
search params
nested layouts
navigation
loaders/actions
```

A useful model:

```text
URL
 ↓
route matching
 ↓
application structure
```

and:

```text
URL parameters
+
search parameters
 ↓
page identity/filter state
```

The URL itself can be part of your application's state model.

---

# 17. Route params vs search params

Route param:

```text
/projects/42
```

means:

```text
which project?
```

Search param:

```text
?status=open&page=2
```

means:

```text
how are we viewing it?
```

That's why:

```text
resource identity
```

and:

```text
view/filter state
```

often naturally map to those two URL mechanisms.

---

# 18. Server state is fundamentally different from UI state

UI state:

```text
modalOpen
selectedTab
inputDraft
```

Server state:

```text
projects
tasks
comments
profile
```

Server state has special properties:

```text
remote ownership
latency
staleness
caching
refetching
deduplication
concurrency
invalidation
```

That's why a server-state library such as TanStack Query is useful.

---

# 19. Query keys are identities for server data

This:

```js
['tasks', projectId, filters]
```

isn't arbitrary syntax.

It means:

> "This specific representation of task data."

The key determines cache identity.

So:

```text
URL/filter state
 ↓
query key
 ↓
cache entry
 ↓
server request
```

can form a coherent data architecture.

---

# 20. Mutations are commands

Queries ask:

```text
"What is the state?"
```

Mutations ask:

```text
"Make something happen."
```

Examples:

```text
create
update
delete
archive
```

After a mutation, you must think about:

```text
What cached representations are now stale?
```

This is why mutation handling and query invalidation are tightly connected.

---

# 21. Cache consistency is part of correctness

Imagine:

```text
detail → Done
list → Open
```

No exception.

No crash.

But the application is still wrong.

This is a very important production-level idea:

> **Consistency bugs can be more subtle than crashes.**

A correct app must keep related representations converging toward server truth.

---

# 22. Optimistic state is prediction, not truth

Optimistic UI:

```text
user action
 ↓
assume success
 ↓
update UI immediately
 ↓
server confirms/rejects
```

Therefore it needs a reconciliation strategy:

```text
success → keep/replace with canonical state
failure → rollback/correct
```

Optimistic UI without a failure model is incomplete architecture.

---

# 23. Suspense solves a different problem

Suspense answers:

> **"What should React render while this subtree isn't ready?"**

It doesn't automatically decide:

```text
how data is fetched
how data is cached
how requests are retried
how authorization works
```

Think:

```text
data/code/resource
       ↓
not ready
       ↓
Suspense boundary
       ↓
fallback
```

Suspense is rendering coordination.

---

# 24. Transitions solve another problem

`startTransition` says roughly:

> "This update is non-urgent."

For example:

```text
typing
 ↓
urgent input update

expensive result update
 ↓
transition/non-urgent
```

A transition is not:

```text
debounce
timeout
cache
server fetch
```

It is about scheduling UI work with different urgency.

---

# 25. `useDeferredValue` is about lagging intentionally

Think:

```text
current value ─────→ immediate UI

deferred value ────→ expensive UI
```

The expensive area can temporarily render based on an older value.

That tradeoff can improve responsiveness.

---

# 26. Server Components answer "where does this code run?"

This is a different axis entirely.

Server Component:

```text
executes on server
```

Client Component:

```text
belongs to client-capable module graph
```

Don't confuse:

```text
Server Component
```

with:

```text
SSR
```

SSR concerns how initial output is produced.

Server Components concern component execution/module boundaries.

---

# 27. `"use client"` means module boundary

This is one of the advanced concepts worth remembering forever.

It does not simply mean:

> "This function runs in the browser."

Rather, it marks where the client module graph begins.

That means an accidental import can pull a large dependency tree into the client.

So:

```text
server/client boundary
```

is also a bundle-size architecture decision.

---

# 28. Server Functions are a network boundary

Think:

```text
client invocation
 ↓
network/server boundary
 ↓
server function
 ↓
business logic
 ↓
database
```

Therefore arguments arriving there are untrusted.

The function must:

```text
authenticate
validate
authorize
mutate
```

The fact that the call looks like a function call does not remove the security boundary.

---

# 29. Authentication and authorization

Memorize the separation:

```text
Authentication
= Who are you?

Authorization
= What are you allowed to do?
```

And:

```text
React UI guard
≠
server authorization
```

Hiding a button is UX.

Protecting the underlying operation is security.

---

# 30. SSR mental model

SSR:

```text
request
 ↓
server render
 ↓
HTML
 ↓
browser
 ↓
hydration
```

Hydration:

```text
existing server HTML
 ↓
attach/reactivate client behavior
```

Therefore:

```text
visible
≠
interactive
```

during initial loading.

And server/client initial output must be compatible.

---

# 31. Hydration bugs usually come from nondeterminism

Be suspicious of things like:

```js
Date.now()
Math.random()
window
localStorage
navigator
timezone-dependent formatting
```

during initial render when server and client can see different values.

Think:

```text
server render
      ↓
client initial render
      ↓
must correspond
```

---

# 32. External stores solve a different ownership problem

When React isn't the owner of the state:

```text
WebSocket manager
browser online state
external app store
third-party library
```

you can bridge it with:

```text
useSyncExternalStore
```

The mental model:

```text
external source
 ↓
subscribe
 ↓
stable snapshot
 ↓
React
```

This keeps React synchronized without pretending the external system is a normal React state variable.

---

# 33. Real-time apps need consistency, not just sockets

WebSocket:

```text
persistent communication
```

doesn't solve:

```text
ordering
duplicates
missed events
reconnection
resync
conflicts
```

A robust real-time architecture often needs:

```text
event IDs
sequence/cursor
reconnect
resynchronization
deduplication
```

The transport is only one layer.

---

# 34. Composition is one of React's greatest design tools

Instead of:

```text
giant configurable component
```

prefer composition where useful:

```jsx
<Modal>
  <ModalHeader />
  <ModalBody />
  <ModalFooter />
</Modal>
```

or:

```jsx
<Layout
  sidebar={<Sidebar />}
  toolbar={<Toolbar />}
>
  <Content />
</Layout>
```

Composition lets the caller decide structure.

That tends to produce more reusable APIs than dozens of boolean props.

---

# 35. Your component API should express intent

Bad:

```jsx
<Button
  leftIcon="save"
  showLoader
  loaderColor="white"
  align="center"
  padding="..."
/>
```

Better:

```jsx
<SaveButton isPending={isPending} />
```

when the underlying behavior is conceptually "save."

A reusable component should expose the important contract, not every implementation detail.

---

# 36. Performance: don't optimize the word "rerender"

A rerender isn't inherently bad.

Ask:

```text
Is the rerender:
  unnecessary?
  expensive?
  causing user-visible delay?
```

A cheap rerender can be completely harmless.

A single expensive calculation can be a major problem even if the component rerenders only once.

So:

```text
rerender count
≠
user-visible performance
```

---

# 37. The performance hierarchy

When something is slow, investigate in roughly this space:

```text
network
server
database
JavaScript download
JavaScript execution
React rendering
DOM/layout/paint
large data structures
```

Don't assume React is responsible just because the product is a React app.

---

# 38. Measure before optimizing

Useful tools/concepts include:

```text
React Profiler
browser Performance tools
network timing
production telemetry
RUM
```

The loop is:

```text
measure
 ↓
identify bottleneck
 ↓
optimize
 ↓
measure again
```

Not:

```text
guess
 ↓
useMemo
 ↓
hope
```

---

# 39. Memoization is an optimization

Remember:

```text
React.memo
useMemo
useCallback
```

do not make incorrect code correct.

They optimize whether certain work can be skipped.

Use them when you understand:

```text
what is expensive
what identity changes
what work can be avoided
```

And current React's compiler tooling can automate some memoization opportunities, which makes understanding **render cost and architecture** even more important than manually sprinkling memoization everywhere.

---

# 40. Accessibility is not an add-on

A good component API includes:

```text
semantic HTML
keyboard behavior
focus management
labels
error states
loading states
accessible names/descriptions
```

For example:

```text
button
```

should behave like a button.

Don't build:

```html
<div onclick="...">
```

and then try to recreate all button semantics manually.

Use the platform first.

---

# 41. Testing: test behavior

The strongest general test principle:

> **Test what users and systems can observe.**

Prefer:

```text
click
 ↓
UI changes
 ↓
assert
```

over:

```text
inspect private component state
```

A good test survives implementation changes.

---

# 42. Test the right boundary

Use:

```text
unit
→ pure logic

integration
→ interacting UI/data boundaries

E2E
→ critical real workflows
```

Don't mock away the behavior you're trying to verify.

Mock external boundaries when appropriate.

For server-state UI:

```text
component
 ↓
real query logic
 ↓
controlled network
```

is often much more useful than:

```text
mock useQuery()
```

---

# 43. Security: trust boundaries matter more than React APIs

A value can come from:

```text
URL
form
localStorage
React state
query params
WebSocket
API
```

and still be untrusted.

Ask:

```text
Can the client manipulate it?
```

Usually, yes.

Therefore:

```text
server validation
+
authorization
```

must protect sensitive operations.

---

# 44. React escaping is useful, but not magic

Normal JSX:

```jsx
<div>{userText}</div>
```

is fundamentally different from:

```jsx
<div
  dangerouslySetInnerHTML={{
    __html: userHtml
  }}
/>
```

The second is a deliberate security-sensitive boundary.

If rich HTML is required:

```text
untrusted content
 ↓
sanitize
 ↓
trusted representation
 ↓
controlled rendering
```

And browser-level defenses such as CSP and Trusted Types can provide additional layers.

---

# 45. Error handling should be classified

Not every failure belongs in the same place.

```text
render error
→ Error Boundary

query failure
→ query/error UI

mutation failure
→ mutation state + recovery

validation failure
→ form/Action state

authorization failure
→ server 401/403

network loss
→ retry/offline behavior

unexpected production error
→ telemetry
```

This is far better than:

```text
try/catch everything
```

---

# 46. Reliability means defining failure behavior

For every important operation, ask:

```text
What if the server is slow?
What if it fails?
What if the request is duplicated?
What if the client disconnects?
What if the cache is stale?
What if another user changes the same resource?
```

Production engineering starts where the happy path ends.

---

# 47. Observability is the system talking back to you

Remember:

```text
logs
metrics
traces
errors
```

A useful production event might contain:

```text
requestId
release
route
operation
duration
error type
safe user/session context
```

Then:

```text
user problem
 ↓
telemetry
 ↓
engineer can reconstruct what happened
```

Without that, debugging becomes guesswork.

---

# 48. Deployment is part of application correctness

A correct source tree can still produce a broken production application through:

```text
wrong env variables
bad SPA fallback
broken lazy chunks
SSR mismatch
asset caching
incompatible API versions
database migration problems
```

So:

```text
source code
```

is only one part of the system.

The deployed environment matters too.

---

# 49. A request should have a life cycle in your head

Whenever a user performs an operation, imagine:

```text
USER
 ↓
EVENT
 ↓
REACT UPDATE
 ↓
RENDER
 ↓
COMMIT
 ↓
NETWORK / SERVER OPERATION
 ↓
AUTHENTICATION
 ↓
VALIDATION
 ↓
AUTHORIZATION
 ↓
DATABASE / EXTERNAL SYSTEM
 ↓
RESPONSE
 ↓
CACHE RECONCILIATION
 ↓
REACT UPDATE
 ↓
RENDER
 ↓
COMMIT
 ↓
USER SEES RESULT
```

If real-time is involved:

```text
SERVER EVENT
 ↓
WebSocket
 ↓
EXTERNAL STORE / CACHE
 ↓
REACT
```

If something fails:

```text
FAILURE
 ↓
RECOVERY UI
+
TELEMETRY
```

This one diagram can explain a surprisingly large percentage of real React applications.

---

# 50. Your ultimate state-classification cheat sheet

When you encounter state:

|Question|Likely answer|
|---|---|
|Can it be calculated from existing state?|Derive it|
|Belongs to one component?|`useState`|
|Many related transitions?|`useReducer`|
|Shared by a subtree?|Context|
|Part of navigation/shareable URL?|Router / search params|
|Owned by backend?|Server-state cache|
|Owned by browser/external system?|`useSyncExternalStore`|
|Temporary prediction?|Optimistic state|
|Form submission lifecycle?|Action|
|Requires server execution?|Server Function/API|

And one extra question:

> **Does this value need to exist at all, or am I storing something I can derive?**

That question eliminates a lot of bad state architecture.

---

# 51. Your ultimate debugging checklist

When the UI is wrong, don't randomly add logs.

Walk the system:

```text
1. Did the event fire?
2. Did state/URL/query state change?
3. What did the next render see?
4. Did React commit the expected output?
5. Did the network request happen?
6. What did the server receive?
7. What did the server authorize?
8. What did the database change?
9. Did the client cache reconcile?
10. Did another external event overwrite it?
```

This turns debugging into tracing data flow.

---

# 52. Your ultimate architecture checklist

Before shipping a feature, ask:

```text
STATE
Who owns it?
How long does it live?

DATA
Where is canonical truth?
How is it cached?

ROUTING
Should this be represented in the URL?

SERVER
Does this cross a server boundary?
Where is validation?
Where is authorization?

UI
What are loading/error/empty states?

ASYNC
What happens on slow network?
What happens on retry?
What happens on duplicate submission?

REAL-TIME
Can events be missed?
How do we resync?

SECURITY
Can the client manipulate this value?
Could it become HTML/code?
Are secrets staying server-side?

ACCESSIBILITY
Can keyboard and assistive technology users use it?

TESTING
What should be unit-tested?
Integration-tested?
E2E-tested?

PERFORMANCE
What is actually expensive?

OBSERVABILITY
How will we know if it breaks in production?

DEPLOYMENT
What happens across builds, environments, migrations, and rollouts?
```

That is a serious engineering checklist.

---

# 53. The biggest mistakes to avoid

## Mistake 1: "Every problem needs a hook"

No.

First identify the problem.

Then choose the smallest appropriate mechanism.

---

## Mistake 2: "UseEffect for everything"

No.

Effects synchronize with external systems.

Don't use an Effect merely because:

> "Something should happen after render."

---

## Mistake 3: "Context is global state"

Not really.

Context is dependency delivery through a tree.

State ownership is a separate question.

---

## Mistake 4: "The client knows the truth"

No.

For protected or durable data:

```text
server/database
```

is authoritative.

---

## Mistake 5: "If the button is hidden, it's secure"

No.

The server must enforce authorization.

---

## Mistake 6: "A rerender means performance problem"

No.

Measure actual cost.

---

## Mistake 7: "More memoization means faster React"

No.

Wrong memoization can add complexity while solving nothing.

---

## Mistake 8: "Tests should verify implementation"

Usually no.

Verify user-visible or contract-level behavior.

---

## Mistake 9: "WebSocket means real-time consistency"

No.

You still need ordering, deduplication, reconnection, and resynchronization.

---

## Mistake 10: "SSR and Server Components are the same"

No.

They concern different architectural dimensions.

---

# 54. How an experienced React developer thinks

A beginner asks:

> "Which hook should I use?"

A more experienced developer asks:

> "What state do I have?"

An even more experienced developer asks:

> "Who owns that state?"

Then:

> "What is the canonical source of truth?"

Then:

> "What happens when it becomes stale?"

Then:

> "What happens when the request fails?"

Then:

> "What happens if two actors change it at once?"

Then:

> "How will I observe and test this in production?"

That progression is the real course.

---

# 55. You no longer need to memorize React as a list

You can reconstruct solutions from principles.

For example, suppose tomorrow React introduces:

```text
useSomethingNew()
```

You don't need to blindly memorize it.

Ask:

```text
What problem does it solve?
What owns the state?
What is its lifetime?
What boundary does it introduce?
What happens during render?
What happens during commit?
How does it interact with async work?
How is failure handled?
```

You already know how to reason about the answer.

---

# 56. The final React map

Keep this mental picture:

```text
                           REACT
                             │
                         RENDERING
                             │
                  ┌──────────┴──────────┐
                  │                     │
                 STATE                INPUT
                  │                     │
        ┌─────────┼─────────┐      events / URL
        │         │         │
      local     shared    server
      state     context    state
        │         │         │
 useState/      Context   Query cache
 useReducer                 │
        │                    │
        └──────────┬─────────┘
                   │
                 UI
                   │
        ┌──────────┼─────────────┐
        ▼          ▼             ▼
     Router      Actions      Effects
        │          │             │
        │          │       external systems
        │          │
        │     Server Functions
        │          │
        └──────────┼─────────────┐
                   ▼             ▼
                SERVER       REAL-TIME
                   │             │
              auth/validate   WebSocket
              authorize          │
                   │             │
                DATABASE         │
                   │             │
                   └──────┬──────┘
                          ▼
                         UI

       + Suspense
       + Transitions
       + SSR
       + Server Components
       + Accessibility
       + Security
       + Testing
       + Performance
       + Observability
       + Deployment
```

---

# 57. The final principle

Everything we've learned reduces to one enormous engineering question:

> **Where does this information or behavior belong, and what guarantees should exist at that boundary?**

For state:

```text
Who owns it?
```

For server data:

```text
What is canonical?
```

For security:

```text
Where is the trust boundary?
```

For async work:

```text
What happens when it is slow or fails?
```

For performance:

```text
Where is the actual bottleneck?
```

For testing:

```text
What boundary proves the behavior?
```

For deployment:

```text
What changes outside local development?
```

For production:

```text
How will we know what happened?
```

---

# 58. You are now at the point where React becomes a design problem

The syntax becomes the easy part.

You can already write:

```jsx
const [value, setValue] = useState('');
```

The harder and more valuable questions are:

```text
Should this even be state?

Should it be URL state?

Should the server own it?

Should it be cached?

Should it be optimistic?

Should it cross the client boundary?

Should it be an external store?

Should this be a Server Component?

Should this be a Client Component?

Should this be an Action?

Should this be a Server Function?

What happens if the operation fails?

How do I test it?

How do I observe it?
```

That is the level at which you should now approach React.

---

# Final course conclusion

You've effectively moved through three stages:

### Stage 1 — Learn React

```text
components
JSX
state
events
hooks
```

### Stage 2 — Understand React

```text
rendering
identity
effects
composition
routing
server state
concurrency
server/client boundaries
```

### Stage 3 — Engineer with React

```text
architecture
security
testing
performance
SSR
real-time systems
deployment
observability
reliability
```

The third stage is where React stops being "a frontend library I know" and becomes "a system I can reason about."

And that is the real end of the course.