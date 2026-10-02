
# Lesson 31 — TypeScript with React: Props, State, Events, Refs, Hooks, and Why Types Actually Matter

We've now reached a point where adding TypeScript makes sense.

Not because TypeScript is required to use React—it isn't—but because the components and data flows we're writing are now complex enough that **types can describe the contracts we've been building**.

And I don't want to teach TypeScript as:

```text
string
number
boolean
interface
done
```

That misses the interesting part.

You already know JavaScript, so we'll focus on:

```text
TypeScript
      ↓
React component contracts
      ↓
props
      ↓
state
      ↓
events
      ↓
refs
      ↓
Context
      ↓
reducers
      ↓
server data
      ↓
generic reusable components
```

Current React supports TypeScript directly through JSX files (`.tsx`), and React's type definitions provide the types for React APIs and DOM interactions. TypeScript's JSX support is built into the language/compiler. ([TypeScript](https://www.typescriptlang.org/docs/handbook/jsx.html?utm_source=chatgpt.com "TypeScript: Documentation - JSX"))

---

# 1. What problem is TypeScript solving?

Suppose JavaScript:

```jsx
function ProductCard({ product }) {
  return (
    <h2>{product.name}</h2>
  );
}
```

You and I know we expect:

```js
product = {
  id: 42,
  name: "Laptop",
  price: 999,
}
```

But JavaScript doesn't enforce that.

Someone could accidentally pass:

```js
product = {
  title: "Laptop",
}
```

and now:

```js
product.name
```

is:

```text
undefined
```

Or perhaps:

```js
product = "hello";
```

and:

```js
product.name
```

is conceptually nonsense.

TypeScript lets us say:

> **"This component expects a product with this structure."**

That's the first major benefit.

---

# 2. TypeScript is mostly about contracts

Imagine this component:

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};

function ProductCard({
  product,
}: {
  product: Product;
}) {
  return (
    <article>
      <h2>{product.name}</h2>
      <p>${product.price}</p>
    </article>
  );
}
```

Now the component has a contract:

```text
ProductCard
expects
    ↓
product
    ↓
{
  id: number
  name: string
  price: number
}
```

The interesting thing is that the type describes **the boundary between components**.

That's exactly the same idea we've been discussing with component API design.

---

# 3. Think of a component as a function

We learned very early:

```jsx
function ProductCard(props) {
  ...
}
```

is conceptually:

```text
component
=
function receiving an argument
```

TypeScript lets us type that argument:

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};

type ProductCardProps = {
  product: Product;
};

function ProductCard({
  product,
}: ProductCardProps) {
  ...
}
```

So:

```text
JavaScript
function
   ↓
input

TypeScript
function
   ↓
typed input
```

That's a very natural connection.

---

# 4. You don't have to create a named props type every time

This:

```tsx
function ProductCard({
  product,
}: {
  product: Product;
}) {
  ...
}
```

is perfectly valid.

Or:

```tsx
type ProductCardProps = {
  product: Product;
};

function ProductCard({
  product,
}: ProductCardProps) {
  ...
}
```

Both express the same basic thing.

Use a named type when:

```text
the type is reused
the props are substantial
you want a descriptive name
```

Use inline typing when:

```text
the props are tiny
the component is simple
```

Don't create ten files containing:

```text
FooProps
BarProps
BazProps
```

just to make the code look "typed."

---

# 5. Optional props

Suppose:

```tsx
type ProductCardProps = {
  product: Product;
  showPrice?: boolean;
};
```

The `?` means:

```text
showPrice
=
boolean OR undefined
```

So:

```tsx
<ProductCard product={product} />
```

is valid.

And:

```tsx
<ProductCard
  product={product}
  showPrice={false}
/>
```

is also valid.

Inside:

```tsx
function ProductCard({
  product,
  showPrice,
}: ProductCardProps) {
  if (showPrice) {
    ...
  }
}
```

you must account for the fact that:

```text
showPrice
```

might be:

```text
undefined
```

This maps very naturally to JavaScript optional values.

---

# 6. Don't use `?` just because you can

Compare:

```tsx
type ButtonProps = {
  label?: string;
};
```

with:

```tsx
type ButtonProps = {
  label: string;
};
```

If a Button genuinely needs a label, make it required.

The type should describe reality.

Otherwise you've told TypeScript:

> "Sometimes this component has no label."

Now your implementation must deal with that possibility.

A useful principle is:

> **A good type describes what is actually allowed, not what would be convenient to type.**

---

# 7. Props and children

Suppose:

```tsx
function Card({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <section className="card">
      {children}
    </section>
  );
}
```

Now:

```tsx
<Card>
  <h2>Hello</h2>
  <p>Content</p>
</Card>
```

The important type here is:

```text
React.ReactNode
```

Conceptually, it means:

> a value that React can render as children.

This is much broader than:

```text
JSX.Element
```

because children can involve things such as:

```text
elements
strings
numbers
arrays/fragments
null
etc.
```

So for a general `children` prop, `React.ReactNode` is usually the appropriate broad type.

---

# 8. Don't automatically type children as `JSX.Element`

Suppose:

```tsx
type Props = {
  children: JSX.Element;
};
```

That's too restrictive for many component APIs.

This would potentially exclude valid children such as:

```tsx
<Card>
  Hello
</Card>
```

or:

```tsx
<Card>
  {condition && <Something />}
</Card>
```

because React's renderable children aren't limited to one JSX element.

So:

```text
React.ReactNode
```

is generally the broader "renderable content" concept.

React's TypeScript guidance emphasizes using React's supplied types for common component patterns. ([React](https://react.dev/learn/typescript?utm_source=chatgpt.com "Using TypeScript – React"))

---

# 9. State is where TypeScript gets especially useful

Consider:

```tsx
const [count, setCount] = useState(0);
```

TypeScript can infer:

```text
count
→ number
```

and:

```text
setCount
→ accepts number or updater function
```

You usually don't need to write:

```tsx
const [count, setCount] =
  useState<number>(0);
```

because the initial value tells TypeScript enough.

This is a very important principle:

> **Use inference whenever TypeScript can already see the correct type.**

Don't annotate everything just because you can.

---

# 10. When `useState` needs an explicit type

Here's a common example:

```tsx
const [user, setUser] =
  useState(null);
```

What does TypeScript infer?

Essentially:

```text
user = null
```

which means you can't later do:

```tsx
setUser({
  id: 42,
  name: "Alice",
});
```

because TypeScript inferred too narrow a type.

So you write:

```tsx
type User = {
  id: number;
  name: string;
};

const [user, setUser] =
  useState<User | null>(null);
```

Now the state model is:

```text
user
 ↓
User OR null
```

That's exactly what the actual application means.

---

# 11. `null` is valuable information

You should not think:

```text
User | null
```

as TypeScript being annoying.

It is encoding a real state machine:

```text
initially:
user = null

after login/load:
user = User
```

So:

```tsx
const [user, setUser] =
  useState<User | null>(null);
```

communicates:

```text
not loaded / not authenticated
        OR
actual user
```

This is much more informative than:

```tsx
const [user, setUser] =
  useState<any>(null);
```

---

# 12. TypeScript forces you to acknowledge impossible assumptions

Suppose:

```tsx
if (user) {
  console.log(user.name);
}
```

TypeScript understands:

```text
inside this branch
user ≠ null
```

This is called **narrowing**.

So:

```text
User | null
```

becomes:

```text
User
```

inside the guarded branch.

TypeScript's control-flow narrowing is one of its most useful features for React code because state frequently moves between several possible values. ([TypeScript](https://www.typescriptlang.org/docs/handbook/?utm_source=chatgpt.com "TypeScript: The starting point for learning TypeScript"))

---

# 13. Nullable async state

This pattern appears constantly:

```tsx
type Product = {
  id: number;
  name: string;
};

const [product, setProduct] =
  useState<Product | null>(null);
```

Then:

```tsx
if (!product) {
  return <Spinner />;
}

return (
  <h1>{product.name}</h1>
);
```

The control flow tells TypeScript:

```text
before check:
Product | null

after:
Product
```

And that corresponds exactly to our React UI model:

```text
no data
 ↓
loading
 ↓
product exists
```

---

# 14. State unions become even more powerful

Instead of:

```tsx
const [loading, setLoading] =
  useState(false);

const [error, setError] =
  useState<string | null>(null);

const [data, setData] =
  useState<Product | null>(null);
```

you can model the **entire state machine**:

```tsx
type ProductState =
  | {
      status: "loading";
    }
  | {
      status: "success";
      data: Product;
    }
  | {
      status: "error";
      error: string;
    };
```

Now:

```tsx
const [state, setState] =
  useState<ProductState>({
    status: "loading",
  });
```

And:

```tsx
if (state.status === "loading") {
  return <Spinner />;
}

if (state.status === "error") {
  return <ErrorMessage />;
}

return (
  <ProductView product={state.data} />
);
```

TypeScript understands:

```text
status = loading
→ only loading fields exist

status = error
→ error exists

status = success
→ data exists
```

This is a beautiful connection to our earlier lessons about:

```text
state machines
useReducer
server-state lifecycle
```

---

# 15. Discriminated unions

The technique above is called a **discriminated union**.

This:

```tsx
type ProductState =
  | { status: "loading" }
  | { status: "success"; data: Product }
  | { status: "error"; error: string };
```

has:

```text
status
```

as the discriminator.

So:

```tsx
if (state.status === "success") {
```

narrows:

```text
state
```

to:

```tsx
{
  status: "success";
  data: Product;
}
```

Now:

```tsx
state.data
```

is guaranteed to exist.

This is much safer than:

```text
data can be anything
maybe null
maybe undefined
maybe error
```

---

# 16. This is especially useful with reducers

Remember our reducer model:

```text
(state, action)
       ↓
   next state
```

TypeScript fits beautifully here.

Suppose:

```tsx
type Todo = {
  id: number;
  text: string;
  completed: boolean;
};

type TodoState = Todo[];

type Action =
  | {
      type: "added";
      todo: Todo;
    }
  | {
      type: "toggled";
      id: number;
    }
  | {
      type: "deleted";
      id: number;
    };
```

Reducer:

```tsx
function todoReducer(
  state: TodoState,
  action: Action
): TodoState {
  switch (action.type) {
    case "added":
      return [...state, action.todo];

    case "toggled":
      return state.map(todo =>
        todo.id === action.id
          ? {
              ...todo,
              completed:
                !todo.completed,
            }
          : todo
      );

    case "deleted":
      return state.filter(
        todo => todo.id !== action.id
      );
  }
}
```

Now TypeScript knows:

```text
action.type === "added"
      ↓
action.todo exists

action.type === "toggled"
      ↓
action.id exists
```

This is a very natural pairing:

```text
useReducer
+
discriminated union
```

---

# 17. Exhaustiveness

Suppose you add:

```tsx
type Action =
  | { type: "added"; todo: Todo }
  | { type: "toggled"; id: number }
  | { type: "deleted"; id: number }
  | { type: "archived"; id: number };
```

but forget to update the reducer.

A carefully typed reducer can expose that you haven't handled every action.

A common technique uses:

```tsx
function assertNever(
  value: never
): never {
  throw new Error(
    "Unhandled action"
  );
}
```

Then:

```tsx
default:
  return assertNever(action);
```

Now if `action` could still be `"archived"`, TypeScript complains.

This is a powerful pattern because it turns:

```text
"Oops, we forgot a case."
```

into:

```text
compile-time warning/error
```

That's one of the real strengths of static typing.

---

# 18. Event types

This is where React + TypeScript becomes immediately practical.

Suppose:

```tsx
function Form() {
  function handleChange(
    event: React.ChangeEvent<HTMLInputElement>
  ) {
    console.log(event.currentTarget.value);
  }

  return (
    <input onChange={handleChange} />
  );
}
```

The event is explicitly typed.

React's TypeScript guidance shows this same style for extracted event handlers. ([React](https://react.dev/learn/typescript?utm_source=chatgpt.com "Using TypeScript – React"))

But there's an important trick:

> **Often TypeScript can infer the event type for you.**

For example:

```tsx
<input
  onChange={event => {
    console.log(
      event.currentTarget.value
    );
  }}
/>
```

Since React knows this is an input's `onChange`, it can infer the event type.

So you often only need an explicit event type when you extract the handler:

```tsx
function handleChange(event: React.ChangeEvent<HTMLInputElement>) {
  ...
}
```

This is a good example of letting inference do useful work.

---

# 19. `currentTarget` vs `target`

This connects to the events lesson.

You may write:

```tsx
event.target.value
```

But TypeScript often gives you a more useful relationship through:

```tsx
event.currentTarget.value
```

For a properly typed input change event:

```text
currentTarget
→ the input receiving the handler

target
→ the actual origin of the event
```

That distinction exists in browser event semantics, and TypeScript reflects it.

So in a typed handler:

```tsx
function handleChange(
  event: React.ChangeEvent<HTMLInputElement>
) {
  setValue(event.currentTarget.value);
}
```

is often clean and explicit.

---

# 20. Button event types

Suppose:

```tsx
function handleClick(
  event: React.MouseEvent<HTMLButtonElement>
) {
  ...
}
```

This says:

```text
MouseEvent
for
HTMLButtonElement
```

You don't need to memorize dozens of event types.

The pattern is mostly:

```text
React.<EventType><HTMLElementType>
```

Examples:

```text
React.ChangeEvent<HTMLInputElement>

React.ChangeEvent<HTMLSelectElement>

React.MouseEvent<HTMLButtonElement>

React.FormEvent<HTMLFormElement>

React.KeyboardEvent<HTMLInputElement>
```

The exact event type depends on the DOM event and element involved.

---

# 21. Form submit events

You can write:

```tsx
function handleSubmit(
  event: React.FormEvent<HTMLFormElement>
) {
  event.preventDefault();
}
```

Now TypeScript knows:

```text
event.currentTarget
=
HTMLFormElement
```

So you can access form-specific properties safely.

Again, this is simply describing the boundary:

```text
browser
 ↓
React handler
 ↓
typed event
```

---

# 22. Refs

We just learned refs in depth.

Now add TypeScript.

For an input:

```tsx
const inputRef =
  useRef<HTMLInputElement | null>(null);
```

Now TypeScript understands:

```text
inputRef.current
=
HTMLInputElement | null
```

So:

```tsx
inputRef.current?.focus();
```

works because optional chaining accounts for the possibility that it is still `null`.

---

# 23. Why the `null` is important

At render time:

```text
input
might not yet be attached
```

So:

```tsx
inputRef.current
```

can legitimately be:

```text
null
```

After mount:

```text
HTMLInputElement
```

TypeScript is encoding the lifecycle we already learned.

That's important:

> **Good types often make lifecycle assumptions explicit.**

---

# 24. DOM element types

You may encounter:

```text
HTMLInputElement
HTMLButtonElement
HTMLDivElement
HTMLFormElement
HTMLSelectElement
HTMLTextAreaElement
HTMLVideoElement
```

These correspond to actual browser DOM interfaces.

So:

```tsx
const buttonRef =
  useRef<HTMLButtonElement | null>(null);
```

means:

```text
ref.current
→ button DOM element or null
```

This is vastly better than:

```tsx
useRef<any>(null)
```

because now TypeScript knows what methods/properties are available.

---

# 25. Custom imperative handles

Now our previous `useImperativeHandle` lesson becomes interesting.

Suppose:

```tsx
type EditorHandle = {
  focus: () => void;
  clear: () => void;
};
```

Then:

```tsx
function Editor({
  ref,
}: {
  ref: React.Ref<EditorHandle>;
}) {
  ...
}
```

And:

```tsx
useImperativeHandle(ref, () => ({
  focus() {
    ...
  },

  clear() {
    ...
  },
}));
```

Now the parent gets:

```text
ref.current
 ↓
EditorHandle
 ↓
focus()
clear()
```

TypeScript can guarantee that the parent only expects capabilities that the component says it exposes.

This is exactly what we discussed as a component's **imperative API**.

---

# 26. Callback props

Suppose:

```tsx
type SearchBoxProps = {
  value: string;
  onChange: (value: string) => void;
};
```

Then:

```tsx
function SearchBox({
  value,
  onChange,
}: SearchBoxProps) {
  return (
    <input
      value={value}
      onChange={event =>
        onChange(
          event.currentTarget.value
        )
      }
    />
  );
}
```

Now the component contract says:

```text
onChange
must be a function
that accepts string
and returns void
```

That is much more precise than:

```text
onChange: Function
```

---

# 27. Don't use `Function` as a prop type

Avoid:

```tsx
type Props = {
  onClick: Function;
};
```

That tells TypeScript almost nothing useful.

Compare:

```tsx
type Props = {
  onClick: () => void;
};
```

Now we know:

```text
takes no arguments
returns nothing
```

Or:

```tsx
type Props = {
  onSelect: (id: number) => void;
};
```

Now we know:

```text
takes number
returns nothing
```

This is a major benefit of explicit function types.

---

# 28. Return types don't always need annotations

You might write:

```tsx
function getFullName(
  firstName: string,
  lastName: string
) {
  return `${firstName} ${lastName}`;
}
```

TypeScript infers:

```text
return type = string
```

No need to write:

```tsx
function getFullName(
  firstName: string,
  lastName: string
): string {
```

unless the explicit return type provides useful documentation or constraints.

Again:

> **Let inference carry information when it can.**

Use annotations where they clarify boundaries.

---

# 29. Props are boundaries; internal calculations often aren't

Suppose:

```tsx
function ProductCard({
  product,
}: {
  product: Product;
}) {
  const priceWithTax =
    product.price * 1.18;

  return ...
}
```

You don't necessarily need:

```tsx
const priceWithTax: number =
```

because TypeScript already knows it.

This gives us a useful rule:

```text
external boundary
→ explicit types are valuable

internal obvious calculation
→ inference is usually enough
```

That keeps TypeScript readable.

---

# 30. Union types for UI states

Imagine a button:

```text
idle
saving
success
error
```

Instead of:

```tsx
const [status, setStatus] =
  useState<string>("idle");
```

use:

```tsx
type Status =
  | "idle"
  | "saving"
  | "success"
  | "error";
```

Then:

```tsx
const [status, setStatus] =
  useState<Status>("idle");
```

Now this:

```tsx
setStatus("saving");
```

is valid.

But:

```tsx
setStatus("banana");
```

is rejected.

This is incredibly useful for UI state because the values aren't arbitrary strings.

They're a **finite state machine**.

---

# 31. This is better than booleans

Consider:

```tsx
const [loading, setLoading] =
  useState(false);

const [error, setError] =
  useState(false);

const [success, setSuccess] =
  useState(false);
```

Now you can accidentally have:

```text
loading = true
success = true
error = true
```

What does that mean?

Your state model permits contradictory combinations.

With:

```tsx
type Status =
  | "idle"
  | "loading"
  | "success"
  | "error";
```

only one state exists at a time.

This is not just a TypeScript trick.

It's **better state modeling**.

---

# 32. TypeScript can expose bad state architecture

Suppose:

```tsx
type State = {
  data: Product | null;
  loading: boolean;
  error: string | null;
};
```

All of this is technically possible:

```text
loading = true
data = valid Product
error = "failed"
```

Is that a valid state?

Maybe.

Maybe not.

TypeScript can't automatically know your business rules from those three fields.

But a discriminated union forces you to explicitly define the valid combinations:

```tsx
type State =
  | { status: "loading" }
  | { status: "success"; data: Product }
  | { status: "error"; error: string };
```

Now the architecture itself becomes clearer.

That's one reason TypeScript is much more than "catch typos."

---

# 33. Typing Context

Suppose:

```tsx
type Theme = "light" | "dark";

type ThemeContextValue = {
  theme: Theme;
  setTheme: (theme: Theme) => void;
};
```

Then:

```tsx
const ThemeContext =
  createContext<ThemeContextValue | null>(
    null
  );
```

Why `null`?

Because before you're inside the provider, there's no actual context value.

Then create a custom hook:

```tsx
function useTheme() {
  const context =
    useContext(ThemeContext);

  if (!context) {
    throw new Error(
      "useTheme must be used inside ThemeProvider"
    );
  }

  return context;
}
```

Now callers get:

```text
theme: Theme
setTheme: (theme: Theme) => void
```

without having to repeatedly deal with `null`.

This is a very nice pattern.

---

# 34. Context + custom hook + TypeScript

The architecture becomes:

```text
ThemeContext
     ↓
typed value
     ↓
ThemeProvider
     ↓
useTheme()
     ↓
components
```

And the custom hook effectively converts:

```text
ThemeContextValue | null
```

into:

```text
ThemeContextValue
```

after validating the provider exists.

That's another example of a custom hook becoming a well-defined API.

---

# 35. Typing server responses

Suppose:

```json
{
  "id": 42,
  "name": "Laptop",
  "price": 999
}
```

You might define:

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};
```

Then:

```tsx
async function fetchProduct(
  id: number
): Promise<Product> {
  const response =
    await fetch(`/api/products/${id}`);

  if (!response.ok) {
    throw new Error(
      "Failed to fetch product"
    );
  }

  return response.json();
}
```

Now the TypeScript contract says:

```text
fetchProduct(number)
        ↓
Promise<Product>
```

So:

```tsx
const product =
  await fetchProduct(42);
```

is understood as:

```text
Promise<Product>
```

and after `await`:

```text
Product
```

This makes your server-state code much easier to reason about.

---

# 36. But here's a very important limitation

This:

```tsx
async function fetchProduct(
  id: number
): Promise<Product> {
  ...
}
```

does **not prove the server actually returned a Product**.

It only tells TypeScript:

> "I'm claiming this function returns Product."

If the API sends:

```json
{
  "banana": true
}
```

the browser doesn't magically stop it.

TypeScript's types are erased at runtime.

So:

```text
TypeScript
→ compile-time/static checking

API response
→ runtime data
```

These are different worlds.

This is one of the most important concepts in typed frontend development.

---

# 37. Why runtime validation sometimes matters

Suppose:

```text
backend
 ↓
external API
 ↓
unknown JSON
```

Your TypeScript type says:

```tsx
type Product = {
  id: number;
  name: string;
};
```

But runtime JSON is untrusted input.

For high-value boundaries, you may use runtime validation libraries or manually validate the response.

Conceptually:

```text
unknown JSON
      ↓
runtime validation
      ↓
validated Product
      ↓
application
```

This gives you:

```text
runtime safety
+
compile-time safety
```

Libraries such as Zod are commonly used for this, but the important idea is broader than any specific library.

We'll come back to schema validation later when we get deeper into production API architecture.

---

# 38. `unknown` is much safer than `any`

Suppose you truly don't know what some value is:

```tsx
const data: unknown = responseData;
```

Now TypeScript says:

> "You don't know what this is yet."

You can't just do:

```tsx
data.name
```

because that hasn't been established.

You must narrow it first.

That's good.

Compare:

```tsx
const data: any = responseData;
```

Now TypeScript essentially stops protecting you.

You can do:

```tsx
data.name
data.foo.bar
data.doWhatever()
```

and TypeScript mostly says:

> "Sure."

So:

```text
unknown
→ "prove what this is"

any
→ "trust me"
```

Prefer `unknown` when the actual type is not known.

---

# 39. Typing reusable generic components

Now we get to one of the most useful advanced TypeScript + React ideas.

Suppose we want a reusable list:

```tsx
function List({ items }) {
  return items.map(item => ...);
}
```

But what type is `item`?

It could be:

```text
Product
User
Todo
Message
Order
```

Generics solve this.

Conceptually:

```tsx
function List<T>({
  items,
  renderItem,
}: {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
}) {
  return (
    <>
      {items.map(renderItem)}
    </>
  );
}
```

Now:

```tsx
<List
  items={products}
  renderItem={product => (
    <ProductCard product={product} />
  )}
/>
```

TypeScript infers:

```text
T = Product
```

So:

```text
items
→ Product[]

renderItem
→ (product: Product) => ReactNode
```

This is a beautiful combination of:

```text
generics
+
map
+
component composition
```

---

# 40. Why generics matter

Without generics, you might write:

```tsx
items: any[]
```

which destroys type information.

Or:

```tsx
items: Product[]
```

which makes the component only useful for products.

Generics allow:

```text
same component
+
preserved type relationship
```

For example:

```text
List<Product>
List<User>
List<Todo>
```

without losing the specific type inside each usage.

That is one of TypeScript's strongest tools for reusable component APIs.

---

# 41. Generic component relationships

Consider:

```tsx
function Select<T>({
  options,
  value,
  onChange,
}: {
  options: T[];
  value: T;
  onChange: (value: T) => void;
}) {
  ...
}
```

Now:

```tsx
const colors = [
  "red",
  "blue",
  "green",
];

<Select
  options={colors}
  value="red"
  onChange={color => {
    ...
  }}
/>
```

TypeScript can infer:

```text
T = string
```

But you can also use objects:

```tsx
type User = {
  id: number;
  name: string;
};

<Select
  options={users}
  value={selectedUser}
  onChange={setSelectedUser}
/>
```

Now:

```text
T = User
```

The same component preserves the relationship between:

```text
options
value
onChange
```

That's a real reusable component contract.

---

# 42. Generics are useful when values are related

This is the core idea.

Bad:

```text
options: any[]
value: any
onChange: (x: any) => void
```

Type information is lost.

Good:

```text
options: T[]
value: T
onChange: (value: T) => void
```

Now TypeScript understands:

```text
these three things must agree on the same type
```

That's what generics are really buying you.

---

# 43. Component props and HTML props

You might want a custom button that supports normal button attributes.

For example:

```tsx
type ButtonProps =
  React.ComponentProps<"button"> & {
    variant?: "primary" | "secondary";
  };
```

Then:

```tsx
function Button({
  variant = "primary",
  ...props
}: ButtonProps) {
  return (
    <button
      data-variant={variant}
      {...props}
    />
  );
}
```

Now the component inherits normal button props such as:

```text
type
disabled
onClick
aria-*
name
value
```

while adding:

```text
variant
```

This can be extremely useful when creating reusable design-system components.

---

# 44. Why not manually type every button prop?

You could:

```tsx
type ButtonProps = {
  onClick?: ...
  disabled?: boolean;
  type?: ...
  ...
};
```

But native HTML already has a rich type definition.

Reusing it means:

```text
browser's button API
+
your component's additions
```

That's much more maintainable.

React's type definitions include these DOM attribute types for JSX elements. ([TypeScript](https://www.typescriptlang.org/docs/handbook/jsx.html?utm_source=chatgpt.com "TypeScript: Documentation - JSX"))

---

# 45. This becomes powerful for design systems

Suppose:

```tsx
type ButtonProps =
  React.ComponentProps<"button"> & {
    variant?: "primary" | "danger";
    loading?: boolean;
  };
```

Now:

```tsx
<Button
  type="submit"
  disabled={loading}
  aria-label="Save changes"
  variant="primary"
  loading={loading}
/>
```

Your Button doesn't need to reinvent the entire native button API.

This is excellent component API design.

---

# 46. But be careful with overly clever generic types

You can create horrifying TypeScript:

```text
type Whatever<T extends ...
  ? ...
  : ...
>
```

that nobody on the team understands.

The goal isn't:

> "Can I encode every possible relationship in the type system?"

The goal is:

> **"Does this type make the component's contract clearer and safer?"**

A simple:

```tsx
type ButtonProps = {
  variant: "primary" | "danger";
  disabled?: boolean;
};
```

is often better than a 60-line conditional generic.

---

# 47. Type aliases vs interfaces

You'll encounter:

```tsx
type Product = {
  id: number;
  name: string;
};
```

and:

```tsx
interface Product {
  id: number;
  name: string;
}
```

For everyday React props and data shapes, either can work.

You don't need to turn this into a philosophical argument.

A practical style is:

```text
type
→ unions
→ function types
→ compositional type expressions

interface
→ object-shaped public contracts
→ extension in some codebases
```

But both can describe object shapes.

Consistency within your project matters more than declaring one globally superior.

---

# 48. Literal types are extremely useful in React

Remember:

```tsx
type Variant =
  | "primary"
  | "secondary"
  | "danger";
```

This is much better than:

```tsx
variant: string;
```

because now:

```tsx
<Button variant="primary" />
```

is allowed.

But:

```tsx
<Button variant="banana" />
```

is rejected.

This is particularly useful for:

```text
status
variant
size
alignment
theme
mode
role
sort direction
```

In other words:

> **When a value can only be one of a small known set, encode the set.**

---

# 49. Typing URL state

This connects to our routing lesson.

Suppose:

```text
?sort=price&page=2
```

The URL gives you strings.

So:

```tsx
const sort =
  searchParams.get("sort");
```

has a potentially broad result.

You can transform it:

```tsx
type Sort =
  | "price"
  | "name"
  | "rating";

function parseSort(
  value: string | null
): Sort {
  if (
    value === "price" ||
    value === "name" ||
    value === "rating"
  ) {
    return value;
  }

  return "name";
}
```

Now:

```text
raw URL
 ↓
string | null
 ↓
validation/parsing
 ↓
Sort
```

This is an excellent boundary pattern.

The browser gives you untrusted serialized input.

Your application converts it into a known domain type.

---

# 50. This same idea works for API data

The pattern is:

```text
outside world
 ↓
unknown/raw data
 ↓
validate/parse
 ↓
domain type
 ↓
React
```

External sources include:

```text
URL
API
localStorage
user input
postMessage
third-party libraries
```

TypeScript helps after you've established what the data is.

It does not magically validate the outside world at runtime.

This is one of the most important concepts to remember.

---

# 51. TypeScript + TanStack Query

Suppose:

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};
```

Then:

```tsx
const productsQuery =
  useQuery<Product[]>({
    queryKey: ["products"],
    queryFn: fetchProducts,
  });
```

Depending on the way the query function itself is typed, TypeScript can often infer the `Product[]` result without you manually providing the generic.

For example:

```tsx
async function fetchProducts():
  Promise<Product[]> {
  const response =
    await fetch("/api/products");

  if (!response.ok) {
    throw new Error("Failed");
  }

  return response.json();
}
```

Then:

```tsx
const query = useQuery({
  queryKey: ["products"],
  queryFn: fetchProducts,
});
```

can infer:

```text
query.data
→ Product[] | undefined
```

Inference is often preferable.

The type flows from:

```text
queryFn return type
 ↓
useQuery
 ↓
query.data
```

That's exactly what we want.

---

# 52. Query state typing

If:

```tsx
query.data
```

can be unavailable initially, TypeScript may represent:

```text
Product[] | undefined
```

So:

```tsx
if (query.isPending) {
  return <Spinner />;
}

return (
  <ProductList
    products={query.data}
  />
);
```

can often narrow the type appropriately depending on the library's supplied typings and control flow.

The point is that the type system can model the lifecycle:

```text
not available
      OR
data exists
```

rather than pretending:

```text
data always exists
```

---

# 53. TypeScript + `useMemo`

Suppose:

```tsx
const expensiveValue =
  useMemo(() => {
    return calculateSomething(
      products
    );
  }, [products]);
```

TypeScript can infer the return type from:

```text
calculateSomething
```

So you usually don't need:

```tsx
useMemo<Product[]>(...)
```

unless there's a specific reason to constrain/inform inference.

Again:

```text
good inference
→ less syntax
→ easier code
```

---

# 54. TypeScript + callbacks

This is a particularly good use of types.

Suppose:

```tsx
const handleSelect =
  useCallback(
    (id: number) => {
      selectProduct(id);
    },
    []
  );
```

TypeScript knows:

```text
handleSelect
=
(number) => void
```

Now passing it to:

```tsx
<ProductList onSelect={handleSelect} />
```

lets TypeScript verify the callback contract.

If:

```tsx
onSelect: (id: string) => void;
```

then you've got a mismatch.

This is a nice example of types flowing through the component tree.

---

# 55. Types move through React like information does

This is the deeper mental model.

Suppose:

```text
API
 ↓
Product[]
 ↓
query
 ↓
ProductList
 ↓
ProductCard
 ↓
Product
```

The type travels with the data:

```text
Product[]
 ↓
Product[]
 ↓
Product
```

Similarly:

```text
parent callback
 ↓
(id: number) => void
 ↓
child callback
 ↓
event
 ↓
number
```

TypeScript is essentially checking that these contracts remain compatible across the data flow.

That maps beautifully onto the way we've been learning React.

---

# 56. A full typed example

Let's put a lot together.

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};

type ProductCardProps = {
  product: Product;
  onSelect: (id: number) => void;
};

function ProductCard({
  product,
  onSelect,
}: ProductCardProps) {
  return (
    <article>
      <h2>{product.name}</h2>

      <p>
        ${product.price}
      </p>

      <button
        onClick={() =>
          onSelect(product.id)
        }
      >
        Select
      </button>
    </article>
  );
}
```

Parent:

```tsx
type ProductListProps = {
  products: Product[];
  onSelect: (id: number) => void;
};

function ProductList({
  products,
  onSelect,
}: ProductListProps) {
  return (
    <div>
      {products.map(product => (
        <ProductCard
          key={product.id}
          product={product}
          onSelect={onSelect}
        />
      ))}
    </div>
  );
}
```

And:

```tsx
function ProductsPage() {
  const [selectedId, setSelectedId] =
    useState<number | null>(null);

  const productsQuery =
    useQuery({
      queryKey: ["products"],
      queryFn: fetchProducts,
    });

  if (productsQuery.isPending) {
    return <p>Loading...</p>;
  }

  if (productsQuery.isError) {
    return <p>Failed.</p>;
  }

  return (
    <>
      <ProductList
        products={productsQuery.data}
        onSelect={setSelectedId}
      />

      {selectedId !== null && (
        <p>
          Selected: {selectedId}
        </p>
      )}
    </>
  );
}
```

Look at the type flow:

```text
fetchProducts
   ↓
Promise<Product[]>
   ↓
useQuery
   ↓
Product[]
   ↓
ProductList
   ↓
Product
   ↓
ProductCard
```

and:

```text
setSelectedId
   ↓
(number | null)
   ↓
onSelect: (id: number) => void
```

This is what TypeScript is doing at its best.

It's not littering the code with annotations.

It's verifying the **relationships** between parts.

---

# 57. One more important concept: TypeScript doesn't replace testing

Suppose your type says:

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};
```

TypeScript can catch:

```tsx
product.prcie
```

because you've misspelled `price`.

Great.

But it cannot tell you:

```text
the button is in the wrong place
the modal doesn't close
the API returns 500
the user can't keyboard-navigate
the CSS is broken
the wrong product is displayed
```

That's why:

```text
TypeScript
+
tests
+
runtime validation
```

solve different problems.

Think:

```text
TypeScript
→ "Is this code structurally/type-wise plausible?"

Tests
→ "Does the behavior actually work?"

Runtime validation
→ "Is the outside-world data actually what we expected?"
```

All three can be useful.

---

# 58. `any` is not a solution to difficult typing

Suppose TypeScript complains:

```text
Type 'X' is not assignable to type 'Y'
```

The worst response is:

```tsx
const value: any = something;
```

because you've silenced the type system instead of understanding the mismatch.

Sometimes `any` is genuinely necessary at a boundary.

But don't make it:

```text
"the fix for TypeScript"
```

A better workflow is:

```text
TypeScript error
 ↓
What does the value actually represent?
 ↓
Model that reality
 ↓
Narrow/transform it
 ↓
keep the type information
```

This mindset will make you much better at TypeScript.

---

# 59. The important TypeScript skill isn't syntax

Eventually you'll learn:

```text
interface
type
union
intersection
generic
conditional type
mapped type
utility type
infer
```

But don't make those the goal.

The real skill is:

> **Can I model the states and relationships in my application accurately?**

For example:

```text
User | null
```

means:

```text
authentication/user loading state
```

while:

```text
status: "success" + data
```

means:

```text
data is guaranteed in this branch
```

and:

```text
(id: number) => void
```

means:

```text
callback contract
```

and:

```text
T
```

means:

```text
these values are related by the same type
```

That's the deeper understanding.

---

# 60. The mental model for React + TypeScript

Think of React as:

```text
data
 ↓
components
 ↓
props
 ↓
state
 ↓
events
 ↓
server
```

And TypeScript draws a contract around each boundary:

```text
             Product API
                  │
                  ↓
            Product[]
                  │
          ┌───────┴───────┐
          ↓               ↓
      ProductList      filters/etc.
          │
          ↓
       Product
          │
          ↓
     ProductCard
          │
          ↓
onSelect: (id: number) => void
```

So instead of types being decoration around React, think:

> **TypeScript describes the shape of the information moving through your React application.**

That is the useful mental model.

---

# 61. The rules worth remembering

```text
1. TypeScript primarily gives compile-time/static checking.
   It does not validate runtime data automatically.

2. Props are component contracts, so typing props is one of the
   most valuable uses of TypeScript in React.

3. Let TypeScript infer obvious types.

4. Explicitly type state when the initial value is too narrow,
   especially for null/empty initial values.

5. Union types are excellent for UI state.

6. Discriminated unions are excellent for modeling state machines.

7. Extracted event handlers often need explicit React event types.

8. Refs should be typed with the actual DOM element or imperative handle.

9. Callback props should describe their exact argument and return types,
   not just use Function.

10. useReducer and discriminated-union actions work extremely well together.

11. Generic components are useful when several values must share the same type.

12. unknown means "prove what this is"; any largely disables type checking.

13. API response types describe your expectation—they don't validate the server.

14. Use runtime validation at important external-data boundaries when needed.

15. Good types describe real application rules instead of merely satisfying
    the compiler.
```

The deepest idea from today's lesson is this:

```text
React tells you:
"how information flows through the UI."

TypeScript lets you describe:
"what that information is allowed to look like."
```

And once those two ideas meet, something quite powerful happens:

```text
props
+
state
+
events
+
reducers
+
context
+
refs
+
queries
+
mutations
        ↓
typed contracts
        ↓
safer refactoring
        ↓
better editor feedback
        ↓
fewer classes of mistakes
```

Current React and TypeScript support JSX/React typing directly, with `.tsx` for TypeScript files containing JSX and React's type definitions supplying the types for React and DOM patterns. ([TypeScript](https://www.typescriptlang.org/docs/handbook/jsx.html?utm_source=chatgpt.com "TypeScript: Documentation - JSX"))

The next topic I want to take further is **TypeScript's narrowing, generics, utility types, and advanced component typing in real React code**—especially patterns like polymorphic components, discriminated unions, typed reducers, typed contexts, and generic data tables, where TypeScript starts becoming genuinely powerful rather than merely descriptive.