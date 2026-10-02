
# Styling with React — Lesson S1

# CSS Fundamentals, Properly Understood

We are starting the styling section from the ground up, but **not from the perspective of someone who doesn't know CSS**. You already know HTML/CSS, so the goal is different:

> Understand exactly how CSS interacts with React's component model, rendering, state, DOM structure, and application architecture.

We will go deep enough that later topics like CSS Modules, Tailwind, design systems, animations, responsive layouts, and component libraries make sense rather than feeling like unrelated tools.

---

# 1. First: React does not replace CSS

This distinction needs to be absolutely clear.

React primarily decides:

```text
what UI exists
what DOM structure exists
what attributes/classes exist
when UI changes
```

CSS decides:

```text
how that UI looks
where it is laid out
how large it is
how it responds to available space
how it changes visually
```

So this:

```jsx
function Button() {
  return <button className="button">Save</button>;
}
```

contains two separate ideas:

```text
React
 ↓
creates <button>
 ↓
attaches class="button"

CSS
 ↓
finds .button
 ↓
applies visual/layout rules
```

React doesn't "style the button."

React creates the DOM.

CSS styles the DOM.

---

# 2. The browser is where CSS actually runs

The architecture is:

```text
React code
    ↓
render
    ↓
DOM
    ↓
browser CSS engine
    ↓
style calculation
    ↓
layout
    ↓
paint
    ↓
compositing
    ↓
pixels
```

So when you write:

```jsx
<div className="card">
  Hello
</div>
```

React doesn't directly know what:

```css
.card {
  padding: 20px;
}
```

means.

The browser does.

This distinction becomes very useful when debugging performance.

---

# 3. `className` is just the React form of HTML `class`

HTML:

```html
<div class="card"></div>
```

JSX:

```jsx
<div className="card"></div>
```

Why `className`?

Because `class` has special meaning in JavaScript syntax and JSX historically uses the DOM property naming convention.

The resulting DOM is essentially:

```html
<div class="card"></div>
```

So:

```jsx
className="card"
```

is not a special React styling language.

It's a way to produce the DOM class attribute.

---

# 4. CSS selectors still work normally

React:

```jsx
function Card() {
  return (
    <article className="card">
      <h2 className="card-title">
        Project Alpha
      </h2>
    </article>
  );
}
```

CSS:

```css
.card {
  border: 1px solid #ddd;
}

.card-title {
  font-size: 1.25rem;
}
```

React doesn't interpret:

```css
.card
```

The browser matches it against the generated DOM.

So:

```text
React JSX
   ↓
DOM
   ↓
CSS selector matching
   ↓
style application
```

---

# 5. Conditional classes are one of the most important React/CSS patterns

Suppose a task can be:

```text
open
in progress
done
blocked
```

React can choose the class:

```jsx
function TaskStatus({ status }) {
  return (
    <span className={`status status--${status}`}>
      {status}
    </span>
  );
}
```

Then:

```css
.status {
  padding: 0.25rem 0.5rem;
  border-radius: 999px;
}

.status--open {
  /* open appearance */
}

.status--in-progress {
  /* in-progress appearance */
}

.status--done {
  /* done appearance */
}

.status--blocked {
  /* blocked appearance */
}
```

The important architecture is:

```text
application state
      ↓
class selection
      ↓
CSS state representation
      ↓
visual output
```

This is much better than making React manually calculate dozens of individual style properties.

---

# 6. React state and CSS state are not the same thing

Suppose:

```jsx
const [open, setOpen] = useState(false);
```

React owns:

```text
open = true/false
```

CSS can represent that state:

```jsx
<div className={open ? 'menu menu--open' : 'menu'}>
```

Then:

```css
.menu {
  opacity: 0;
  pointer-events: none;
}

.menu--open {
  opacity: 1;
  pointer-events: auto;
}
```

Notice the separation:

```text
React
→ determines state

CSS
→ determines presentation of that state
```

That separation is one of the healthiest React styling patterns.

---

# 7. Don't use React state for purely visual CSS values unnecessarily

Bad:

```jsx
const [isHovered, setIsHovered] = useState(false);

<div
  onMouseEnter={() => setIsHovered(true)}
  onMouseLeave={() => setIsHovered(false)}
>
```

just to change:

```text
background on hover
```

CSS already understands hover:

```css
.card:hover {
  transform: translateY(-2px);
}
```

You don't need React involved.

The architecture should be:

```text
interaction belongs to browser/CSS
→ CSS

application meaning belongs to React
→ React
```

For example:

```text
hover
focus
active
disabled
visited
```

are often CSS concerns.

Whereas:

```text
selected task
current user
editor open
server status
```

are React/application concerns.

---

# 8. The cascade

CSS stands for Cascading Style Sheets.

"Cascade" is not just a name.

When multiple rules match the same element, the browser determines which declarations win.

Suppose:

```css
p {
  color: blue;
}

.message {
  color: red;
}
```

and:

```jsx
<p className="message">
  Hello
</p>
```

Both rules match:

```text
p
.message
```

The class selector has higher specificity than the element selector, so:

```text
red
```

wins.

This is the beginning of understanding CSS conflicts.

---

# 9. The cascade is a decision process

A simplified mental model is:

```text
candidate declarations
        ↓
cascade origin/layer
        ↓
importance
        ↓
specificity
        ↓
source order
        ↓
winning declaration
```

You don't need to memorize the entire browser specification to use CSS well.

But you do need to understand why:

```css
.foo { color: red; }
.bar { color: blue; }
```

can result in either color depending on specificity and ordering.

---

# 10. Specificity

Think of specificity as the strength of a selector.

Very roughly:

```text
element
    ↓
class / attribute / pseudo-class
    ↓
ID
```

For example:

```css
button {
  color: black;
}
```

versus:

```css
.button {
  color: blue;
}
```

The class selector wins.

Then:

```css
#save-button {
  color: green;
}
```

wins over the class.

The exact specificity calculation is more precise than this, but this model is enough to begin reasoning about conflicts.

---

# 11. Why over-specific CSS becomes painful

Suppose you write:

```css
.page .content .toolbar .actions .button {
  color: red;
}
```

Now another component needs to override it.

Perhaps:

```css
.page .content .toolbar .actions .button.primary {
  color: blue;
}
```

Then:

```text
specificity keeps increasing
```

Eventually you get:

```text
!important
```

and then:

```text
!important everywhere
```

Now your stylesheet becomes a specificity war.

Good CSS architecture tries to avoid needing that.

---

# 12. Avoid `!important` as normal architecture

This:

```css
.button {
  color: red !important;
}
```

can be necessary in certain integration scenarios.

But if you're using it because:

> "My CSS isn't applying."

the real question is:

```text
Why isn't it winning the cascade?
```

Perhaps:

```text
wrong selector
wrong layer
wrong specificity
wrong source order
inline style
third-party stylesheet
```

Understand the cause first.

---

# 13. CSS layers

A modern CSS architecture can deliberately control cascade layers:

```css
@layer reset, base, components, utilities;
```

Then:

```css
@layer base {
  body {
    margin: 0;
  }
}

@layer components {
  .button {
    ...
  }
}
```

Now you're not relying purely on accidental source ordering.

This is particularly useful in larger applications where CSS comes from:

```text
reset
framework
design system
component styles
utilities
third-party packages
```

We'll revisit layers when discussing CSS architecture.

---

# 14. Inheritance

Some CSS properties naturally inherit from parent to child.

For example:

```css
body {
  color: #222;
  font-family: system-ui;
}
```

A child:

```html
<div>
  <p>Hello</p>
</div>
```

will often inherit:

```text
color
font-family
```

unless another rule overrides it.

This gives you:

```text
parent
 ↓
default visual environment
 ↓
children inherit
```

That's extremely useful for themes.

---

# 15. Not everything inherits

For example:

```css
.parent {
  margin: 20px;
}
```

doesn't mean:

```text
children also get margin: 20px
```

Likewise many layout/box properties don't inherit.

So don't think:

```text
CSS = everything flows from parent
```

Think:

```text
specific properties have inheritance behavior
```

---

# 16. React component boundaries do not stop CSS inheritance

This is important.

Suppose:

```jsx
function App() {
  return (
    <Theme>
      <Card />
    </Theme>
  );
}
```

Even though:

```text
App
 ↓
Theme
 ↓
Card
```

are React components, the browser only sees the final DOM.

If the DOM is:

```html
<div class="theme-dark">
  <article class="card">
    ...
  </article>
</div>
```

then CSS inheritance can flow through it normally.

React component boundaries do not create CSS isolation by themselves.

---

# 17. The box model

Every normal CSS box can be understood as:

```text
┌───────────────────────────────┐
│            margin             │
│   ┌───────────────────────┐   │
│   │        border         │   │
│   │  ┌─────────────────┐  │   │
│   │  │     padding     │  │   │
│   │  │ ┌─────────────┐ │  │   │
│   │  │ │   content   │ │  │   │
│   │  │ └─────────────┘ │  │   │
│   │  └─────────────────┘  │   │
│   └───────────────────────┘   │
└───────────────────────────────┘
```

This explains a huge amount of CSS behavior.

---

# 18. Content-box vs border-box

By default:

```css
box-sizing: content-box;
```

means:

```text
declared width
=
content width
```

Padding and border are added outside that width.

Suppose:

```css
.card {
  width: 300px;
  padding: 20px;
  border: 2px solid;
}
```

Total width becomes:

```text
300
+ 20 left
+ 20 right
+ 2 left border
+ 2 right border

= 344px
```

This surprises people constantly.

---

# 19. Why `border-box` is so popular

With:

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

the declared width includes:

```text
content
+
padding
+
border
```

So:

```css
.card {
  width: 300px;
  padding: 20px;
  border: 2px solid;
}
```

has:

```text
total width = 300px
```

and the content area becomes:

```text
300
- 40 padding
- 4 border
=
256px
```

This is generally easier to reason about.

A common reset includes this rule.

---

# 20. Margin vs padding

Think:

```text
padding
= space inside the box

margin
= space outside the box
```

For:

```text
card
```

padding controls:

```text
content ↔ card edge
```

margin controls:

```text
card ↔ neighboring elements
```

For example:

```css
.card {
  padding: 1rem;
  margin-bottom: 1rem;
}
```

means:

```text
content
   ↕
inside space
   ↕
card boundary
   ↕
outside space
   ↕
next element
```

This distinction should become instinctive.

