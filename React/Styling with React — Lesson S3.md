
# Styling with React — Lesson S3

# Responsive Design in Depth

Responsive design is often taught as:

```css
@media (max-width: 768px) {
  ...
}
```

That is only the surface.

A production React application needs to answer much deeper questions:

```text
What should change when space decreases?
Who should decide that — React or CSS?
Should layout react to viewport size or container size?
How should typography scale?
What happens on mobile browsers where viewport height is dynamic?
What happens to tables, boards, dialogs, navigation, forms?
When is a different component actually needed?
```

The central principle of this lesson is:

> **Responsive design is primarily about constraints and available space, not about device names.**

---

# 1. Stop thinking in devices

Avoid designing around:

```text
mobile
tablet
desktop
```

as if these were fixed categories.

A user could have:

```text
phone in portrait
phone in landscape
small laptop
split-screen desktop
large monitor
browser window resized to 700px
embedded dashboard inside 500px container
```

So instead of:

```text
"At tablet width, do X."
```

think:

> **"When this layout no longer has enough space for its current arrangement, change it."**

That's a much stronger responsive principle.

---

# 2. The viewport is not the only source of available space

Imagine:

```text
Viewport = 1440px
```

but your component sits inside:

```text
Sidebar
  ↓
Panel
   width = 500px
       ↓
CardGrid
```

The viewport is huge.

The card grid only has:

```text
500px
```

available.

A viewport media query knows:

```text
1440px
```

but your component actually needs to respond to:

```text
500px
```

This is the fundamental motivation for **container queries**.

---

# 3. Media queries

The traditional tool is:

```css
@media (min-width: 768px) {
  .sidebar {
    display: block;
  }
}
```

The browser asks:

```text
Is the viewport at least 768px?
```

If yes, the rule applies.

This is useful for page-level layout.

---

# 4. Mobile-first CSS

Suppose a card is initially designed for a narrow screen.

Write the base rules for narrow layouts:

```css
.card-grid {
  display: grid;
  grid-template-columns: 1fr;
}
```

Then progressively enhance:

```css
@media (min-width: 700px) {
  .card-grid {
    grid-template-columns:
      repeat(2, 1fr);
  }
}
```

and:

```css
@media (min-width: 1100px) {
  .card-grid {
    grid-template-columns:
      repeat(4, 1fr);
  }
}
```

This is called mobile-first because the base stylesheet represents the smaller layout.

---

# 5. Why mobile-first is useful

Your CSS becomes conceptually:

```text
base
 ↓
add capacity
 ↓
add more capacity
```

rather than:

```text
desktop
 ↓
remove things
 ↓
remove more things
 ↓
make smaller
```

This often encourages simpler base rules.

But "mobile-first" is not a moral rule. The important thing is that the responsive system remains understandable.

---

# 6. Breakpoints should come from layout failure

Bad:

```text
iPhone = 390
tablet = 768
desktop = 1024
```

and then mechanically use those values everywhere.

Better:

```text
Card layout works
 ↓
space becomes insufficient
 ↓
switch layout
```

For example:

```css
.cards {
  grid-template-columns:
    repeat(
      auto-fit,
      minmax(240px, 1fr)
    );
}
```

You may not need a breakpoint at all.

CSS can calculate how many cards fit.

---

# 7. Responsive design can often be intrinsic

Compare these two approaches.

### Breakpoint-heavy

```css
.cards {
  grid-template-columns: 1fr;
}

@media (min-width: 700px) {
  .cards {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (min-width: 1100px) {
  .cards {
    grid-template-columns: repeat(4, 1fr);
  }
}
```

### Intrinsic

```css
.cards {
  display: grid;

  grid-template-columns:
    repeat(
      auto-fit,
      minmax(240px, 1fr)
    );
}
```

The second says:

> "Fit as many columns as reasonably possible, but never make one smaller than 240px."

That is a constraint.

It's often more resilient.

---

# 8. A breakpoint should represent a behavioral change

Suppose our navigation:

```text
desktop:
Logo | Projects Tasks Activity | Profile
```

becomes too crowded.

At some point:

```text
Logo | ... | Menu
```

The breakpoint exists because:

```text
the previous composition stops working
```

not because:

```text
"768 is the tablet breakpoint."
```

This difference is critical.

---

# 9. Container queries

Now the more modern model.

HTML:

```jsx
<div className="card-container">
  <ProjectCard />
</div>
```

CSS:

```css
.card-container {
  container-type: inline-size;
}
```

Now descendants can query the container:

```css
@container (min-width: 500px) {
  .project-card {
    display: flex;
  }
}
```

The browser asks:

```text
How wide is my query container?
```

rather than:

```text
How wide is the entire viewport?
```

---

# 10. Why container queries are a big deal for React

React components are reusable.

Suppose:

```jsx
<ProjectCard />
```

appears in:

```text
homepage
sidebar
dashboard
modal
embedded widget
```

Each parent may provide different available width.

Viewport-based CSS can't directly express:

