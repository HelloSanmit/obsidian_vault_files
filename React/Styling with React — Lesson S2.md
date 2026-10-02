
# Styling with React — Lesson S2

# Flexbox + Grid, Deep Dive

This is one of the most important CSS lessons for React because most real interfaces are built from combinations of:

```text
normal flow
+
Flexbox
+
Grid
```

React gives you the structure.

CSS lays that structure out.

We'll learn the models rather than memorizing property lists.

---

# 1. The first thing to understand: layout is hierarchical

Suppose React renders:

```jsx
function Dashboard() {
  return (
    <div className="dashboard">
      <aside className="sidebar">
        Sidebar
      </aside>

      <main className="main">
        Main
      </main>
    </div>
  );
}
```

The browser sees:

```text
.dashboard
├── .sidebar
└── .main
```

CSS might say:

```css
.dashboard {
  display: flex;
}
```

Now the layout relationship is:

```text
dashboard
  ↓
flex container
  ↓
sidebar + main
```

This is the first fundamental rule:

> **The parent usually controls how its children are laid out.**

This is true for both Flexbox and Grid.

---

# 2. Flexbox is primarily one-dimensional

A flex container lays children out along a primary axis.

Think:

```text
┌─────────────────────────────────────────→
│              MAIN AXIS
│
│   A       B       C
│
└─────────────────────────────────────────
```

Or vertically:

```text
MAIN AXIS
   ↓
   A
   B
   C
```

The direction depends on:

```css
flex-direction
```

Default:

```css
flex-direction: row;
```

So the main axis is horizontal.

---

# 3. Main axis and cross axis

If:

```css
.container {
  display: flex;
  flex-direction: row;
}
```

then:

```text
main axis   → horizontal
cross axis  ↓ vertical
```

If:

```css
.container {
  flex-direction: column;
}
```

then:

```text
main axis   ↓ vertical
cross axis  → horizontal
```

This is the conceptual foundation of Flexbox.

---

# 4. Why this matters for `justify-content` and `align-items`

People often memorize:

```text
justify-content = horizontal
align-items = vertical
```

That's wrong.

The correct rule is:

```text
justify-content
→ main axis

align-items
→ cross axis
```

So with:

```css
flex-direction: row;
```

you get:

```text
justify-content → horizontal
align-items     → vertical
```

But with:

```css
flex-direction: column;
```

you get:

```text
justify-content → vertical
align-items     → horizontal
```

Once you understand this, Flexbox becomes much easier.

---

# 5. The simplest Flexbox example

```jsx
<div className="toolbar">
  <button>Back</button>
  <h1>Projects</h1>
  <button>Settings</button>
</div>
```

```css
.toolbar {
  display: flex;
  align-items: center;
  gap: 1rem;
}
```

Browser:

```text
Back    Projects    Settings
```

The browser places the children along the main axis.

---

# 6. `gap` is usually better than child margins

Instead of:

```css
.toolbar > * {
  margin-right: 1rem;
}
```

use:

```css
.toolbar {
  display: flex;
  gap: 1rem;
}
```

Why?

Because `gap` describes the **relationship between items**.

That means:

```text
A ↔ B
B ↔ C
C ↔ D
```

without having to manually manage:

```text
last child margin
first child margin
direction
wrapping
```

It becomes even more useful with Grid.

---

# 7. `justify-content`

Suppose:

```css
.toolbar {
  display: flex;
  justify-content: center;
}
```

Children move toward the center of the main axis.

Possible values include:

```text
flex-start
center
flex-end
space-between
space-around
space-evenly
```

The important concept is not the memorization.

It is:

```text
available main-axis space
          ↓
justify-content decides distribution
```

---

# 8. `space-between`

Suppose:

```text
container width = 600px
items = 100px, 100px, 100px
```

Total item width:

```text
300px
```

Remaining space:

```text
300px
```

With:

```css
justify-content: space-between;
```

the browser distributes that remaining space **between** the items.

Conceptually:

```text
A -------- B -------- C
```

There is no leading/trailing space under the basic interpretation.

---

# 9. `space-around` and `space-evenly`

These differ in how free space is distributed.

```text
space-around
```

gives half-sized outer spaces relative to inner gaps.

```text
space-evenly
```

makes the spaces equal.

Visually:

```text
space-evenly:

|-- A -- B -- C --|
```

This matters in navigation bars and evenly distributed controls.

---

# 10. `align-items`

Now the cross axis.

Suppose:

```css
.toolbar {
  display: flex;
  align-items: center;
}
```

For a row:

```text
cross axis
    ↓
┌─────────────────────────────┐
│          A B C              │
└─────────────────────────────┘
```

Items are centered vertically within the flex line.

Common values:

```text
stretch
flex-start
center
flex-end
baseline
```

---

# 11. `stretch`

Default behavior in many row layouts is:

```css
align-items: stretch;
```

If children don't have their own cross-axis size, they may stretch across the available cross-axis dimension.

This surprises people when they expect:

```text
small child
```

but get:

```text
child fills container height
```

That's Flexbox doing exactly what it was asked to do.

---

# 12. `baseline`

Typography-heavy layouts often benefit from:

```css
align-items: baseline;
```

Example:

```text
$  100.00   Buy
```

The visual alignment follows text baselines rather than box centers.

This becomes valuable when components contain different font sizes.

---

# 13. `flex-direction`

```css
flex-direction: row;
```

means:

```text
A B C
```

while:

```css
flex-direction: column;
```

means:

```text
A
B
C
```

You can also use:

```text
row-reverse
column-reverse
```

but don't use reverse directions as a substitute for proper semantic ordering when accessibility/reading order matters.

Visual order and document order are not always the same thing.

---

# 14. `flex-wrap`

By default:

```css
flex-wrap: nowrap;
```

means the browser tries to keep items on one line.

Suppose:

```text
container = 400px

items:
150px
150px
150px
```

Total:

```text
450px
```

Something has to happen.

With:

```css
flex-wrap: wrap;
```

the items can move onto a second line.

```text
A       B

C
```

This is useful for:

```text
tag lists
button groups
filters
responsive toolbars
```

---

# 15. Wrapped Flexbox creates multiple flex lines

Once wrapping happens, the cross-axis model changes.

Think:

```text
line 1:  A B
line 2:  C D
line 3:  E
```

Now there are multiple flex lines.

That's where:

```css
align-content
```

becomes relevant.

This property is often confused with `align-items`.

---

# 16. `align-items` vs `align-content`

### `align-items`

Controls items **within a flex line**.

### `align-content`

Controls the **collection of flex lines** when there is extra cross-axis space.

Think:

```text
align-items
→ items inside a line

align-content
→ lines inside the container
```

If there is only one line, `align-content` often has nothing interesting to do.

---

# 17. `flex-grow`

Now we reach the more important part.

Suppose:

```text
container = 600px
A = 100px
B = 100px
C = 100px
```

Remaining space:

```text
300px
```

If:

```css
A {
  flex-grow: 1;
}
```

A can absorb available free space.

Conceptually:

```text
A grows
B stays
C stays
```

---

# 18. Multiple grow values

Suppose:

```css
A {
  flex-grow: 1;
}

B {
  flex-grow: 2;
}
```

Total grow factor:

```text
1 + 2 = 3
```

Free space is divided proportionally.

So:

```text
A gets 1/3
B gets 2/3
```

of the distributable free space, subject to the rest of the Flexbox sizing algorithm and constraints.

---

# 19. `flex-shrink`

This handles lack of space.

Suppose:

```text
container = 300px

A = 200px
B = 200px
```

Total:

```text
400px
```

There is:

```text
100px
```

too much content for the available size.

Flexbox can shrink items according to:

```css
flex-shrink
```

Default:

```text
1
```

means items are shrinkable.

---

# 20. `flex-grow` and `flex-shrink` aren't percentages of width

This misconception is common.

You don't say:

```text
flex-shrink: 20 means shrink by 20%
```

The algorithm uses these factors to distribute free/negative space.

So think:

```text
grow
→ how strongly this item participates in extra space

shrink
→ how strongly this item participates in shortage
```

---

# 21. `flex-basis`

`flex-basis` provides the item's initial main-axis sizing basis before free space distribution.

For example:

```css
.item {
  flex-basis: 200px;
}
```

This says, roughly:

> Start considering this item as 200px along the main axis before grow/shrink resolution.

This is more conceptually precise than treating it as just:

```text
"another width property"
```

---

# 22. The `flex` shorthand

You frequently see:

```css
flex: 1;
```

This is shorthand for a grow/shrink/basis combination according to the CSS shorthand expansion rules.

The practical behavior is often approximately:

```text
grow = 1
shrink = 1
basis = 0%
```

depending on the shorthand form.

The important practical meaning:

> "This item participates in distributing available space."

---

# 23. The classic `flex: 1` layout

Suppose:

```jsx
<div className="layout">
  <aside>Sidebar</aside>
  <main>Main</main>
</div>
```

```css
.layout {
  display: flex;
}

aside {
  width: 240px;
}

main {
  flex: 1;
}
```

Conceptually:

```text
layout = total width
          │
          ├── sidebar = 240px
          │
          └── remaining space
                  ↓
                main
```

This is one of the most useful application-layout patterns.

---

# 24. Why `min-width: 0` is so important in Flexbox

This is a classic real-world React bug.

Suppose:

```jsx
<div className="row">
  <aside>...</aside>
  <main>
    <VeryLongContent />
  </main>
</div>
```

and:

```css
.row {
  display: flex;
}

main {
  flex: 1;
}
```

You may expect:

```text
sidebar | main shrinks
```

but instead:

```text
sidebar | main causes horizontal overflow
```

Why?

Flex items have an automatic minimum size behavior that can prevent shrinking below their content's intrinsic minimum in situations such as long unbroken content or large descendants.

Adding:

```css
main {
  min-width: 0;
}
```

allows the flex item to actually shrink.

This single property fixes many dashboard/layout problems.

---

# 25. Why this happens

Conceptually:

```text
flex item
   ↓
"default minimum size"
   ↓
content says "I need lots of room"
   ↓
item refuses to shrink enough
   ↓
overflow
```

With:

```css
min-width: 0;
```

you're saying:

> The item is allowed to become smaller than its content's intrinsic minimum.

Then the inner content can handle the overflow.

This is especially common in:

```text
React dashboards
sidebar + content layouts
nested flex containers
tables
code blocks
long URLs
```

---

# 26. `min-height: 0` is the vertical equivalent

Suppose:

```text
app shell
 ↓
flex column
 ↓
header
main
```

and `main` should scroll.

You may write:

```css
.app {
  display: flex;
  flex-direction: column;
  height: 100vh;
}

main {
  flex: 1;
  min-height: 0;
  overflow: auto;
}
```

Without:

```text
min-height: 0
```

the content can sometimes force the flex item to grow rather than allowing the intended internal scrolling.

This is one of the most valuable practical Flexbox tricks.

---

# 27. Flexbox and intrinsic sizing

Browsers don't think only in arbitrary pixels.

Content has intrinsic sizes.

For example:

```text
"supercalifragilistic..."
```

has some minimum space requirement depending on wrapping opportunities.

The browser considers:

```text
content size
minimum size
available size
flex constraints
```

So many layout bugs are really:

> **The browser is resolving constraints differently from the assumptions in your mental model.**

---

# 28. `auto` margins in Flexbox

This is a beautiful technique.

Suppose:

```jsx
<header className="header">
  <Logo />
  <Nav />
  <Profile />
</header>
```

```css
.header {
  display: flex;
  align-items: center;
}

.profile {
  margin-left: auto;
}
```

The auto margin absorbs available main-axis space.

Result:

```text
Logo   Nav                         Profile
```

No JavaScript.

No `justify-content: space-between`.

This is especially useful when only one item needs to be pushed away.

---

# 29. `margin-inline-start: auto`

In production CSS, a logical property can be better:

```css
.profile {
  margin-inline-start: auto;
}
```

This respects writing direction more naturally.

We'll keep using logical properties where they improve internationalization.

---

# 30. Flexbox and component APIs

Suppose your React component:

```jsx
<Toolbar
  start={<Search />}
  end={<Actions />}
/>
```

renders:

```jsx
<div className="toolbar">
  <div className="toolbar__start">{start}</div>
  <div className="toolbar__end">{end}</div>
</div>
```

Then:

```css
.toolbar {
  display: flex;
  align-items: center;
}

.toolbar__end {
  margin-inline-start: auto;
}
```

This is a nice example of:

```text
React composition
+
CSS Flexbox
```

working together.

---

# 31. Now: Grid

Flexbox is primarily one-dimensional.

Grid is primarily two-dimensional.

Think:

```text
       columns →
     ┌────┬────┬────┐
rows │ A  │ B  │ C  │
 ↓   ├────┼────┼────┤
     │ D  │ E  │ F  │
     ├────┼────┼────┤
     │ G  │ H  │ I  │
     └────┴────┴────┘
```

Grid lets you reason about:

```text
rows
+
columns
```

simultaneously.

---

# 32. The simplest Grid

```css
.card-grid {
  display: grid;
  grid-template-columns:
    repeat(3, 1fr);
  gap: 1rem;
}
```

Result:

```text
A | B | C
D | E | F
G | H | I
```

The browser creates three columns and places children into the grid.

---

# 33. What is `fr`?

`fr` means fractional share of available grid space.

Suppose:

```css
grid-template-columns:
  1fr 2fr;
```

Total fractions:

```text
1 + 2 = 3
```

The available track space is divided:

```text
first = 1/3
second = 2/3
```

subject to track sizing constraints.

It's conceptually similar to proportional space allocation, but Grid's sizing algorithm is richer.

---

# 34. Fixed + flexible Grid

A very common application layout:

```css
.dashboard {
  display: grid;
  grid-template-columns:
    240px minmax(0, 1fr);
}
```

This means:

```text
240px sidebar
+
remaining flexible content
```

The `minmax(0, 1fr)` deserves attention.

---

# 35. Why `minmax(0, 1fr)`?

You might write:

```css
grid-template-columns:
  240px 1fr;
```

and sometimes experience overflow because the flexible track's automatic minimum sizing can remain larger than expected.

Using:

```css
minmax(0, 1fr)
```

says:

```text
minimum = 0
maximum = 1fr
```

meaning the track is genuinely allowed to shrink to zero if necessary before overflow handling elsewhere.

This is one of the most useful production Grid patterns.

---

# 36. Grid rows

You can explicitly define rows:

```css
grid-template-rows:
  auto
  1fr
  auto;
```

For an application shell:

```text
┌──────────────────────────┐
│ header                   │ auto
├──────────────────────────┤
│                          │
│ main                     │ 1fr
│                          │
├──────────────────────────┤
│ footer                   │ auto
└──────────────────────────┘
```

Very useful.

---

# 37. Grid areas

For complex layouts:

```css
.layout {
  display: grid;
  grid-template-areas:
    "header header"
    "sidebar main"
    "footer footer";

  grid-template-columns:
    240px 1fr;

  grid-template-rows:
    auto 1fr auto;
}
```

Then:

```css
.header {
  grid-area: header;
}

.sidebar {
  grid-area: sidebar;
}

.main {
  grid-area: main;
}

.footer {
  grid-area: footer;
}
```

This is very readable for page-level layouts.

---

# 38. Grid is about tracks

A Grid consists conceptually of:

```text
grid container
├── column tracks
└── row tracks
```

For example:

```css
grid-template-columns:
  200px 1fr 300px;
```

creates three column tracks:

```text
200px | flexible | 300px
```

Children are placed into those tracks.

---

# 39. Explicit vs implicit Grid

You might explicitly define:

```css
grid-template-columns:
  repeat(3, 1fr);
```

but you may only have:

```text
3 rows
```

The browser creates additional rows automatically.

Those are **implicit tracks**.

You can control them with:

```css
grid-auto-rows
grid-auto-columns
grid-auto-flow
```

This is particularly useful for automatically generated React lists.

---

# 40. `repeat()`

Instead of:

```css
grid-template-columns:
  1fr 1fr 1fr 1fr;
```

use:

```css
grid-template-columns:
  repeat(4, 1fr);
```

Much easier to maintain.

---

# 41. Responsive Grid without React

Suppose:

```css
.card-grid {
  display: grid;
  grid-template-columns:
    repeat(3, 1fr);
}
```

Instead of React computing:

```text
if width < 768
  1 column
else
  3 columns
```

let CSS handle viewport layout:

```css
.card-grid {
  display: grid;
  grid-template-columns:
    1fr;
}

@media (min-width: 768px) {
  .card-grid {
    grid-template-columns:
      repeat(3, 1fr);
  }
}
```

No React state required.

---

# 42. `auto-fit` and `minmax`

A more fluid approach:

```css
.card-grid {
  display: grid;

  grid-template-columns:
    repeat(
      auto-fit,
      minmax(240px, 1fr)
    );

  gap: 1rem;
}
```

Now the browser decides how many columns fit.

Conceptually:

```text
wide
[A][B][C][D]

medium
[A][B][C]

narrow
[A][B]

very narrow
[A]
```

You didn't specify:

```text
2 columns
3 columns
4 columns
```

You specified:

> Each column should be at least about 240px and otherwise share available space.

That's a much more resilient design.

---

# 43. `auto-fit` vs `auto-fill`

These are subtle.

```text
auto-fit
```

generally collapses empty tracks, allowing existing items to expand.

```text
auto-fill
```

preserves the track structure even when some tracks are empty.

For many card-grid use cases:

```css
repeat(auto-fit, minmax(...))
```

is the intuitive choice.

But understanding the distinction helps when layouts behave unexpectedly.

---

# 44. Grid gap

Grid:

```css
grid {
  gap: 1rem;
}
```

can use:

```css
row-gap
column-gap
```

or:

```css
gap: 1rem 2rem;
```

meaning:

```text
row gap = 1rem
column gap = 2rem
```

Again, describe relationships directly.

---

# 45. Alignment in Grid

Grid has:

```text
justify-items
align-items
justify-content
align-content
```

This sounds similar to Flexbox but is conceptually richer.

Think:

### `justify-items`

How each grid item aligns horizontally within its grid area.

### `align-items`

How each grid item aligns vertically within its grid area.

### `justify-content`

How the whole grid tracks are distributed horizontally inside the container.

### `align-content`

How the whole grid tracks are distributed vertically.

So:

```text
item alignment
vs
grid alignment
```

is the distinction.

---

# 46. `place-items`

Shortcut:

```css
place-items: center;
```

roughly sets:

```text
align-items: center
justify-items: center
```

Extremely useful for:

```text
loading indicators
empty states
centered icons
simple hero content
```

---

# 47. Centering with Grid

This:

```css
.empty-state {
  display: grid;
  place-items: center;
}
```

can center content in both axes when the container has appropriate dimensions.

It's often simpler than old approaches involving:

```text
absolute
top: 50%
left: 50%
transform
```

---

# 48. When should you use Flexbox vs Grid?

A useful default:

### Flexbox

Use when layout is primarily:

```text
one dimension
```

Examples:

```text
navbar
toolbar
button group
row of metadata
vertical stack
```

### Grid

Use when layout is primarily:

```text
two dimensions
```

Examples:

```text
dashboard
card grid
page shell
form with rows/columns
calendar
complex content layout
```

This isn't a strict rule.

But it's a powerful starting point.

---

# 49. You can combine them

Real applications do this constantly.

Example:

```text
Dashboard → Grid
  │
  ├── Sidebar
  └── Main
       │
       └── Toolbar → Flex
```

CSS:

```css
.dashboard {
  display: grid;
  grid-template-columns:
    240px minmax(0, 1fr);
}

.toolbar {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}
```

This is normal.

There's no requirement to choose one layout system for the whole application.

---

# 50. Nested layout systems are expected

Imagine:

```text
Page Grid
 ├── Sidebar
 └── Main
      ├── Toolbar Flex
      ├── Stats Grid
      └── TaskBoard Grid
           └── TaskCard Flex
```

This is often exactly how complex interfaces should be built.

Each component can own its local layout.

---

# 51. React composition naturally maps to nested layout

React:

```jsx
<Dashboard>
  <Sidebar />
  <main>
    <Toolbar />
    <Stats />
    <TaskBoard />
  </main>
</Dashboard>
```

CSS:

```text
Dashboard → Grid
main      → Stack/Flex
Stats     → Grid
Toolbar   → Flex
TaskBoard → Grid
TaskCard  → Flex
```

This is one reason componentization works so well with CSS layout.

Each component establishes a local layout boundary.

---

# 52. Don't build every layout with absolute positioning

Bad:

```css
.sidebar {
  position: absolute;
  left: 0;
  top: 0;
}

.main {
  position: absolute;
  left: 240px;
}
```

This is brittle.

You have manually recreated layout arithmetic.

Better:

```css
.dashboard {
  display: grid;
  grid-template-columns:
    240px minmax(0, 1fr);
}
```

Let the layout engine do the arithmetic.

Absolute positioning should usually be reserved for elements that are genuinely overlaid or removed from normal flow.

---

# 53. A real Orbit layout

Let's design:

```text
┌─────────────────────────────────────────────┐
│ Header                                      │
├──────────────┬──────────────────────────────┤
│              │ Toolbar                      │
│ Sidebar      ├──────────────────────────────┤
│              │ Stats                        │
│              ├──────────────────────────────┤
│              │ Tasks                        │
│              │                              │
└──────────────┴──────────────────────────────┘
```

CSS:

```css
.app {
  min-height: 100vh;

  display: grid;

  grid-template:
    "header header" auto
    "sidebar main" minmax(0, 1fr)
    / 240px minmax(0, 1fr);
}

.header {
  grid-area: header;
}

.sidebar {
  grid-area: sidebar;
}

.main {
  grid-area: main;

  min-width: 0;
  min-height: 0;
}
```

Notice both:

```text
min-width: 0
min-height: 0
```

Those are deliberately there.

---

# 54. Why `min-width: 0` belongs in real layouts

Suppose TaskBoard contains:

```text
long task title
large table
code block
wide filter
```

Without the correct minimum sizing behavior:

```text
content → pushes layout wider
```

With:

```css
.main {
  min-width: 0;
}
```

the main region can shrink and let its children handle overflow appropriately.

This tiny property is one of the biggest differences between:

```text
tutorial CSS
```

and:

```text
production CSS
```

---

# 55. The "stack" pattern

A huge percentage of UI consists of:

```text
vertical children
```

You can define:

```css
.stack {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
```

Then:

```jsx
<div className="stack">
  <h2>Project</h2>
  <p>Description</p>
  <button>Archive</button>
</div>
```

This creates a reusable layout primitive.

---

# 56. Horizontal cluster

Similarly:

```css
.cluster {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}
```

Now:

```text
icon + text + button
```

can share a layout primitive.

This is the beginning of a design-system layout vocabulary.

---

# 57. Don't confuse spacing with arbitrary margins

Instead of:

```css
.title {
  margin-bottom: 17px;
}

.description {
  margin-bottom: 13px;
}

.button {
  margin-top: 9px;
}
```

you can often model the relationship:

```css
.form-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
```

Then spacing becomes a property of the layout relationship.

This is much easier to maintain.

---

# 58. `gap` and responsive layouts

Suppose the layout switches:

```text
desktop:
row

mobile:
column
```

With Flexbox:

```css
.actions {
  display: flex;
  gap: 1rem;
}

@media (max-width: 700px) {
  .actions {
    flex-direction: column;
  }
}
```

The same `gap` works in the new direction.

That's another reason `gap` is elegant.

---

# 59. The relationship between React and media queries

React doesn't need to know:

```text
"the viewport is 700px"
```

for ordinary visual layout.

CSS already knows.

So don't automatically write:

```jsx
const isMobile = window.innerWidth < 700;
```

just to change layout.

That creates unnecessary JavaScript state and synchronization problems.

Use CSS:

```css
@media (max-width: 700px) {
  ...
}
```

when the behavior is purely presentational.

---

# 60. But sometimes React genuinely needs viewport information

There are cases where application behavior truly depends on viewport/environment.

For example:

```text
virtualization strategy
different interaction model
mobile-specific functionality
hardware capability
```

Then a hook such as `matchMedia` integration may make sense.

But don't use JavaScript merely to do something CSS already handles.

