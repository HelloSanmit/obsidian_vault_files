
# Lesson 25 — Portals, Event Propagation, and Modals

This topic is deceptively small in API surface but very important for understanding that **the React tree and the browser DOM tree are not always the same thing**.

This matters for things like:

```text
modals
dialogs
dropdowns
tooltips
context menus
popover UI
toast systems
```

Let's build the idea from scratch.

---

# 1. The problem portals solve

Suppose you have:

```jsx
function ProductPage() {
  return (
    <div className="page">
      <ProductCard />
    </div>
  );
}
```

Inside the card:

```jsx
function ProductCard() {
  const [open, setOpen] = useState(false);

  return (
    <div className="card">
      <button onClick={() => setOpen(true)}>
        Delete
      </button>

      {open && <DeleteDialog />}
    </div>
  );
}
```

Structurally, your React tree is:

```text
ProductPage
└── ProductCard
    ├── button
    └── DeleteDialog
```

That seems fine.

But the browser DOM might look like:

```html
<body>
  <div id="root">
    <div class="page">
      <div class="card">
        <button>Delete</button>

        <div class="dialog">
          ...
        </div>
      </div>
    </div>
  </div>
</body>
```

Now imagine the dialog needs:

```text
position: fixed
z-index: 9999
```

and lives inside a deeply nested element with:

```css
overflow: hidden;
transform: ...;
position: relative;
```

Suddenly you can get clipping, stacking-context problems, awkward positioning, and layout constraints.

The UI logically belongs to:

```text
ProductCard
```

but physically, it may be much easier to place its DOM somewhere closer to:

```html
<body>
```

That's what a portal gives you.

---

# 2. A portal changes the DOM location, not the React ownership

The basic API is:

```jsx
createPortal(children, domNode)
```

For example:

```jsx
import { createPortal } from "react-dom";

function Modal({ children }) {
  return createPortal(
    <div className="modal">
      {children}
    </div>,
    document.body
  );
}
```

Now imagine:

```jsx
function ProductCard() {
  return (
    <Modal>
      Delete this product?
    </Modal>
  );
}
```

The React relationship still looks like:

```text
ProductCard
└── Modal
    └── dialog content
```

But the DOM can look more like:

```html
<body>
  <div id="root">
    ...
    <div class="card">
      <button>Delete</button>
    </div>
  </div>

  <div class="modal">
    Delete this product?
  </div>
</body>
```

That is the key idea:

> **A portal changes where the JSX is mounted in the DOM, not where it lives in the React tree.**

Current React provides `createPortal` specifically for rendering children into a different DOM node while preserving their place in the React tree.

---

# 3. React tree vs DOM tree

This distinction is worth drawing carefully.

Without a portal:

```text
React tree             DOM tree

App                     body
└── Card                └── root
    └── Dialog              └── Card
                              └── Dialog
```

With a portal:

```text
React tree             DOM tree

App                     body
└── Card                ├── root
    └── Dialog          │   └── Card
                        │
                        └── Dialog
```

So:

```text
React relationship:
Card → Dialog

DOM relationship:
root → Card

body → Dialog
```

This is why portals are sometimes called a way to **escape a DOM hierarchy without escaping React's hierarchy**.

---

# 4. Why this is useful for modals

A modal often wants to behave like:

```text
full-screen layer
    ↓
above application UI
```

Putting it near the document root makes it easier to manage:

```css
.modal {
  position: fixed;
  inset: 0;
}
```

and:

```css
.modal-backdrop {
  position: fixed;
  inset: 0;
}
```

Instead of fighting with the layout ancestors of the component that happened to open the modal.

The portal therefore solves a **DOM layout concern**, not a state-management concern.

Your modal state can still belong exactly where it did before:

```jsx
const [open, setOpen] = useState(false);
```

---

# 5. A complete basic modal

```jsx
import { createPortal } from "react-dom";

function Modal({ open, onClose, children }) {
  if (!open) {
    return null;
  }

  return createPortal(
    <div className="backdrop" onClick={onClose}>
      <div
        className="dialog"
        onClick={e => e.stopPropagation()}
      >
        {children}
      </div>
    </div>,
    document.body
  );
}
```

Then:

```jsx
function ProductCard() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(true)}>
        Delete
      </button>

      <Modal
        open={open}
        onClose={() => setOpen(false)}
      >
        <h2>Delete product?</h2>
        <button onClick={() => setOpen(false)}>
          Cancel
        </button>
      </Modal>
    </>
  );
}
```

There are two important concepts hidden here:

```text
portal
+
event propagation
```

The portal is only half the story.

---

# 6. React events follow the React tree

This is where portals become especially interesting.