---

# 21. `display`

The `display` property fundamentally affects how an element participates in layout.

Important values:

```text
block
inline
inline-block
flex
grid
none
contents
```

---

# 22. Block

Typical block elements:

```text
div
section
article
p
h1
```

Conceptually:

```text
block
 ↓
takes its own line
```

For example:

```html
<div>A</div>
<div>B</div>
```

typically produces:

```text
A
B
```

---

# 23. Inline

Inline content flows within a line:

```html
<span>A</span>
<span>B</span>
```

produces something like:

```text
A B
```

Inline layout follows text-flow rules rather than behaving like independent block rectangles.

This matters when you try to apply:

```text
width
height
vertical alignment
```

to inline elements and wonder why behavior differs from a block.

---

# 24. `inline-block`

This combines ideas:

```text
inline participation
+
box-like dimensions
```

So multiple elements can sit on the same line while still accepting box dimensions more like blocks.

It's less central today because Flexbox and Grid are generally superior layout tools, but you should understand it.

---

# 25. `display: none`

```css
.menu {
  display: none;
}
```

means the element isn't participating in normal layout.

Compare that with:

```css
visibility: hidden;
```

The second usually keeps layout space while making the element invisible.

And:

```css
opacity: 0;
```

makes it transparent, but it still occupies layout space and can still participate in interaction depending on other properties.

So:

```text
display:none
→ removed from layout

visibility:hidden
→ box remains, not visible

opacity:0
→ box remains, visually transparent
```

These are not interchangeable.

---

# 26. `pointer-events`

Suppose:

```css
.overlay {
  opacity: 0;
}
```

The element might still intercept pointer interaction.

You may therefore use:

```css
.overlay {
  opacity: 0;
  pointer-events: none;
}

.overlay--visible {
  opacity: 1;
  pointer-events: auto;
}
```

Now the visual and interaction states agree.

This becomes particularly useful for menus, overlays, and transitions.

---

# 27. Normal flow

Before learning positioning, understand normal flow.

The browser generally lays things out according to:

```text
block flow
inline flow
available space
content dimensions
parent constraints
```

For example:

```html
<div>Header</div>
<div>Main</div>
<div>Footer</div>
```

naturally stacks in normal block flow.

Most UI should start in normal flow.

Then use:

```text
Flexbox
Grid
positioning
```

only when the layout actually requires them.

---

# 28. Positioning

Important values:

```text
static
relative
absolute
fixed
sticky
```

---

# 29. `static`

The default.

```css
.element {
  position: static;
}
```

The element participates in normal layout.

Offsets like:

```css
top: 10px;
left: 10px;
```

do not meaningfully reposition a static element.

---

# 30. `relative`

```css
.element {
  position: relative;
}
```

The element still occupies its normal location, but now offsets can reposition it relative to that position.

More importantly:

> A positioned ancestor can become the containing block for absolutely positioned descendants.

This second use is extremely important.

---

# 31. `absolute`

Suppose:

```jsx
<div className="card">
  <button className="close">×</button>
</div>
```

CSS:

```css
.card {
  position: relative;
}

.close {
  position: absolute;
  top: 8px;
  right: 8px;
}
```

The browser conceptually sees:

```text
.card
  ↓
positioned containing block
  ↓
.close absolute positioning
```

So:

```text
top: 8px
right: 8px
```

are relative to the relevant containing block.

---

# 32. Why `position: relative` appears everywhere

You may see:

```css
.card {
  position: relative;
}
```

even when no visible offset is applied.

Often the reason is:

```text
"I need this element to become the positioning context for a child."
```

Example:

```text
card
 ├── image
 ├── content
 └── badge  ← absolute
```

The card establishes the containing block.

---

# 33. `fixed`

```css
position: fixed;
```

places an element relative to the viewport or appropriate containing context in ways distinct from normal document flow.

Typical examples:

```text
toast
floating action button
fixed header
modal overlay
```

Because fixed elements are removed from normal flow, they don't push surrounding content around.

---

# 34. `sticky`

`sticky` is fascinating because it combines normal flow with a threshold.

```css
.header {
  position: sticky;
  top: 0;
}
```

Conceptually:

```text
normal document flow
      ↓
scroll
      ↓
reaches threshold
      ↓
sticks within its scroll container
```

It's not simply:

```text
fixed
```

because its behavior depends on its scroll container and containing environment.

---

# 35. Containing blocks

Many confusing CSS positioning bugs become understandable once you ask:

> **"What is the containing block?"**

For an absolutely positioned element:

```text
absolute child
      ↓
find relevant containing block
      ↓
apply top/right/bottom/left
```

This explains why:

```css
position: relative;
```

on a parent changes where the child appears.

---

# 36. Overflow

Suppose:

```css
.card {
  width: 300px;
  height: 200px;
}
```

and content is larger.

What happens?

`overflow` controls it:

```css
overflow: visible;
overflow: hidden;
overflow: auto;
overflow: scroll;
```

---

# 37. `overflow: hidden`

```css
.card {
  overflow: hidden;
}
```

means content extending outside the box can be clipped.

This is useful for:

```text
image cropping
rounded-card clipping
custom scroll areas
animation masking
```

But it can have side effects.

For example, it can interfere with:

```text
positioned descendants
shadows
sticky behavior
```