> "When the component itself gets narrow, change its layout."

Container queries can.

That makes them extremely natural for component-driven systems.

---

# 11. Component responsiveness

Imagine:

```text
ProjectCard
```

At 300px:

```text
┌─────────────┐
│ Project     │
│ Alpha       │
│ 23 tasks    │
└─────────────┘
```

At 600px:

```text
┌───────────────────────────────┐
│ Project Alpha │ 23 tasks      │
└───────────────────────────────┘
```

The component can adapt to its container.

React stays the same:

```jsx
<ProjectCard project={project} />
```

Only CSS changes.

This is one of the strongest examples of keeping presentation logic in CSS.

---

# 12. When should React know the viewport?

Usually:

```text
if the difference is only visual layout
```

→ CSS.

For example:

```text
one column
vs
three columns
```

should generally be CSS.

But if the application behavior genuinely changes:

```text
desktop interaction model
vs
mobile interaction model
```

React may need to know.

For example:

```text
desktop:
drag-and-drop board

mobile:
tap-to-move workflow
```

Now that's no longer purely presentation.

---

# 13. Why `window.innerWidth` is often the wrong first move

You may see:

```jsx
const isMobile =
  window.innerWidth < 768;
```

Then:

```jsx
return isMobile
  ? <MobileNav />
  : <DesktopNav />;
```

This creates several problems.

The server doesn't have the same browser information.

SSR can mismatch.

The value can become stale after resizing.

You now need synchronization logic.

And you're using JavaScript for a problem CSS may already solve.

---

# 14. A CSS-first approach

Instead:

```jsx
<nav className="navigation">
  <DesktopNavigation />
  <MobileNavigation />
</nav>
```

and:

```css
.mobile-navigation {
  display: none;
}

@media (max-width: 700px) {
  .desktop-navigation {
    display: none;
  }

  .mobile-navigation {
    display: block;
  }
}
```

However, there is another issue:

> Both components exist in the DOM unless you structure the semantics carefully.

For some UI this is fine.

For others, especially interactive controls, you need to think about:

```text
accessibility
focusability
duplicate navigation
screen reader behavior
performance
```

Responsive rendering is not purely a visual question.

---

# 15. Sometimes one component is better

Instead of:

```text
DesktopMenu
MobileMenu
```

you can have one:

```jsx
<Navigation />
```

with CSS changing the layout.

That avoids duplicate markup.

For example:

```jsx
<nav className="navigation">
  <Logo />

  <div className="navigation__links">
    ...
  </div>

  <button className="navigation__menu-button">
    Menu
  </button>
</nav>
```

CSS controls what is visible and how the pieces are arranged.

---

# 16. But different interaction models may deserve different DOM

Suppose desktop has:

```text
hover menus
```

while mobile has:

```text
tap-to-open accordion
```

That's a genuine interaction difference.

In that case, forcing one DOM structure to behave identically at every size can create complicated logic.

The right decision depends on semantics and behavior, not on avoiding duplicate code at all costs.

---

# 17. Responsive typography

A naive approach:

```text
desktop = 48px
mobile = 32px
```

using a breakpoint.

A more fluid approach is:

```css
.hero-title {
  font-size:
    clamp(2rem, 5vw, 4rem);
}
```

`clamp()` expresses:

```text
minimum
preferred fluid value
maximum
```

So:

```text
2rem ≤ actual size ≤ 4rem
```

while `5vw` acts as the preferred scaling value.

---

# 18. Why `clamp()` is powerful

Suppose:

```css
font-size:
  clamp(2rem, 5vw, 4rem);
```

On a small viewport:

```text
5vw may be smaller than 2rem
→ use 2rem
```

On medium:

```text
5vw fits range
→ use fluid value
```

On very large screens:

```text
5vw may exceed 4rem
→ cap at 4rem
```

One declaration expresses a responsive range.

---

# 19. `clamp()` works for spacing too

For example:

```css
.section {
  padding-block:
    clamp(2rem, 6vw, 6rem);
}
```

Now section spacing grows smoothly rather than jumping at arbitrary breakpoints.

This can create much more natural layouts.

---

# 20. Fluid width

You might use:

```css
.container {
  width: min(
    100% - 2rem,
    1200px
  );
  margin-inline: auto;
}
```

This means approximately:

```text
available viewport width minus gutters
but never wider than 1200px
```

This is an excellent pattern for page containers.

---

# 21. Why `max-width` matters

Without:

```css
max-width: 1200px;
```

a large monitor could produce:

```text
very long text lines
huge navigation width
uncomfortable reading
```

A centered max-width container gives you:

```text
large viewport
     ↓
controlled reading/layout width
```

---

# 22. `width: 100%` plus `max-width`

A very common pattern:

```css
.container {
  width: 100%;
  max-width: 1200px;
  margin-inline: auto;
  padding-inline: 1rem;
}
```

Now:

```text
small screen
→ nearly full width

large screen
→ capped width
```

This is a foundational responsive pattern.

---

# 23. Don't forget `box-sizing`

