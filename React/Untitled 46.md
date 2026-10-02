
# Lesson 43 — Production React Architecture: How to Structure a Large Application

We've accumulated enough React knowledge that the next problem is no longer:

> "How does this hook work?"

It becomes:

> **"Where does all this code actually go in a real application?"**

A small React project can survive almost any folder structure.

A large project cannot.

At some point, you start seeing things like:

```text
App.jsx
 ├── 900 lines
 ├── 17 useState calls
 ├── 8 useEffect calls
 ├── router logic
 ├── API calls
 ├── validation
 ├── modal logic
 ├── analytics
 └── business rules
```

Then:

```text
utils/
  helpers.js
  helpers2.js
  helpers-final.js
  common.js
```

And:

```text
hooks/
  useData.js
  useThing.js
  useManager.js
  useHandler.js
```

This is where React architecture becomes important.

The goal of architecture is not:

```text
"Put files into beautiful folders."
```

The goal is:

> **Make responsibilities, ownership, dependencies, and boundaries obvious.**

---

# 1. The first architectural mistake: organizing by file type

A beginner structure often looks like:

```text
src/
├── components/
├── hooks/
├── contexts/
├── reducers/
├── services/
├── utils/
├── pages/
└── api/
```

At first this feels clean.

But imagine a feature called:

```text
billing
```

You may end up with:

```text
components/BillingForm.jsx
hooks/useBilling.js
services/billing.js
contexts/BillingContext.jsx
utils/billing.js
pages/Billing.jsx
api/billing.js
```

The feature is scattered everywhere.

A developer working on billing has to jump through the entire project.

That's a scaling problem.

---

# 2. Feature/domain organization

A different approach is:

```text
src/
├── features/
│   ├── auth/
│   ├── billing/
│   ├── projects/
│   ├── notifications/
│   └── reports/
│
├── app/
├── shared/
└── routes/
```

Now billing can contain:

```text
features/
└── billing/
    ├── components/
    ├── hooks/
    ├── api/
    ├── state/
    ├── types/
    ├── validation/
    └── index.js
```

The exact names aren't important.

The important idea is:

```text
feature
   ↓
its UI
its state
its data logic
its validation
its types
```

stay together.

---

# 3. Why feature boundaries matter

Suppose the application has:

```text
auth
projects
reports
billing
notifications
```

A project developer mostly works within:

```text
projects/
```

instead of navigating across:

```text
components/
hooks/
services/
reducers/
utils/
```

That improves something very important:

```text
change locality
```

Meaning:

> How much of the codebase must I understand or touch to make one change?

Good architecture makes that number smaller.

---

# 4. Think in dependencies, not folders

Folders are only a visualization.

The real architecture is the dependency graph:

```text
Component A
   ↓
Hook B
   ↓
Service C
   ↓
API D
```

A healthy dependency graph tends to point in understandable directions.

For example:

```text
UI
 ↓
feature logic
 ↓
data access
 ↓
server/API
```

What you want to avoid is:

```text
UI
 ↔ state
 ↔ API
 ↔ UI
 ↔ utils
 ↔ feature B
 ↔ feature C
```

Circular dependencies and tangled ownership are architecture smells.

---

# 5. A useful layered model

For a typical application:

```text
Presentation
     ↓
Feature/application logic
     ↓
Domain logic
     ↓
Data access
```

For example:

```text
Button
 ↓
useCheckout()
 ↓
calculateOrder()
 ↓
checkout API
```

But don't turn this into six unnecessary layers for every feature.

The point is:

```text
UI shouldn't secretly contain database/application rules.
```

---

# 6. UI logic vs business logic

This distinction is extremely important.

Suppose:

```js
function Checkout() {
  const total =
    items.reduce(...);

  ...
}
```

Maybe:

```text
total = subtotal + tax + shipping - discount
```

is business logic.

That rule might also be needed by:

```text
checkout summary
invoice
receipt
order confirmation
admin order page
```

If you bury it inside one component:

```text
Checkout.jsx
```

you make the rule difficult to reuse and test.

Instead:

```js
function calculateOrderTotal(order) {
  ...
}
```

can live closer to the domain.

Then:

```text
UI
 ↓
domain function
```

The component presents the result.

---

# 7. Pure business logic is especially valuable

Suppose:

```js
function calculateDiscount(
  subtotal,
  coupon
) {
  ...
}
```

If it's pure:

```text
same input
 ↓
same output
```

you can test it without:

```text
React
browser
DOM
network
```

That makes business rules cheap to reason about.

This connects directly to your reducer lesson.

Reducers are useful partly because they isolate:

```text
state transition logic
```