Suppose:

```jsx
function Parent() {
  function handleClick() {
    console.log("Parent clicked");
  }

  return (
    <div onClick={handleClick}>
      <Modal>
        <button>Inside modal</button>
      </Modal>
    </div>
  );
}
```

The button may physically be somewhere under:

```html
<body>
  ...
  <button>Inside modal</button>
</body>
```

rather than inside the parent's DOM element.

Yet clicking it can still bubble through the **React tree** to the parent.

Conceptually:

```text
React tree:

Parent
└── Modal
    └── Button
```

Click:

```text
Button
 ↓
React event propagation
 ↓
Modal
 ↓
Parent
```

even if the DOM nesting is different.

That is one of the most important properties of portals.

Current React's portal behavior preserves event propagation according to the React tree rather than the portal's physical DOM location.

---

# 7. Why this matters

Suppose you have:

```jsx
<div onClick={() => console.log("page")}>
  <Modal>
    <button>Save</button>
  </Modal>
</div>
```

A click on the modal button can still reach the outer `div`'s React event handler.

That can create surprising bugs.

For example:

```text
click inside modal
       ↓
modal button handler runs
       ↓
event bubbles
       ↓
page click handler runs
       ↓
something else happens unexpectedly
```

So portals don't create an isolated event universe.

---

# 8. `stopPropagation()`

That's why you often see:

```jsx
<div
  className="dialog"
  onClick={e => e.stopPropagation()}
>
```

This prevents the click from continuing through the propagation path.

For our modal:

```jsx
<div className="backdrop" onClick={onClose}>
  <div
    className="dialog"
    onClick={e => e.stopPropagation()}
  >
    ...
  </div>
</div>
```

The intended interaction is:

```text
click backdrop
   ↓
onClose()

click dialog
   ↓
stopPropagation()
   ↓
don't close
```

This is a very common modal pattern.

---

# 9. Event propagation itself deserves a mental model

When you click:

```html
<button>
```

the browser's event system has propagation phases.

At a high level:

```text
capturing
    ↓
target
    ↓
bubbling
```

In React you'll most commonly work with:

```jsx
onClick
```

for bubbling behavior.

And:

```jsx
onClickCapture
```

for the capture phase.

For now, the practical mental model is:

```text
parent
  ↓
child
  ↓
target

then on the way back:

target
  ↓
child
  ↓
parent
```

So bubbling is the upward part.

---

# 10. Why event delegation matters

React doesn't necessarily attach an independent native event listener directly to every JSX element.

React has its own event system and uses event delegation strategies to handle many events efficiently.

You don't need to memorize the implementation details.

The important point is:

> **The event handler you write is part of React's event model, not simply a raw DOM listener copied one-for-one onto every element.**

That's one reason React can preserve event relationships across portals.

---

# 11. Portal does not mean “new React app”

Suppose:

```jsx
createPortal(
  <Dialog />,
  document.body
)
```

It does **not** mean:

```text
create a completely independent React application
```

The dialog still has access to:

```text
props
state
context
hooks
event relationships
```

from its React ancestry.

This is crucial.

For example:

```jsx
<ThemeContext value="dark">
  <Page>
    <Modal>
      ...
    </Modal>
  </Page>
</ThemeContext>
```

Even if the modal DOM gets mounted under `document.body`, the modal is still inside the React provider's tree.

So:

```jsx
const theme = useContext(ThemeContext);
```

can still receive:

```text
"dark"
```

The portal moved the DOM node.

It did not move the component out of its React ownership/context structure.

---

# 12. Portal + Context

Let's make that concrete.

```jsx
const ThemeContext = createContext("light");

function App() {
  return (
    <ThemeContext value="dark">
      <Page />
    </ThemeContext>
  );
}

function Page() {
  return <Modal />;
}

function Modal() {
  const theme = useContext(ThemeContext);

  return createPortal(
    <div className={theme}>
      Dark modal
    </div>,
    document.body
  );
}
```

The modal still receives:

```text
ThemeContext = dark
```

even though its DOM isn't inside the provider's DOM subtree.

That's a very strong demonstration of:

```text
React tree ≠ DOM tree
```

---

# 13. Portal + state

Same principle.

Suppose:

```jsx
function Parent() {
  const [count, setCount] = useState(0);

  return (
    <>
      <button onClick={() => setCount(count + 1)}>
        {count}
      </button>

      {createPortal(
        <p>{count}</p>,
        document.body
      )}
    </>
  );
}
```

The portal content still reads:

```text
count
```

from the same component render/state model.

When state changes:

```text
count
 ↓
Parent renders
 ↓
portal content gets updated
```