If:

```css
width: 100%;
padding: 1rem;
```

with:

```text
content-box
```

the total rendered size can exceed the container.

That is one reason:

```css
* {
  box-sizing: border-box;
}
```

is such a common baseline.

Responsive design amplifies sizing mistakes because available space is already constrained.

---

# 24. Viewport units

Traditional:

```text
vw
vh
```

mean:

```text
viewport width
viewport height
```

But mobile browsers introduced complications because browser UI can appear/disappear as the user scrolls.

That's why newer viewport units exist:

```text
svh
lvh
dvh
```

Think of them conceptually as:

```text
svh
→ small viewport height

lvh
→ large viewport height

dvh
→ dynamic viewport height
```

---

# 25. Why `100vh` can be problematic on mobile

Suppose the browser's visible area changes:

```text
URL bar visible
```

then:

```text
user scrolls
URL bar collapses
```

The effective visible viewport changes.

A fixed:

```css
height: 100vh;
```

can produce awkward sizing.

For interfaces that genuinely need dynamic visible height, something like:

```css
min-height: 100dvh;
```

can be more appropriate.

But don't replace every `vh` mechanically.

Choose the viewport model based on the intended behavior.

---

# 26. `svh`, `lvh`, `dvh`

A useful conceptual picture:

```text
small viewport
┌──────────────┐
│ browser UI   │
├──────────────┤
│              │
│ content      │
│              │
└──────────────┘

large viewport
┌──────────────┐
│              │
│ content      │
│              │
│              │
└──────────────┘

dynamic viewport
changes with current browser UI
```

Again:

```text
svh = more conservative visible height
lvh = largest viewport size
dvh = current dynamic viewport
```

---

# 27. Full-screen app shell

For Orbit:

```css
.app {
  min-height: 100dvh;
}
```

may be a good modern baseline when the application should at least fill the visible viewport.

Then:

```text
header
sidebar
main
```

can be laid out inside that shell.

---

# 28. Safe areas

On devices with display cutouts/notches, the browser can expose safe-area insets.

You may encounter:

```css
padding-bottom:
  env(safe-area-inset-bottom);
```

or combined values:

```css
padding-bottom:
  calc(
    1rem +
    env(safe-area-inset-bottom)
  );
```

This matters for:

```text
fixed bottom navigation
mobile toolbars
full-screen applications
```

---

# 29. Responsive spacing

Instead of:

```text
mobile = 16px
desktop = 32px
```

you can sometimes use:

```css
padding-inline:
  clamp(1rem, 4vw, 3rem);
```

This produces fluid spacing.

The design becomes:

```text
minimum
→ fluid
→ maximum
```

instead of a sudden jump.

---

# 30. Responsive grids without breakpoints

This remains one of my favorite patterns:

```css
.grid {
  display: grid;

  grid-template-columns:
    repeat(
      auto-fit,
      minmax(16rem, 1fr)
    );

  gap: 1rem;
}
```

Let's imagine:

```text
width = 400
→ 1 column

width = 800
→ 2 columns

width = 1200
→ 3 columns

width = 1600
→ 4 columns
```

No JavaScript.

No React state.

Possibly no media query.

---

# 31. Responsive cards

A card itself can change layout based on container width.

```css
.card {
  container-type: inline-size;
}

.card__body {
  display: grid;
  gap: 1rem;
}

@container (min-width: 500px) {
  .card__body {
    grid-template-columns:
      120px minmax(0, 1fr);
  }
}
```

The same React component can be used anywhere.

This is excellent for reusable design-system components.

---

# 32. Responsive navigation

Let's design Orbit.

Desktop:

```text
┌────────────────────────────────────────────┐
│ Logo │ Projects │ Tasks │ Activity │ User │
└────────────────────────────────────────────┘
```

Small screen:

```text
┌────────────────────────────────────────────┐
│ Logo                                  ☰   │
└────────────────────────────────────────────┘
```

CSS may hide/show navigation controls based on available width.

But the **interaction model** needs consideration.

The menu button must:

```text
have an accessible name
manage expanded state
control focus appropriately
```

This is where responsive design meets accessibility.

---

# 33. Responsive sidebar

Desktop:

```text
sidebar │ main
        │
        │
```

Small screen:

```text
main
```

perhaps with the sidebar becoming:

```text
drawer
```

Now there is a genuine interaction change.

React may own:

```text
sidebarOpen
```

while CSS owns:

```text
sidebar width
position
animation
desktop/mobile layout
```

Again:

```text
React = state
CSS = presentation
```

---

# 34. Responsive tables

Tables are particularly difficult.

Suppose:

```text
Task | Assignee | Priority | Status | Updated
```

On mobile, horizontal overflow may be perfectly appropriate:

```css
.table-container {
  overflow-x: auto;
}
```

Don't automatically convert every table into cards.

Ask:

> Is horizontal scrolling acceptable for this data?

Sometimes yes.

Sometimes the information architecture genuinely needs to change.

---

