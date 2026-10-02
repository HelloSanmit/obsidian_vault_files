
# Lesson 32 — Advanced TypeScript for React: Narrowing, Utility Types, Generics, and Flexible Component APIs

Last lesson established the basic connection:

```text
TypeScript
   ↓
React component contracts
   ↓
props
state
events
refs
context
server data
```

Today we're going to make those types **actually useful for designing React code**.

The four ideas we'll build are:

```text
1. narrowing
2. utility types
3. generics
4. advanced component API design
```

The important theme throughout is:

> **A type should describe the relationships and valid states of your program, not merely silence compiler errors.**

TypeScript's current type system uses control-flow analysis for narrowing, and its generic/utility-type features are designed specifically for reusable and transformed types. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing?utm_source=chatgpt.com "TypeScript: Documentation - Narrowing"))

---

# 1. Narrowing is one of the most important TypeScript skills

Suppose:

```tsx
function formatValue(
  value: string | number
) {
  return value.toUpperCase();
}
```

TypeScript complains.

Why?

Because:

```text
value
=
string OR number
```

and numbers don't have:

```js
.toUpperCase()
```

So you need to **narrow** the type.

```tsx
function formatValue(
  value: string | number
) {
  if (typeof value === "string") {
    return value.toUpperCase();
  }

  return value.toFixed(2);
}
```

Now TypeScript understands:

```text
before if:
string | number

inside if:
string

after:
number
```

This is called control-flow narrowing. TypeScript uses checks such as `typeof`, equality checks, `in`, `instanceof`, truthiness, and user-defined type predicates to refine union types. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing?utm_source=chatgpt.com "TypeScript: Documentation - Narrowing"))

---

# 2. React uses narrowing everywhere

Consider:

```tsx
type User = {
  id: number;
  name: string;
};

function Profile({
  user,
}: {
  user: User | null;
}) {
  if (!user) {
    return <p>No user.</p>;
  }

  return <h1>{user.name}</h1>;
}
```

Before the check:

```text
user = User | null
```

After:

```tsx
if (!user) {
  return;
}
```

TypeScript knows:

```text
user = User
```

for the rest of that branch.

This is exactly the lifecycle we already know:

```text
not available
   ↓
available
```

The type reflects the application state.

---

# 3. Truthiness narrowing

You can also write:

```tsx
if (user) {
  return <p>{user.name}</p>;
}
```

TypeScript understands that a truthy `user` isn't `null` or `undefined`.

But be careful when the valid value can itself be falsy.

For example:

```tsx
function Example({
  count,
}: {
  count: number | null;
}) {
  if (count) {
    return <p>{count}</p>;
  }

  return <p>No count</p>;
}
```

This treats:

```text
count = 0
```

as if it were absent.

That's not necessarily what you want.

In that case:

```tsx
if (count !== null) {
  ...
}
```

is more accurate.

This is a good example of an important distinction:

```text
TypeScript narrowing
+
JavaScript truthiness
```

are related, but your condition still has to represent the actual business rule.

---

# 4. Equality narrowing

Suppose:

```tsx
type Status = "idle" | "loading" | "success";
```

Then:

```tsx
function StatusMessage({
  status,
}: {
  status: Status;
}) {
  if (status === "loading") {
    return <Spinner />;
  }

  if (status === "success") {
    return <p>Done.</p>;
  }

  return null;
}
```

TypeScript knows which literal remains in each branch.

This becomes very useful when your state isn't arbitrary text.

Instead of:

```text
status: string
```

you have:

```text
"idle" | "loading" | "success"
```

Now the compiler can reason about every valid case.

TypeScript's control-flow system explicitly uses equality checks such as `===` for narrowing. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing?utm_source=chatgpt.com "TypeScript: Documentation - Narrowing"))

---

# 5. Discriminated unions are the real power move

Suppose we have:

```tsx
type State =
  | {
      status: "loading";
    }
  | {
      status: "success";
      data: Product[];
    }
  | {
      status: "error";
      message: string;
    };
```

Now:

```tsx
function ProductsView({
  state,
}: {
  state: State;
}) {
  switch (state.status) {
    case "loading":
      return <Spinner />;

    case "success":
      return (
        <ProductList
          products={state.data}
        />
      );

    case "error":
      return (
        <ErrorMessage
          message={state.message}
        />
      );
  }
}
```

Why is this so good?

Because:

```text
status = success
```

guarantees:

```text
data exists
```

and:

```text
status = error
```

guarantees:

```text
message exists
```

We have encoded the relationship:

```text
state.status
      ↕
state's available fields
```

This is much stronger than three unrelated variables.

---

# 6. Compare it with a weak state model

Weak:

```tsx
type State = {
  loading: boolean;
  data?: Product[];
  error?: string;
};
```

Now these are all technically possible:

```text
loading = true
data = [...]
error = "Failed"
```