Again, its physical DOM location is not its React ownership boundary.

---

# 14. Portal + custom hooks

Nothing special is required.

A component rendered through a portal can use:

```jsx
useState()
useEffect()
useRef()
useContext()
useReducer()
custom hooks
```

normally.

The portal affects **DOM placement**, not the basic rules for hooks.

---

# 15. When you should not use a portal

A portal is not the default solution for:

```text
every component
```

You generally reach for it when the DOM hierarchy itself causes problems.

Typical examples:

```text
modal/dialog
tooltip
popover
dropdown escaping overflow
floating UI
toast layer
```

If a normal component works correctly inside its parent:

```jsx
<div>
  <Card />
</div>
```

don't portal it just because you can.

Remember:

> **Portals solve a DOM-placement problem.**

---

# 16. The modal example exposes an architecture question

Where should modal state live?

Imagine:

```jsx
function ProductCard() {
  const [open, setOpen] = useState(false);

  return (
    <>
      <button onClick={() => setOpen(true)}>
        Delete
      </button>

      <DeleteModal
        open={open}
        onClose={() => setOpen(false)}
      />
    </>
  );
}
```

That's often perfectly reasonable.

But if your application has a global modal system:

```text
open profile modal
open edit-product modal
open confirmation modal
open image viewer
```

you might eventually use:

```text
context
state machine/reducer
global UI store
```

The portal doesn't determine this.

Again:

```text
Portal
=
DOM placement
```

while:

```text
Context/reducer/store
=
state ownership
```

These are separate architectural concerns.

---

# 17. A deeper modal pattern

A mature modal component often separates:

```text
state
accessibility
DOM placement
visual presentation
event handling
```

For example:

```text
Modal
├── open/close state
├── backdrop
├── dialog
├── Escape handling
├── focus management
├── portal
└── content
```

We're not going deep into accessibility yet, but one thing is worth knowing now:

> A modal is not truly production-ready just because it visually overlays the screen.

There are additional behaviors such as:

```text
keyboard interaction
focus management
screen-reader semantics
background interaction
focus restoration
```

We'll cover those in the accessibility portion later.

---

# 18. Escape-to-close

One classic modal feature:

```text
press Escape
 ↓
close modal
```

This often uses an effect to synchronize with a DOM event:

```jsx
useEffect(() => {
  function handleKeyDown(event) {
    if (event.key === "Escape") {
      onClose();
    }
  }

  if (open) {
    document.addEventListener(
      "keydown",
      handleKeyDown
    );
  }

  return () => {
    document.removeEventListener(
      "keydown",
      handleKeyDown
    );
  };
}, [open, onClose]);
```

Look at what we're doing:

```text
React state
  ↓
effect
  ↓
external DOM system
  ↓
keyboard event
  ↓
callback
  ↓
state change
```

This is another perfect example of what `useEffect` is actually for:

> **synchronizing React with something outside React.**

---

# 19. Cleanup is essential here

Suppose:

```jsx
document.addEventListener(
  "keydown",
  handleKeyDown
);
```

but you never remove it.

Then you can accumulate listeners:

```text
open modal
 → add listener

close modal
 → listener remains

open modal
 → another listener

close modal
 → both remain
```

Eventually one key press might invoke multiple callbacks.

So:

```jsx
return () => {
  document.removeEventListener(
    "keydown",
    handleKeyDown
  );
};
```

mirrors the setup.

This connects directly to our earlier `useEffect` lesson.

---

# 20. Focus management

Imagine opening:

```text
Delete dialog
```

User clicks:

```text
Delete
```

The user's focus is on that button.

After opening the modal, you usually want keyboard focus moved into the dialog.

Then when the dialog closes, you often want focus restored to the thing that opened it.

This is where:

```jsx
useRef()
```

becomes useful.

Conceptually:

```text
button that opened modal
       ↓
remember it with ref
       ↓
modal opens
       ↓
focus dialog
       ↓
modal closes
       ↓
restore focus
```

So our earlier lessons are converging:

```text
portal
+
state
+
effect
+
ref
+
event handling
+
context
```

can all participate in one real component.

---

# 21. A common portal bug: assuming DOM ancestry

Suppose you write code like:

```js
const element = event.target.closest(".card");
```

Now your element has been portaled elsewhere.

Its physical DOM ancestors may no longer resemble the React ancestors you had mentally associated with it.

That means code based on DOM ancestry needs extra care.

The React relationship:

```text
Card
└── Modal
```

doesn't imply:

```text
DOM:
Card
└── Modal
```

after a portal.

This is one of the main reasons portal-related debugging can feel strange at first.

---

# 22. Another common mistake: using z-index as a universal fix