depending on the layout.

So don't add `overflow: hidden` as a magical "fix."

Understand what you're clipping.

---

# 38. `overflow: auto`

```css
.panel {
  overflow: auto;
}
```

creates scrolling when content exceeds available space.

This means the browser now has a scroll container.

That detail matters later for:

```text
sticky
scroll snapping
infinite lists
virtualization
```

---

# 39. Why nested scroll containers cause problems

Imagine:

```text
viewport
 └── page
      └── panel
           └── content
```

and both:

```css
page {
  overflow: auto;
}

panel {
  overflow: auto;
}
```

Now there can be multiple scroll contexts.

Then features like:

```text
sticky
scrollIntoView
wheel behavior
keyboard scrolling
```

can become confusing.

A good UI architecture avoids unnecessary scroll containers.

---

# 40. `z-index`

People often think:

```text
z-index: 999999;
```

means:

> "Put this on top of everything."

Not necessarily.

`z-index` operates within **stacking contexts**.

This is a much deeper concept.

---

# 41. Stacking contexts

Imagine:

```text
Page
 ├── Header
 ├── Content
 └── Modal
```

You might do:

```css
.modal {
  position: fixed;
  z-index: 1000;
}
```

and expect it on top.

But if another ancestor creates a different stacking context, the actual layering can behave differently than you expect.

That's why:

```text
z-index: 9999
```

sometimes loses to:

```text
z-index: 10
```

It isn't necessarily a numeric comparison across the entire page.

The elements may belong to different stacking contexts.

---

# 42. What can create stacking contexts?

Common triggers include things like:

```text
position + z-index
opacity less than 1
transform
filter
isolation
certain containment properties
```

This matters for:

```text
dropdowns
modals
tooltips
sticky headers
portals
animations
```

We'll revisit stacking contexts in the advanced UI lesson.

---

# 43. React portals solve some layering problems differently

Recall our earlier portal lesson:

```jsx
createPortal(
  <Modal />,
  document.body
)
```

The modal can move in the DOM to:

```text
body
```

while remaining logically part of the same React tree.

This can simplify layering because the modal is no longer trapped inside a deeply nested DOM container with:

```text
overflow: hidden
transform
stacking contexts
```

But portals don't automatically solve every positioning problem.

---

# 44. Width and height

A common mistake is:

```css
width: 100%;
```

thinking:

> "Take all available screen width."

It really means:

> "Take 100% of the relevant containing block's available sizing context."

So:

```text
parent = 500px
child width = 100%
```

means roughly:

```text
child = 500px
```

not:

```text
child = viewport width
```

---

# 45. `height: 100%` is trickier

This:

```css
.child {
  height: 100%;
}
```

requires a definite parent height in many cases.

For example:

```css
.parent {
  height: 500px;
}

.child {
  height: 100%;
}
```

is straightforward.

But:

```css
.parent {
  height: auto;
}

.child {
  height: 100%;
}
```

doesn't give the browser a definite reference height.

This is why:

```text
"height: 100% doesn't work"
```

is such a common complaint.

---

# 46. `min-height: 100vh` vs `height: 100vh`

Suppose:

```css
.page {
  height: 100vh;
}
```

You are telling the element:

```text
exactly viewport height
```

That can cause problems if content becomes taller.

Whereas:

```css
.page {
  min-height: 100vh;
}
```

means:

```text
at least viewport height
but can grow
```

For page layouts, `min-height` is often more flexible.

Modern mobile viewport behavior also makes viewport units more nuanced, which we'll cover in responsive design.

---

# 47. Units

Important CSS units include:

```text
px
%
em
rem
vw
vh
dvh
svh
lvh
ch
ex
fr
```

We'll learn them carefully later, but the basic distinction is:

### `px`

Absolute CSS pixel unit.

### `%`

Relative to another dimension/property context.

### `rem`

Relative to root font size.

### `em`

Relative to the relevant element's font size, which can compound.

### `vw`

Viewport width.

### `vh`

Viewport height.

### `fr`

Grid's fractional track unit.

---

# 48. `rem` vs `em`

Suppose root font size is:

```text
16px
```

Then:

```css
padding: 2rem;
```

is:

```text
32px
```

because:

```text
2 × 16 = 32
```

Now `em`:

```css
.parent {
  font-size: 20px;
}

.child {
  font-size: 1.5em;
}
```

Child becomes:

```text
30px
```

because:

```text
1.5 × 20 = 30
```

This is why nested `em` values can compound.

---

# 49. Why design systems often use `rem`

Suppose your spacing scale is:

```text
0.25rem
0.5rem
0.75rem
1rem
1.5rem
2rem
```

Now spacing remains tied to the root sizing system rather than arbitrary pixel values everywhere.

For example:

```css
.card {
  padding: 1rem;
  gap: 0.75rem;
}
```

This becomes easier to standardize.

---

# 50. CSS variables

This will become extremely important later.

```css
:root {
  --color-primary: #2563eb;
  --surface: white;
  --text: #111827;
  --space-md: 1rem;
}
```

Then:

```css
.button {
  background: var(--color-primary);
}

.card {
  background: var(--surface);
  color: var(--text);
  padding: var(--space-md);
}
```

Now the values become part of a design vocabulary.

---

# 51. CSS variables are not React state