---

# 61. Responsive component design

A component like:

```jsx
<Card />
```

should ideally work when placed into:

```text
300px container
600px container
1000px container
```

This is why modern CSS features like:

```text
container queries
```

are powerful.

The component can respond to **its own available space**, not only the viewport.

We'll cover this in the next responsive-design lesson.

---

# 62. Grid `minmax()` is architectural

Consider a dashboard:

```css
grid-template-columns:
  repeat(3, minmax(0, 1fr));
```

The `0` isn't arbitrary.

You're telling each track:

```text
you may shrink fully if necessary
```

This helps prevent content from unexpectedly forcing the entire grid wider.

A large percentage of reliable CSS is about expressing **minimum constraints correctly**.

---

# 63. Intrinsic sizing keywords

Later you'll see:

```text
min-content
max-content
fit-content()
```

These are ways of asking the browser to size based on content's intrinsic dimensions.

For example:

```css
grid-template-columns:
  max-content 1fr;
```

could give the first column enough space for its content while leaving the remainder flexible.

This is an advanced but very powerful part of Grid.

We'll cover it deeply later.

---

# 64. Why content often breaks your layout

Imagine:

```jsx
<h2>
  ExtremelyLongProjectNameWithoutAnyBreakOpportunity
</h2>
```

The browser now has conflicting goals:

```text
keep content intact
vs
fit available width
```

CSS needs a strategy:

```text
wrapping
overflow
truncation
minimum sizing
```

React doesn't solve this.

This is a browser layout problem.

---

# 65. Common production bug: flex + long text

Example:

```css
.item {
  display: flex;
}

.title {
  flex: 1;
}
```

Then:

```text
long text
```

causes overflow.

Try:

```css
.title {
  min-width: 0;
}
```

and if truncation is intended:

```css
.title {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

This is one of those patterns you'll use constantly in real React applications.

---

# 66. Don't fix layout with random widths

A beginner often does:

```css
width: 73%;
```

until the screen "looks right."

That's usually a symptom of not modeling the layout relationship properly.

Ask:

```text
Should this be:
fixed?
flexible?
content-sized?
grid track?
intrinsic?
clamped?
scrollable?
```

Then choose the correct layout model.

---

# 67. The difference between size and constraint

This is an advanced but powerful thought.

Instead of:

```text
"make this 420px"
```

think:

```text
"it should never be smaller than 240px,
but may grow as space permits."
```

Then write:

```css
minmax(240px, 1fr)
```

That's a constraint-based layout mindset.

Modern CSS becomes much easier when you think in constraints rather than coordinates.

---

# 68. Flexbox vs Grid in Orbit

We can classify our capstone UI.

```text
App shell
 → Grid

Top navigation
 → Flex

Task toolbar
 → Flex

Stats cards
 → Grid

Task columns
 → Grid or Flex depending on behavior

Task metadata
 → Flex

Comment form
 → Flex / vertical stack

Modal
 → Grid/Flex internally

Buttons
 → inline-flex
```

This is realistic.

There is no conflict in using both.

---

# 69. `inline-flex`

Suppose:

```jsx
<button className="button">
  <Icon />
  Save