What does that mean?

Your type doesn't know.

The discriminated union:

```tsx
type State =
  | { status: "loading" }
  | { status: "success"; data: Product[] }
  | { status: "error"; message: string };
```

makes invalid combinations much harder to represent.

This is an important principle:

> **Good typing can improve your state architecture, not just describe it.**

---

# 7. Exhaustiveness with `never`

Suppose:

```tsx
type Action =
  | { type: "added"; todo: Todo }
  | { type: "deleted"; id: number }
  | { type: "toggled"; id: number };
```

Your reducer:

```tsx
function reducer(
  state: Todo[],
  action: Action
) {
  switch (action.type) {
    case "added":
      return [...state, action.todo];

    case "deleted":
      return state.filter(
        todo => todo.id !== action.id
      );

    case "toggled":
      return state.map(todo =>
        todo.id === action.id
          ? {
              ...todo,
              completed: !todo.completed,
            }
          : todo
      );
  }
}
```

Now imagine somebody later adds:

```tsx
| { type: "archived"; id: number }
```

but forgets to update the reducer.

You can force the compiler to notice.

```tsx
function assertNever(
  value: never
): never {
  throw new Error(
    "Unexpected value"
  );
}
```

Then:

```tsx
default:
  return assertNever(action);
```

Why does this work?

Once all valid cases are handled, TypeScript knows:

```text
action = impossible
```

and represents that impossible state as:

```text
never
```

If a new action remains unhandled, `action` is no longer `never`, and TypeScript reports the problem. Current TypeScript explicitly documents `never` as the basis of exhaustive checking over discriminated unions. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing?utm_source=chatgpt.com "TypeScript: Documentation - Narrowing"))

---

# 8. This is particularly good with reducers

Our earlier reducer architecture:

```text
dispatch(action)
      ↓
reducer(state, action)
      ↓
next state
```

becomes:

```text
Action union
      ↓
typed dispatch
      ↓
typed reducer
      ↓
typed next state
```

That's a very strong combination.

For example:

```tsx
type Action =
  | {
      type: "added";
      text: string;
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

Now:

```tsx
dispatch({
  type: "toggled",
  id: 42,
});
```

is valid.

But:

```tsx
dispatch({
  type: "toggled",
  text: "hello",
});
```

is not.

TypeScript understands that a `"toggled"` action requires an `id`.

That's much stronger than:

```tsx
dispatch({
  type: "anything",
  whatever: ...
});
```

with `any`.

---

# 9. User-defined type guards

Sometimes TypeScript doesn't know enough from ordinary conditions.

Suppose:

```tsx
type Product = {
  id: number;
  name: string;
};

type User = {
  id: number;
  email: string;
};

type Entity = Product | User;
```

You could write:

```tsx
function isProduct(
  entity: Entity
): entity is Product {
  return "name" in entity;
}
```

Now:

```tsx
function renderEntity(
  entity: Entity
) {
  if (isProduct(entity)) {
    return entity.name;
  }

  return entity.email;
}
```

The special return type:

```text
entity is Product
```

tells TypeScript:

> "If this function returns true, treat `entity` as a Product."

That's a **type predicate**. TypeScript supports user-defined type guards of this form. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing?utm_source=chatgpt.com "TypeScript: Documentation - Narrowing"))

---

# 10. This becomes useful with API data

Suppose some endpoint returns:

```text
Product | ErrorResponse
```

You can create a guard:

```tsx
function isProduct(
  value: unknown
): value is Product {
  if (
    typeof value !== "object" ||
    value === null
  ) {
    return false;
  }

  if (!("id" in value)) {
    return false;
  }

  if (!("name" in value)) {
    return false;
  }

  return (
    typeof value.id === "number" &&
    typeof value.name === "string"
  );
}
```

Then:

```tsx
if (isProduct(data)) {
  // data is Product
}
```

Notice what's happening:

```text
unknown
 ↓
runtime checks
 ↓
validated enough for this code
 ↓
Product
```

This is the bridge between our earlier discussion of:

```text
TypeScript
```

and:

```text
runtime validation
```

---

# 11. A type doesn't validate runtime data

Suppose:

```tsx
const data: Product = await response.json();
```

This does **not** magically validate the JSON.

You've merely asserted:

> "I expect this to be a Product."

The server could still return nonsense.

So for untrusted input:

```text
URL
API
localStorage
user input
third-party library
```

you should think:

```text
unknown
 ↓
check/parse
 ↓