# 35. Responsive task boards

Orbit's task board:

```text
Open | Doing | Done
```

On desktop:

```text
┌─────┬─────┬─────┐
│Open │Doing│Done │
└─────┴─────┴─────┘
```

On small screens, forcing three 280px columns may produce:

```text
← huge horizontal scroll →
```

Maybe that's okay.

Maybe the mobile experience should instead become:

```text
Open
  task
  task

Doing
  task
  task

Done
  task
```

Or perhaps a horizontally scrolling board is exactly what users want.

The key is:

> **Responsive design isn't "make everything fit." It is "make the interaction usable at different sizes."**

---

# 36. Responsive modals

Desktop:

```text
             ┌─────────────────┐
             │ Edit Task        │
             │                  │
             │ fields           │
             │                  │
             │ Cancel  Save      │
             └─────────────────┘
```

Mobile:

```text
┌──────────────────────────────┐
│ Edit Task                  × │
├──────────────────────────────┤
│                              │
│ fields                       │
│                              │
├──────────────────────────────┤
│       Cancel      Save       │
└──────────────────────────────┘
```

This may involve:

```text
width changes
max-width changes
height constraints
overflow
padding
button arrangement
```

The modal behavior can stay React-driven.

CSS handles the responsive presentation.

---

# 37. Responsive typography hierarchy

A production interface shouldn't have:

```text
h1 = 48px everywhere
h2 = 32px everywhere
body = 16px
```

without considering viewport and reading conditions.

You might define tokens:

```css
:root {
  --text-body:
    clamp(1rem, 0.95rem + 0.2vw, 1.125rem);

  --text-title:
    clamp(2rem, 1.5rem + 2vw, 4rem);
}
```

Then:

```css
body {
  font-size: var(--text-body);
}

h1 {
  font-size: var(--text-title);
}
```

This gives the design system a fluid typography foundation.

---

# 38. `clamp()` as a design-token tool

Instead of only:

```text
--spacing-lg = 32px
```

you can have:

```css
--spacing-section:
  clamp(2rem, 6vw, 6rem);
```

Then every section using:

```css
padding-block:
  var(--spacing-section);
```

responds fluidly.

This turns responsive behavior into part of the design system.

---

# 39. `min()`, `max()`, and `clamp()`

These three functions are worth learning as a family.

### `min()`

Choose the smaller result:

```css
width: min(100%, 1200px);
```

### `max()`

Choose the larger result:

```css
padding-inline:
  max(1rem, 4vw);
```

### `clamp()`

Keep a value inside a range:

```css
font-size:
  clamp(2rem, 5vw, 4rem);
```

These let CSS express constraints directly.

---

# 40. Responsive images

Images need responsive behavior too.

A classic baseline:

```css
img {
  max-width: 100%;
  height: auto;
}
```

This prevents an image from exceeding its containing width.

But production image strategy also involves:

```text
intrinsic dimensions
aspect-ratio
object-fit
object-position
responsive image sources
loading strategy
```

---

# 41. `aspect-ratio`

Suppose project cards have thumbnails.

Instead of manually calculating heights:

```css
.thumbnail {
  aspect-ratio: 16 / 9;
}
```

The browser maintains that ratio.

This is especially useful for:

```text
cards
video previews
avatars
media placeholders
skeletons
```

---

# 42. `object-fit`

Suppose:

```jsx
<img src={project.image} alt="" />
```

and your container is:

```text
16:9
```

Use:

```css
img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

Now the image fills the box while preserving its aspect ratio, cropping overflow as necessary.

Other common behavior:

```css
object-fit: contain;
```

keeps the entire image visible.

---

# 43. Responsive image cropping

You can combine:

```text
aspect-ratio
object-fit
object-position
```

with responsive design.

For example:

```css
.hero-image {
  aspect-ratio: 16 / 7;
  object-fit: cover;
}

@media (max-width: 700px) {
  .hero-image {
    aspect-ratio: 4 / 3;
  }
}
```

Same image.

Different presentation.

---

# 44. Art direction vs simple responsiveness

Sometimes mobile needs a **different crop or even different image**.

That's called art direction.

This is different from simply resizing an image.

Conceptually:

```text
desktop
→ wide landscape image

mobile
→ portrait-focused image
```

This may involve:

```text
<picture>
source media
different assets
```

The browser can then choose the appropriate resource.

This is not primarily a React problem.

---

# 45. Responsive content vs responsive layout

These are different.

### Layout change

```text
3 columns
→ 1 column
```

### Content change

```text
"Save changes"
→
"Save"
```

The second changes actual content.

Be careful with CSS `content` or hiding text merely to shorten UI.

For accessible interfaces, semantic content shouldn't depend purely on visual tricks.

---

# 46. Don't use CSS to solve semantic content changes

Suppose:

```text
desktop:
"Create new project"

mobile:
"New"
```

You _can_ visually manipulate text, but if the accessible name or meaning becomes inconsistent, you can create problems.

Sometimes a React decision is more appropriate:

```jsx
<button>
  <span className="desktop-label">
    Create new project
  </span>

  <span className="mobile-label">
    New
  </span>