Before portals, people often try:

```css
z-index: 999999;
```

and:

```css
position: absolute;
```

to force a modal above everything.

But stacking contexts can make this fail.

For example, an ancestor with certain properties can establish a separate stacking context.

Then:

```text
child z-index = 999999
```

doesn't magically leap outside the ancestor's stacking context.

A portal can avoid much of this by placing the modal in a more appropriate DOM location.

The point isn't:

> "Portals fix all CSS problems."

It's:

> **They let the overlay participate in a different DOM layout context while preserving React ownership.**

---

# 23. Portals and SSR

There's another topic you'll eventually encounter.

If code runs in a server-rendered environment, this:

```js
document.body
```

doesn't exist during server rendering.

So code such as:

```jsx
createPortal(..., document.body)
```

needs to be used with awareness of the rendering environment.

This is one of the reasons portal implementations in SSR applications sometimes use a dedicated client-side container or otherwise ensure the DOM target exists before rendering into it.

We're not going deep into SSR yet, but I want you to recognize the issue when we reach it.

---

# 24. A reusable modal architecture

A practical structure can look like:

```jsx
function Modal({ open, onClose, children }) {
  useEffect(() => {
    if (!open) return;

    function handleEscape(event) {
      if (event.key === "Escape") {
        onClose();
      }
    }

    document.addEventListener(
      "keydown",
      handleEscape
    );

    return () => {
      document.removeEventListener(
        "keydown",
        handleEscape
      );
    };
  }, [open, onClose]);

  if (!open) {
    return null;
  }

  return createPortal(
    <div
      className="backdrop"
      onClick={onClose}
    >
      <div
        className="dialog"
        role="dialog"
        aria-modal="true"
        onClick={e => e.stopPropagation()}
      >
        {children}
      </div>
    </div>,
    document.body
  );
}
```

We're intentionally not making this a complete production accessibility implementation yet.

But look at what the component now demonstrates:

```text
open
 ↓
conditional rendering

onClose
 ↓
callback prop

Escape
 ↓
effect

cleanup
 ↓
event listener removal

createPortal
 ↓
DOM relocation

stopPropagation
 ↓
event behavior

children
 ↓
composition

role/aria
 ↓
accessibility semantics
```

This is exactly what good React learning should eventually look like: separate concepts cooperating inside one real component.

---

# 25. Portal isn't a styling trick

This distinction is worth making explicit.

You might think:

> "Portal = put the modal at the top of the page."

That's the visual consequence.

The actual concept is:

> **Render a React subtree into a different DOM node while keeping it in the same React tree.**

That explains:

```text
context still works
state still works
hooks still work
React events still follow React relationships
```

while:

```text
DOM ancestors change
CSS/layout context can change
```

That's the core.

---

# 26. The three trees/systems worth distinguishing

At this point you've encountered:

### React component tree

```text
Parent
└── Child
```

### Browser DOM tree

```text
div
└── button
```

### State/data relationships

```text
Context
Query cache
local state
URL
```

These can overlap, but they're not identical.

For example:

```text
React tree
   ↓
Modal belongs to ProductCard

DOM tree
   ↓
Modal lives under body

Server state
   ↓
Modal may be editing product #42

Local state
   ↓
modal is open
```

Thinking in separate layers like this is extremely useful.

---

# 27. The deepest lesson

This topic connects to something broader we've been learning:

> **Logical ownership does not always have to match physical placement.**

We've already seen similar ideas elsewhere.

State can be:

```text
owned by parent
```

while UI is rendered:

```text
deep inside child
```

Server data can be:

```text
cached centrally
```

while consumed:

```text
by many distant components
```

URL state can be:

```text
stored in the URL
```

while affecting:

```text
a deeply nested product list
```

Portals add another version:

```text
React ownership
≠
DOM placement
```

That's a very useful architectural way to think about React.

---

# 28. Final mental model

Keep this diagram:

```text
                    React Tree
                        │
             ┌──────────┴──────────┐
             │                     │
          Product               Modal
             │                     │
             └──────────┬──────────┘
                        │
                 same React logic
                        │
                        ↓
                 React event system
                        │
                        ↓
                  Context / State
                        
                        BUT

                    DOM Tree
                        │
              ┌─────────┴─────────┐
              │                   │
             #root              <body>
              │                   │
           Product             Modal
```

Same React tree.

Different DOM location.

That is the portal.

And the most important practical consequence is:

> **Events and React relationships can cross the portal boundary because the portal changes DOM placement, not React ancestry.**

Once this makes sense, modals, popovers, overlays, and floating UI stop feeling like weird exceptions to React and start looking like ordinary React components with a different DOM placement strategy.