from:

```text
UI rendering
```

---

# 8. "God components"

One of the most common large-project problems is the **god component**.

For example:

```jsx
function Dashboard() {
  // fetch user
  // fetch projects
  // fetch billing
  // manage filters
  // manage modals
  // calculate permissions
  // transform API data
  // validate forms
  // render tables
  // render charts
  // submit mutations
  // handle websocket events
}
```

It becomes impossible to answer:

```text
What does Dashboard actually own?
```

A better structure might be:

```text
Dashboard
├── DashboardHeader
├── ProjectSummary
├── BillingSummary
├── ActivityFeed
└── ReportsWidget
```

with feature-specific hooks/services below them.

---

# 9. But don't make "one component per div"

The opposite problem is **over-fragmentation**.

You might see:

```text
UserName.jsx
UserEmail.jsx
UserAvatar.jsx
UserButton.jsx
UserLabel.jsx
UserContainer.jsx
```

for trivial markup.

Now you have 30 files to understand a simple card.

Component boundaries should usually correspond to:

```text
meaningful behavior
reusable UI
independent state
independent lifecycle
clear API
```

Not:

```text
"this JSX has 7 lines."
```

---

# 10. Component boundaries are API boundaries

You've already learned composition and component API design.

For example:

```jsx
<UserTable
  users={users}
  onSelect={handleSelect}
/>
```

The public contract is:

```text
users
onSelect
```

Everything inside:

```text
sorting
row rendering
empty state
loading state
```

can be internal.

This means a good component boundary reduces the amount of implementation detail other code needs to know.

That's one of the most powerful uses of abstraction.

---

# 11. Don't make custom hooks the new god object

We've covered custom hooks.

Now you may be tempted to do this:

```js
function useDashboard() {
  // user
  // billing
  // projects
  // filters
  // websocket
  // modal
  // navigation
  // permissions
  // analytics
}
```

You've simply moved the god component problem from:

```text
Dashboard.jsx
```

to:

```text
useDashboard.js
```

The solution isn't "use more hooks."

The solution is **clear ownership**.

---

# 12. A healthy hook composition

Instead:

```js
const user = useCurrentUser();
const projects = useProjects();
const filters = useProjectFilters();
const modal = useModal();
```

Now each hook answers one meaningful question:

```text
useCurrentUser
→ current identity

useProjects
→ project data

useProjectFilters
→ filter state

useModal
→ modal state
```

Then a page composes them.

This is similar to function composition:

```text
small clear pieces
   ↓
larger behavior
```

---

# 13. A page doesn't have to own everything it renders

Imagine:

```text
DashboardPage
├── useUser()
├── useProjects()
├── useNotifications()
├── useReports()
└── useBilling()
```

Maybe that's reasonable.

But if:

```text
BillingWidget
```

needs billing state only, it may own:

```text
useBilling()
```

instead of forcing:

```text
DashboardPage
```

to understand billing internals.

This is another form of **state colocation**.

---

# 14. Server vs client code should be physically obvious

After the RSC lessons, this becomes important.

You may have:

```text
features/projects/
├── components/
├── server/
├── client/
├── shared/
└── data/
```

You don't necessarily need exactly these folders.

But the concept should be obvious:

```text
server-only logic
client-only logic
shared logic
```

should not be casually mixed.

For example, don't create:

```js
shared/api.js
```

and then import:

```text
database credentials
```

through code that's supposed to run in the browser.

---

# 15. Shared code has to actually be shareable

A useful test:

```text
Could this file run in both environments?
```

If yes:

```text
shared
```

might make sense.

If it uses:

```text
window
localStorage
document
```

it's client-oriented.

If it uses:

```text
database
filesystem
secret credentials
```

it's server-oriented.

If it contains:

```text
pure functions
types
validation schemas
domain calculations
```

it may be shared.

This classification becomes extremely valuable in RSC applications.

---

# 16. Avoid generic `utils`

This:

```text
utils/
```

often becomes:

```text
utils/date.js
utils/api.js
utils/format.js
utils/helpers.js
utils/common.js
utils/misc.js
```

Eventually nobody knows where anything belongs.

Instead, organize by **meaning**.

Bad:

```text
utils/calculatePrice.js
```

Better:

```text
features/billing/domain/pricing.js
```

if price calculation is specifically billing logic.

The name and location communicate ownership.

---

# 17. A domain function should belong to its domain

Suppose:

```js
function canCancelSubscription(subscription) {
  ...
}
```

This isn't a generic utility.

It's a billing/domain rule.

So:

```text
billing/
  canCancelSubscription.ts
```

is more informative than:

```text
utils/
  subscription.ts
```

Architecture should help answer:

> **Who owns this rule?**

---

# 18. Shared code should earn its way into `shared`

Don't create:

```text
shared/
  Button.jsx
```

just because two components use a button.

A component becomes genuinely shared when:

```text
multiple features depend on it
+
its API is stable
+
it has no feature-specific assumptions
```

Otherwise it may belong inside the feature.

This prevents premature abstraction.

---

# 19. The "shared" graveyard

A common architecture failure:

```text
shared/
├── Button
├── Modal
├── useData
├── constants
├── helpers
├── types
├── hooks
└── random stuff
```

Everything ends up there because:

> "It's shared."

Eventually:

```text
shared
=
nothing has clear ownership
```

A shared module should be shared **intentionally**, not simply because its location is convenient.

---

# 20. Feature boundaries can be enforced with imports

Imagine:

```text
features/auth
features/billing
features/projects
```

You don't want:

```text
billing
 ↓
imports internal implementation
 ↓
auth/private-file.js
```

Instead, features can expose a deliberate public API:

```text
features/auth/index.ts
```

and consumers import:

```js
import {
  useCurrentUser
} from "@/features/auth";
```

rather than:

```js
import {
  useCurrentUser
} from "@/features/auth/internal/session/useUser";
```

This creates an architectural boundary.

---

# 21. Barrel files: useful but dangerous

You may see:

```js
export * from "./components";
export * from "./hooks";
export * from "./api";
```

This is a barrel.

It can make imports pleasant:

```js
import {
  useProjects,
  ProjectCard
} from "@/features/projects";
```

But excessive barrels can create:

```text
large dependency graphs
circular imports
poor tree-shaking in some setups
unclear dependency direction
```

So use them as intentional public interfaces, not automatically for every folder.

---

# 22. The public API idea

A feature should ideally say:

```text
"These are the things other parts
of the application are allowed to use."
```

For example:

```text
features/projects/
├── index.ts
├── components/
├── hooks/
├── data/
└── domain/
```

`index.ts`:

```js
export {
  ProjectCard
} from "./components/ProjectCard";

export {
  useProjects
} from "./hooks/useProjects";
```

Internal files remain internal.

This is very similar to designing a component API.

Architecture is basically **API design at the module level**.

---

# 23. State architecture should be explicit

We've learned many state systems:

```text
useState
useReducer
Context
useSyncExternalStore
TanStack Query
URL state
Actions
optimistic state
router state
```

A large application should not blindly mix them.

Each should answer a particular ownership question.

---

# 24. A practical state ownership map

```text
UI-only
→ useState

Complex local transitions
→ useReducer

Subtree-wide configuration
→ Context

URL-driven state
→ search params / route params

Server cache
→ TanStack Query or route data

External shared state
→ external store / useSyncExternalStore

Temporary Action state
→ useActionState

Temporary optimistic display
→ useOptimistic

Router lifecycle
→ router state
```

This is one of the most useful summaries of the entire course.

---

# 25. Don't create Context for everything

A common large-app pattern is:

```text
GlobalContext
```

containing:

```js
{
  user,
  theme,
  settings,
  notifications,
  cart,
  search,
  modal,
  projects
}
```

This becomes a giant shared dependency.

Instead ask:

```text
Does every consumer truly need this?
```

If not, narrow ownership.

Maybe:

```text
ThemeContext
UserContext
```

and:

```text
projects → TanStack Query
search → URL state
modal → local state
```

A good architecture doesn't merely reduce code.

It reduces unnecessary coupling.

---

# 26. Don't create a global store just because Redux exists

The same principle applies to global state libraries.

Ask:

```text
Does this state actually need global ownership?
```

If a state belongs to:

```text
one dialog
```

keep it local.

If it belongs to:

```text
one route
```

maybe route state.

If it belongs to:

```text
server
```

don't duplicate it into a giant global client store without reason.

Global state is powerful.

It is also a coupling mechanism.

---

# 27. Avoid the "everything in Redux/store" problem

Imagine:

```text
store
├── user
├── projects
├── modal
├── form
├── filters
├── server response
├── current tab
└── random flags
```

Now the store becomes the application's second runtime.

Developers begin asking:

```text
"Where should this value actually live?"
```

The answer becomes:

```text
"Probably in the store."
```

That's not architecture.

That's centralization.

Centralization and good ownership are not the same thing.

---

# 28. Route boundaries are useful architectural boundaries

You've now learned React Router deeply enough to use route modules as units.

For example:

```text
routes/
├── dashboard.tsx
├── projects.tsx
├── project.$id.tsx
└── settings.tsx
```