</button>
```

But then you need to carefully manage accessible naming so you're not exposing duplicate content to assistive technology.

There isn't a universal CSS-only answer.

---

# 47. Responsive visibility is different from conditional rendering

This distinction:

```text
display: none
```

versus:

```jsx
{condition && <Component />}
```

matters.

CSS:

```text
element exists
but not participating in layout
```

React:

```text
element may not exist at all
```

Choosing between them affects:

```text
state preservation
mount/unmount effects
accessibility
performance
animation
```

This directly connects responsive CSS to the React rendering model.

---

# 48. Preserve state vs reset state

Suppose your mobile sidebar contains:

```text
search input
filters
expanded groups
```

If you only hide it:

```css
display: none;
```

its React state can remain because the component remains mounted.

If React conditionally removes it:

```jsx
{mobile && <Sidebar />}
```

you may reset its state depending on component identity/placement.

So responsive implementation is also a state-lifetime decision.

---

# 49. Responsive design and `Activity`

Recall React's `Activity` from the earlier advanced lesson.

Suppose:

```text
sidebar
```

is temporarily hidden.

You might want:

```text
preserve state
pause expensive effects
```

rather than fully unmounting it.

That gives another architecture option beyond:

```text
CSS hide
```

or:

```text
conditional unmount
```

This is where React's newer capabilities can complement responsive UI architecture.

---

# 50. Responsive state should not be duplicated

Avoid:

```text
React says mobile
+
CSS says mobile
```

unless both are independently needed.

Otherwise you now have:

```text
two sources of truth
```

For purely visual breakpoints:

```text
CSS owns it.
```

If React needs an actual application-level mode:

```text
React may own it.
```

Try not to maintain the same fact in both places.

---

# 51. Media query hooks: when they are justified

A hook might conceptually subscribe to:

```js
window.matchMedia(
  '(prefers-reduced-motion: reduce)'
);
```

This is legitimate because React needs to respond to an external browser signal.

That's fundamentally different from:

```text
useEffect just to copy viewport width into state
```

The first is:

```text
external browser state
→ React
```

which matches our `useSyncExternalStore` mental model.

---

# 52. `prefers-reduced-motion`

Responsive design isn't only about width.

The browser exposes user preferences.

For example:

```css
@media (prefers-reduced-motion: reduce) {
  .card {
    transition: none;
  }
}
```

Now the application respects a user preference without JavaScript.

This is another example of letting CSS respond to environment state.

---

# 53. `prefers-color-scheme`

Likewise:

```css
@media (prefers-color-scheme: dark) {
  ...
}
```

allows CSS to respond to OS/browser theme preference.

React can still offer an explicit theme toggle.

A common architecture is:

```text
system preference
        ↓
default theme

user selection
        ↓
override theme
```

CSS variables then control the visuals.

---

# 54. Container query units

Once you've learned container queries, you'll eventually encounter:

```text
cqw
cqh
cqi
cqb
cqmin
cqmax
```

These are relative to query containers rather than the viewport.

For example:

```css
font-size: 5cqw;
```

can scale based on container width.

These are more advanced and aren't required everywhere, but they're part of modern component-oriented responsive CSS.

---

# 55. `rem`, `vw`, and container units solve different relationships

Think:

```text
rem
→ root typography system

vw
→ viewport

cqw
→ component/container
```

Choose the unit according to what should control the value.

This is much better than:

```text
"px everywhere"
```

or:

```text
"vw everywhere"
```

---

# 56. Responsive spacing architecture

A design system might define:

```css
:root {
  --space-xs: 0.25rem;
  --space-sm: 0.5rem;
  --space-md: 1rem;
  --space-lg: 1.5rem;
  --space-xl: 2rem;
}
```

Then responsive spacing can layer on:

```css
.page {
  padding-inline:
    clamp(
      var(--space-md),
      4vw,
      var(--space-xl)
    );
}
```

The result is:

```text
design token
+
responsive constraint
```

---

# 57. Responsive layout using intrinsic widths

Suppose a toolbar contains:

```text
Search
Filter
Sort
New Task
```

Instead of:

```text
button width: 120px
search width: 320px
```

you can let content and constraints participate:

```css
.toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.search {
  flex: 1 1 16rem;
  min-width: min(100%, 16rem);
}
```

Now the toolbar can wrap naturally.

---

# 58. `flex-wrap` + `flex-basis` is a responsive tool

This:

```css
.search {
  flex: 1 1 20rem;
}
```

means roughly:

```text
start around 20rem
grow when space exists
shrink when necessary
```

Combined with:

```css
.toolbar {
  display: flex;
  flex-wrap: wrap;
}
```

you often get responsive behavior without explicit breakpoints.

---

# 59. A responsive toolbar

```text
wide:

[ Search........................ ] [Filter] [Sort] [New Task]

narrow:

[ Search......................... ]