This distinction matters.

CSS variable:

```css
--surface: white;
```

belongs to:

```text
CSS/browser styling system
```

React state:

```js
const [theme, setTheme] = useState('dark');
```

belongs to:

```text
React/application logic
```

React may change a class or data attribute:

```jsx
<div data-theme={theme}>
```

and CSS variables respond:

```css
[data-theme="dark"] {
  --surface: #111;
  --text: #fff;
}
```

This is an excellent division of responsibility.

---

# 52. Data attributes are useful styling hooks

React:

```jsx
<button
  data-variant="danger"
  data-size="small"
>
  Delete
</button>
```

CSS:

```css
button[data-variant="danger"] {
  /* danger appearance */
}

button[data-size="small"] {
  /* size */
}
```

This can be useful for state/variant styling.

However, don't add data attributes randomly.

Use them when the DOM state is useful as part of the component contract or CSS architecture.

---

# 53. Pseudo-classes

CSS can react to browser interaction state:

```css
button:hover
button:focus
button:focus-visible
button:active
button:disabled
input:invalid
```

React doesn't have to track these states.

For example:

```css
.button:hover {
  transform: translateY(-1px);
}
```

The browser handles it.

This dramatically reduces unnecessary React state.

---

# 54. `:focus` vs `:focus-visible`

This is important for accessibility.

```css
button:focus {
  outline: 2px solid;
}
```

can show focus styles whenever focus exists.

But:

```css
button:focus-visible {
  outline: 2px solid;
}
```

lets the browser apply focus visibility based on interaction modality heuristics.

This often produces a better experience for keyboard users without unnecessarily displaying strong focus indication for every pointer interaction.

We'll go much deeper into this during the design/accessibility lesson.

---

# 55. Pseudo-elements

CSS can generate visual content with:

```css
::before
::after
```

For example:

```css
.badge::before {
  content: '';
  display: inline-block;
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}
```

This is useful for purely decorative pieces.

Don't use pseudo-elements to store meaningful application content that users need to access semantically.

---

# 56. `currentColor`

A beautiful CSS feature:

```css
.icon {
  color: blue;
}

.icon-circle {
  fill: currentColor;
}
```

`currentColor` means:

> Use the element's current `color`.

This lets icons, borders, SVGs, and decorative pieces naturally follow the component's text color.

It's very useful in design systems.

---

# 57. CSS nesting

Modern CSS supports nesting patterns like:

```css
.card {
  padding: 1rem;

  &:hover {
    transform: translateY(-2px);
  }

  .title {
    font-weight: 600;
  }
}
```

Whether you use nesting directly depends on your tooling/browser support targets and team conventions.

The underlying concept is still ordinary CSS selectors.

Don't confuse nesting with component-scoped CSS.

---

# 58. CSS does not know your React component boundaries

Suppose:

```text
<Card>
  <Button />
</Card>
```

You might think:

```text
Card CSS
Button CSS
```

are inherently isolated because they're different components.

They are not.

With global CSS:

```css
.card button {
  ...
}
```

can affect `Button`.

That's one reason larger React projects need a styling architecture.

We'll eventually compare:

```text
global CSS
CSS Modules
CSS-in-JS
utility CSS
design-system CSS
```

---

# 59. CSS selectors can accidentally leak

Imagine:

```css
button {
  border: none;
}
```

That affects:

```text
every button in the application
```

not just your `PrimaryButton`.

This can be useful for global resets:

```text
all buttons
```

but dangerous if accidental.

So when writing component-specific CSS, scope it intentionally.

---

# 60. A practical component stylesheet

Suppose:

```jsx
function UserCard({ user }) {
  return (
    <article className="user-card">
      <img
        className="user-card__avatar"
        src={user.avatar}
        alt=""
      />

      <div className="user-card__content">
        <h2 className="user-card__name">
          {user.name}
        </h2>
      </div>
    </article>
  );
}
```

Then:

```css
.user-card {
  display: flex;
  gap: 1rem;
  padding: 1rem;
  border-radius: 0.75rem;
}

.user-card__avatar {
  width: 48px;
  height: 48px;
  border-radius: 50%;
}

.user-card__name {
  margin: 0;
}
```

This style is deliberately boring.

That's good.

Boring CSS is often easier to maintain than clever CSS.

---

# 61. Naming classes

A convention such as:

```text
user-card
user-card__avatar
user-card__name
user-card--compact
```

can make relationships obvious.

This resembles BEM:

```text
block
block__element
block--modifier
```

You don't have to use BEM.

But the underlying idea is useful:

```text
component
 ↓
part
 ↓
variant
```

We'll compare naming systems later.

---

# 62. React variant props

You can combine that with a component API:

```jsx
<Card variant="compact" />
```

and:

```jsx
function Card({ variant = 'default' }) {
  return (
    <article
      className={[
        'card',
        `card--${variant}`
      ].join(' ')}
    >
      ...
    </article>
  );
}
```

CSS:

```css
.card {
  padding: 1rem;
}

.card--compact {
  padding: 0.5rem;
}
```

This gives a clear flow:

```text
component API
 ↓
variant
 ↓
class
 ↓
CSS rules
```

---

# 63. But don't create dozens of variants

Avoid APIs like:

```jsx
<Card
  size="small"
  density="tight"
  tone="dark"
  border="subtle"
  elevation="medium"
  rounded="large"
  spacing="compact"
/>
```

Now the component has a combinatorial design surface.

A design system should expose meaningful concepts, not every possible combination of CSS properties.

---

# 64. Shorthands

CSS has many shorthand properties:

```css
margin: 1rem 2rem;
padding: 1rem;
border: 1px solid;
background: ...;
font: ...;
```

For example:

```css
margin: top right bottom left;
```

with missing values being inferred according to CSS's shorthand rules.

When debugging, DevTools expands these into their individual sides.

Learning to read both forms is important.

---

# 65. Logical properties

This is a more advanced but very important modern CSS concept.

Instead of:

```css
margin-left: 1rem;
```

you can use:

```css
margin-inline-start: 1rem;
```

Instead of:

```css
padding-top: 1rem;
```

you can use:

```css
padding-block-start: 1rem;
```

Why?

Because `left/right` assume a particular writing direction.

Logical properties work better with:

```text
LTR
RTL
different writing modes
internationalization
```

We'll cover this much more deeply when discussing production UI architecture.

---

# 66. CSS and internationalization

Suppose:

```text
English:
icon → text

Arabic:
text ← icon
```

If your layout blindly relies on:

```css
margin-left
```

you may create problems.

Logical properties:

```css
margin-inline-start
```

allow CSS to adapt more naturally to writing direction.

This is one reason good CSS architecture isn't only about appearance.

It's about representing UI structure correctly.

---

# 67. Color is more than hex values

You already know:

```css
color: #2563eb;
```

But production systems often think in terms of:

```text
semantic roles
```

such as:

```text
--color-text
--color-surface
--color-border
--color-primary
--color-danger
--color-success
```

rather than:

```text
--blue-500
--gray-700
```

Why?

Because components care about meaning.

```text
danger
```

is a semantic role.

The actual red tone can change later.

---

# 68. CSS custom properties enable theming

For example:

```css
:root {
  --color-surface: white;
  --color-text: #111;
  --color-primary: #2563eb;
}

[data-theme='dark'] {
  --color-surface: #111827;
  --color-text: white;
  --color-primary: #60a5fa;
}
```

Then:

```css
body {
  background: var(--color-surface);
  color: var(--color-text);
}
```

React only needs to choose:

```jsx
document.documentElement.dataset.theme =
  theme;
```

or render a theme attribute somewhere appropriate.

CSS handles the actual visual transformation.

---

# 69. React + CSS architecture

Now let's define the relationship cleanly.

```text
React
├── component structure
├── state
├── events
├── semantics
├── data attributes/classes
└── conditional rendering

CSS
├── layout
├── colors
├── typography
├── spacing
├── visual states
├── responsive behavior
├── animation
└── theme presentation
```

That division is a very strong default.

---

# 70. What belongs in CSS vs React?

A useful decision table:

|Problem|Usually belongs in|
|---|---|
|Hover background|CSS|
|Keyboard focus ring|CSS|
|Responsive columns|CSS|
|Grid layout|CSS|
|Dark theme colors|CSS|
|Modal open/closed state|React|
|Selected task|React|
|Which variant to render|React|
|Whether an element exists|React|
|Whether a panel is disabled|React + CSS|
|Server error message|React|
|Animation between application states|React triggers + CSS/animation system|

This isn't absolute.

It's a starting point.

---

# 71. React controls _whether_ something exists

Suppose:

```jsx
{open && <Menu />}
```

React decides:

```text
Menu exists
```

CSS decides:

```text
how Menu looks
```

Alternatively, CSS can keep it mounted and change appearance:

```jsx
<Menu open={open} />
```

with:

```css
.menu {
  opacity: 0;
}

.menu--open {
  opacity: 1;
}
```

Choosing between these approaches affects:

```text
accessibility
animation
layout
state preservation
performance
```

We'll study this deeply in the animation/modal lesson.

---

# 72. CSS transitions

Suppose:

```css
.button {
  background: blue;
  transition: background-color 150ms ease;
}

.button:hover {
  background: darkblue;
}
```

The browser interpolates the transition.

React doesn't need to manage:

```text
start timer
calculate frame
update state
```

The browser handles that.

This is another recurring principle:

> **Don't use JavaScript to manually implement behavior that the browser already provides efficiently.**

---

# 73. Transforms

Transforms:

```css
transform: translateY(-2px);
transform: scale(1.02);
transform: rotate(5deg);
```

change visual geometry without directly changing normal document flow in the same way as layout properties.

That's why transforms are frequently used for animations.

Example:

```css
.card {
  transition: transform 150ms ease;
}

.card:hover {
  transform: translateY(-2px);
}
```

---

# 74. Layout properties vs visual properties

This becomes important for performance.

Changes to things like:

```text
width
height
top
left
```

can interact with layout.

Changes to:

```text
transform
opacity
```

are often better candidates for efficient animation paths.

This isn't a guarantee that every transform is "free," but it is a useful browser-rendering principle.

We'll go deeper into this later.

---

# 75. The browser rendering pipeline

A simplified model:

```text
style recalculation
      ↓
layout
      ↓
paint
      ↓
composite
```

Different CSS changes can affect different stages.

For example:

```text
color
→ often style + paint

width
→ style + layout + paint

transform
→ often style + composite-oriented work
```

The exact behavior depends on the browser and context, but this model is useful for performance reasoning.

---

# 76. `will-change`

You may encounter:

```css
will-change: transform;
```

It tells the browser:

> "This property may change soon."

But don't sprinkle it across everything.

Excessive `will-change` can consume resources and create unnecessary complexity.

It's an optimization hint, not a magic performance switch.

---

# 77. CSS reset / baseline

Browsers apply default styles.

For example:

```text
body margin
heading margins
button styles
form control appearance
```

Applications commonly normalize some of these defaults.

A minimal example:

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}

html,
body {
  margin: 0;
  padding: 0;
}

button,
input,
textarea,
select {
  font: inherit;
}
```

The exact reset should be intentional.

Don't remove browser defaults blindly.

---

# 78. Don't destroy useful browser behavior

For example:

```css
button {
  outline: none;
}
```

is often a mistake.

It can remove visible keyboard focus.

Instead, define an accessible focus style:

```css
button:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```

A reset should normalize, not destroy accessibility.

---

# 79. Typography is layout

People often treat typography as decoration.

It isn't.

Changing:

```text
font
font-size
line-height
font-weight
letter-spacing
```

changes:

```text
element dimensions
wrapping
layout
vertical rhythm
```

For example:

```css
.title {
  font-size: 2rem;
  line-height: 1.1;
}
```

The `line-height` directly influences how text occupies vertical space.

Typography therefore belongs in the layout system.

---

# 80. `line-height`

You can write:

```css
line-height: 1.5;
```

rather than:

```css
line-height: 24px;
```

The unitless approach scales relative to the element's font size.

For example:

```text
font-size = 16px
line-height = 1.5
→ 24px
```

Then:

```text
font-size = 20px
line-height = 1.5
→ 30px
```

This is often useful for scalable typography.

---

# 81. Text overflow

Suppose:

```css
.title {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}
```

Then:

```text
Very long project name that doesn't...
```

appears.

Notice three properties working together:

```text
white-space
overflow
text-overflow
```

This is a good example of why memorizing single properties isn't enough.

CSS behavior often emerges from **combinations**.

---

# 82. Why CSS debugging should be systematic

Suppose your button isn't red.

Don't randomly try:

```css
!important
```

Instead inspect:

```text
1. Is the class actually on the DOM?
2. Does the selector match?
3. Is the stylesheet loaded?
4. Is the declaration crossed out in DevTools?
5. Which rule wins?
6. Is the value inherited?
7. Is a CSS variable resolving correctly?
8. Is another state selector winning?
9. Is a pseudo-element responsible for what you're looking at?
```

This is the CSS equivalent of the React debugging methodology we learned.

---

# 83. React DevTools vs browser DevTools

When debugging a visual bug, you need both perspectives.

React DevTools tells you:

```text
component
props
state
tree
```

Browser DevTools tells you:

```text
DOM
computed styles
matched CSS rules
layout
box model
events
network
paint/performance
```

For styling problems, the browser DevTools is usually the authoritative place to inspect CSS behavior.

---

# 84. Computed styles

Suppose:

```css
.button {
  color: blue;
}

button {
  color: black;
}
```

DevTools may show:

```text
color: blue
```

as the final computed value.

You can inspect the cascade and see which declaration won.

This is vastly faster than guessing.

---

# 85. Why a CSS value can look correct but still be wrong

Consider:

```css
color: var(--text);
```

If:

```css
--text: red;
```

is inherited from an ancestor, the component may visually respond to a value you don't see in the local stylesheet.

CSS variables participate in the cascade and inheritance.

That's why design systems often deliberately scope variables.

---

# 86. Parent variables

Suppose:

```css
.theme {
  --surface: white;
}

.theme-dark {
  --surface: #111;
}
```

Children automatically see the appropriate variable:

```css
.card {
  background: var(--surface);
}
```

This gives you:

```text
ancestor
 ↓
CSS variable
 ↓
descendants
```

which is essentially dependency injection for styles.

That's why CSS variables are so powerful in React component systems.

---

# 87. CSS and component libraries

Suppose you use a component library:

```jsx
<Button variant="primary">
  Save
</Button>
```

The library may internally use:

```text
CSS
CSS variables
CSS-in-JS
utility classes
data attributes
```

You don't need to know the implementation immediately.

But understanding CSS fundamentals lets you reason about:

```text
why an override works
why it doesn't
how variants are implemented
where theme values come from
why specificity matters
```

That is much more valuable than memorizing library props.

---

# 88. Inline styles

React supports:

```jsx
<div
  style={{
    width: `${progress}%`,
    transform: `translateX(${offset}px)`
  }}
/>
```

This is useful for values that are naturally data-driven.

For example:

```text
progress = 73%
```

You could produce:

```css
width: 73%;
```

from React.

But don't use inline styles for your entire application.

---

# 89. Why not?

Inline styles make things like:

```text
pseudo-classes
media queries
complex selectors
keyframes
cascade organization
```

awkward or impossible to express directly.

For example:

```css
button:hover { ... }
```

doesn't naturally map to a plain inline style object.

So a good pattern is often:

```text
CSS
→ stable visual rules