Each route can own:

```text
loader
action
Component
error boundary
pending UI
```

This can make the route a natural application boundary.

Then inside:

```text
projects/
```

you can keep domain logic.

---

# 29. Don't duplicate route and feature ownership blindly

Suppose:

```text
routes/projects.tsx
features/projects/
```

Who owns project data?

You should be able to answer.

Maybe:

```text
route
→ navigation/data lifecycle

feature
→ business/domain/UI logic
```

For example:

```text
route loader
 ↓
feature data function
 ↓
server
```

and:

```text
route component
 ↓
feature components
```

This keeps route concerns and domain concerns distinct.

---

# 30. API access should not be scattered throughout components

Bad:

```jsx
function UserCard() {
  fetch("/users/42");
}

function Sidebar() {
  fetch("/users/42");
}

function Profile() {
  fetch("/users/42");
}
```

Now the same API knowledge is duplicated.

Instead:

```text
features/users/
   data/
      getUser
```

or a query hook:

```js
useUser(userId)
```

Then:

```text
UI
 ↓
feature data abstraction
 ↓
API
```

The UI doesn't need to know endpoint URLs everywhere.

---

# 31. Don't over-abstract your data layer

The opposite extreme is:

```text
Component
 ↓
hook
 ↓
service
 ↓
repository
 ↓
adapter
 ↓
gateway
 ↓
HTTP client
 ↓
fetch
```

to make a simple:

```text
GET /users/42
```

request.

Abstractions have a cost.

A layer is useful when it provides:

```text
meaningful isolation
reusability
testability
security boundary
environment boundary
consistency
```

Otherwise it may just make navigation harder.

---

# 32. Architecture should follow change patterns

This is a powerful way to judge structure.

Imagine billing changes frequently:

```text
tax rules
coupons
discounts
invoices
payment methods
```

You want those changes to mostly stay within:

```text
features/billing
```

If a small billing rule change forces edits in:

```text
20 unrelated directories
```

your boundaries may be wrong.

A good architecture isn't defined by folder aesthetics.

It's defined by **how change propagates through the codebase**.

---

# 33. Coupling

Two modules are coupled when one depends on the implementation/details of the other.

High coupling:

```text
Billing
 ↓
imports 8 internal files from Auth
```

Lower coupling:

```text
Billing
 ↓
Auth public API
```

Coupling isn't automatically bad.

You need dependencies.

The goal is:

> **Make dependencies intentional and stable.**

---

# 34. Cohesion

Cohesion asks:

> Does this module contain things that belong together?

Bad:

```text
utils/
```

contains:

```text
date formatting
password rules
billing math
modal state
API parsing
```

Low cohesion.

Better:

```text
billing/
  pricing
  coupons
  invoice

auth/
  password
  session
  permissions
```

Higher cohesion.

A strong architecture generally aims for:

```text
high cohesion
+
manageable coupling
```

---

# 35. A practical feature structure

Here's one reasonable example:

```text
src/
├── app/
│   ├── router/
│   ├── providers/
│   ├── layout/
│   └── startup/
│
├── features/
│   ├── auth/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── data/
│   │   ├── domain/
│   │   └── index.ts
│   │
│   ├── projects/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── data/
│   │   ├── domain/
│   │   ├── types/
│   │   └── index.ts
│   │
│   └── billing/
│       ├── components/
│       ├── data/
│       ├── domain/
│       └── index.ts
│
└── shared/
    ├── ui/
    ├── lib/
    ├── types/
    └── browser/
```

Again, this isn't a canonical React structure.

It's an example of **ownership-oriented architecture**.

---

# 36. What belongs in `app/`?

Usually application-wide wiring.

For example:

```text
router
providers
root layout
error boundary
global styles
startup
analytics initialization
```

Think:

```text
"What assembles the application?"
```

rather than:

```text
"What feature does this belong to?"
```

---

# 37. What belongs in `features/`?

Things with domain meaning.

```text
auth
projects
billing
reports
notifications
```

A feature can own:

```text
UI
hooks
state
data access
business rules
types
tests
```

as long as those concerns genuinely belong together.

---

# 38. What belongs in `shared/`?

Things with no single feature owner.

Examples:

```text
Button
Modal
Date formatting
generic validation helper
shared type
common browser utility
```

But shared should stay relatively small.

Why?

Because feature ownership becomes the default.

---

# 39. Domain logic vs application orchestration

Another useful distinction.

### Domain logic

```js
calculateInvoiceTotal(...)
canRefund(order)
isSubscriptionActive(subscription)
```

This answers:

```text
"What is true according to the business rules?"
```

### Application orchestration