[Filter] [Sort] [New Task]
```

The browser can achieve much of this automatically with Flexbox.

This is often better than maintaining:

```text
desktopToolbar
mobileToolbar
```

as separate structures.

---

# 60. Responsive card content

Suppose:

```jsx
<article className="card">
  <img />
  <div className="content">
    <h2 />
    <p />
  </div>
  <button />
</article>
```

Desktop:

```text
image | content       | button
```

Small:

```text
image
content
button
```

CSS can often switch:

```css
.card {
  display: grid;
  grid-template-columns:
    120px minmax(0, 1fr) auto;
}

@media (max-width: 600px) {
  .card {
    grid-template-columns: 1fr;
  }
}
```

Again the breakpoint exists because the arrangement stops working.

---

# 61. Responsive content width is as important as screen width

A common mistake:

```text
"desktop = big"
```

but a 2000px monitor can still need:

```text
max-width: 70ch
```

for comfortable reading.

So responsive design also includes:

```text
readability constraints
```

For long-form text:

```css
.article {
  max-width: 70ch;
  margin-inline: auto;
}
```

`ch` is tied approximately to character width in the current font context.

This is useful for readable line lengths.

---

# 62. `ch` is not a universal measurement

It's tempting to say:

```text
70ch = exactly 70 characters
```

That's not guaranteed in a strict typographic sense because glyph widths vary.

It's better to think:

> `ch` gives a character-width-related typographic measure.

It's useful for controlling line length, not exact character counting.

---

# 63. Responsive page architecture

A mature responsive application might use:

```text
App Shell
  ↓
Grid
  ↓
Sidebar + Main

Main
  ↓
Stack
  ↓
Toolbar
  ↓
Stats Grid
  ↓
Task Board
```

and each layer has its own response model:

```text
App Shell
 → viewport/page-level

Toolbar
 → flex wrapping

Stats
 → intrinsic grid

Task cards
 → container query

Task board
 → overflow or alternate layout
```

This is much better than one giant:

```text
@media (max-width: 768px)
```

block controlling everything.

---

# 64. Avoid giant media-query overrides

This:

```css
@media (max-width: 768px) {
  .header { ... }
  .sidebar { ... }
  .toolbar { ... }
  .stats { ... }
  .card { ... }
  .modal { ... }
  .table { ... }
  .footer { ... }
  ...
}
```

can become difficult to reason about.

A more scalable approach is:

```text
each component
 → owns its own responsive behavior
```

while global responsive tokens handle system-wide patterns.

---

# 65. Responsive behavior belongs near the component

For:

```text
TaskCard
```

its stylesheet should ideally describe:

```text
default card
narrow card
wide card
```

not require a separate giant application-wide stylesheet to understand.

This is one of the arguments for component-oriented CSS organization.

---

# 66. CSS Modules become especially useful here

Later we'll write:

```jsx
import styles from './TaskCard.module.css';

<article className={styles.card}>
```

Then the responsive rules stay near the component:

```css
.card {
  ...
}

@container (min-width: 500px) {
  .card {
    ...
  }
}
```

This is one reason CSS Modules work nicely with React component architecture.

But we'll dedicate an entire lesson to that later.

---

# 67. Responsive state machine example

Suppose the sidebar has:

```text
closed
open
```

React:

```jsx
<aside
  className="sidebar"
  data-open={open}
>
```

CSS:

```css
.sidebar {
  transform: translateX(-100%);
}

.sidebar[data-open="true"] {
  transform: translateX(0);
}

@media (min-width: 900px) {
  .sidebar {
    transform: none;
  }
}
```

Now:

```text
mobile
→ state controls open/closed

desktop
→ sidebar permanently visible
```

The responsive rule changes the default presentation, while React still controls the application's open state.

---

# 68. This avoids unnecessary React logic

You don't need:

```js
if (window.innerWidth >= 900) {
  setSidebarOpen(true);
}
```

The CSS can make the sidebar permanently visible on large screens.

React only needs to care about:

```text
"Is the mobile drawer open?"
```

when that concept actually exists.

---

# 69. SSR makes this separation even more valuable

Suppose your app server renders:

```text
mobile HTML
```

because the server doesn't know the browser width precisely.

If you make layout purely CSS-driven:

```text
same markup
+
responsive CSS
```

then the browser can adapt after loading without changing the React tree.

This reduces hydration complexity.

---

# 70. Why viewport branching during render is dangerous in SSR

Imagine:

```jsx
if (window.innerWidth < 768) {
  return <MobileLayout />;
}