</button>
```

CSS:

```css
.button {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
}
```

This gives you a flex layout **while participating in inline formatting**.

It's extremely useful for:

```text
buttons
badges
chips
small controls
icons + text
```

---

# 70. `inline-grid`

Same idea:

```css
.inline-layout {
  display: inline-grid;
}
```

Less common, but useful when an element needs Grid behavior while participating inline.

---

# 71. Alignment is not spacing

Suppose:

```css
display: flex;
align-items: center;
```

This doesn't mean:

```text
"put 1rem between children"
```

It means:

```text
align children along cross axis
```

Spacing:

```text
gap
margin
padding
```

Alignment:

```text
align-items
justify-content
place-items
```

Keeping these concepts separate makes layouts much easier to reason about.

---

# 72. A useful layout vocabulary

Instead of thinking:

```text
"How do I position this div?"
```

ask:

```text
Stack?
Cluster?
Grid?
Sidebar layout?
Centered content?
Split pane?
Overlay?
Scrollable region?
```

These are layout patterns.

Once you recognize the pattern, CSS becomes simpler.

---

# 73. Pattern: sidebar layout

```css
.shell {
  display: grid;
  grid-template-columns:
    240px minmax(0, 1fr);
}
```

---

# 74. Pattern: stack

```css
.stack {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
```

---

# 75. Pattern: cluster

```css
.cluster {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}
```

---

# 76. Pattern: split

```css
.split {
  display: flex;
  gap: 1rem;
}

.split__right {
  margin-inline-start: auto;
}
```

---

# 77. Pattern: responsive cards

```css
.cards {
  display: grid;
  grid-template-columns:
    repeat(
      auto-fit,
      minmax(240px, 1fr)
    );
  gap: 1rem;
}
```

These patterns become reusable primitives in a design system.

---

# 78. CSS layout and React component composition

Suppose:

```jsx
<Stack gap="lg">
  <Heading />
  <TaskList />
</Stack>
```

The component API might eventually translate:

```text
gap="lg"
 ↓
CSS custom property / utility
 ↓
CSS gap
```

The component doesn't need to manually position children.

This is how React and CSS can cooperate without making React responsible for the layout algorithm.

---

# 79. A mistake: putting layout in every child

Bad architecture:

```css
.child-a {
  margin-right: 17px;
}

.child-b {
  margin-right: 23px;
}

.child-c {
  margin-left: 11px;
}
```

Better:

```css
.parent {
  display: flex;
  gap: 1rem;
}
```

The parent expresses:

> "These items belong in a horizontal group with a 1rem relationship."

That's a stronger semantic model.

---

# 80. Flexbox and accessibility

CSS visual reordering can be dangerous.

Suppose DOM order:

```text
1. Main
2. Sidebar
```

and CSS does:

```css
grid-template-areas:
  "sidebar main";
```

Visual order changes.

That's often fine.

But using:

```css
order
```

or:

```css
row-reverse
```

can create a mismatch between:

```text
visual order
vs
DOM/accessibility order
```

So visual layout should not casually change semantic reading order.

---

# 81. Flex item `order`

```css
.item {
  order: 2;
}
```

can visually reorder flex items.

But screen readers and keyboard navigation may follow DOM order rather than your visual order.

Therefore:

> Use CSS ordering primarily for presentation adjustments where semantic order remains understandable.

If actual logical order differs, change the React/DOM structure.

---

# 82. Grid can also create visual order differences

Grid placement:

```css
grid-column: 2;
grid-row: 1;
```

doesn't necessarily change DOM order.

Again:

```text
visual position
≠
semantic/document order
```

This is another reason accessibility must be part of layout thinking.

---

# 83. Layout and forms

Forms often work beautifully with Grid.

Example:

```jsx
<form className="form">
  <label>First name</label>
  <input />

  <label>Last name</label>
  <input />

  <label>Email</label>
  <input />
</form>
```

CSS:

```css
.form {
  display: grid;
  grid-template-columns:
    160px minmax(0, 1fr);
  gap: 1rem;
}
```

Result:

```text
First name | [             ]
Last name  | [             ]
Email      | [             ]
```

This is much cleaner than manually positioning every field.

---

# 84. Responsive forms

At smaller widths:

```css
@media (max-width: 700px) {
  .form {
    grid-template-columns: 1fr;
  }
}
```

Now:

```text
First name
[              ]

Last name
[              ]
```

React doesn't need to know about the viewport.

---

# 85. Complex forms: Grid + sub-layouts

For example:

```text
Address
┌─────────────┬──────────────┐
│ Street      │ Apartment    │
├─────────────┼──────────────┤
│ City        │ State        │
├─────────────┼──────────────┤
│ Postal      │ Country      │
└─────────────┴──────────────┘
```

Grid handles the major structure.

Individual field groups can use Flexbox internally.

This nested approach is extremely common.

---

# 86. Task board layout

A project management board is interesting:

```text
┌──────────┬──────────┬──────────┐
│ Open     │ Doing    │ Done     │
│ ──────── │ ──────── │ ──────── │
│ Task A   │ Task C   │ Task F   │
│ Task B   │ Task D   │ Task G   │
│          │ Task E   │          │
└──────────┴──────────┴──────────┘
```

Possible CSS:

```css
.task-board {
  display: grid;
  grid-template-columns:
    repeat(
      3,
      minmax(280px, 1fr)
    );
  gap: 1rem;

  overflow-x: auto;
}
```

Each column:

```css
.task-column {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;

  min-width: 0;
}
```

Each card:

```css
.task-card {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}
```

Three different layout levels:

```text
board → Grid
column → Flex
card → Flex
```

This is exactly what production CSS often looks like.

---

# 87. The most useful browser mental model

When a layout breaks:

```text
1. Who is the layout container?
2. Is it normal flow, Flexbox, or Grid?
3. What axis/tracks exist?
4. What are the item's intrinsic sizes?
5. What minimum constraints apply?
6. Where is free space?
7. Who gets that free space?
8. What is allowed to shrink?
9. What is overflowing?
10. Which element is actually scrolling?
```

This is dramatically better than randomly tweaking margins.

---

# 88. A concrete debugging example

Suppose:

```text
sidebar = 240px
main = flex: 1
```

but the page overflows horizontally.

Check:

```text
main
```

Does it have:

```css
min-width: 0;
```

If not, add it.

Then inspect:

```text
child table
child code block
long text
fixed-width element
```

One descendant may be forcing the minimum width.

This is a constraint problem, not a React problem.

---

# 89. A second debugging example

Suppose:

```text
main should scroll vertically
```

but instead:

```text
entire page grows
```

Check:

```text
flex column parent
```

and make sure the scrolling child can shrink:

```css
main {
  min-height: 0;
  overflow: auto;
}
```

Again:

```text
layout constraint
→ browser behavior
```

not a React state problem.

---

# 90. CSS layout should usually remain declarative

This:

```jsx
const width = window.innerWidth * 0.7;
```

and:

```jsx
style={{
  width
}}
```

should not be your first solution to layout.

Prefer:

```css
width: 70%;
```

or:

```css
grid-template-columns:
  minmax(0, 0.7fr) 0.3fr;
```

The browser is exceptionally good at solving layout constraints.

Let it.

---

# 91. But CSS and React are complementary

Sometimes the application genuinely knows something CSS doesn't.

For example:

```text
"show the sidebar"
```

is application state.

React:

```jsx
<div data-sidebar={open ? 'open' : 'closed'}>
```

CSS:

```css
.layout[data-sidebar='open'] {
  ...
}
```

So:

```text
React decides state
CSS solves layout
```

This division is our recurring pattern.

---

# 92. What you should now understand about Flexbox

Not just:

```text
display: flex;
```

but:

```text
container
 ↓
main axis
cross axis
 ↓
free space
negative space
 ↓
grow/shrink
 ↓
minimum constraints
 ↓
alignment
 ↓
final layout
```

And especially:

```text
min-width: 0
min-height: 0
```

---

# 93. What you should now understand about Grid

Not just:

```text
display: grid;
```

but:

```text
grid container
 ↓
tracks
 ↓
explicit/implicit grid
 ↓
fr units
 ↓
minmax()
 ↓
intrinsic sizing
 ↓
item placement
 ↓
alignment
```

This is the level at which Grid becomes predictable.

---

# 94. The React styling architecture emerging

We're building toward:

```text
React
 ↓
semantic structure
 ↓
component boundaries
 ↓
classes / attributes / CSS variables
 ↓
CSS
 ↓
layout engine
```

And within CSS:

```text
normal flow
 ↓
Flexbox
 ↓
Grid
 ↓
positioning
 ↓
responsive constraints
```

Each layer has a role.

---

# 95. A practical rule for choosing the layout tool

Ask:

> **Am I arranging things in a line or designing an area?**

Line:

```text
Flexbox
```

Area:

```text
Grid
```

Then:

> **Does a child need to overlay another child?**

Maybe:

```text
positioning / portal
```

Then:

> **Could normal flow solve this?**

Try that first.

This simple sequence prevents a lot of overengineering.

---

# 96. One final combined example

React:

```jsx
function ProjectPage() {
  return (
    <div className="project-page">
      <aside className="project-page__sidebar">
        <ProjectNavigation />
      </aside>

      <main className="project-page__main">
        <div className="toolbar">
          <h1>Project Alpha</h1>

          <div className="toolbar__actions">
            <button>Filter</button>
            <button>New task</button>
          </div>
        </div>

        <section className="stats">
          <StatCard />
          <StatCard />
          <StatCard />
        </section>

        <TaskBoard />
      </main>
    </div>
  );
}
```

CSS:

```css
.project-page {
  min-height: 100vh;

  display: grid;

  grid-template-columns:
    240px minmax(0, 1fr);
}

.project-page__main {
  min-width: 0;
  min-height: 0;

  display: flex;
  flex-direction: column;

  gap: 1.5rem;
}

.toolbar {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.toolbar__actions {
  display: flex;
  align-items: center;
  gap: 0.5rem;

  margin-inline-start: auto;
}

.stats {
  display: grid;

  grid-template-columns:
    repeat(
      auto-fit,
      minmax(220px, 1fr)
    );

  gap: 1rem;
}
```

Look at the architecture:

```text
ProjectPage
│
├── page shell       → Grid
│
├── main             → Flex column
│
├── toolbar          → Flex row
│
├── actions          → Flex row
│
└── stats            → responsive Grid
```

This is exactly how you want to think about production UI.

Not:

> "Which random CSS property makes this look right?"

But:

> **"What is the layout relationship between these elements?"**

---

# 97. Final mental model for this lesson

Memorize this:

```text
NORMAL FLOW
→ default layout

FLEXBOX
→ distribute items along one primary axis

GRID
→ design rows + columns together

POSITIONING
→ take something out of normal flow / overlay it

GAP
→ spacing between layout children

AUTO MARGIN
→ absorb available space

MIN-WIDTH: 0
→ allow flex/grid content to actually shrink

MINMAX()
→ express minimum + flexible maximum

1FR
→ flexible share of grid space
```

And the most important rule:

> **Let CSS solve layout. Let React express application state and semantic structure.**

---

# Next — Styling with React S3

Next we move into **responsive design in depth**, including:

```text
media queries
mobile-first architecture
breakpoints
container queries
viewport units
dvh/svh/lvh
fluid typography
clamp()
responsive spacing
responsive grids
intrinsic sizing
art-direction vs responsive layout
mobile navigation patterns
responsive tables/task boards
when JavaScript should and should not know the viewport
```

We'll then connect all of that back to the Orbit capstone rather than learning responsive CSS as isolated examples.