```js
submitCheckout()
```

might coordinate:

```text
validate
 ↓
call payment API
 ↓
create order
 ↓
refresh cart
 ↓
navigate
```

This answers:

```text
"How do we perform this workflow?"
```

Keeping these separate can make complex features easier to test.

---

# 40. UI should ideally consume domain/application APIs

Imagine:

```jsx
function RefundButton({ order }) {
  const canRefund =
    canRefundOrder(order);

  ...
}
```

The UI asks:

```text
canRefundOrder()
```

instead of embedding:

```js
if (
  order.status === "paid" &&
  order.daysSincePurchase < 30 &&
  !order.isDisputed &&
  ...
)
```

inside the JSX.

Why?

Because now:

```text
business rule
```

is centralized.

The UI expresses:

```text
"show refund button if refund is allowed."
```

That's much easier to reason about.

---

# 41. Don't hide business rules inside generic hooks

Bad:

```js
const {
  canRefund,
  canCancel,
  canEdit,
  canDelete
} = usePermissions(order);
```

when that hook mixes:

```text
network
user state
business rules
UI assumptions
```

A better separation might be:

```text
useCurrentUser()
        ↓
user permissions

canRefundOrder(order, user)
        ↓
domain rule
```

Each piece has a clear responsibility.

---

# 42. Tests should follow architecture

You've already learned React Testing Library.

Architecture affects testing.

### Pure domain function

```js
calculateTotal(...)
```

can use a tiny unit test.

### Feature hook

```text
useProjects()
```

might need a data/query test environment.

### Component

```text
ProjectCard
```

needs user interaction testing.

### Route

```text
project route
```

may deserve an integration test.

### Full workflow

```text
create project
→ redirect
→ project appears
```

may be an E2E test.

Architecture gives you natural testing boundaries.

---

# 43. Avoid testing implementation wiring

Suppose:

```text
ProjectPage
```

calls:

```text
useProjects()
```

and that internally uses:

```text
TanStack Query
```

A component test should usually care:

```text
projects appear
error appears
loading UI appears
```

not:

```text
"useQuery was called once."
```

Your architecture should allow internal implementation to change without rewriting the entire test suite.

That's another sign of a good boundary.

---

# 44. Dependency direction

A very useful rule:

```text
high-level policy
        ↓
lower-level mechanism
```

For example:

```text
Project UI
 ↓
Project domain
 ↓
data access
```

The UI shouldn't need to know:

```text
how database requests are constructed
```

And the database layer shouldn't know:

```text
which React component renders the result
```

That's separation of concerns.

---

# 45. Keep React out of pure domain code when possible

Suppose:

```js
function calculateLateFee(invoice) {
  ...
}
```

There is usually no reason for:

```js
import React from "react";
```

inside it.

Pure domain logic becomes:

```text
framework-independent
```

which makes it easier to:

```text
test
reuse
execute server-side
execute in scripts
reason about
```

This is particularly valuable as your application grows.

---

# 46. Keep server logic out of client modules

With RSC, accidentally crossing this boundary is particularly dangerous.

Imagine:

```js
// sharedProject.js

import db from "./db";

export function getProject() {
  return db.project.find(...);
}
```

and then:

```jsx
"use client";

import {
  getProject
} from "./sharedProject";
```

That is an architectural boundary mistake.

The module isn't truly shared.

Name and structure should make the environment explicit.

---

# 47. A useful folder split for RSC applications

For a feature:

```text
features/projects/
├── server/
│   ├── queries.ts
│   └── mutations.ts
│
├── client/
│   ├── ProjectEditor.tsx
│   └── ProjectFilters.tsx
│
├── shared/
│   ├── types.ts
│   └── validation.ts
│
└── components/
```

Again, the exact names are flexible.

The important idea is:

```text
server-only
client-only
shared
```

are visible in the architecture.

---

# 48. Don't let routes become domain modules

Another common problem:

```text
routes/project.tsx
```

contains:

```text
database calls
business rules
authorization
validation
rendering
forms
```

The route becomes a mini-god module.

Instead:

```text
route
 ↓
feature/server API
 ↓
domain
```

For example:

```js
loader: async ({ params }) => {
  return getProjectPageData(params.id);
}
```

Then:

```text
getProjectPageData()
```

owns the actual data orchestration.

The route describes the route.

---

# 49. Server authorization belongs close to mutation logic

Suppose:

```text
deleteProject()
```

requires authorization.

Don't rely on:

```text
route guard
```

alone.

Don't rely on:

```text
client permission UI
```

either.

The server mutation itself should enforce:

```text
authenticate
validate
authorize
mutate
```