React style / CSS variable
→ dynamic runtime values
```

---

# 90. CSS custom properties bridge React and CSS beautifully

Instead of:

```jsx
<div
  style={{
    backgroundColor: userColor
  }}
/>
```

you can sometimes use:

```jsx
<div
  style={{
    '--accent': userColor
  }}
  className="profile-card"
>
```

then:

```css
.profile-card {
  border-color: var(--accent);
}

.profile-card__avatar {
  box-shadow:
    0 0 0 3px var(--accent);
}
```

Now React supplies one dynamic value and CSS uses it throughout the component.

This is an extremely useful pattern.

---

# 91. React + CSS variables + themes

This gives us a powerful architecture:

```text
React
 ↓
theme = dark
 ↓
data-theme="dark"
 ↓
CSS variables change
 ↓
entire visual system changes
```

Rather than:

```text
React
 ↓
set 500 individual style properties
```

This is exactly the kind of separation we want.

---

# 92. CSS architecture matters because applications get large

Imagine:

```text
Orbit
├── Header
├── Sidebar
├── ProjectCard
├── TaskBoard
├── TaskCard
├── TaskEditor
├── Modal
├── Toast
├── CommentBox
└── Settings
```

At first:

```text
one CSS file
```

may be fine.

Eventually:

```text
4000 selectors
```

becomes difficult to reason about.

Then you need decisions around:

```text
scope
naming
layers
tokens
components
utilities
themes
```

That's what the later styling lessons will solve.

---

# 93. What not to do: style based on DOM accidents

Suppose:

```css
.project-page > div > div > button {
  ...
}
```

This is fragile.

A React refactor:

```text
add wrapper
```

can break styling.

Prefer intentional hooks:

```css
.project-toolbar {
  ...
}
```

This is analogous to React's architecture lesson:

> Depend on meaningful contracts, not incidental structure.

---

# 94. CSS should reflect component intent

For:

```jsx
<TaskCard />
```

styles should represent concepts such as:

```text
task-card
task-card__title
task-card__meta
task-card--selected
task-card--dragging
```

rather than:

```text
div-3
row-middle
padding-left-thing
```

Semantic styling is easier to maintain.

---

# 95. State classes

A powerful convention:

```text
component
component--variant
component.is-active
component[data-state="open"]
```

For example:

```jsx
<div
  className="dialog"
  data-state={open ? 'open' : 'closed'}
>
```

Then:

```css
.dialog[data-state="open"] {
  opacity: 1;
}
```

This is especially useful when a component has a well-defined state machine.

---

# 96. CSS and React state machines

Remember our reducer lesson:

```text
idle
pending
success
error
```

You can represent visual state:

```jsx
<div data-state={state}>
```

then:

```css
.form[data-state="pending"] { ... }
.form[data-state="error"] { ... }
.form[data-state="success"] { ... }
```

This connects styling directly to the application's state model.

That's a very clean architecture.

---

# 97. A complete mini-example

React:

```jsx
function SaveButton({ state }) {
  return (
    <button
      className="save-button"
      data-state={state}
      disabled={state === 'pending'}
    >
      {state === 'pending'
        ? 'Saving...'
        : 'Save'}
    </button>
  );
}
```

CSS:

```css
.save-button {
  padding: 0.625rem 1rem;
  border: 0;
  border-radius: 0.5rem;
  background: var(--color-primary);
  color: white;
  transition:
    transform 150ms ease,
    opacity 150ms ease;
}

.save-button:hover:not(:disabled) {
  transform: translateY(-1px);
}

.save-button[data-state="pending"] {
  opacity: 0.7;
}

.save-button[data-state="error"] {
  --color-primary: var(--color-danger);
}

.save-button:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```

Observe the responsibilities.

React decides:

```text
state
label
disabled
```

CSS decides:

```text
spacing
color
hover
focus
animation
visual pending/error appearance
```

That is exactly the separation we want.

---

# 98. What this teaches us about React styling

A lot of "React styling" is actually:

```text
React
→ produce meaningful DOM state

CSS
→ represent that state visually
```

This is much cleaner than:

```text
React
→ manually manipulate every visual property
```

---

# 99. The three-layer styling model we'll use

For the rest of the styling course, think in three layers.

### Layer 1 — Foundation

```text
CSS reset
typography
colors
spacing
variables
tokens
```

### Layer 2 — Layout

```text
flow
Flexbox
Grid
positioning
responsive behavior
containers
```

### Layer 3 — Components

```text
buttons
cards
forms
modals
menus
task boards
tables
states
animations
```

This gives us structure.

---

# 100. What we're going to cover next

Now that the browser/CSS/React relationship is clear, we'll go much deeper into actual layout.

## Next lesson — S2: Flexbox + Grid in React

We'll build and dissect:

```text
navbar
sidebar
dashboard
responsive card grid
task board
toolbar
form layouts
modal layout
sticky regions
```

We'll cover not only:

```text
display: flex
display: grid
```

but the reasoning behind:

```text
main axis
cross axis
flex-basis
grow/shrink
min-width: 0
auto margins
intrinsic sizing
min-content
max-content
minmax()
auto-fit
auto-fill
grid tracks
alignment
gap
nested layouts
```

and, critically, **why layouts that look simple often break inside real React components**.

That is where the styling section starts becoming production-level rather than just "some CSS around JSX."