return <DesktopLayout />;
```

Problems:

```text
window doesn't exist during SSR
```

and even if guarded:

```text
server output
≠
client initial output
```

can lead to hydration mismatch.

This is another reason CSS should handle purely visual responsiveness.

---

# 71. Orientation

The viewport can also respond to:

```css
@media (orientation: landscape) {
  ...
}
```

But again, don't overuse device-specific media conditions.

Ask whether the layout actually needs a different composition.

---

# 72. Pointer/hover capabilities

Modern media queries can also describe interaction capabilities.

Conceptually:

```css
@media (hover: hover) {
  ...
}
```

or:

```css
@media (pointer: coarse) {
  ...
}
```

These are useful because "mobile" is an imperfect proxy for interaction capability.

A tablet might support a pointer.

A laptop might have touch.

Responsive design should respond to actual capabilities when appropriate.

---

# 73. Responsive ≠ mobile-only

This is worth repeating.

Responsive interfaces can react to:

```text
width
height
container size
orientation
hover capability
pointer precision
color scheme
reduced motion
safe areas
writing direction
```

"Mobile responsive" is only one part of the broader idea.

---

# 74. Accessibility and responsive behavior

Imagine a desktop dropdown:

```text
hover
```

On mobile:

```text
tap
```

You shouldn't depend purely on hover because some users can't hover.

A robust component may use:

```text
button
keyboard
focus
click/tap
```

with CSS enhancing the visual behavior.

Responsive interaction should preserve semantics.

---

# 75. Responsive focus management

A mobile sidebar opening:

```text
click menu
 ↓
drawer appears
```

should typically consider:

```text
move focus into drawer
trap focus when modal-like
close on Escape
restore focus to trigger
```

This is not CSS.

It's React/browser interaction logic.

CSS handles:

```text
position
size
animation
visibility
```

Again the boundary is clear.

---

# 76. Responsive animations

Suppose the sidebar animation:

```css
.sidebar {
  transition:
    transform 200ms ease;
}
```

On reduced motion:

```css
@media (prefers-reduced-motion: reduce) {
  .sidebar {
    transition: none;
  }
}
```

Now:

```text
React
→ open/closed state

CSS
→ transition

user preference
→ animation policy
```

Three concerns cooperate cleanly.

---

# 77. A full responsive Orbit example

Let's combine what we've learned.

```css
.app {
  min-height: 100dvh;

  display: grid;

  grid-template:
    "header" auto
    "main" minmax(0, 1fr);
}

.main {
  min-width: 0;
  min-height: 0;
}

.cards {
  display: grid;

  grid-template-columns:
    repeat(
      auto-fit,
      minmax(220px, 1fr)
    );

  gap: 1rem;
}

.toolbar {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 0.75rem;
}

.search {
  flex: 1 1 18rem;
  min-width: min(100%, 18rem);
}

.page-title {
  font-size:
    clamp(1.75rem, 4vw, 3rem);
}

.page {
  width: min(
    100% - 2rem,
    1200px
  );

  margin-inline: auto;
}

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

@media (max-width: 800px) {
  .task-board {
    grid-template-columns:
      repeat(
        3,
        minmax(260px, 85vw)
      );
  }
}
```

Notice the philosophy:

```text
intrinsic sizing where possible
+
minimal breakpoint intervention
+
fluid typography
+
proper minimum constraints
+
CSS-owned visual responsiveness
```

---

# 78. Why the task board may intentionally remain horizontally scrollable

A three-column Kanban board has a conceptual structure:

```text
Open | Doing | Done
```

On a phone, trying to compress them into tiny columns may destroy usability.

Horizontal scrolling can preserve:

```text
column width
task readability
drag interactions
```

So responsive design doesn't mean:

```text
everything = one column
```

It means:

> Choose a layout that preserves the actual interaction model.

---

# 79. Responsive design is information architecture

Suppose a desktop dashboard has:

```text
10 columns
40 fields
```

You can't simply scale it down.

You may need to decide:

```text
what stays
what collapses
what scrolls
what moves
what becomes secondary
what becomes a separate interaction
```

This is a product-design decision as much as a CSS decision.

React may participate because the component tree or interaction model changes.

---

# 80. There are three kinds of responsive changes

This distinction is useful.

### Type 1 — Same content, different layout

```text
row → column
```

Mostly CSS.

### Type 2 — Same feature, different interaction

```text
sidebar → drawer
```

React + CSS.

### Type 3 — Different information architecture

```text
desktop analytics dashboard
→ mobile summary screen
```

Potentially different React components/structure.

Don't force Type 3 into a few media-query overrides.

---

# 81. The "CSS first, React when necessary" rule

For responsive behavior:

```text
Can CSS solve it?
        ↓ yes
Use CSS.
        ↓ no
Does React need different behavior?
        ↓ yes
Use React.
```

This avoids a large amount of unnecessary complexity.

---

# 82. Responsive testing

Earlier we learned testing.

Now apply it here.

You should test representative viewports:

```text
small
medium
large
```

but test **behavior**, not every pixel.

For example:

```text
mobile:
menu button appears

desktop:
sidebar visible

mobile:
task board remains usable

all sizes:
Save button accessible
```

---

# 83. E2E responsive testing

A browser test can run at different viewport sizes.

Conceptually:

```js
await page.setViewportSize({
  width: 390,
  height: 844
});
```

Then:

```text
open app
 ↓
mobile navigation
 ↓
open menu
 ↓