trusted domain value
```

This distinction is one of the most important things to understand about TypeScript.

---

# 12. Now let's move to utility types

TypeScript provides built-in utility types for transforming existing types. Current TypeScript includes utilities such as `Partial`, `Required`, `Readonly`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`, and others. ([TypeScript](https://www.typescriptlang.org/docs/handbook/utility-types?utm_source=chatgpt.com "TypeScript: Documentation - Utility Types"))

These become very useful in React component/API code.

---

# 13. `Partial`

Suppose:

```tsx
type User = {
  id: number;
  name: string;
  email: string;
};
```

`Partial<User>` becomes conceptually:

```tsx
{
  id?: number;
  name?: string;
  email?: string;
}
```

So:

```tsx
type UserPatch = Partial<User>;
```

is useful when representing:

```text
"some subset of User fields"
```

For example:

```tsx
function updateUser(
  id: number,
  changes: Partial<User>
) {
  ...
}
```

Now this is valid:

```tsx
updateUser(42, {
  name: "Alice",
});
```

because you're representing a partial update.

This maps very naturally to the PATCH concept we learned earlier.

---

# 14. But `Partial` can be dangerous if used carelessly

Suppose:

```tsx
type CreateProductInput =
  Partial<Product>;
```

You've now made every field optional.

Maybe that isn't what the API means.

Perhaps creating a product requires:

```text
name
price
category
```

but:

```text
id
```

is generated by the server.

Then blindly using `Partial<Product>` loses important domain rules.

So:

```text
Partial
=
all fields optional
```

not:

```text
"somehow this is the correct create/update type."
```

You still have to model the domain correctly.

---

# 15. `Pick`

Suppose:

```tsx
type User = {
  id: number;
  name: string;
  email: string;
  passwordHash: string;
};
```

A profile card needs only:

```text
id
name
email
```

You can write:

```tsx
type ProfileUser =
  Pick<User, "id" | "name" | "email">;
```

Now:

```text
ProfileUser
=
{
  id: number;
  name: string;
  email: string;
}
```

This can be useful when a component genuinely needs only part of a larger domain object.

---

# 16. `Omit`

The inverse:

```tsx
type PublicUser =
  Omit<User, "passwordHash">;
```

means:

```text
take User
but remove passwordHash
```

Now:

```tsx
type PublicUser = {
  id: number;
  name: string;
  email: string;
};
```

This can be useful when constructing types for different application boundaries.

For example:

```text
database model
      ↓
public API model
```

But again, don't blindly use `Omit` for security.

If the server accidentally sends a sensitive field, TypeScript doesn't remove it from the runtime object.

The utility type changes the type system, not the actual JSON.

---

# 17. `Record`

Suppose:

```tsx
type Status =
  | "todo"
  | "doing"
  | "done";
```

and you want a label for every status.

```tsx
const labels: Record<
  Status,
  string
> = {
  todo: "To do",
  doing: "In progress",
  done: "Completed",
};
```

Now TypeScript requires:

```text
todo
doing
done
```

to all exist.

If you forget one, you'll get a type error.

This is a beautiful combination:

```text
union
+
Record
=
"exactly one value for every known case"
```

---

# 18. `Record` and UI configuration

Suppose:

```tsx
type Variant =
  | "primary"
  | "secondary"
  | "danger";
```

Then:

```tsx
const styles: Record<
  Variant,
  string
> = {
  primary: "btn-primary",
  secondary: "btn-secondary",
  danger: "btn-danger",
};
```

Now every supported variant must have an entry.

Later you add:

```text
"ghost"
```

and TypeScript forces you to deal with it.

That's much better than:

```tsx
const styles = {
  primary: "...",
  secondary: "...",
};
```

and silently discovering later that:

```text
ghost
```

has no style.

---

# 19. `Omit` + props

Suppose you want a custom button:

```tsx
type ButtonProps =
  React.ComponentProps<"button"> & {
    variant:
      | "primary"
      | "danger";
  };
```

Now imagine you want to stop users from supplying:

```text
type
```

because your component always controls the type.

You can write:

```tsx
type ButtonProps =
  Omit<
    React.ComponentProps<"button">,
    "type"
  > & {
    variant:
      | "primary"
      | "danger";
  };
```

Now you're composing:

```text
native button props
   ↓
remove one prop
   ↓
add your own API
```

This pattern is extremely common in component libraries.

---

# 20. Why utility types are powerful in React

Your codebase often has related shapes:

```text
Product
ProductCard props
ProductCreateInput
ProductUpdateInput
ProductPreview
ProductSummary
```

Instead of manually rewriting:

```tsx
{
  id: number;
  name: string;
  price: number;
}
```

everywhere, you can derive related types when the relationships are genuinely meaningful.

This reduces duplication.

But don't turn every type into:

```text
Pick<Omit<Partial<...>>>
```

until nobody understands it.

Readability still matters.

---

# 21. Now generics

We've touched generics before.

Let's really understand them.

Suppose:

```tsx
function identity(
  value: unknown
) {
  return value;
}
```

You lose the relationship between:

```text
input
```

and:

```text
output
```

because both are just `unknown`.

Now:

```tsx
function identity<T>(
  value: T
): T {
  return value;
}
```

means:

> Whatever type comes in as `T`, that same type comes back out.

So:

```tsx
const user =
  identity({
    id: 42,
    name: "Alice",
  });
```

`T` becomes:

```text
{
  id: number;
  name: string;
}
```

and the result preserves that type.

Generics are fundamentally about **preserving relationships between types**. TypeScript's current generics model is built around this idea of reusable components that retain type information across inputs and outputs. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/generics?utm_source=chatgpt.com "TypeScript: Documentation - Generics"))

---

# 22. Generic React component

Suppose:

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

Use it with products:

```tsx
<List
  items={products}
  renderItem={product => (
    <ProductCard
      product={product}
    />
  )}
/>
```

TypeScript figures out:

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

Same component:

```tsx
<List
  items={users}
  renderItem={user => (
    <Profile
      user={user}
    />
  )}
/>
```

Now:

```text
T = User
```

The component is reusable without throwing away type information.

---

# 23. The relationship is the important part

Suppose:

```tsx
type SelectProps<T> = {
  options: T[];
  value: T;
  onChange: (value: T) => void;
};
```

Now:

```tsx
function Select<T>({
  options,
  value,
  onChange,
}: SelectProps<T>) {
  ...
}
```

This says:

```text
options contain T
value is T
onChange receives T
```

All three must agree.

So if:

```text
options = User[]
```

then:

```text
value = User
onChange = (user: User) => ...
```

This is much more powerful than:

```text
options: any[]
value: any
onChange: (x: any) => void
```

because the latter loses the relationship.

---

# 24. Generics are especially useful for tables

Imagine a generic table:

```tsx
type Column<T> = {
  header: string;
  render: (item: T) => React.ReactNode;
};

type TableProps<T> = {
  rows: T[];
  columns: Column<T>[];
};

function Table<T>({
  rows,
  columns,
}: TableProps<T>) {
  ...
}
```

Product table:

```tsx
const columns: Column<Product>[] = [
  {
    header: "Name",
    render: product =>
      product.name,
  },
  {
    header: "Price",
    render: product =>
      `$${product.price}`,
  },
];
```

Now a typo:

```tsx
render: product =>
  product.prcie
```

is caught.

The generic is carrying:

```text
Table
  ↓
Product
  ↓
every column renderer
```

That's exactly what generics are good at.

---

# 25. Constraining generics

Sometimes you don't want **any** `T`.

Suppose a table needs an ID.

You can write:

```tsx
type WithId = {
  id: string | number;
};

function Table<T extends WithId>({
  rows,
}: {
  rows: T[];
}) {
  return rows.map(row => (
    <div key={row.id}>
      ...
    </div>
  ));
}
```

Now TypeScript requires:

```text
T
must contain
id
```

So:

```tsx
Table<Product>
```

works if Product has an `id`.

But:

```tsx
Table<{
  name: string;
}>
```

doesn't.

This is called a **generic constraint**.

Think:

```text
T
=
some type

T extends WithId
=
some type that has at least the WithId shape
```

---

# 26. This is much better than `any`

Without the constraint:

```tsx
function Table<T>({
  rows,
}: {
  rows: T[];
}) {
  return rows.map(row => (
    <div>{row.id}</div>
  ));
}
```

TypeScript doesn't know that `T` has `id`.

With:

```tsx
T extends { id: string | number }
```

you've given the generic a guaranteed minimum structure.

This is a very common generic pattern.

---

# 27. Generic hooks

Generics are not just for components.

Suppose:

```tsx
function useSelection<T>(
  items: T[]
) {
  const [selected, setSelected] =
    useState<T | null>(null);

  return {
    selected,
    select: setSelected,
  };
}
```

If you call:

```tsx
const {
  selected,
  select,
} = useSelection(products);
```

TypeScript can infer:

```text
T = Product
```

so:

```text
selected
→ Product | null

select
→ (product: Product) => void
```

Now the hook preserves the type of the data passed into it.

This is an extremely useful pattern for reusable custom hooks.

---

# 28. Generic API helpers

Imagine:

```tsx
async function fetchJson<T>(
  url: string
): Promise<T> {
  const response =
    await fetch(url);

  if (!response.ok) {
    throw new Error("Request failed");
  }

  return response.json();
}
```

Then:

```tsx
const products =
  await fetchJson<Product[]>(
    "/api/products"
  );
```

and:

```tsx
const user =
  await fetchJson<User>(
    "/api/users/42"
  );
```

This looks elegant.

But remember our earlier warning:

```text
<T>
```

doesn't validate the response.

You've simply told TypeScript:

```text
"Treat the result as T."
```

So a generic API helper is useful for static typing, but it doesn't replace runtime validation.

---

# 29. Generic component + callback + narrowing

Now combine everything.

```tsx
type SelectProps<T> = {
  items: T[];
  getLabel: (item: T) => string;
  onSelect: (item: T) => void;
};
```

Component:

```tsx
function Select<T>({
  items,
  getLabel,
  onSelect,
}: SelectProps<T>) {
  return (
    <div>
      {items.map(item => (
        <button
          key={getLabel(item)}
          onClick={() =>
            onSelect(item)
          }
        >
          {getLabel(item)}
        </button>
      ))}
    </div>
  );
}
```

Usage:

```tsx
<Select
  items={users}
  getLabel={user => user.name}
  onSelect={user => {
    console.log(user.id);
  }}
/>
```

Everything stays connected:

```text
users
 ↓
T = User
 ↓
getLabel(User)
 ↓
onSelect(User)
```

That is the real power of generic component APIs.

---

# 30. But generic components can become over-engineered

You might eventually encounter something like:

```tsx
<T,
  TValue extends keyof T,
  TFilter extends ...
>
```

with a 30-line prop type.

That's not automatically better.

Remember:

```text
generic power
+
complexity cost
```

A generic abstraction is valuable when it genuinely preserves a useful relationship across many usages.

If the component is used once:

```text
generic abstraction
```

may be unnecessary.

If it's a reusable table/form/select library:

```text
generic abstraction
```

can be extremely valuable.

---

# 31. A particularly useful React pattern: `ComponentProps`

Suppose you want your custom button to accept everything a normal HTML button accepts.

```tsx
type NativeButtonProps =
  React.ComponentProps<"button">;
```

Now:

```tsx
type ButtonProps =
  NativeButtonProps & {
    variant:
      | "primary"
      | "danger";
  };
```

Your component can support:

```tsx
<Button
  disabled={loading}
  type="submit"
  onClick={handleClick}
  aria-label="Save"
  variant="primary"
/>
```

This is a great pattern for design systems because the custom component builds on the platform's existing API rather than manually reproducing it.

---

# 32. A custom Link component

Suppose:

```tsx
type LinkProps =
  React.ComponentProps<"a"> & {
    external?: boolean;
  };
```

Then:

```tsx
function Link({
  external,
  ...props
}: LinkProps) {
  return (
    <a
      {...props}
      target={
        external
          ? "_blank"
          : props.target
      }
    />
  );
}
```

Now the component inherits:

```text
href
target
rel
aria-*
className
onClick
...
```

and adds:

```text
external
```

This is useful component API design.

---

# 33. Polymorphic components

Now we're approaching an advanced pattern.

Imagine:

```tsx
<Text as="p">
  Hello
</Text>
```

or:

```tsx
<Text as="h1">
  Hello
</Text>
```

or:

```tsx
<Text as="span">
  Hello
</Text>
```

The component changes the underlying DOM element.

A fully typed polymorphic component might conceptually look like:

```tsx
type TextProps<
  T extends React.ElementType
> = {
  as?: T;
  children: React.ReactNode;
} & React.ComponentPropsWithoutRef<T>;
```

Then:

```tsx
function Text<
  T extends React.ElementType = "span"
>({
  as,
  children,
  ...props
}: TextProps<T>) {
  const Component =
    as ?? "span";

  return (
    <Component {...props}>
      {children}
    </Component>
  );
}
```

This is powerful because:

```tsx
<Text as="button" type="button">
  Save
</Text>
```

can receive button-specific props.

And:

```tsx
<Text as="a" href="/products">
  Products
</Text>
```

can receive anchor-specific props.

This pattern is common in sophisticated component libraries.

But it's also significantly more complex.

---

# 34. Why polymorphic typing is hard

Because now the type relationship is:

```text
"what element did you choose?"
        ↓
which props are legal?
        ↓
which ref type is valid?
```

For:

```text
as="button"
```

you want:

```text
HTMLButtonElement
```

For:

```text
as="a"
```

you want:

```text
HTMLAnchorElement
```

Now we're combining:

```text
generics
+
union/conditional behavior
+
HTML prop types
+
refs
```

This is exactly the kind of thing TypeScript can model—but it isn't something I'd recommend writing from scratch unless your design system genuinely needs polymorphism.

---

# 35. Another important type: `keyof`

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
type ProductKey = keyof Product;
```

becomes conceptually:

```text
"id" | "name" | "price"
```

Now:

```tsx
function getValue(
  product: Product,
  key: keyof Product
) {
  return product[key];
}
```

This prevents:

```tsx
getValue(product, "banana");
```

because `"banana"` isn't a key of Product.

This becomes useful for generic tables, sortable columns, filters, forms, and configuration APIs.

---

# 36. Generic key access

We can make this relationship even stronger:

```tsx
function getValue<
  T,
  K extends keyof T
>(
  object: T,
  key: K
): T[K] {
  return object[key];
}
```

Now:

```tsx
const name =
  getValue(product, "name");
```

has type:

```text
string
```

while:

```tsx
const price =
  getValue(product, "price");
```

has type:

```text
number
```

The generic relationship is:

```text
T
 ↓
object type

K extends keyof T
 ↓
valid key of that object

T[K]
 ↓
type of the selected property
```

This is advanced TypeScript, but it demonstrates exactly why the language has generics.

It's encoding relationships that ordinary `string`/`any` types cannot express.

---

# 37. This can power a typed sorting function

For example:

```tsx
function sortBy<
  T,
  K extends keyof T
>(
  items: T[],
  key: K
) {
  return [...items].sort(
    (a, b) => {
      const first = a[key];
      const second = b[key];

      // domain-specific comparison
      ...
    }
  );
}
```

Then:

```tsx
sortBy(products, "price");
```

is valid.

But:

```tsx
sortBy(products, "banana");
```

isn't.

The compiler knows the second argument has to be a real key of Product.

This kind of relationship becomes extremely useful in reusable libraries.

---

# 38. Utility type: `ReturnType`

Suppose:

```tsx
function createUser() {
  return {
    id: 42,
    name: "Alice",
  };
}
```

You can write:

```tsx
type User =
  ReturnType<typeof createUser>;
```

Now TypeScript derives:

```text
{
  id: number;
  name: string;
}
```

from the function itself.

This can be useful when the function is the canonical source of a complex returned type.

But again, use it thoughtfully.

Sometimes a named domain type:

```tsx
type User = ...
```

is clearer.

Utility types should improve maintainability, not obscure the source of truth.

---

# 39. `Awaited`

We've learned:

```tsx
async function fetchProducts():
  Promise<Product[]>
```

`Awaited<ProductPromise>` conceptually gives you:

```text
Product[]
```

because it models what happens when a Promise is awaited.

Current TypeScript includes `Awaited<T>` for recursively modeling Promise unwrapping. ([TypeScript](https://www.typescriptlang.org/docs/handbook/utility-types?utm_source=chatgpt.com "TypeScript: Documentation - Utility Types"))

You may occasionally see:

```tsx
type Product =
  Awaited<
    ReturnType<typeof fetchProduct>
  >;
```

which effectively means:

```text
"What is the actual resolved value of fetchProduct?"
```

That's useful in infrastructure code, although I wouldn't use it automatically in ordinary components.

---

# 40. `Readonly`

Suppose:

```tsx
type Product = {
  id: number;
  name: string;
};
```

Then:

```tsx
type ReadonlyProduct =
  Readonly<Product>;
```

conceptually means:

```tsx
{
  readonly id: number;
  readonly name: string;
}
```

This can communicate:

> "This object shouldn't be mutated through this reference."

That aligns nicely with the immutable update patterns we've used throughout React.

But `Readonly` is a type-level restriction.

It doesn't physically freeze the JavaScript object at runtime.

Again:

```text
type-level
≠
runtime behavior
```

---

# 41. React props are often naturally readonly

You may eventually see:

```tsx
type Props = {
  readonly user: User;
};
```

or:

```tsx
type Props = Readonly<{
  user: User;
}>;
```

This can communicate that the component shouldn't mutate its props.

And conceptually that fits React very well:

```text
props
→ inputs

not:
props
→ mutable internal storage
```

You don't strictly need to use `readonly` everywhere, but understanding the idea is useful.

---

# 42. TypeScript + immutability

Consider:

```tsx
type User = {
  name: string;
};
```

and:

```tsx
function updateUser(
  user: User
) {
  user.name = "Bob";
}
```

TypeScript allows it.

Now:

```tsx
type User = Readonly<{
  name: string;
}>;
```

and:

```tsx
user.name = "Bob";
```

TypeScript complains.

You could then create a new object:

```tsx
const updatedUser = {
  ...user,
  name: "Bob",
};
```

This fits our React mental model:

```text
old value
 ↓
copy
 ↓
change
 ↓
new value
```

Again, types can help reinforce an architectural principle.

---

# 43. Don't turn every object into `Readonly`

There's a cost to overdoing this.

Some code legitimately mutates local temporary objects before returning them.

TypeScript is a tool, not a moral system.

The question is:

> "Would this restriction communicate and protect something valuable?"

If yes:

```text
Readonly
```

can help.

If not:

```text
don't add ceremony
```

---

# 44. A practical generic form component

Let's make something more realistic.

Suppose we want:

```tsx
<FormField
  label="Email"
  value={email}
  onChange={setEmail}
/>
```

We could define:

```tsx
type FormFieldProps<T> = {
  label: string;
  value: T;
  onChange: (value: T) => void;
  format?: (value: T) => string;
};
```

Then:

```tsx
function FormField<T>({
  label,
  value,
  onChange,
  format,
}: FormFieldProps<T>) {
  ...
}
```

Now:

```text
T
```

could be:

```text
string
number
Date
custom domain type
```

But here's an important architectural question:

Can your component actually edit arbitrary `T` values?

If the component is an HTML text input, the browser gives you a string.

So:

```text
generic API
```

doesn't automatically mean:

```text
generic implementation
```

You still need a sensible conversion boundary.

This is a key TypeScript lesson:

> **A type can express a relationship, but it cannot remove the real constraints of the underlying UI.**

---

# 45. A better generic select

A select component can be designed around values:

```tsx
type SelectProps<T> = {
  value: T | null;
  options: T[];
  getLabel: (item: T) => string;
  getValue: (item: T) => string;
  onChange: (item: T) => void;
};
```

Now:

```text
T = User
```

and:

```text
options
→ User[]

getLabel
→ User → string

getValue
→ User → string

onChange
→ User → void
```

The component can translate:

```text
DOM string value
```

back into:

```text
User
```

using its typed configuration.

That's a much more realistic generic UI API.

---

# 46. Why generic APIs should preserve meaning

A terrible generic component:

```tsx
function Component<T, U, V, W>(...)
```

doesn't automatically impress anyone.

A good generic component answers:

```text
"What relationship am I preserving?"
```

For our Select:

```text
T
=
the type of the selected domain object
```

For our Table:

```text
T
=
the row type shared by rows and column renderers
```

For our List:

```text
T
=
the item type shared by items and renderItem
```

That's why those generics make sense.

---

# 47. A useful advanced pattern: `satisfies`

TypeScript also has the `satisfies` operator, which lets you validate that a value conforms to a type while preserving more specific inferred information.

For example:

```tsx
type Status =
  | "todo"
  | "done";

const labels = {
  todo: "To do",
  done: "Completed",
} satisfies Record<
  Status,
  string
>;
```

Now TypeScript checks:

```text
all Status keys exist
values are strings
```

while still retaining the concrete object's inferred properties.

This can be especially useful for configuration objects.

It's often preferable to certain broad annotations when you want validation **without unnecessarily widening the value's type**.

---

# 48. Why `satisfies` is useful for React configuration

Suppose:

```tsx
type Variant =
  | "primary"
  | "danger";

const buttonConfig = {
  primary: {
    className: "btn-primary",
  },
  danger: {
    className: "btn-danger",
  },
} satisfies Record<
  Variant,
  {
    className: string;
  }
>;
```

Now the object is checked against the expected shape.

If you add:

```text
secondary
```

to the union, TypeScript can tell you the configuration is incomplete.

But the values still retain their specific inferred structure.

This is a nice tool when building:

```text
component variants
route configuration
form schemas
status maps
feature configuration
```

---

# 49. Let's combine everything into a typed data table

```tsx
type Product = {
  id: number;
  name: string;
  price: number;
};

type Column<T> = {
  key: keyof T;
  header: string;
  render?: (
    item: T
  ) => React.ReactNode;
};

type TableProps<T> = {
  rows: T[];
  columns: Column<T>[];
};

function Table<T>({
  rows,
  columns,
}: TableProps<T>) {
  return (
    <table>
      <thead>
        <tr>
          {columns.map(column => (
            <th key={String(column.key)}>
              {column.header}
            </th>
          ))}
        </tr>
      </thead>

      <tbody>
        {rows.map(row => (
          <tr key={String(row.id)}>
            {columns.map(column => (
              <td key={String(column.key)}>
                {column.render
                  ? column.render(row)
                  : String(
                      row[column.key]
                    )}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

Now:

```tsx
const columns: Column<Product>[] = [
  {
    key: "name",
    header: "Name",
  },
  {
    key: "price",
    header: "Price",
    render: product =>
      `$${product.price}`,
  },
];
```

We've combined:

```text
generic
+
keyof
+
indexed access
+
React.ReactNode
+
component props
```

The result isn't just "typed."

It expresses real relationships:

```text
columns belong to Product
keys must be Product keys
render receives Product
rows are Product[]
```

That's advanced TypeScript doing useful work.

---

# 50. But there's a subtle issue in our table

We used:

```tsx
key={String(row.id)}
```

because we assumed `T` has an `id`.

But our generic says:

```text
T = any object
```

So TypeScript shouldn't allow us to assume `id`.

We can fix that:

```tsx
type HasId = {
  id: string | number;
};

type TableProps<T extends HasId> = {
  rows: T[];
  columns: Column<T>[];
};

function Table<T extends HasId>({
  rows,
  columns,
}: TableProps<T>) {
  ...
}
```

Now our implementation's assumption is encoded in the type.

This is exactly what generic constraints are for.

---

# 51. This lesson has a recurring pattern

Notice how often the solution was:

```text
Don't say:
"this is any."

Say:
"what relationship actually exists?"
```

Examples:

```text
User | null
```

instead of:

```text
User
```

or:

```text
status: "loading" | "success" | "error"
```

instead of:

```text
status: string
```

or:

```text
(id: number) => void
```

instead of:

```text
Function
```

or:

```text
T
```

to connect:

```text
items
renderItem
```

or:

```text
K extends keyof T
```

to connect:

```text
object
key
selected value
```

That's the heart of strong TypeScript.

---

# 52. TypeScript should make illegal states difficult

This is probably the biggest lesson of all.

Suppose your application has:

```text
payment status
```

Possible values:

```text
pending
paid
failed
refunded
```

Instead of:

```tsx
status: string
```

use:

```tsx
type PaymentStatus =
  | "pending"
  | "paid"
  | "failed"
  | "refunded";
```

Now:

```text
UI branches
API handling
buttons
labels
icons
permissions
```

can all share the same finite vocabulary.

And with:

```tsx
Record<PaymentStatus, ...>
```

you can make configuration exhaustive.

TypeScript's narrowing and exhaustive checking are specifically designed to make this kind of finite-state modeling practical. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/narrowing?utm_source=chatgpt.com "TypeScript: Documentation - Narrowing"))

---

# 53. React + TypeScript is strongest when types follow domain concepts

For example:

```tsx
type OrderStatus =
  | "pending"
  | "processing"
  | "shipped"
  | "delivered"
  | "cancelled";
```

Then:

```tsx
type Order = {
  id: number;
  status: OrderStatus;
};
```

Then:

```tsx
function OrderBadge({
  status,
}: {
  status: OrderStatus;
}) {
  ...
}
```

Then:

```tsx
const statusLabels:
  Record<OrderStatus, string> = {
    pending: "Pending",
    processing: "Processing",
    shipped: "Shipped",
    delivered: "Delivered",
    cancelled: "Cancelled",
  };
```

Now one domain concept flows through:

```text
API
 ↓
Order
 ↓
OrderStatus
 ↓
OrderBadge
 ↓
statusLabels
```

Everything agrees.

That's exactly what we want.

---

# 54. The mental model for advanced TypeScript

Think of the type system as describing **relationships**.

### Narrowing

```text
Union
 ↓
condition
 ↓
specific member
```

### Discriminated union

```text
status
 ↓
determines available fields
```

### Generic

```text
T
 ↓
preserves relationship between inputs/outputs
```

### Constraint

```text
T extends X
 ↓
T can vary,
but must contain X
```

### `keyof`

```text
type
 ↓
set of valid keys
```

### `T[K]`

```text
object type + key type
 ↓
property value type
```

### Utility types

```text
existing type
 ↓
derived type
```

That is much more useful than memorizing utility-type names one by one.

---

# 55. The final architecture

By now, our React application can look like:

```text
                     EXTERNAL WORLD
                           │
                ┌──────────┴──────────┐
                ↓                     ↓
               API                  URL
                │                     │
                ↓                     ↓
             unknown               string
                │                     │
          validate/parse          parse
                │                     │
                └──────────┬──────────┘
                           ↓
                     DOMAIN TYPES
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
           state          props        server data
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                    React components
                           │
               ┌───────────┼───────────┐
               ↓           ↓           ↓
            narrowing    generics    utility types
               │           │           │
               └───────────┼───────────┘
                           ↓
                    safer contracts
```

This is where TypeScript becomes genuinely powerful.

---

# 56. Rules I want you to carry forward

```text
1. Narrow instead of asserting whenever possible.

2. Use discriminated unions when states have different valid fields.

3. Use never/exhaustiveness checks when every case must be handled.

4. Use user-defined type guards when runtime checks need to communicate
   more precise information to TypeScript.

5. Use Partial/Pick/Omit/Record to derive types when those relationships
   are actually meaningful.

6. Don't use Partial just because it is convenient.

7. Use generics when multiple values need to remain related by the same type.

8. Use constraints when generic code requires a minimum structure.

9. Use keyof when an API accepts only keys of a particular object type.

10. Use T[K] when the output type should follow the chosen key.

11. Don't use any merely to silence a difficult error.

12. Remember that TypeScript types do not validate runtime API data.

13. Don't build complicated generic abstractions unless they solve a
    real reuse/relationship problem.

14. Types should model domain rules, not merely satisfy the compiler.
```

The deepest idea is:

> **Strong TypeScript doesn't mean writing more types. It means expressing more of the truth of your application in the type system.**

So instead of:

```text
"something"
```

you want:

```text
Product
User | null
PaymentStatus
(id: number) => void
Product[]
Record<Status, Label>
T extends HasId
K extends keyof T
```

Each one tells the rest of the code something meaningful.

And when those types flow through React:

```text
API → query → state → props → component → event → mutation
```

the compiler can continuously check that the information moving through your application still satisfies the contracts you've designed. TypeScript's current generics, narrowing, and utility-type features are particularly well suited to this kind of reusable, relationship-preserving code. ([TypeScript](https://www.typescriptlang.org/docs/handbook/2/generics?utm_source=chatgpt.com "TypeScript: Documentation - Generics"))