This keeps security close to the protected operation.

---

# 50. Architecture should survive team growth

Imagine:

```text
developer A
→ projects

developer B
→ billing

developer C
→ notifications
```

A feature-oriented architecture lets each person reason mostly within the feature boundary.

But a structure like:

```text
components/
hooks/
services/
utils/
```

forces everyone to navigate shared global directories.

As a team grows, this difference becomes significant.

Architecture is partly about **human scalability**.

---

# 51. Naming is an architectural tool

Consider:

```text
getData()
```

versus:

```text
loadProjectActivity()
```

The second name communicates purpose.

Similarly:

```text
useManager()
```

is vague.

While:

```text
useProjectFilters()
```

has an obvious ownership and purpose.

Good naming reduces the amount of code you have to mentally inspect.

---

# 52. Avoid "manager", "handler", "helper", "service" without meaning

These names often conceal unclear ownership.

For example:

```text
ProjectManager
```

Could mean:

```text
state manager
API client
domain service
UI coordinator
cache manager
```

Instead name by behavior:

```text
ProjectRepository
ProjectService
useProjectFilters
loadProjectActivity
calculateProjectBudget
```

The exact term should reflect the actual responsibility.

---

# 53. Keep APIs narrow

Suppose:

```js
projectService.updateProject(...)
```

accepts:

```js
{
  everything,
  anything,
  optionalFlags,
  randomContext
}
```

That's a weak boundary.

Prefer a clear contract:

```js
updateProject({
  projectId,
  title
});
```

Narrow interfaces make dependencies easier to understand.

---

# 54. Architecture and TypeScript work together

You learned advanced TypeScript.

Now use it architecturally.

Define:

```ts
type Project = {
  id: string;
  name: string;
  status: ProjectStatus;
};
```

and:

```ts
type ProjectStatus =
  | "draft"
  | "active"
  | "archived";
```

Now your domain logic:

```ts
function canArchiveProject(
  project: Project
) {
  return project.status === "active";
}
```

has a clear contract.

Types become part of the architecture.

---

# 55. Runtime validation belongs at boundaries

TypeScript says:

```text
"I expect this data to be Project."
```

The network says:

```text
"here is some runtime data."
```

So API boundaries should validate external input.

Conceptually:

```text
HTTP
 ↓
unknown
 ↓
runtime validation
 ↓
Project
 ↓
domain
```

That boundary should live close to your data layer rather than scattered across components.

---

# 56. Architecture should minimize duplicated interpretation

Suppose five components each independently transform:

```text
API response
```

into:

```text
display model
```

You now have five interpretations.

Instead, centralize the transformation where its ownership is clear:

```text
API response
 ↓
feature data adapter
 ↓
domain model
 ↓
UI
```

This reduces subtle inconsistencies.

---

# 57. But don't create an "enterprise DTO layer" for everything

Again, don't overdo it.

If your API returns:

```json
{
  "id": 42,
  "name": "React"
}
```

and your UI uses:

```js
{
  id: 42,
  name: "React"
}
```

you don't need:

```text
API DTO
 ↓
transport DTO
 ↓
application DTO
 ↓
domain DTO
 ↓
view model
```

unless those transformations solve a real problem.

Architecture is not measured in layers.

It's measured in clarity.

---

# 58. The "change together" heuristic

Here's one of my favorite architecture rules.

> **Things that change together should generally live together.**

Suppose:

```text
billing tax rules
billing UI
billing validation
```

always change together.

Keep them near billing.

Suppose:

```text
generic Button
```

is used by many features and evolves independently.

That belongs in shared UI.

This heuristic is more useful than blindly following folder templates.

---

# 59. The "protect change" heuristic

Another one:

> **Things that change for different reasons should be separated.**

Suppose:

```text
database implementation
```

changes for infrastructure reasons.

While:

```text
checkout UI
```

changes for product/design reasons.

Don't tightly couple them.

The abstraction boundary protects each side from unrelated changes.

---

# 60. Architectural boundaries should correspond to failure boundaries too

This is an advanced but important idea.

Suppose:

```text
Reports
```

fails.

You don't necessarily want:

```text
entire application
```

to fail.

Your boundaries should allow:

```text
Reports error
```

while:

```text
Navigation
Sidebar
Profile
```

remain usable.

This connects directly to:

```text
Error Boundaries
route boundaries
Suspense boundaries
feature boundaries
```

Good architecture often creates good failure isolation naturally.

---

# 61. Architectural boundaries also help performance

Suppose:

```text
billing feature
```

is only used on:

```text
/billing
```

A clean feature boundary makes it easier to:

```text
lazy-load it
route-split it
keep it client-local
keep server-only code server-side
```

So architecture affects:

```text
performance
testability
maintainability
security
```

all at once.

---

# 62. A real application architecture

Let's build one.

```text
src/
│
├── app/
│   ├── router/
│   ├── providers/
│   ├── layout/
│   └── error/
│
├── routes/
│   ├── dashboard/
│   ├── projects/
│   ├── billing/
│   └── settings/
│
├── features/
│   ├── auth/
│   ├── projects/
│   ├── billing/
│   ├── notifications/
│   └── reports/
│
└── shared/
    ├── ui/
    ├── lib/
    ├── validation/
    └── types/
```

Possible dependency direction:

```text
routes
  ↓
features
  ↓
shared
```

with:

```text
features
  ↓
server/data layer
```

depending on the application.

The important thing is to define and enforce your intended direction.

---

# 63. Example feature

```text
features/projects/
│
├── components/
│   ├── ProjectCard.tsx
│   ├── ProjectList.tsx
│   └── ProjectForm.tsx
│
├── hooks/
│   ├── useProjects.ts
│   └── useProjectFilters.ts
│
├── data/
│   ├── queries.ts
│   ├── mutations.ts
│   └── adapters.ts
│
├── domain/
│   ├── permissions.ts
│   ├── projectRules.ts
│   └── calculations.ts
│
├── types/
│   └── project.ts
│
└── index.ts
```

Again:

```text
components
→ UI

hooks
→ React integration

data
→ server communication

domain
→ business rules

types
→ contracts

index
→ public API
```

This is much more meaningful than:

```text
components/
hooks/
utils/
```

spread across the whole application.

---

# 64. Where TanStack Query fits

Inside:

```text
features/projects/data/
```

you might have:

```js
export function useProjects() {
  return useQuery({
    queryKey: ["projects"],
    queryFn: fetchProjects
  });
}
```

Then:

```jsx
function ProjectList() {
  const {
    data: projects
  } = useProjects();

  ...
}
```

The component doesn't need to know:

```text
endpoint
retry
cache
query key
```

unless those are genuinely UI concerns.

---

# 65. Where server functions fit

Inside:

```text
features/projects/server/
```

you might have:

```js
export async function archiveProject(
  projectId
) {
  "use server";

  const user =
    await getCurrentUser();

  assertCanArchive(
    user,
    projectId
  );

  await db.projects.archive(
    projectId
  );
}
```

Then a client feature might invoke it.

Now:

```text
features/projects
```

owns the project-specific server capability.

The route doesn't need to know its implementation.

---

# 66. Where domain logic fits

```js
export function canArchiveProject(
  project
) {
  return (
    project.status === "active"
  );
}
```

That function doesn't care:

```text
React
TanStack Query
WebSocket
Router
DOM
```

It's just a domain rule.

This is a beautiful property.

Your domain logic can sit beneath many different UI mechanisms.

---

# 67. Where browser-only code fits

Suppose:

```text
project filters
```

are stored in:

```text
URL search params
```

The React hook might live in:

```text
features/projects/client/
```

because it requires:

```text
router/browser APIs
```

Now the environment boundary is explicit.

---

# 68. The "thin page" idea

Try to keep routes/pages relatively orchestration-focused.

For example:

```jsx
function ProjectsPage() {
  const projects =
    useLoaderData();

  return (
    <ProjectsScreen
      projects={projects}
    />
  );
}
```

or:

```jsx
function ProjectsPage() {
  return (
    <ProjectsFeature />
  );
}
```

The route shouldn't become the entire application.

Think:

```text
route
→ composition/orchestration

feature
→ actual product behavior
```

Not an absolute rule, but a useful default.

---

# 69. Architecture isn't about hiding every detail

A bad abstraction can make debugging harder.

Suppose:

```text
useProjects()
```

internally calls:

```text
service
 ↓
repository
 ↓
adapter
 ↓
query manager
 ↓
request factory
 ↓
fetcher
```

and nobody knows where a request actually comes from.

Now your abstraction has become a maze.

A good abstraction should make the **important behavior easier to understand**, not merely make direct dependencies disappear.

---

# 70. A healthy application has recognizable paths

For reading:

```text
route
 ↓
feature
 ↓
data
 ↓
server
```

For user interaction:

```text
button
 ↓
feature hook/action
 ↓
mutation
 ↓
server
 ↓
state/cache update
```

For pure rules:

```text
UI
 ↓
domain function
```

For real-time:

```text
socket
 ↓
external store
 ↓
React
```

When you can trace those paths easily, the architecture is doing its job.

---