focus moves correctly
 ↓
navigate
```

And another test at:

```text
1440 × 900
```

checks:

```text
desktop navigation
sidebar
task board
```

---

# 84. Don't test every CSS declaration

You do not need:

```text
expect gap = 16px
expect padding = 24px
expect width = 320px
```

unless those exact values are part of a critical design contract.

Instead test:

```text
element is visible
layout is usable
button is accessible
content isn't clipped
critical interaction works
```

Visual regression can handle exact appearance when appropriate.

---

# 85. Responsive performance

Mobile users often have:

```text
slower CPU
slower network
less memory
```

Responsive design therefore intersects performance.

If your mobile page loads:

```text
10MB of JS
1000 DOM nodes
huge images
```

making the layout one column doesn't solve the real problem.

Responsive performance may require:

```text
lazy loading
code splitting
image optimization
virtualization
reduced DOM
```

So responsive engineering is not just visual.

---

# 86. Mobile layout can expose hidden assumptions

A desktop layout may hide:

```text
overflow bugs
hardcoded widths
minimum-size bugs
text wrapping problems
focus problems
```

Mobile testing is therefore useful even if most of your users are desktop users.

It stresses your layout constraints.

---

# 87. The biggest responsive CSS bugs

Keep an eye out for:

```text
fixed widths
fixed heights
min-content surprises
flex overflow
grid overflow
nested scroll containers
large unbreakable text
absolute positioning
100vh assumptions
duplicated responsive state
```

Many of these are really constraint problems.

---

# 88. The responsive debugging procedure

When something breaks on a smaller screen:

```text
1. What is the available width?
2. Which element is too large?
3. Is it fixed-width?
4. What is its min-width?
5. Is it a flex item?
6. Is it a grid track?
7. Is content refusing to shrink?
8. Is something overflowing?
9. Which element should scroll?
10. Is the layout actually supposed to change here?
```

Then inspect:

```text
computed width
box model
overflow
flex/grid tracks
container dimensions
```

This is far more reliable than adding another breakpoint.

---

# 89. A responsive mindset shift

Instead of:

> "How do I make desktop look like mobile?"

think:

> "What constraints does this UI have at different available sizes?"

For example:

```text
Toolbar:
buttons require ~300px
search needs ~250px
gap needs ~32px
```

Total needed:

```text
~582px
```

If container becomes:

```text
450px
```

something must change.

Flex-wrap might solve it.

Grid might solve it.

A breakpoint might solve it.

A different interaction model might solve it.

The numbers reveal the actual constraint.

---

# 90. Final mental model

Responsive CSS can now be thought of as:

```text
AVAILABLE SPACE
      ↓
constraints
      ↓
intrinsic sizing
      ↓
Flexbox / Grid
      ↓
responsive functions
(min/max/clamp)
      ↓
media/container queries
      ↓
final layout
```

And React sits above that:

```text
React
 ↓
semantic structure
 ↓
application state
 ↓
interaction model
 ↓
CSS
 ↓
responsive presentation
```

---

# 91. The rules I want you to remember

```text
1. Responsive ≠ mobile breakpoints.

2. Let layout respond to available space.

3. Prefer intrinsic CSS solutions before adding breakpoints.

4. Use media queries for viewport-level behavior.

5. Use container queries for component-level behavior.

6. Use CSS for visual differences.

7. Use React when behavior or information architecture actually changes.

8. Avoid duplicating viewport state in both React and CSS.

9. Be careful with fixed widths/heights.

10. Treat min-width: 0 and min-height: 0 as important layout tools.

11. Use min(), max(), and clamp() to express constraints.

12. Understand dvh/svh/lvh for modern viewport behavior.

13. Responsive design must preserve accessibility and interaction quality.

14. Test representative sizes, not every possible device.

15. Don't shrink a bad desktop layout; redesign the constraints.
```

---

# 92. Where this connects to our Orbit project

Our eventual Orbit styling architecture should look roughly like:

```text
Orbit
│
├── global foundation
│   ├── reset
│   ├── typography
│   ├── tokens
│   └── themes
│
├── layout primitives
│   ├── Stack
│   ├── Cluster
│   ├── Grid
│   └── Container
│
├── responsive components
│   ├── Navigation
│   ├── Sidebar
│   ├── TaskBoard
│   ├── TaskCard
│   └── TaskEditor
│
└── responsive behavior
    ├── media queries
    ├── container queries
    ├── fluid sizing
    └── accessibility preferences
```

That is where we're headed.

---

# Next — Styling with React S4

Next is the **big styling-strategy lesson**:

```text
Global CSS
CSS Modules
Inline styles
CSS-in-JS
Utility CSS
Tailwind
Component libraries
CSS variables
style props
data attributes
scoping
specificity
CSS layers
composition
```

We'll build the **same React component using each approach**, inspect what actually reaches the DOM/CSS system, compare maintainability, runtime/build-time behavior, theming, dynamic styles, SSR, performance, and explain **when each approach makes sense and when it becomes painful**.