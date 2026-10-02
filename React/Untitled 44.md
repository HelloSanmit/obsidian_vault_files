
# Lesson 41 — React 19.3: `<Activity>` and `<ViewTransition>`

We've reached a good point to cover two newer React features that are small in API surface but important conceptually:

```text
<Activity>
<ViewTransition>
```

They build directly on things you've already learned:

```text
state preservation
Suspense
startTransition
useDeferredValue
rendering
DOM behavior
UX/performance
```

As of React 19.3, both **Activity** and **ViewTransition** are part of the current React direction, with `<ViewTransition>` now stable. ([React 19.3 release notes](https://react.dev/blog/2026/09/09/react-19-3) )

These aren't "must use everywhere" features. They're useful when you have UI that is repeatedly hidden/shown or UI whose visual movement should feel connected to a React transition.

---

# 1. Start with the problem `<Activity>` solves

Suppose you have:

```jsx
function App() {
  const [showSidebar, setShowSidebar] =
    useState(true);

  return (
    <>
      <button
        onClick={() =>
          setShowSidebar(
            value => !value
          )
        }
      >
        Toggle sidebar
      </button>

      {showSidebar && <Sidebar />}
    </>
  );
}
```

And `Sidebar` contains:

```jsx
function Sidebar() {
  const [search, setSearch] =
    useState("");

  return (
    <input
      value={search}
      onChange={e =>
        setSearch(e.target.value)
      }
    />
  );
}
```

Suppose the user types:

```text
react architecture
```

Then hides the sidebar.

With ordinary conditional rendering:

```jsx
{showSidebar && <Sidebar />}
```

React **unmounts** the sidebar.

When it is shown again:

```text
Sidebar mounted again
   ↓
useState initial value
   ↓
search = ""
```

The user's text is gone.

---

# 2. Mounting and hiding are different concepts

This distinction is extremely useful.

Normal conditional rendering:

```jsx
{visible && <Component />}
```

means:

```text
visible
  ↓
component exists

hidden
  ↓
component doesn't exist
```

So the component's local state disappears.

But sometimes what you actually want is:

```text
component still exists
but
not currently visible
```

That's what `<Activity>` is designed for.

React describes Activity as a way to make a subtree either `"visible"` or `"hidden"` while preserving its state; hidden content is taken out of the visible UI while its state can remain preserved. ([React Activity reference](https://react.dev/reference/react/Activity))

---

# 3. Basic `<Activity>`

```jsx
import {
  Activity,
  useState
} from "react";

function App() {
  const [showSidebar, setShowSidebar] =
    useState(true);

  return (
    <>
      <button
        onClick={() =>
          setShowSidebar(
            value => !value
          )
        }
      >
        Toggle sidebar
      </button>

      <Activity
        mode={
          showSidebar
            ? "visible"
            : "hidden"
        }
      >
        <Sidebar />
      </Activity>
    </>
  );
}
```

Now the mental model changes from:

```text
visible
 ↓
mount

hidden
 ↓
unmount
```

to:

```text
visible
 ↓
show subtree

hidden
 ↓
keep subtree's state
but hide/deprioritize it
```

---

# 4. Trace the state

Suppose:

```text
search = "react architecture"
```

User clicks Hide.

With ordinary conditional rendering:

```text
Sidebar
 ↓
unmount
 ↓
state destroyed
```

With Activity:

```text
Sidebar
 ↓
Activity → hidden
 ↓
state preserved
```

User clicks Show again:

```text
Activity → visible
 ↓
Sidebar reappears
 ↓
search still =
"react architecture"
```

This is the most important feature of Activity.

---

# 5. Activity is not just CSS `display: none`

You might think:

```jsx
<div style={{ display: show ? "block" : "none" }}>
```

Why not simply do that?

Because Activity is tied to React's rendering model.

React can treat hidden content as an **inactive piece of UI**, rather than simply leaving a fully visible DOM subtree around.

The current Activity API is specifically designed to preserve state while hiding a subtree and to make hidden work lower priority relative to visible UI. ([React Activity reference](https://react.dev/reference/react/Activity))

So Activity is closer to:

```text
"this UI still belongs to the application,
but it isn't currently active."
```

rather than merely:

```text
"hide these pixels."
```

---

# 6. Activity and effects

There's another important behavioral difference.

Imagine:

```jsx
function Panel() {
  useEffect(() => {
    console.log("connect");

    return () =>
      console.log("disconnect");
  }, []);

  return <div>Panel</div>;
}
```

With unmounting:

```text
hide
 ↓
unmount
 ↓
cleanup effect
```

show:

```text
mount
 ↓
effect runs
```

With Activity, the subtree remains mounted but becomes hidden.

React can clean up certain Effects while hidden and re-run them when the subtree becomes visible again. The important idea is that **state can persist even though Effects are not necessarily kept actively running while the subtree is hidden**. ([React Activity reference](https://react.dev/reference/react/Activity))

That gives you a useful distinction:

```text
state
→ preserve

active external synchronization
→ can pause while hidden
```

This is a very interesting design.

---

# 7. Why would you want that?

Imagine tabs:

```text
Profile
Settings
Notifications
```

Typical approach:

```jsx
{tab === "profile" && <Profile />}
{tab === "settings" && <Settings />}
{tab === "notifications" && <Notifications />}
```

Switching tabs:

```text
Profile
 ↓
unmount

Settings
 ↓
mount
```

Then:

```text
Settings
 ↓
unmount

Notifications
 ↓
mount
```

If each tab contains rich state:

```text
form values
scroll position
expanded sections
drafts
filters
```

you may not want them to reset every time.

Activity gives you:

```text
Tab A
 └── Activity hidden

Tab B
 └── Activity visible
```

so the application can preserve the tab's state.

---

# 8. Activity is particularly interesting with routing

Imagine:

```text
Dashboard
   │
   ├── Overview
   ├── Analytics
   └── Settings
```

You might want a navigation system where:

```text
/overview
```

becomes:

```text
Overview visible
Analytics hidden
Settings hidden
```

Rather than destroying and rebuilding every screen.

This can provide:

```text
state preservation
faster return to recently visited UI
less remounting
```

The tradeoff is that you're retaining those subtrees rather than throwing them away.

So memory usage matters.

---

# 9. Activity is not free

Suppose you preserve:

```text
10 huge dashboards
```

all at once.

Now you're keeping more application state around.

So:

```text
unmount
→ lower memory
→ lose local state

Activity hidden
→ preserve local state
→ potentially more memory/work
```

The correct decision depends on the UI.

That's an architecture tradeoff, not a magic performance optimization.

---

# 10. Now let's move to `<ViewTransition>`

Activity solves:

```text
"I want hidden UI to retain its state."
```

ViewTransition solves:

```text
"I want React updates to have a meaningful visual transition."
```

It integrates React's Transition system with the browser's native **View Transition API**.

React 19.3 made `<ViewTransition>` stable. ([React 19.3 release notes](https://react.dev/blog/2026/09/09/react-19-3))

---

# 11. Basic ViewTransition

```jsx
import {
  ViewTransition,
  useState,
  startTransition
} from "react";

function App() {
  const [show, setShow] =
    useState(false);

  return (
    <>
      <button
        onClick={() => {
          startTransition(() => {
            setShow(value => !value);
          });
        }}
      >
        Toggle
      </button>

      {show && (
        <ViewTransition>
          <div className="card">
            Hello
          </div>
        </ViewTransition>
      )}
    </>
  );
}
```

Now React can animate the UI change.

By default, the browser's View Transition mechanism provides a cross-fade style transition unless you customize it. ([React 19.3 release notes](https://react.dev/blog/2026/09/09/react-19-3))

---

# 12. Why does `startTransition` matter here?

This is one of the coolest connections to our earlier lessons.

We previously learned:

```js
startTransition(() => {
  setSomething(...);
});
```

means:

```text
"This update is non-urgent."
```

Now React can also use that Transition as an opportunity to animate the UI.

So:

```text
startTransition
       ↓
React transition
       ↓
ViewTransition can animate
```

A normal urgent update doesn't automatically activate the ViewTransition boundary. React's current docs specifically note that `<ViewTransition>` responds to updates marked as Transitions, Suspense reveals, and `useDeferredValue` updates. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 13. The relationship is elegant

You've learned:

```text
startTransition
→ scheduling / priority

ViewTransition
→ visual animation of the transition
```

So one concept controls:

```text
when/how React renders the transition
```

and the other controls:

```text
how the visual result moves between states
```

That's a very coherent API design.

---

# 14. Enter animation

Suppose:

```jsx
{show && (
  <ViewTransition>
    <Card />
  </ViewTransition>
)}
```

Initial state:

```text
Card doesn't exist
```

Then:

```text
startTransition
 ↓
show = true
 ↓
Card enters tree
```

ViewTransition can identify that as an:

```text
enter
```

transition.

---

# 15. Exit animation

Now:

```text
show = true
```

User closes it:

```js
startTransition(() => {
  setShow(false);
});
```

The component is removed.

React can identify:

```text
exit
```

and animate that disappearance.

This is important because ordinary React mounting/unmounting is instantaneous from the application's point of view.

ViewTransition gives the browser a visual representation of that state change.

---

# 16. Update animation

Suppose:

```jsx
<ViewTransition>
  <div className={expanded ? "large" : "small"}>
    ...
  </div>
</ViewTransition>
```

State changes:

```text
small
 ↓
large
```

That's not enter or exit.

The component remains.

Its content/style changes.

React identifies this as an:

```text
update
```

transition.

So ViewTransition distinguishes:

```text
enter
exit
update
```

and also supports:

```text
share
```

for shared elements. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 17. Shared element transitions

This is where the feature gets really interesting.

Imagine a product list:

```text
┌──────────────┐
│  Product A   │
│   [image]    │
└──────────────┘
```

User clicks it.

Now detail page:

```text
┌──────────────────────┐
│       Product A      │
│       [BIG IMAGE]    │
└──────────────────────┘
```

You want the image to visually appear to move from:

```text
small card
```

to:

```text
large detail image
```

This can be represented using a named ViewTransition.

Conceptually:

```jsx
<ViewTransition name="product-image">
  <img ... />
</ViewTransition>
```

on the old screen and:

```jsx
<ViewTransition name="product-image">
  <img ... />
</ViewTransition>
```

on the new screen.

React can interpret matching names across the transition as a shared-element transition. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 18. Why the name matters

React normally generates unique View Transition names automatically.

But if you explicitly use:

```jsx
<ViewTransition name="product-image">
```

you're telling React:

```text
"This visual thing corresponds
to the same conceptual element
across these states."
```

So:

```text
old product image
      │
      │ same transition name
      ▼
new product image
```

can animate as one conceptual object.

This is very similar to how you learned keys, but do not confuse the two.

---

# 19. `key` vs `ViewTransition name`

This is an excellent distinction.

### `key`

React uses:

```text
identity
```

for reconciliation and state preservation.

### `ViewTransition name`

React/browser uses:

```text
visual correspondence
```

for animation.

So:

```text
key
→ "Which component/entity is this?"

ViewTransition name
→ "Which visual thing should animate between states?"
```

Different concepts.

---

# 20. ViewTransition isn't a replacement for CSS animations

React's own guidance is clear here: View Transitions are intended for transitions between UI states such as navigation, expansion, opening, and reordering; they aren't meant to replace ordinary CSS/JS animations for every element. ([React Labs / View Transitions](https://react.dev/blog/2025/04/23/react-labs-view-transitions-activity-and-more))

For example, a continuously spinning loading indicator:

```css
.spinner {
  animation: spin 1s linear infinite;
}
```

is still a perfectly normal CSS animation.

You don't need:

```text
<ViewTransition>
```

for everything.

---

# 21. Think of ViewTransition as "state-change animation"

A useful distinction is:

### CSS animation

```text
"This element should continuously animate."
```

### ViewTransition

```text
"The UI changed from state A to state B.
Animate the visual transition."
```

So:

```text
A
 ↓
B
```

is where ViewTransition shines.

---

# 22. Navigation is a perfect use case

Suppose:

```text
/home
 ↓
/details/42
```

Without a transition:

```text
Home disappears
Details appears
```

With a transition-aware router and ViewTransition:

```text
Home
   ↘
    visual transition
   ↗
Details
```

React Router can integrate navigations with React Transitions, which makes `<ViewTransition>` a natural fit for route changes. The React team's current View Transition guidance specifically describes wrapping route content so navigation transitions can animate between pages. ([React Labs](https://react.dev/blog/2025/04/23/react-labs-view-transitions-and-more?utm_source=chatgpt.com))

---

# 23. A simple navigation example

Conceptually:

```jsx
<ViewTransition>
  <Page />
</ViewTransition>
```

Then the router changes:

```text
Page A
 ↓
transition
 ↓
Page B
```

Because navigation is typically treated as a Transition, the ViewTransition boundary can animate that update.

This is one reason your earlier lessons on:

```text
Router
+
startTransition
+
Suspense
```

were not isolated concepts.

They're converging.

---

# 24. Suspense + ViewTransition

This is another elegant combination.

Suppose:

```jsx
<ViewTransition>
  <Suspense fallback={<Skeleton />}>
    <ProductDetails />
  </Suspense>
</ViewTransition>
```

Navigation happens.

Product data isn't ready.

Suspense shows:

```text
Skeleton
```

Then the product resolves.

ViewTransition can animate the revealed UI as part of that Transition. React's current ViewTransition behavior explicitly includes Suspense reveals as animation triggers. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

So:

```text
navigation
   ↓
Suspense
   ↓
skeleton
   ↓
data ready
   ↓
ViewTransition
   ↓
content reveal
```

This can produce much more coherent navigation UX.

---

# 25. ViewTransition + `useDeferredValue`

You learned:

```js
const deferred = useDeferredValue(value);
```

Suppose:

```text
urgent value
 ↓
deferred result
```

When the deferred content catches up, ViewTransition can animate that update because deferred updates are among the transitions that can activate a ViewTransition boundary. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

That means:

```text
urgent state
   ↓
deferred UI catches up
   ↓
smooth visual transition
```

Again, scheduling and visual presentation are separate layers.

---

# 26. ViewTransition has four conceptual animation categories

Keep these:

```text
ENTER
```

Something appears.

```text
EXIT
```

Something disappears.

```text
UPDATE
```

Existing UI changes.

```text
SHARE
```

A visual element corresponds between old and new locations.

React determines which category applies based on how the tree changes. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 27. Customizing animations

By default:

```text
smooth cross-fade
```

But you can customize.

For example:

```jsx
<ViewTransition
  enter="slide-in"
  exit="slide-out"
>
  <Panel />
</ViewTransition>
```

Then CSS can define the animation behavior for the corresponding View Transition classes.

React 19.3's stable API supports View Transition Classes and event callbacks such as:

```text
onEnter
onExit
onShare
onUpdate
```

for more advanced animation control. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 28. You don't normally call `document.startViewTransition()` yourself

This is another subtle but important point.

The browser provides:

```js
document.startViewTransition(...)
```

But when using React's `<ViewTransition>`, React coordinates this internally.

You generally shouldn't manually start a browser View Transition for the same React update.

The point of React's API is:

```text
React knows
when the UI changes
what boundary participates
and when the transition should occur
```

React's current implementation itself coordinates the underlying browser API. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 29. ViewTransition creates a visual snapshot

Here's the deeper browser concept.

The browser's View Transition system can capture a visual representation of the old/new states.

Conceptually:

```text
OLD UI
   ↓
snapshot

NEW UI
   ↓
snapshot

animate
   ↓
OLD → NEW
```

That's why it can create effects that would otherwise be cumbersome to implement manually.

You aren't necessarily animating every individual DOM node.

You're giving the browser the ability to transition between visual states.

React then maps its component/transition model onto that mechanism.

---

# 30. Why this can look smoother than manual layout animation

Suppose a card moves:

```text
position A
      ↓
position B
```

A manual system might animate:

```text
top
left
width
height
```

for many individual elements.

View Transitions can instead operate on a visual representation of the UI state change.

This can produce smoother transitions in cases where animating the full DOM structure manually would be cumbersome. The React ViewTransition guidance specifically describes this as animating the view as an image-like representation rather than independently animating every internal element. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 31. Accessibility: reduced motion

This is important.

Not everyone wants:

```text
large visual transitions
```

React's current guidance specifically recommends respecting:

```css
@media (prefers-reduced-motion: reduce)
```

and reducing or disabling ViewTransition animations where appropriate. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

For example:

```css
@media (prefers-reduced-motion: reduce) {
  /* use minimal or no transition */
}
```

This connects directly to our accessibility lesson.

Animation is part of accessibility.

---

# 32. Now combine Activity + ViewTransition

This is where the APIs become very interesting together.

Suppose:

```jsx
<Activity
  mode={show ? "visible" : "hidden"}
>
  <ViewTransition
    enter="slide-in"
    exit="slide-out"
  >
    <Sidebar />
  </ViewTransition>
</Activity>
```

When shown:

```text
hidden
 ↓
visible
 ↓
enter animation
```

When hidden:

```text
visible
 ↓
hidden
 ↓
exit animation
```

And the sidebar's state can remain preserved.

So you get:

```text
state preservation
+
visual transitions
```

That's a powerful combination.

React's current ViewTransition documentation specifically demonstrates Activity + ViewTransition for showing/hiding stateful content while preserving the contained state. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

---

# 33. Example: persistent sidebar

Imagine:

```text
┌───────────┬───────────────────┐
│ Sidebar   │ Main content      │
│           │                   │
│ search:   │                   │
│ React     │                   │
└───────────┴───────────────────┘
```

User collapses sidebar.

Instead of:

```text
remove sidebar
```

you can have:

```text
Activity hidden
+
ViewTransition exit
```

Then reopen:

```text
Activity visible
+
ViewTransition enter
```

and:

```text
search = "React"
```

is still preserved.

That is a genuinely useful UI behavior.

---

# 34. Activity and state preservation vs keys

You should connect this to Lesson 13.

Ordinary React state preservation depends on identity:

```text
same component
+
same position/key
→ state survives
```

But when you conditionally unmount:

```jsx
{show && <Panel />}
```

the component disappears entirely.

Activity gives you a way to keep the subtree's logical existence around while changing its visibility/activity state.

So the conceptual difference is:

```text
conditional rendering
→ existence changes

Activity
→ visibility/activity changes
```

That is why state can survive.

---

# 35. When Activity is a great fit

Think:

```text
tabbed interfaces
persistent sidebars
recently visited panels
multi-step UI
complex forms
navigation caches
```

especially where preserving:

```text
draft
scroll
expanded sections
local state
```

is valuable.

---

# 36. When not to use Activity

If the subtree is:

```text
huge
rarely revisited
memory-heavy
expensive to keep around
```

ordinary unmounting may be preferable.

For example:

```text
massive analytics dashboard
```

that users visit once every 20 minutes may not justify retaining every internal stateful component in memory.

Again:

```text
preserved state
↔
memory cost
```

Tradeoff.

---

# 37. ViewTransition vs CSS transition

Consider a simple hover:

```css
button {
  transition: transform 150ms;
}

button:hover {
  transform: scale(1.05);
}
```

Use CSS.

Why?

Because this is:

```text
continuous/local element animation
```

ViewTransition is more appropriate for:

```text
list item inserted
route changed
panel expanded
element moved
UI switched
```

where the **state transition itself** is what you want to animate.

---

# 38. ViewTransition vs React animation libraries

You may also encounter:

```text
Framer Motion / Motion
GSAP
React Spring
```

These are still relevant.

ViewTransition is not intended to replace every animation library.

For example:

```text
complex gesture-driven animation
physics
timeline animation
scroll-linked animation
```

may still be better handled by dedicated animation tools or native browser APIs.

ViewTransition is particularly elegant for:

```text
old UI state
      ↓
new UI state
```

---

# 39. ViewTransition and React state

Let's see a complete simple example.

```jsx
import {
  ViewTransition,
  startTransition,
  useState
} from "react";

function App() {
  const [expanded, setExpanded] =
    useState(false);

  function toggle() {
    startTransition(() => {
      setExpanded(
        value => !value
      );
    });
  }

  return (
    <button onClick={toggle}>
      <ViewTransition>
        <div
          className={
            expanded
              ? "card expanded"
              : "card"
          }
        >
          Product
        </div>
      </ViewTransition>
    </button>
  );
}
```

The state transition:

```text
collapsed
 ↓
expanded
```

becomes:

```text
React Transition
 ↓
ViewTransition
 ↓
visual interpolation
```

Without this concept, you'd have to manually think:

```text
measure old layout
measure new layout
capture old state
animate
restore
```

View Transition infrastructure handles much of that complexity.

---

# 40. Important pitfall: only the right DOM boundaries animate

ViewTransition operates around the nearest DOM structure inside its boundary.

Its placement matters.

For enter/exit transitions especially, React's current docs note that the `ViewTransition` needs to be appropriately positioned in the rendered tree; putting unrelated DOM nodes before it can prevent the intended enter/exit behavior. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

So don't think:

```text
<ViewTransition>
```

is an invisible wrapper with unlimited semantic reach.

It defines a specific visual boundary.

---

# 41. Another pitfall: don't reuse names carelessly

Suppose:

```jsx
<ViewTransition name="item">
```

appears in five places.

Now you've potentially told the browser:

```text
all these visuals are "item"
```

That's ambiguous.

For shared transitions, names should be globally unique within the relevant transition context. React's current guidance specifically warns about name collisions. ([React ViewTransition reference](https://react.dev/reference/react/ViewTransition))

So:

```text
product-image-42
```

is much safer than:

```text
item
```

when many items are present.

---

# 42. Activity + ViewTransition + Router

Now imagine:

```text
/products
      ↓
/products/42
```

Architecture:

```text
Router
  ↓
navigation Transition
  ↓
ViewTransition
  ↓
new route
```

Meanwhile a product page could contain:

```text
Activity
  ↓
preserve a sidebar/filter state
```

and:

```text
Suspense
  ↓
load reviews
```

Now we're combining:

```text
Router
+
Transition
+
ViewTransition
+
Activity
+
Suspense
```

These features each have a separate responsibility.

That's the key.

---

# 43. The responsibilities

### `startTransition`

```text
How should this update be scheduled?
```

### `<ViewTransition>`

```text
How should this state change look visually?
```

### `<Activity>`

```text
Should this subtree remain around
while temporarily inactive?
```

### `<Suspense>`

```text
What should happen while required work
isn't ready?
```

Once you see those as separate questions, the APIs stop feeling redundant.

---

# 44. Activity and performance

Activity has an interesting performance implication.

Suppose you have:

```text
visible UI
+
hidden UI
```

The hidden subtree can be deprioritized so the visible application gets more attention.

This fits the broader React philosophy we've been learning:

```text
not all UI has the same priority
```

The application can retain:

```text
state
```

without treating hidden work as equally important as visible work.

That is one of the deeper reasons Activity exists.

---

# 45. Activity and Suspense

Imagine a hidden Activity contains something expensive:

```jsx
<Activity mode="hidden">
  <Suspense fallback={<Skeleton />}>
    <HeavyPanel />
  </Suspense>
</Activity>
```

This gives React a structure for:

```text
hidden UI
+
work that may not yet be ready
```

As it becomes relevant, React can progressively bring it into the visible application.

This is part of a larger direction toward treating UI as **active/inactive work with different priorities**, rather than everything being equally live all the time.

---

# 46. The bigger picture of React scheduling

At this point, we've accumulated a very useful model.

React can distinguish:

```text
urgent
```

from:

```text
transition
```

and now:

```text
visible Activity
```

versus:

```text
hidden Activity
```

So the UI is no longer simply:

```text
mounted
or
unmounted
```

It can increasingly be thought of as:

```text
active
inactive
pending
suspended
transitioning
```

That is a much richer rendering model.

---

# 47. A practical example: settings tabs

Let's make a realistic design.

```jsx
function Settings() {
  const [tab, setTab] =
    useState("profile");

  return (
    <>
      <TabButtons
        value={tab}
        onChange={next =>
          startTransition(() =>
            setTab(next)
          )
        }
      />

      <Activity
        mode={
          tab === "profile"
            ? "visible"
            : "hidden"
        }
      >
        <ViewTransition>
          <ProfileSettings />
        </ViewTransition>
      </Activity>

      <Activity
        mode={
          tab === "security"
            ? "visible"
            : "hidden"
        }
      >
        <ViewTransition>
          <SecuritySettings />
        </ViewTransition>
      </Activity>
    </>
  );
}
```

Now:

```text
Profile
 ↓
type draft
 ↓
Security
 ↓
return Profile
```

and:

```text
Profile draft is preserved
```

while the switch can animate.

That's a concrete use case for both APIs.

---

# 48. Another example: master/detail

Imagine:

```text
Inbox
 ├── Message A
 ├── Message B
 └── Message C
```

Click Message B:

```text
list
 ↓
detail
```

ViewTransition can animate the transition between:

```text
message preview
```

and:

```text
full message
```

while Activity could preserve:

```text
inbox filters
selection
draft state
```

when the detail view becomes active.

That's exactly the kind of application interaction these primitives are good at.

---

# 49. Another subtle detail: ViewTransition only animates Transition-driven updates

This:

```jsx
setOpen(true);
```

is an ordinary synchronous update.

A `<ViewTransition>` surrounding the changed content won't necessarily animate it automatically.

But:

```jsx
startTransition(() => {
  setOpen(true);
});
```

marks it as a Transition.

Then the ViewTransition can participate.

This is another reason not to view:

```text
<ViewTransition>
```

as a generic CSS wrapper.

It is tied to React's scheduling model.

---

# 50. Why this design is interesting

Traditional UI animation often looks like:

```text
state change
 ↓
animation library notices
 ↓
manipulates DOM/styles
```

React's direction is more like:

```text
state change
 ↓
React knows this is a Transition
 ↓
ViewTransition participates
 ↓
browser animates the resulting visual change
```

The animation becomes part of the rendering model.

That's conceptually cleaner.

---

# 51. Current limitation

`<ViewTransition>` is currently a DOM-oriented feature. React 19.3's stable API does not yet apply to React Native or other renderers. ([React 19.3 release notes](https://react.dev/blog/2026/09/09/react-19-3))

So:

```text
React DOM
✅

React Native
not currently supported
```

This is worth knowing because React itself is multi-platform.

---

# 52. Mental model: ordinary conditional rendering vs Activity

Keep this comparison.

### Conditional rendering

```jsx
{show && <Panel />}
```

means:

```text
show
→ panel exists

hide
→ panel removed
```

### Activity

```jsx
<Activity mode={show ? "visible" : "hidden"}>
  <Panel />
</Activity>
```

means:

```text
show
→ panel active/visible

hide
→ panel inactive/hidden
  state can persist
```

---

# 53. Mental model: CSS animation vs ViewTransition

### CSS

```text
animate properties over time
```

### ViewTransition

```text
animate visual relationship
between old and new UI states
```

---

# 54. Mental model: `startTransition` vs ViewTransition

### `startTransition`

```text
scheduling
```

### `ViewTransition`

```text
visual presentation
```

So:

```text
startTransition
      +
ViewTransition
```

means:

```text
non-urgent UI transition
+
animate the visual transition
```

---

# 55. Where Activity fits in our entire course

Earlier:

```text
state preservation
→ keys/identity
```

Now:

```text
state preservation
→ identity
+
Activity for intentionally inactive subtrees
```

Earlier:

```text
Suspense
→ not ready
```

Now:

```text
Suspense
→ not ready

Activity
→ not active
```

Earlier:

```text
Transition
→ scheduling
```

Now:

```text
Transition
→ scheduling

ViewTransition
→ animation of the transition
```

That is why these features make sense now rather than earlier in the course.

---

# 56. One complete picture

```text
                         REACT UI

                  ┌───────────────────┐
                  │                   │
               visible             hidden
                  │                   │
                  ▼                   ▼
              Activity            Activity
                  │                   │
                  ▼                   │
             render work             │
                  │                   │
         ┌────────┴─────────┐         │
         │                  │         │
      ready             suspended     │
         │                  │         │
         ▼                  ▼         │
        UI               Suspense     │
         │                  │         │
         └────────┬─────────┘         │
                  ▼                   │
            React Transition          │
                  │                   │
                  ▼                   │
            ViewTransition            │
                  │                   │
                  ▼                   │
             visual change            │
                                      │
                         state can persist
```

This is the key insight:

> React is increasingly modeling UI not just as "mounted or unmounted", but as work with different **visibility, priority, readiness, and visual-transition states**.

---

# 57. What to remember from this lesson

```text
<Activity>
→ keep a subtree's state while making it inactive/hidden

<Activity mode="visible">
→ active visible UI

<Activity mode="hidden">
→ hidden/inactive UI whose state can remain preserved

<ViewTransition>
→ animate React state transitions using the browser View Transition API

startTransition
→ marks non-urgent work

ViewTransition
→ gives that transition a visual animation

enter
→ something appears

exit
→ something disappears

update
→ existing UI changes

share
→ same conceptual visual element moves between states
```

And the most useful distinction:

```text
Unmounting
→ destroy this UI instance

Activity hidden
→ keep this UI instance, but make it inactive

ViewTransition
→ animate the visual change between states
```

That completes the important React 19.3 rendering/interaction primitives.

The next topic can move into **real-time React applications: WebSockets, Server-Sent Events, external stores, reconnection, optimistic updates, and how to prevent real-time streams from causing render storms**.