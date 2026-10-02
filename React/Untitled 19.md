
# Lesson 16 — URL State, Search Params, and Route Params

We’ve now reached an important point in React:

> **Not every piece of state belongs in `useState`.**

Some state belongs in the **URL**.

This becomes extremely useful for things like:

```text
search
filter
sort
page
selected tab
category
date range
```

For example:

```text
/products?search=laptop&category=electronics&page=2
```

That URL itself contains state.

And unlike ordinary component state, URL state has a major property:

> **It can be bookmarked, shared, refreshed, and navigated with browser history.**

React Router currently provides `useSearchParams()` for reading and updating the query-string portion of the URL. ([React Router](https://reactrouter.com/api/hooks/useSearchParams?utm_source=chatgpt.com "useSearchParams | React Router"))

---

# 1. First understand the URL

Take this:

```text
https://example.com/products/42?category=electronics&page=2
```

Break it apart:

```text
https://example.com
        │
        ↓
     /products/42
        │
        ↓
     ?category=electronics&page=2
```

There are two important parts for us.

### Path

```text
/products/42
```

### Search/query parameters

```text
?category=electronics&page=2
```

We've already seen that `/products/:id` uses a **route parameter**.

Now we're focusing on the part after `?`.

---

# 2. Route params vs search params

These are different concepts.

## Route parameter

```text
/products/42
```

with:

```jsx
<Route
  path="/products/:productId"
  element={<Product />}
/>
```

Then:

```jsx
const { productId } = useParams();
```

gives:

```text
"42"
```

Route params usually identify **which resource/page you're looking at**.

---

## Search params

```text
/products?category=electronics&page=2
```

These usually describe **how you want that page configured**.

For example:

```text
category=electronics
page=2
sort=price
search=laptop
```

So a useful mental distinction is:

```text
Route param
→ "Which thing?"

Search param
→ "How should I view/filter this thing?"
```

This isn't an absolute rule, but it's a very useful design heuristic. React Router treats dynamic path segments and URL search params as separate pieces of URL state. ([React Router](https://reactrouter.com/start/declarative/url-values?utm_source=chatgpt.com "URL Values | React Router"))

---

# 3. Why put state in the URL?

Suppose your product page has:

```text
Search: laptop
Category: electronics
Page: 2
Sort: price
```

You could store all four in React state:

```jsx
const [search, setSearch] = useState("");
const [category, setCategory] = useState("all");
const [page, setPage] = useState(1);
const [sort, setSort] = useState("name");
```

That works.

But now the URL might still be:

```text
/products
```

If the user refreshes the browser:

```text
/products
```

your state may go back to its initial values.

And if they copy the URL to a friend:

```text
/products
```

the friend doesn't get the same search/filter/page.

That's a sign that some of this information may belong in the URL.

---

# 4. URL state is shareable state

Imagine:

```text
/products?search=laptop&category=electronics&page=2
```

You can:

```text
bookmark it
refresh it
share it
open it in another tab
use browser Back/Forward
```

and the URL still tells the application how the page should look.

That's a huge advantage.

So:

```text
useState
→ component/application state

URL
→ navigation/shareable state
```

---

# 5. `useSearchParams`

Here's the React Router API:

```jsx
import { useSearchParams } from "react-router";

function Products() {
  const [searchParams, setSearchParams] =
    useSearchParams();

  // ...
}
```

This looks suspiciously like:

```jsx
const [value, setValue] = useState(...)
```

and that's intentional.

You get:

```text
searchParams
setSearchParams
```

The first lets you read the current query parameters.

The second lets you update them.

Updating them causes a navigation. ([React Router](https://reactrouter.com/api/hooks/useSearchParams?utm_source=chatgpt.com "useSearchParams | React Router"))

---

# 6. Reading a parameter

Suppose URL:

```text
/products?search=laptop
```

Then:

```jsx
const search =
  searchParams.get("search");
```

gives:

```text
"laptop"
```

So:

```jsx
function Products() {
  const [searchParams] =
    useSearchParams();

  const search =
    searchParams.get("search");

  return (
    <h1>
      Searching for: {search}
    </h1>
  );
}
```

Result:

```text
Searching for: laptop
```

`URLSearchParams.get()` is standard browser JavaScript, and React Router gives you the current URL's `URLSearchParams` object through `useSearchParams`. ([React Router](https://reactrouter.com/api/hooks/useSearchParams?utm_source=chatgpt.com "useSearchParams | React Router"))

---

# 7. What if the parameter doesn't exist?

Suppose URL:

```text
/products
```

and you do:

```js
searchParams.get("search")
```

you get:

```text
null
```

So you might write:

```jsx
const search =
  searchParams.get("search") ?? "";
```

Now:

```text
no search param
   ↓
""
```

That gives you a useful default for your UI.

---

# 8. Updating search params

Suppose the user types into the search box.

You could do:

```jsx
setSearchParams({
  search: "laptop"
});
```

The URL becomes approximately:

```text
/products?search=laptop
```

React Router's current `setSearchParams` API accepts object-like initializers and causes navigation when the params are updated. ([React Router](https://reactrouter.com/api/hooks/useSearchParams?utm_source=chatgpt.com "useSearchParams | React Router"))

---

# 9. Build a searchable product page

Let's start with:

```jsx
function Products() {
  const [searchParams, setSearchParams] =
    useSearchParams();

  const search =
    searchParams.get("search") ?? "";

  return (
    <div>
      <input
        value={search}
        onChange={e =>
          setSearchParams({
            search: e.target.value
          })
        }
      />

      <p>
        Searching for: {search}
      </p>
    </div>
  );
}
```

Now the state lives in the URL.

For example:

```text
User types "lap"
       ↓
setSearchParams(...)
       ↓
URL becomes:
/products?search=lap
       ↓
component gets new URL state
       ↓
search becomes "lap"
       ↓
UI updates
```

Notice how similar this is to `useState`.

But the storage location is different.

---

# 10. This is a useful comparison

Normal state:

```jsx
const [search, setSearch] =
  useState("");
```

URL state:

```jsx
const [searchParams, setSearchParams] =
  useSearchParams();
```

Then:

```text
useState
→ value stored in React component state

useSearchParams
→ value represented in the URL
```

This distinction is important.

---

# 11. Add category

Suppose:

```text
/products?search=laptop&category=electronics
```

Read:

```jsx
const search =
  searchParams.get("search") ?? "";

const category =
  searchParams.get("category") ?? "all";
```

Now:

```jsx
<select
  value={category}
  onChange={e => {
    setSearchParams({
      search,
      category: e.target.value
    });
  }}
>
  <option value="all">
    All
  </option>

  <option value="electronics">
    Electronics
  </option>

  <option value="accessories">
    Accessories
  </option>
</select>
```

Now both pieces of UI are synchronized with the URL.

---

# 12. But there's a subtle problem

Suppose current URL is:

```text
/products?search=laptop&category=electronics&page=2
```

Then you execute:

```jsx
setSearchParams({
  search: "phone"
});
```

You may end up replacing the search params so that:

```text
category
page
```

are gone.

This is why it's important to understand that `setSearchParams` isn't exactly a `setState` replacement.

You should deliberately construct the complete desired query-string state.

For example:

```jsx
setSearchParams({
  search: "phone",
  category,
  page: "1"
});
```

Now the URL becomes:

```text
/products?search=phone&category=electronics&page=1
```

---

# 13. Why would page become `1`?

Imagine:

```text
?page=4
```

and the user changes the search from:

```text
laptop
```

to:

```text
phone
```

Keeping:

```text
page=4
```

might make little sense because the new search results may have only two pages.

So your state transition can intentionally be:

```text
search changed
   ↓
page reset to 1
```

This is a nice example of application logic built around URL state.

---

# 14. Search params can replace a lot of React state

Instead of:

```jsx
const [search, setSearch] = useState("");
const [category, setCategory] = useState("all");
const [sort, setSort] = useState("name");
const [page, setPage] = useState(1);
```

you can potentially make:

```text
URL
/products
  ?search=laptop
  &category=electronics
  &sort=price
  &page=2
```

the source of truth.

Then:

```text
URL
 ↓
searchParams
 ↓
derived UI/data
```

That's often much cleaner for state that represents navigation/filtering.

---

# 15. A very important classification of state

We're now at the point where "state" needs to be split into categories.

### Local UI state

Examples:

```text
isModalOpen
isDropdownOpen
input currently being edited
```

Often:

```jsx
useState()
```

---

### URL state

Examples:

```text
search
filter
sort
page
tab
```

Often:

```jsx
useSearchParams()
```

---

### Server state

Examples:

```text
products from API
logged-in account data
orders
comments
```

This is data originating from a server.

We'll study this much more deeply later.

---

### Contextual state

Examples:

```text
theme
locale
authentication context
```

Potentially:

```jsx
Context
```

---

### Derived data

Examples:

```text
filteredProducts
totalPrice
isEmpty
```

Usually:

```text
ordinary calculation
```

This is a very important architecture distinction.

---

# 16. The URL is part of application state

Suppose:

```text
/products?search=laptop
```

and your UI shows:

```text
Search: laptop
```

Then the UI is effectively derived from:

```text
URL
```

So:

```text
URL
 ↓
search
 ↓
filtered products
 ↓
rendered UI
```

The URL itself has become an input into your application's rendering.

---

# 17. Browser Back/Forward now becomes meaningful

Suppose the user searches:

```text
laptop
```

then:

```text
phone
```

then:

```text
keyboard
```

The browser history can now reflect those navigations.

So pressing Back can return the user through previous search states.

That is much more natural than:

```text
React state changed
```

with no corresponding URL/history representation.

This is one of the strongest reasons URL state is useful.

---

# 18. Search params aren't only for search

The name makes them sound search-specific.

They aren't.

You can use:

```text
?tab=overview
?sort=price
?page=4
?category=electronics
?view=grid
?date=2026-09-21
```

So think:

> **Search params = state encoded in the query string.**

Not:

> "Things used only for searching."

---

# 19. Example: tabs

Suppose:

```text
/settings?tab=security
```

Then:

```jsx
const [searchParams, setSearchParams] =
  useSearchParams();

const tab =
  searchParams.get("tab") ?? "general";
```

And:

```jsx
<button
  onClick={() =>
    setSearchParams({ tab: "security" })
  }
>
  Security
</button>
```

The URL tells us which tab is active.

Now a user can bookmark:

```text
/settings?tab=security
```

and go straight to that view.

---

# 20. Example: pagination

Suppose:

```text
/products?page=3
```

You can read:

```jsx
const page =
  Number(searchParams.get("page") ?? 1);
```

Now:

```jsx
<button
  onClick={() =>
    setSearchParams({
      page: String(page + 1)
    })
  }
>
  Next
</button>
```

Notice something:

URL values are strings.

So:

```text
"3"
```

is not:

```text
3
```

That's why:

```js
Number(...)
```

may be necessary.

This is one reason you should think of URL values as **serialized data**.

---

# 21. URL state is serialization

This is a deeper concept.

React state can contain:

```js
{
  selectedProducts: [...],
  complexObject: ...
}
```

The URL can't naturally store that JavaScript object directly.

It needs a string representation:

```text
?product=42
```

or:

```text
?category=electronics
```

So you're converting:

```text
JavaScript value
      ↓
string in URL
```

and back:

```text
string in URL
      ↓
JavaScript value
```

This is essentially serialization/deserialization.

---

# 22. Don't put everything in the URL

This is just as important as knowing when to use it.

Don't turn:

```text
isModalOpen
```

into:

```text
?modal=true
```

unless there is a real navigation/shareability reason.

For example, a temporary confirmation dialog usually belongs in local state:

```jsx
const [isOpen, setIsOpen] =
  useState(false);
```

Whereas:

```text
/products?category=electronics
```

is a good URL state candidate because it's part of the user's navigable view.

---

# 23. A useful decision rule

Ask:

> **Would it make sense for someone to copy/paste/bookmark this state as a URL?**

If yes, URL state may be appropriate.

Examples:

```text
search results
filters
sorting
pagination
selected tab
```

If no:

```text
modal open
dropdown open
hover state
temporary animation state
```

local React state is usually more appropriate.

---

# 24. Combining URL state with ordinary state

You don't need to choose one system for the entire application.

For example:

```text
URL:
search
category
page

React state:
isMobileMenuOpen
isDeleteModalOpen
draftComment
```

So:

```text
                    App
                      │
          ┌───────────┴────────────┐
          ↓                        ↓
       URL state             local state
          │                        │
 search/category/page     modal/open/draft
```

This is perfectly normal.

---

# 25. Search input + URL state

Let's make a better implementation.

```jsx
function ProductSearch() {
  const [searchParams, setSearchParams] =
    useSearchParams();

  const search =
    searchParams.get("search") ?? "";

  function handleChange(e) {
    const nextSearch = e.target.value;

    setSearchParams({
      search: nextSearch
    });
  }

  return (
    <input
      value={search}
      onChange={handleChange}
      placeholder="Search products"
    />
  );
}
```

Now notice the data flow:

```text
URL
 ↓
searchParams
 ↓
search
 ↓
input value
```

When user types:

```text
input
 ↓
handleChange
 ↓
setSearchParams
 ↓
URL changes
 ↓
searchParams changes
 ↓
search changes
 ↓
input renders with new value
```

It's the same feedback-loop idea you learned with controlled inputs.

The difference is where the value lives.

---

# 26. This also means refresh works

Suppose the URL is:

```text
/products?search=laptop
```

The browser is refreshed.

React starts again.

The router reads:

```text
search=laptop
```

Your component gets:

```js
searchParams.get("search")
```

and therefore:

```text
search = "laptop"
```

The UI reconstructs itself from the URL.

This is a very strong property.

---

# 27. Compare with ordinary state

With:

```jsx
const [search, setSearch] =
  useState("");
```

a refresh normally gives you the initial state again unless you've persisted it elsewhere.

With:

```jsx
const [searchParams] =
  useSearchParams();
```

the browser URL itself preserves the information.

So:

```text
useState
→ memory associated with React state

useSearchParams
→ browser URL
```

This distinction will matter a lot in real applications.

---

# 28. One important current React Router detail

`useSearchParams` returns a stable `URLSearchParams` object, but that object is also mutable.

So don't do:

```js
searchParams.set("search", "laptop");
```

and expect the URL to update automatically.

You need:

```jsx
setSearchParams(...)
```

to actually navigate/update the URL. ([React Router](https://reactrouter.com/api/hooks/useSearchParams?utm_source=chatgpt.com "useSearchParams | React Router"))

This is a subtle but important difference from simply mutating a JavaScript object.

---

# 29. A complete product page

Let's put everything together.

```jsx
import { useSearchParams } from "react-router";

const products = [
  {
    id: 1,
    name: "Laptop",
    category: "electronics",
    price: 80000
  },
  {
    id: 2,
    name: "Phone",
    category: "electronics",
    price: 50000
  },
  {
    id: 3,
    name: "Laptop Stand",
    category: "accessories",
    price: 3000
  }
];

function Products() {
  const [searchParams, setSearchParams] =
    useSearchParams();

  const search =
    searchParams.get("search") ?? "";

  const category =
    searchParams.get("category") ?? "all";

  const visibleProducts = products.filter(
    product => {
      const matchesSearch =
        product.name
          .toLowerCase()
          .includes(
            search.toLowerCase()
          );

      const matchesCategory =
        category === "all" ||
        product.category === category;

      return (
        matchesSearch &&
        matchesCategory
      );
    }
  );

  function handleSearchChange(e) {
    setSearchParams({
      search: e.target.value,
      category,
      page: "1"
    });
  }

  function handleCategoryChange(e) {
    setSearchParams({
      search,
      category: e.target.value,
      page: "1"
    });
  }

  return (
    <main>
      <h1>Products</h1>

      <input
        value={search}
        onChange={handleSearchChange}
        placeholder="Search"
      />

      <select
        value={category}
        onChange={handleCategoryChange}
      >
        <option value="all">
          All
        </option>

        <option value="electronics">
          Electronics
        </option>

        <option value="accessories">
          Accessories
        </option>
      </select>

      <p>
        Showing {visibleProducts.length} products
      </p>

      <ul>
        {visibleProducts.map(product => (
          <li key={product.id}>
            {product.name} — ₹{product.price}
          </li>
        ))}
      </ul>
    </main>
  );
}
```

Now we have:

```text
URL
 │
 ├── search
 ├── category
 └── page
 │
 ↓
useSearchParams()
 │
 ↓
derived values
 │
 ↓
filter()
 │
 ↓
map()
 │
 ↓
UI
```

That's a very realistic React pattern.

---

# 30. Notice what we did NOT store in state

We didn't write:

```jsx
const [visibleProducts, setVisibleProducts] =
  useState([]);
```

because:

```text
products
+
search
+
category
↓
visibleProducts
```

is derived.

That's exactly the state-design principle you've already learned.

---

# 31. URL state + route params

Now imagine:

```text
/products/42?tab=reviews
```

There are two kinds of URL state:

```text
/products/42
     ↓
productId = "42"

?tab=reviews
     ↓
tab = "reviews"
```

So:

```jsx
const { productId } = useParams();

const [searchParams] =
  useSearchParams();

const tab =
  searchParams.get("tab");
```

Now the component has:

```text
productId
tab
```

from two different parts of the URL.

This distinction is important:

```text
/products/:productId
→ route parameter

?tab=reviews
→ search parameter
```

React Router exposes route params through `useParams` and search params through `useSearchParams`. ([React Router](https://reactrouter.com/start/declarative/url-values?utm_source=chatgpt.com "URL Values | React Router"))

---

# 32. URL state, React state, and server state

This is probably the biggest architectural concept introduced in today's lesson.

Suppose you're building:

```text
/products?search=laptop&category=electronics&page=2
```

You might have:

```text
URL state:
search
category
page

local React state:
isCartOpen

server state:
products
```

This gives:

```text
                Product Page
                     │
        ┌────────────┼─────────────┐
        ↓            ↓             ↓
      URL          React         Server
     state         state          data
        │            │             │
 search/category   cartOpen     products
 page
```

These are different kinds of state.

Trying to manage all three identically is a common source of messy React architecture.

We're going to keep that distinction throughout the rest of the course.

---

# 33. The decision-making framework

When you encounter some value, ask:

### Is it derived?

```text
filteredProducts
totalPrice
isEmpty
```

→ calculate it.

### Does it need to drive the UI and belong only to this component/subtree?

```text
modalOpen
menuOpen
draft
```

→ `useState` / `useReducer`.

### Does it represent navigation or something worth bookmarking/sharing?

```text
search
filter
sort
page
tab
```

→ consider URL state.

### Does it come from a backend?

```text
products
orders
profile
comments
```

→ server state.

That last category is where we're going next.

---

# 34. You are now past the "basic React" stage

At this point, you're no longer just learning:

```text
useState
useEffect
useContext
```

as isolated APIs.

You're starting to learn:

```text
"What kind of data is this?"
        ↓
"Where should it live?"
        ↓
"Who owns it?"
        ↓
"How should components access it?"
```

That's the real skill behind React architecture.

---

# Practice challenge

Build:

```text
/products
```

with:

```text
Search: [____________]

Category:
[All ▼]

Sort:
[Price: Low → High ▼]

Page:
[ 1 ] [ 2 ] [ 3 ]

Products...
```

Make these URL-driven:

```text
search
category
sort
page
```

So a URL might become:

```text
/products?search=laptop&category=electronics&sort=price_asc&page=2
```

Then refresh the browser.

The page should reconstruct the same view from the URL.

While doing it, consciously separate:

```text
URL state
derived data
ordinary React state
```

That's the important part of the exercise, not the visual design.

---

## Next lesson — Data Fetching and Server State

Now we're ready for the first big real-world problem.

So far our products have been hard-coded:

```js
const products = [...]
```

Real applications get them from:

```text
GET /api/products
```

We'll build the entire flow:

```text
component
 ↓
request
 ↓
loading
 ↓
success / error
 ↓
server data
 ↓
UI
```

Then we'll tackle the problems that appear immediately:

```text
What if the request fails?
What if the component disappears while the request is running?
What if the user changes the URL quickly?
What if two requests race?
Should server data be in useState?
When does useEffect belong here?
```

That will take us into **server state**, which is one of the most important concepts for building serious React applications.