# 71. Architecture should make ownership obvious

For any value, you should be able to answer:

```text
Who owns this?
```

For any mutation:

```text
Who is allowed to perform it?
```

For any route:

```text
Who loads its data?
```

For any interaction:

```text
Who controls its pending state?
```

For any external source:

```text
Who manages the subscription?
```

For any business rule:

```text
Which domain owns it?
```

That is architecture.

---

# 72. One final end-to-end example

Imagine a project-management application.

User opens:

```text
/projects/42
```

### Router

```text
route loader
 ↓
load project
```

### Feature

```text
Projects feature
 ↓
ProjectScreen
```

### Server

```text
database
 ↓
project
```

### UI

```text
ProjectScreen
├── ProjectHeader
├── TaskList
├── ActivityFeed
└── TaskForm
```

### Task mutation

```text
TaskForm
 ↓
Server Function
 ↓
authorize
 ↓
database
```

### Server state

```text
TanStack Query
 ↓
task cache
```

### Real-time

```text
WebSocket
 ↓
external store
 ↓
ActivityFeed
```

### Optimistic interaction

```text
TaskForm
 ↓
useOptimistic
 ↓
instant task UI
```

### Code splitting

```text
Reports route
 ↓
lazy route chunk
```

### UX

```text
startTransition
 ↓
ViewTransition
```

Now all the topics we've studied have a natural place.

---

# 73. The architecture isn't one giant tree anymore

At this point, a sophisticated application can be visualized as several interacting graphs.

### Component tree

```text
App
 └── Page
      └── Feature
           └── Components
```

### Dependency graph

```text
UI
 ↓
feature
 ↓
domain/data
```

### Server/client graph

```text
server
  │
  └── client boundary
```

### State ownership graph

```text
local
URL
server
external
optimistic
router
```

### Data flow

```text
user
 ↓
action
 ↓
server
 ↓
state
 ↓
UI
```

A senior React developer is constantly reasoning across these graphs.

---

# 74. The five architecture rules I'd keep

### 1. Keep ownership close to the owner

```text
local UI
→ local

route data
→ route

server cache
→ query/data layer

external state
→ external store
```

### 2. Keep feature concerns together

```text
projects
→ project UI + project logic + project data
```

rather than scattering them by technical type.

### 3. Keep environment boundaries explicit

```text
server
client
shared
```

should be easy to distinguish.

### 4. Keep domain rules independent

```text
business rule
→ pure function where possible
```

rather than burying it in React components.

### 5. Make dependency direction intentional

```text
UI
 ↓
feature
 ↓
domain/data
```

rather than arbitrary modules importing each other.

---

# 75. Final mental model

A production React application can be thought of like this:

```text
                         APPLICATION
                              │
              ┌───────────────┼────────────────┐
              │               │                │
           ROUTES          FEATURES         SHARED
              │               │                │
          navigation        product          generic
          data             behavior           pieces
          actions             │
              │        ┌──────┼──────┐
              │        │      │      │
              │       UI    domain   data
              │               │      │
              │               │      ├── Query
              │               │      ├── API
              │               │      └── Server Function
              │               │
              │               └── business rules
              │
              └───────────────┐
                              ▼
                          SERVER/CLIENT
                          BOUNDARIES
                              │
                     ┌────────┴────────┐
                     │                 │
                  SERVER             CLIENT
                     │                 │
                   DB/API        state/events/DOM
                                      │
                                external stores
```

And the most important question in every folder is not:

> "Which type of file is this?"

It is:

> **"Who owns this behavior, who depends on it, and which boundary should protect it?"**

That's the architecture mindset.

---

# What to remember

```text
Feature-oriented architecture
→ organize around product/domain boundaries

Cohesion
→ things that belong together stay together

Coupling
→ dependencies should be intentional

State colocation
→ keep state near where it is used

Domain logic
→ keep business rules independent from UI where possible

Shared code
→ genuinely shared, not a dumping ground

Public feature API
→ expose intentional entry points

Server/client separation
→ make execution boundaries obvious

Route boundary
→ navigation/data lifecycle

Feature boundary
→ product behavior

Dependency direction
→ make the flow of responsibility obvious

Good architecture
→ makes change local, ownership clear,
  failures isolated, and dependencies understandable
```

The deeper lesson is that **React architecture is mostly about ownership and boundaries**. The hooks, router, query library, Actions, Server Components, external stores, and performance tools we've learned all become much easier once those two ideas are clear.

Next, we'll cover another important piece that production applications need: **authentication and authorization in React applications**—sessions, cookies, login/logout flow, protected routes, server-side authorization, permission checks, and why client-side route guards are not security.