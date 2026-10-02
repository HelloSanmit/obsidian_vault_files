
# Lesson 29 — Accessibility in React: Semantics, Keyboard Interaction, Focus, ARIA, and `useId`

We've covered portals, refs, imperative handles, Suspense, transitions, and rendering behavior.

Now let's tackle something that is much broader than a React API:

> **How do you build React components that are actually usable by people who don't interact with your application using a mouse and visual cues alone?**

Accessibility is often taught as a collection of `aria-*` attributes.

That's the wrong mental model.

Good accessibility usually starts much earlier:

```text
HTML semantics
    ↓
correct interaction model
    ↓
keyboard behavior
    ↓
focus management
    ↓
accessible names/instructions
    ↓
ARIA where necessary
```

React's role is mostly to help us **preserve those relationships while the UI is dynamic**.

---

# 1. The first principle: use HTML for what it already knows how to do

Suppose you need a button.

You can write:

```jsx
<button onClick={save}>
  Save
</button>
```

Or:

```jsx
<div onClick={save}>
  Save
</div>
```

They may look identical.

They aren't equivalent.

A real `<button>` already has a browser-defined interaction model:

```text
keyboard focus
activation
button semantics
assistive-technology information
disabled behavior
form interaction
```

A `<div>` is just a generic container.

So when you turn:

```jsx
<div onClick={save}>Save</div>
```

into a "button", you are taking responsibility for recreating behavior the platform already provides.

That's usually the wrong tradeoff.

W3C's accessibility guidance emphasizes using appropriate native controls and ensuring all functionality is available from a keyboard. ([W3C](https://www.w3.org/WAI/WCAG21/Understanding/keyboard-accessible.html?utm_source=chatgpt.com "Understanding Guideline 2.1: Keyboard Accessible | WAI | W3C"))

---

# 2. Semantic HTML is an accessibility feature

Consider:

```jsx
<a href="/products">
  Products
</a>
```

This tells the browser:

```text
this is a link
```

Whereas:

```jsx
<div onClick={() => navigate("/products")}>
  Products
</div>
```

says:

```text
this is a generic container
```

You can add:

```jsx
role="link"
tabIndex={0}
```

and keyboard handlers, but now you're rebuilding a link.

Whenever possible:

```text
native semantic element
>
custom imitation
```

This principle becomes especially important for:

```text
button
a
input
select
textarea
form
nav
main
header
footer
fieldset
legend
table
```

---

# 3. React doesn't change the browser's semantics

This is a useful perspective.

When you write:

```jsx
<button>Save</button>
```

React isn't creating its own special "React button".

Eventually the browser has:

```html
<button>Save</button>
```

React is rendering normal web-platform elements.

So accessibility knowledge from HTML remains directly relevant.

React actually makes some HTML relationships easier to generate dynamically.

That's where `useId` becomes useful.

---

# 4. `useId`: the missing React API we haven't properly covered

We've used IDs in examples:

```jsx
<input id="email" />
<label htmlFor="email">
  Email
</label>
```

That's fine for a page with exactly one such field.

But reusable components create a problem.

Imagine:

```jsx
function TextField() {
  return (
    <>
      <label htmlFor="email">
        Email
      </label>

      <input id="email" />
    </>
  );
}
```

Now:

```jsx
<TextField />
<TextField />
<TextField />
```

gives you:

```text
id="email"
id="email"
id="email"
```

That's invalid because IDs are supposed to uniquely identify elements within the document.

And now the labels are ambiguous.

React's `useId` exists specifically to generate unique IDs for relationships such as form labels and descriptions. ([React](https://react.dev/reference/react/useId?utm_source=chatgpt.com "useId – React"))

---

# 5. Using `useId`

```jsx
import { useId } from "react";

function EmailField() {
  const id = useId();

  return (
    <div>
      <label htmlFor={id}>
        Email
      </label>

      <input
        id={id}
        type="email"
      />
    </div>
  );
}
```

Now:

```jsx
<EmailField />
<EmailField />
```

produces distinct IDs.

Conceptually:

```text
EmailField #1
   ↓
id A

EmailField #2
   ↓
id B
```

The label points to the correct control in each instance.

---

# 6. `useId` is about relationships, not arbitrary uniqueness

Don't think:

> "`useId` = React's random ID generator."

That's too simplistic.

Its important use is establishing relationships between DOM elements.

For example:

```text
input
  ↕
label

input
  ↕
description

input
  ↕
error message
```

You can create:

```jsx
const id = useId();

const inputId = `${id}-input`;
const hintId = `${id}-hint`;
const errorId = `${id}-error`;
```

Then:

```jsx
<label htmlFor={inputId}>
  Password
</label>

<input
  id={inputId}
  aria-describedby={`${hintId} ${errorId}`}
/>

<p id={hintId}>
  Use at least 12 characters.
</p>

{error && (
  <p id={errorId}>
    {error}
  </p>
)}
```

This creates a meaningful relationship graph.

`useId` is specifically documented for generating IDs used by accessibility attributes such as `aria-describedby`. ([React](https://react.dev/reference/react/useId?utm_source=chatgpt.com "useId – React"))

---

# 7. `useId` should NOT generate list keys

This is an important distinction.

Don't do:

```jsx
items.map(item => (
  <Row
    key={useId()}
    item={item}
  />
))
```

And you shouldn't try to use `useId` as a replacement for:

```jsx
key={item.id}
```

Those solve different problems.

Remember:

```text
key
→ React identity within a collection

id
→ DOM identity / relationship

useId
→ stable IDs for DOM relationships
```

React's current `useId` guidance explicitly says not to use it for list keys. ([React](https://react.dev/reference/react/useId?utm_source=chatgpt.com "useId – React"))

---

# 8. Labels are much more important than placeholders

Consider:

```jsx
<input placeholder="Email" />
```

You might visually think:

```text
Email
```

is the label.

But placeholder text isn't a substitute for a proper form label.

A better component is:

```jsx
<label htmlFor={id}>
  Email address
</label>

<input
  id={id}
  type="email"
/>
```

A proper label gives the input an accessible name and provides a larger clickable association in normal browser behavior. W3C specifically recommends associated labels for form controls. ([W3C](https://www.w3.org/WAI/tutorials/forms/labels/?utm_source=chatgpt.com "Labeling Controls | Web Accessibility Initiative (WAI) | W3C"))

This also helps sighted users.

So accessibility isn't:

```text
special UX for a tiny group
```

Good semantic structure frequently improves the experience generally.

---

# 9. React's `htmlFor`

In HTML:

```html
<label for="email">
```

In JSX:

```jsx
<label htmlFor="email">
```

This is because React uses the DOM property name `htmlFor`. ([React](https://react.dev/reference/react-dom/components/common?utm_source=chatgpt.com "Common components (e.g. <div>) – React"))

So:

```jsx
<label htmlFor={inputId}>
```

pairs with:

```jsx
<input id={inputId} />
```

The relationship is:

```text
label
  ↓ htmlFor
input id
```

---

# 10. `aria-describedby`: connecting instructions to controls

Suppose:

```text
Password
Must contain at least 12 characters.
```

The user sees the relationship visually.

But we also want the relationship to be programmatically clear.

```jsx
function PasswordField() {
  const id = useId();

  return (
    <>
      <label htmlFor={`${id}-input`}>
        Password
      </label>

      <input
        id={`${id}-input`}
        type="password"
        aria-describedby={`${id}-hint`}
      />

      <p id={`${id}-hint`}>
        Must contain at least 12 characters.
      </p>
    </>
  );
}
```

Now:

```text
input
  ↓
aria-describedby
  ↓
hint
```

The browser/assistive technology can understand that the paragraph describes the input. React's `useId` example uses precisely this pattern. ([React](https://react.dev/reference/react/useId?utm_source=chatgpt.com "useId – React"))

---

# 11. `aria-describedby` is not the same as `aria-label`

These solve different problems.

### `aria-label`

Provides an accessible name.

```jsx
<button aria-label="Close">
  <XIcon />
</button>
```

Visually:

```text
[X]
```

Accessible name:

```text
"Close"
```

### `aria-describedby`

Provides supplementary description.

```jsx
<input
  aria-describedby={hintId}
/>
```

Think:

```text
aria-label
    =
"What is this?"

aria-describedby
    =
"What additional information should I know?"
```

---

# 12. Prefer visible text when possible

Suppose:

```jsx
<button aria-label="Save">
  Save
</button>
```

That's redundant.

The button already contains:

```text
Save
```

so the accessible name can come from its visible content.

`aria-label` becomes especially useful when there is no visible textual name.

For example:

```jsx
<button aria-label="Close">
  <CloseIcon />
</button>
```

This creates a meaningful name where the icon alone isn't sufficient.

The general rule:

> **Use real visible text and native semantics first; use ARIA to supply information the native structure doesn't provide.**

---

# 13. ARIA doesn't create behavior

This is a very important misconception.

Suppose:

```jsx
<div
  role="button"
  aria-disabled="true"
>
  Save
</div>
```

You have told assistive technology:

```text
"This should be understood as a button."
```

You have **not automatically created all button behavior**.

You may still need:

```text
keyboard activation
focus behavior
disabled semantics/behavior
event handling
```

That's why:

```jsx
<button disabled>
  Save
</button>
```

is generally preferable.

The native element already supplies the correct behavior.

This is one of the most valuable accessibility rules:

> **ARIA describes semantics; it doesn't magically implement the widget.**

---

# 14. Keyboard accessibility

W3C's core keyboard principle is straightforward:

> Functionality available by mouse should also be available by keyboard. ([W3C](https://www.w3.org/WAI/WCAG21/Understanding/keyboard-accessible.html?utm_source=chatgpt.com "Understanding Guideline 2.1: Keyboard Accessible | WAI | W3C"))

So suppose you create:

```jsx
<div onClick={openMenu}>
  Menu
</div>
```

Mouse works.

Keyboard?

Potentially no.

A keyboard user may navigate to:

```text
<div>
```

and nothing happens.

You have created an interaction that depends on pointer input.

---

# 15. The correct solution usually isn't adding keyboard handlers

You could attempt:

```jsx
<div
  tabIndex={0}
  role="button"
  onClick={openMenu}
  onKeyDown={e => {
    if (
      e.key === "Enter" ||
      e.key === " "
    ) {
      openMenu();
    }
  }}
>
  Menu
</div>
```

Now you're manually recreating button semantics.

Much simpler:

```jsx
<button onClick={openMenu}>
  Menu
</button>
```

This is the recurring pattern:

```text
native element
→ existing behavior

custom element
→ you must recreate behavior
```

Use custom widgets only when their interaction model truly requires one.

---

# 16. Keyboard focus is a real UI state

Consider:

```text
Button A
Button B
Button C
```

With a mouse, the user can visually click whatever they want.

With a keyboard:

```text
Tab
 ↓
A focused
 ↓
Tab
 ↓
B focused
 ↓
Tab
 ↓
C focused
```

The user needs a visible indication of where they are.

W3C's guidance explicitly requires a mode where keyboard focus is visibly indicated. ([W3C](https://www.w3.org/WAI/WCAG22/Understanding/focus-visible?utm_source=chatgpt.com "Understanding Success Criterion 2.4.7: Focus Visible | WAI | W3C"))

So be extremely careful with CSS such as:

```css
outline: none;
```

when you haven't provided another visible focus indication.

---

# 17. `:focus-visible`

For modern UIs, you can often style keyboard focus using:

```css
button:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```

The exact visual design is up to you.

The important requirement is:

```text
keyboard user
 ↓
focus moves
 ↓
user can see where focus is
```

That is not cosmetic.

It's navigation information. ([W3C](https://www.w3.org/WAI/WCAG22/Understanding/focus-visible?utm_source=chatgpt.com "Understanding Success Criterion 2.4.7: Focus Visible | WAI | W3C"))

---

# 18. Why focus management matters so much in React

React applications frequently insert and remove DOM nodes.

For example:

```text
click Delete
 ↓
modal opens
 ↓
new DOM subtree appears
```

Now where is keyboard focus?

It may still be on the button underneath the modal.

That's bad.

A modal usually needs to manage focus deliberately.

And this is where the ref lesson we just did becomes directly relevant.

---

# 19. Modal focus with refs

Suppose:

```jsx
function Dialog({ open, onClose }) {
  const headingRef = useRef(null);

  useEffect(() => {
    if (open) {
      headingRef.current?.focus();
    }
  }, [open]);

  if (!open) {
    return null;
  }

  return createPortal(
    <div className="backdrop">
      <div
        role="dialog"
        aria-modal="true"
      >
        <h2
          ref={headingRef}
          tabIndex={-1}
        >
          Delete product?
        </h2>

        ...
      </div>
    </div>,
    document.body
  );
}
```

The flow is:

```text
open changes
 ↓
render dialog
 ↓
DOM exists
 ↓
Effect runs
 ↓
ref points to heading
 ↓
focus()
```

This is a perfect example of:

```text
state
+
effect
+
ref
+
portal
+
accessibility
```

all working together.

---

# 20. Why `tabIndex={-1}` is useful

Suppose:

```jsx
<h2>Delete product?</h2>
```

Normally, a heading isn't a normal keyboard-focusable element.

But sometimes we want to programmatically move focus to it.

That's what:

```jsx
tabIndex={-1}
```

allows.

It means, conceptually:

```text
not part of normal Tab sequence
+
can receive programmatic focus
```

So:

```js
headingRef.current.focus();
```

can work.

This is often useful for placing focus at a meaningful starting point after a UI transition.

---

# 21. The modal focus lifecycle

A properly designed modal typically needs to think about:

```text
before opening:
where was focus?

opening:
where should focus go?

while open:
can focus escape into the background?

closing:
where should focus return?
```

So the lifecycle is roughly:

```text
Open modal
   ↓
save opener
   ↓
focus inside dialog
   ↓
keyboard interaction stays within modal
   ↓
close
   ↓
restore opener focus
```

That's much more sophisticated than:

```jsx
{open && <Modal />}
```

The rendering part is easy.

Focus behavior is where production quality begins.

---

# 22. Focus trapping

Imagine:

```text
Modal
├── Cancel
├── Delete
└── Close
```

The modal is active.

A keyboard user presses:

```text
Tab
```

You generally don't want the focus to silently jump into the page behind the modal.

You need an appropriate focus-management strategy so keyboard interaction remains within the active modal while it is modal.

The exact implementation can be surprisingly complicated, which is one reason mature UI libraries often provide dialog primitives rather than expecting every application to write focus trapping from scratch.

The important React lesson is simply:

> **Visual overlays and keyboard focus are separate systems.**

A portal solves the DOM-placement issue.

It does not automatically solve focus management.

---

# 23. `aria-modal`

For an actual modal dialog:

```jsx
<div
  role="dialog"
  aria-modal="true"
>
```

you're communicating important semantics:

```text
this is a dialog
+
it is modal
```

But again:

```text
aria-modal
```

doesn't itself implement:

```text
focus trapping
Escape handling
background interaction rules
visual overlay
```

You need behavior as well.

This is another instance of:

```text
semantics ≠ behavior
```

---

# 24. The Escape key

We already built this in our modal lesson:

```jsx
useEffect(() => {
  if (!open) return;

  function handleKeyDown(event) {
    if (event.key === "Escape") {
      onClose();
    }
  }

  document.addEventListener(
    "keydown",
    handleKeyDown
  );

  return () => {
    document.removeEventListener(
      "keydown",
      handleKeyDown
    );
  };
}, [open, onClose]);
```

Now we can interpret it from an accessibility perspective.

It's not merely a convenience.

It's part of the keyboard interaction model for a dialog.

Notice how the effect's lifecycle mirrors the modal state:

```text
closed
 ↓
no listener

opened
 ↓
listener added

closed
 ↓
listener removed
```

Again:

```text
Effect
+
cleanup
```

is doing exactly what we learned it should do.

---

# 25. Accessible forms

Forms are one of the biggest places React developers accidentally create inaccessible UI.

Consider:

```jsx
<form>
  <input />
  <button>Submit</button>
</form>
```

What's wrong?

The input has no clear label.

Better:

```jsx
<form>
  <label htmlFor={emailId}>
    Email address
  </label>

  <input
    id={emailId}
    type="email"
  />

  <button type="submit">
    Submit
  </button>
</form>
```

W3C recommends labels or instructions for form controls and emphasizes properly associating labels with controls. ([W3C](https://www.w3.org/WAI/tutorials/forms/?utm_source=chatgpt.com "Forms Tutorial | Web Accessibility Initiative (WAI) | W3C"))

---

# 26. Errors need to be connected to the field

Suppose the user enters an invalid email.

You might show:

```text
Email address

[abc]

Invalid email address
```

Visually, it's obvious.

But the DOM should communicate that relationship too.

```jsx
const id = useId();

const inputId = `${id}-input`;
const errorId = `${id}-error`;

return (
  <>
    <label htmlFor={inputId}>
      Email address
    </label>

    <input
      id={inputId}
      type="email"
      aria-invalid={hasError}
      aria-describedby={
        hasError ? errorId : undefined
      }
    />

    {hasError && (
      <p id={errorId}>
        Invalid email address.
      </p>
    )}
  </>
);
```

Now the relationships are explicit:

```text
input
 ├── invalid
 └── described by
       ↓
   error message
```

This is a very strong React pattern because the relationships can change dynamically as state changes.

W3C's forms guidance covers labels, instructions, grouping, and feedback as core aspects of accessible forms. ([W3C](https://www.w3.org/WAI/tutorials/forms/?utm_source=chatgpt.com "Forms Tutorial | Web Accessibility Initiative (WAI) | W3C"))

---

# 27. `aria-invalid` isn't validation itself

This:

```jsx
aria-invalid="true"
```

doesn't validate the field.

Your application still needs:

```text
validation rules
error state
server validation
```

The ARIA attribute communicates:

```text
"This control is currently invalid."
```

So again:

```text
application state
   ↓
accessibility representation
```

React is often the bridge between the two.

---

# 28. Required fields

You can use native HTML:

```jsx
<input required />
```

and optionally:

```jsx
aria-required="true"
```

when appropriate in more complex custom-widget scenarios.

But prefer the native behavior where possible:

```jsx
<input required />
```

The browser already understands that semantics.

Likewise:

```jsx
<button disabled>
```

is preferable to a generic element with only:

```jsx
aria-disabled="true"
```

when the element genuinely is a button.

---

# 29. Grouping related controls

Suppose you have:

```text
Shipping method

○ Standard
○ Express
○ Overnight
```

These belong together semantically.

HTML gives us:

```jsx
<fieldset>
  <legend>
    Shipping method
  </legend>

  <label>
    <input
      type="radio"
      name="shipping"
      value="standard"
    />
    Standard
  </label>

  ...
</fieldset>
```

This is much better than:

```jsx
<div>
  <p>Shipping method</p>
  ...
</div>
```

with a pile of custom ARIA.

Again:

```text
native semantics
first
```

W3C's forms guidance specifically recommends `<fieldset>` and `<legend>` for grouping related controls. ([W3C](https://www.w3.org/WAI/tutorials/forms/?utm_source=chatgpt.com "Forms Tutorial | Web Accessibility Initiative (WAI) | W3C"))

---

# 30. Dynamic React forms make this even more important

Suppose you have a reusable:

```jsx
<FormField />
```

You want:

```text
label
input
hint
error
```

all associated correctly.

A well-designed component can generate one ID:

```jsx
const id = useId();
```

then derive:

```jsx
const inputId = `${id}-input`;
const hintId = `${id}-hint`;
const errorId = `${id}-error`;
```

Now your component can be rendered:

```jsx
<FormField />
<FormField />
<FormField />
```

without manually coordinating globally unique identifiers.

That is exactly the type of dynamic relationship `useId` is intended to support. ([React](https://react.dev/reference/react/useId?utm_source=chatgpt.com "useId – React"))

---

# 31. Accessible buttons with icons

Suppose:

```jsx
<button>
  <TrashIcon />
</button>
```

Visually, a human may understand the icon.

But what does the control mean without the visual icon?

You can give it a name:

```jsx
<button aria-label="Delete">
  <TrashIcon />
</button>
```

Now:

```text
visual:
[trash icon]

accessible name:
Delete
```

You can also use visually hidden text:

```jsx
<button>
  <TrashIcon />
  <span className="sr-only">
    Delete
  </span>
</button>
```

The important concept is:

> Interactive controls need an understandable accessible name.

---

# 32. Don't use icons as the only meaning of a destructive action

This:

```text
🗑️
```

may be visually familiar.

But:

```text
Delete
```

is clearer.

You can still use:

```jsx
<TrashIcon aria-hidden="true" />
```

with:

```jsx
<span className="sr-only">
  Delete
</span>
```

The icon is visual decoration.

The text supplies the meaning.

This separation between:

```text
visual representation
+
semantic representation
```

is a very useful accessibility concept.

---

# 33. `aria-hidden`

Suppose an SVG icon is purely decorative:

```jsx
<TrashIcon aria-hidden="true" />
```

This tells assistive technology:

```text
don't treat this decorative icon
as meaningful content
```

The button still gets its name from:

```jsx
Delete
```

So:

```text
button name
→ Delete

icon
→ decoration
```

That's usually cleaner.

---

# 34. Don't hide meaningful content with CSS and assume it's gone from accessibility

There are multiple ways to visually hide things:

```css
display: none;
```

```css
visibility: hidden;
```

and:

```css
position: absolute;
...
```

These don't all have the same accessibility consequences.

For example, `display: none` removes the element from the rendered layout and accessibility tree.

But a typical screen-reader-only utility deliberately keeps content available to assistive technology while visually hiding it.

So:

```text
"hidden visually"
≠
"hidden from assistive technology"
```

That's why reusable `sr-only` CSS utilities exist.

---

# 35. `disabled` vs `aria-disabled`

Consider:

```jsx
<button disabled>
  Save
</button>
```

versus:

```jsx
<button aria-disabled="true">
  Save
</button>
```

These communicate different things.

Native:

```text
disabled
```

actually makes the control disabled according to the browser's button semantics.

ARIA:

```text
aria-disabled="true"
```

communicates the state but doesn't automatically implement all behavior associated with native disabling.

So if you genuinely have a button that is disabled:

```jsx
<button disabled={isSaving}>
```

is usually the more appropriate choice.

---

# 36. Tabs are a good example of why ARIA is not just decoration

Imagine:

```text
[Overview] [Reviews] [Specs]
```

You want:

```text
click tab
 ↓
panel changes
```

A simple implementation might use:

```jsx
<button onClick={() => setTab("reviews")}>
  Reviews
</button>
```

That's already good as a button.

But if you're implementing an actual tab interface, there are additional semantics:

```text
tablist
tab
tabpanel
selected tab
associated panel
keyboard navigation
```

Now ARIA can become appropriate because you're creating a widget whose semantics aren't fully represented by a single native element.

You might have:

```jsx
<button
  role="tab"
  aria-selected={active}
  aria-controls={panelId}
>
  Reviews
</button>
```

The exact full tab implementation needs more than one attribute.

That's the point:

> **ARIA becomes useful when you're implementing a custom interaction pattern, not as a substitute for ordinary HTML.**

---

# 37. `aria-selected` vs `aria-current`

These are easy to confuse.

Suppose navigation:

```text
Home | Products | Settings
```

You can indicate the current page appropriately with `aria-current="page"` on the current link. ([MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-current?utm_source=chatgpt.com "ARIA: aria-current attribute - ARIA | MDN"))

But for a tab interface:

```text
Overview | Reviews | Specs
```

the active tab is selected:

```text
aria-selected="true"
```

These concepts aren't interchangeable.

Think:

```text
current
→ "this is the current item/page"

selected
→ "this option is currently selected"
```

That distinction is useful when building:

```text
navigation
tabs
steps
breadcrumbs
```

---

# 38. Accessible navigation with React Router

We've already learned:

```jsx
<NavLink to="/products">
  Products
</NavLink>
```

A router-aware navigation component can also reflect the current location appropriately.

The important accessibility lesson is:

```text
navigation link
+
current-page indication
```

not simply:

```text
change text color
```

If your design visually makes the current navigation item bold but provides no programmatic/current-state information, keyboard and assistive-technology users may lose that distinction.

That's why semantic state matters alongside visual state.

---

# 39. Live regions: announcing dynamic changes

React applications frequently update things without changing the URL.

For example:

```text
"Saved successfully."
```

appears after a button click.

A sighted mouse user sees it.

A screen-reader user may not automatically know something changed.

A live region can communicate dynamic updates:

```jsx
<div aria-live="polite">
  {message}
</div>
```

Then:

```text
message = ""
        ↓
message = "Saved successfully."
```

the browser/assistive technology can announce the change according to the live-region behavior.

This is useful for:

```text
status messages
async operation results
validation summaries
notifications
search result counts
```

Use it deliberately; constantly changing live regions can become noisy.

---

# 40. `aria-live="assertive"` is not a louder success message

There are different live-region politeness levels.

For example:

```jsx
<div aria-live="polite">
```

generally means:

```text
announce when appropriate
without aggressively interrupting
```

Whereas:

```jsx
<div aria-live="assertive">
```

is intended for more urgent information.

Don't default to assertive.

A normal:

```text
"Saved!"
```

shouldn't necessarily interrupt whatever the user is currently hearing.

Accessibility often involves prioritizing information, not merely making everything announced.

---

# 41. Search results are an interesting React case

Imagine:

```jsx
<input
  value={query}
  onChange={...}
/>

<p>
  42 results
</p>
```

The result count changes dynamically.

A live region could be useful:

```jsx
<p aria-live="polite">
  {resultCount} results
</p>
```

But think about whether the result count changing on every keystroke is helpful or overwhelming.

This is why accessibility is not simply:

```text
"add aria-live everywhere"
```

It requires thinking about **what information the user needs and when**.

---

# 42. Focus and dynamic rendering

Suppose:

```jsx
{showResults && <Results />}
```

When results suddenly appear, where should keyboard focus go?

You don't automatically want:

```text
focus jumps into Results
```

because the user might still be typing in the search field.

Sometimes the best solution is:

```text
don't move focus
+
announce result count
```

Other times, a page transition genuinely requires focus to move to new content.

This is why accessibility decisions depend on the interaction's purpose.

---

# 43. Route changes and focus

Single-page applications have another accessibility challenge.

Traditional navigation:

```text
click link
 ↓
new document
 ↓
browser starts new page
```

A screen reader/browser naturally gets a new document context.

Client-side routing:

```text
click Link
 ↓
React changes route
 ↓
same document remains loaded
```

The page visually changes, but the browser hasn't performed a traditional full document navigation.

That means your application needs to think about:

```text
where keyboard focus ends up
what gets announced
which heading is now meaningful
```

Good router/framework components can help with some of this, but application-level focus management may still matter.

This is a great example of why SPAs require more deliberate accessibility engineering than simply making individual buttons accessible.

---

# 44. Headings are navigation landmarks for many users

Consider:

```jsx
<h1>Products</h1>

<h2>Featured</h2>
<h2>New arrivals</h2>
```

Headings aren't merely font-size choices.

They create document structure.

So avoid:

```jsx
<div className="big-heading">
  Products
</div>
```

when this is actually a page heading.

The semantic structure can help users navigate the page by headings.

This also makes your UI easier to reason about as a document rather than a pile of styled `<div>` elements.

---

# 45. `<main>`, `<nav>`, `<header>`, `<footer>`

Likewise:

```jsx
<header />
<nav />
<main />
<footer />
```

communicate landmark structure.

Instead of:

```jsx
<div className="header">
<div className="nav">
<div className="main">
<div className="footer">
```

You're giving the browser more semantic information.

This is especially valuable in large React applications because component boundaries can otherwise hide the document structure behind abstractions.

---

# 46. Don't create an abstraction that destroys semantics

Suppose you build:

```jsx
<Box />
```

and use it everywhere:

```jsx
<Box>Heading</Box>
<Box>Navigation</Box>
<Box>Button</Box>
```

If the abstraction always renders:

```html
<div>
```

you've made it easy for your application to lose semantic meaning.

A better API might explicitly support:

```jsx
<Box as="main">
```

or:

```jsx
<Stack as="nav">
```

or simply use semantic components:

```jsx
<Main>
<Nav>
```

The architecture lesson is:

> **Component abstraction should preserve the semantics of the UI, not erase them.**

---

# 47. Accessible component APIs

Think back to our component composition lesson.

Suppose you build:

```jsx
<Card />
```

Now imagine:

```jsx
<Card
  onClick={...}
/>
```

Should Card become a clickable interactive element?

That depends.

If it's just a visual container with a click callback, you've potentially created an accessibility problem.

Maybe it should actually be:

```jsx
<button className="card">
```

or:

```jsx
<Link className="card">
```

depending on the intended action.

This is another example of:

```text
API design
+
semantics
```

being tightly connected.

---

# 48. Don't make an entire card clickable with hidden semantics

Suppose:

```jsx
<div onClick={openProduct}>
  <h2>iPhone</h2>
  <p>...</p>
</div>
```

Maybe the intended interaction is navigation.

Then:

```jsx
<Link to="/products/42">
  ...
</Link>
```

is usually a better semantic model.

If there are multiple actions:

```text
Card
├── product link
├── favorite button
└── delete button
```

then each action can have its own semantic control.

Don't make:

```text
whole card
```

a pseudo-button when the actual UI contains multiple independent actions.

---

# 49. Accessibility and event propagation

This also connects to our portal lesson.

Suppose:

```jsx
<div onClick={selectCard}>
  <Modal>
    <button>Delete</button>
  </Modal>
</div>
```

The button's event can propagate through the React tree even though the modal may be portaled into `document.body`.

So:

```text
Delete
 ↓
modal
 ↓
parent React handler
```

could produce unintended behavior.

Now our earlier:

```jsx
e.stopPropagation()
```

becomes not just a JavaScript trick, but part of interaction architecture.

Again:

```text
accessibility
+
event model
+
portal
```

intersect.

---

# 50. Accessibility + refs

We're seeing the same pattern repeatedly.

Ref:

```js
inputRef.current.focus();
```

is not accessibility by itself.

But used correctly:

```text
validation failure
 ↓
focus first invalid field
```

can greatly improve keyboard usability.

Likewise:

```text
route change
 ↓
focus page heading
```

can provide a meaningful starting point.

So refs are an **implementation mechanism**.

Accessibility is the **user-experience requirement**.

---

# 51. Accessibility + Suspense

This gets interesting too.

Suppose:

```jsx
<Suspense fallback={<Spinner />}>
  <Results />
</Suspense>
```

A spinner may be visually obvious.

But a screen-reader user may not automatically understand:

```text
"Results are currently loading."
```

You might need an accessible status mechanism or an appropriately structured loading state.

The key lesson:

```text
visual loading indicator
≠
automatic accessible communication
```

So every dynamic UI state needs us to ask:

> **What does a non-visual user perceive when this changes?**

That's a very powerful question.

---

# 52. Accessibility + transitions

Likewise:

```text
startTransition(...)
```

is a React rendering concept.

It does not automatically communicate:

```text
"content is updating"
```

to assistive technology.

React handles scheduling.

Your UI still needs meaningful semantics for:

```text
loading
saving
updated
error
navigation
```

Again:

```text
React mechanics
≠
user communication
```

---

# 53. A practical accessibility checklist for a React component

When building an interactive component, mentally walk through:

```text
What semantic element is this?

Does it have an accessible name?

Can I operate it using a keyboard?

Is keyboard focus visible?

If it appears/disappears, where does focus go?

If state changes, does the user receive the necessary information?

Are labels/instructions associated with controls?

Are errors associated with their fields?

Does the DOM structure reflect the visual structure?

Am I using ARIA because it is actually necessary?
```

That checklist is far more valuable than memorizing dozens of ARIA attributes.

---

# 54. A complete accessible field component

Let's combine the ideas we've learned.

```jsx
import { useId } from "react";

function TextField({
  label,
  hint,
  error,
  value,
  onChange,
  type = "text",
}) {
  const id = useId();

  const inputId = `${id}-input`;
  const hintId = `${id}-hint`;
  const errorId = `${id}-error`;

  const describedBy = [
    hint ? hintId : null,
    error ? errorId : null,
  ]
    .filter(Boolean)
    .join(" ") || undefined;

  return (
    <div>
      <label htmlFor={inputId}>
        {label}
      </label>

      {hint && (
        <p id={hintId}>
          {hint}
        </p>
      )}

      <input
        id={inputId}
        type={type}
        value={value}
        onChange={onChange}
        aria-invalid={Boolean(error)}
        aria-describedby={describedBy}
      />

      {error && (
        <p id={errorId}>
          {error}
        </p>
      )}
    </div>
  );
}
```

This component has:

```text
unique ID generation
 ↓
label relationship
 ↓
hint relationship
 ↓
error relationship
 ↓
invalid state
```

Now render:

```jsx
<TextField
  label="Email address"
  hint="Use your work email."
  error={emailError}
  value={email}
  onChange={handleEmailChange}
/>
```

The component API stays simple.

The accessibility plumbing stays inside the reusable component.

That's exactly where abstractions are useful.

---

# 55. A complete button API

Likewise:

```jsx
function Button({
  children,
  disabled,
  onClick,
  type = "button",
}) {
  return (
    <button
      type={type}
      disabled={disabled}
      onClick={onClick}
    >
      {children}
    </button>
  );
}
```

This may look boring.

That's a compliment.

You don't need:

```text
role="button"
tabIndex
keyboard handler
aria-role
custom focus management
```

because:

```jsx
<button>
```

already gives you a huge amount of correct behavior.

---

# 56. The accessibility philosophy I want you to develop

Don't ask:

> "Which ARIA attribute fixes this?"

Ask:

> **"What is this UI element semantically, and how should a user interact with it?"**

Then:

```text
choose native element
        ↓
add correct keyboard behavior
        ↓
manage focus when dynamic UI requires it
        ↓
provide labels/instructions
        ↓
use ARIA only where native semantics aren't enough
```

This makes accessibility much more systematic.

---

# 57. And this gives us a powerful React-specific mental model

React applications have a lot of dynamic behavior:

```text
component appears
component disappears
state changes
route changes
modal opens
error appears
results refresh
content suspends
```

Accessibility asks:

```text
What does the user know about each transition?
Where is focus?
What can be operated?
What is the semantic structure?
```

So accessibility isn't a final polishing step you sprinkle on after the application is finished.

It's part of the component's behavior.

---

# 58. The deepest connection to our previous lessons

Look at this one interaction:

```text
User submits form
       ↓
React event
       ↓
mutation
       ↓
server responds
       ↓
validation/error state
       ↓
render
       ↓
error message appears
       ↓
aria-describedby connects error to input
       ↓
ref focuses invalid field
       ↓
screen-reader/browser semantics communicate state
```

That uses concepts from:

```text
events
state
forms
TanStack Query
mutations
rendering
refs
effects
useId
ARIA
HTML semantics
```

This is what I mean when I say we're not learning APIs independently.

They're pieces of one system.

---

# 59. The rules worth remembering

```text
1. Prefer native semantic HTML.

2. A <button> is generally better than a clickable <div>.

3. A <a> is generally better than a clickable <div> for navigation.

4. Every form control should have an appropriate label.

5. useId is useful for unique DOM IDs and relationships between elements.

6. Do not use useId for list keys.

7. aria describes semantics; it does not automatically implement behavior.

8. Keyboard functionality must exist for functionality available by pointer.

9. Keyboard focus should remain visibly identifiable.

10. Dynamic UI needs deliberate focus management.

11. Modals require more than visual overlay:
    semantics + keyboard + focus + behavior.

12. Form errors should be associated with their controls.

13. Use live regions deliberately for important dynamic status changes.

14. Don't use memoization, portals, refs, or ARIA as substitutes for good structure.

15. Accessibility is part of component behavior and API design.
```

The two questions I'd especially like you to internalize are:

```text
"What is this element?"

and

"How does a keyboard/screen-reader user experience this interaction?"
```

Once you start asking those two questions while building React components, accessibility stops being a pile of special cases and becomes part of normal UI engineering. W3C's current forms and keyboard guidance reinforces the same foundations: meaningful labels/instructions, proper grouping, and keyboard-operable functionality. ([W3C](https://www.w3.org/WAI/tutorials/forms/?utm_source=chatgpt.com "Forms Tutorial | Web Accessibility Initiative (WAI) | W3C"))

And `useId` gives React a particularly useful tool for maintaining those semantic relationships when reusable components are rendered multiple times. ([React](https://react.dev/reference/react/useId?utm_source=chatgpt.com "useId – React"))