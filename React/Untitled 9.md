
# Lesson 7 — Forms in React, Properly

Forms are where all the things you've learned so far come together:

```text
state
props
events
callbacks
conditional rendering
derived values
```

And they expose a very important question:

> **What is the source of truth for what the user has typed?**

We'll answer that first.

---

# 1. The basic React form model

In plain HTML, you can write:

```html
<input type="text">
```

and the browser internally keeps track of what is inside the input.

In React, a common pattern is:

```jsx
const [name, setName] = useState("");

<input
  value={name}
  onChange={e => setName(e.target.value)}
/>
```

Now the flow is:

```text
                    React state
                        │
                        ↓
                   input value
                        │
                user types something
                        ↓
                     onChange
                        │
                        ↓
                  setName(...)
                        │
                        ↓
                    new state
                        │
                        └────→ render again
```

This is the **controlled input** pattern.

---

# 2. What makes an input "controlled"?

This:

```jsx
<input
  value={name}
  onChange={e => setName(e.target.value)}
/>
```

means React owns the value.

The input is essentially saying:

> "My displayed value is whatever `name` currently says it should be."

And:

```jsx
onChange={e => setName(e.target.value)}
```

means:

> "When the user changes me, tell React about the new value."

So React state becomes the source of truth.

---

# 3. Why not just use the DOM value?

You might ask:

> Why not just let the browser manage the input and read it when I submit?

That's also possible.

For example, React supports **uncontrolled inputs**, usually accessed through a ref or form APIs.

But controlled inputs are very useful when the UI needs to react to what the user is typing.

For example:

```text
password strength
live validation
character count
search results
enable/disable submit
formatting
conditional fields
```

If React already knows:

```text
email = "alex@example.com"
```

then it can immediately use that state elsewhere.

---

# 4. First complete form

Let's build a simple registration form:

```jsx
import { useState } from "react";

function SignupForm() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  function handleSubmit(e) {
    e.preventDefault();

    console.log({
      name,
      email
    });
  }

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="name">
          Name
        </label>

        <input
          id="name"
          value={name}
          onChange={e => setName(e.target.value)}
        />
      </div>

      <div>
        <label htmlFor="email">
          Email
        </label>

        <input
          id="email"
          type="email"
          value={email}
          onChange={e => setEmail(e.target.value)}
        />
      </div>

      <button type="submit">
        Create account
      </button>
    </form>
  );
}
```

Let's understand every part.

---

# 5. `value`

Here:

```jsx
value={name}
```

React takes the current state:

```text
name
```

and uses it as the input's value.

Suppose:

```text
name = "Alex"
```

Then conceptually:

```jsx
<input value="Alex" />
```

If state changes to:

```text
name = "Sarah"
```

the input becomes:

```jsx
<input value="Sarah" />
```

So the rendered UI reflects state.

---

# 6. `onChange`

This:

```jsx
onChange={e => setName(e.target.value)}
```

does two things:

### `e`

The event object.

### `e.target.value`

The current value of the input.

So if the user types:

```text
A
```

then:

```text
e.target.value
```

is:

```text
"A"
```

and:

```jsx
setName("A");
```

is called.

---

# 7. Why does typing trigger a render?

Every time the input changes:

```text
user types
   ↓
onChange
   ↓
setName(...)
   ↓
state changes
   ↓
React renders
```

On the new render:

```jsx
value={name}
```

gets the new value.

So the input and React state remain synchronized.

---

# 8. Multiple fields

Now suppose you have:

```text
Name
[____________]

Email
[____________]

Password
[____________]

Age
[____________]
```

You could simply have:

```jsx
const [name, setName] = useState("");
const [email, setEmail] = useState("");
const [password, setPassword] = useState("");
const [age, setAge] = useState("");
```

For small forms, that's perfectly reasonable.

Don't try to optimize away perfectly readable code.

---

# 9. One object for the entire form

For larger forms, you can group the fields:

```jsx
const [form, setForm] = useState({
  name: "",
  email: "",
  password: "",
  age: ""
});
```

Then:

```jsx
<input
  value={form.name}
  onChange={e =>
    setForm({
      ...form,
      name: e.target.value
    })
  }
/>
```

And:

```jsx
<input
  value={form.email}
  onChange={e =>
    setForm({
      ...form,
      email: e.target.value
    })
  }
/>
```

Remember what happens here:

```jsx
{
  ...form,
  name: e.target.value
}
```

You're creating a new object.

The existing fields are copied, and `name` is replaced.

---

# 10. A cleaner generic handler

You can make this reusable:

```jsx
function handleChange(e) {
  const { name, value } = e.target;

  setForm(form => ({
    ...form,
    [name]: value
  }));
}
```

Then:

```jsx
<input
  name="name"
  value={form.name}
  onChange={handleChange}
/>

<input
  name="email"
  value={form.email}
  onChange={handleChange}
/>
```

Let's unpack:

```js
const { name, value } = e.target;
```

Suppose the input is:

```html
<input name="email" value="alex@example.com">
```

Then:

```text
name  = "email"
value = "alex@example.com"
```

And:

```js
[name]: value
```

means:

```js
email: "alex@example.com"
```

So:

```jsx
setForm(form => ({
  ...form,
  [name]: value
}));
```

becomes conceptually:

```js
setForm(form => ({
  ...form,
  email: "alex@example.com"
}));
```

This is just JavaScript computed property syntax combined with a React state update.

---

# 11. Why use the functional state form here?

We write:

```jsx
setForm(form => ({
  ...form,
  [name]: value
}));
```

instead of:

```jsx
setForm({
  ...form,
  [name]: value
});
```

because we're calculating the new state from the previous state.

The functional form explicitly says:

> Give me the current state, and I'll calculate the next state from it.

This is especially useful when state updates may be queued or when several updates interact.

You should get comfortable with this pattern.

---

# 12. Checkboxes

Checkboxes work slightly differently.

Suppose:

```jsx
const [subscribed, setSubscribed] = useState(false);
```

Then:

```jsx
<input
  type="checkbox"
  checked={subscribed}
  onChange={e => setSubscribed(e.target.checked)}
/>
```

Notice:

```text
text input → value
checkbox   → checked
```

So:

```js
e.target.value
```

isn't what you usually care about for a checkbox.

You care about:

```js
e.target.checked
```

which gives:

```text
true / false
```

---

# 13. Checkbox with a label

A good accessible structure:

```jsx
<label>
  <input
    type="checkbox"
    checked={subscribed}
    onChange={e => setSubscribed(e.target.checked)}
  />

  Subscribe to newsletter
</label>
```

Now clicking the text also interacts with the checkbox.

---

# 14. Radio buttons

Suppose the user chooses a plan:

```text
○ Basic
○ Pro
○ Enterprise
```

State:

```jsx
const [plan, setPlan] = useState("basic");
```

Then:

```jsx
<label>
  <input
    type="radio"
    name="plan"
    value="basic"
    checked={plan === "basic"}
    onChange={e => setPlan(e.target.value)}
  />
  Basic
</label>

<label>
  <input
    type="radio"
    name="plan"
    value="pro"
    checked={plan === "pro"}
    onChange={e => setPlan(e.target.value)}
  />
  Pro
</label>
```

The state stores:

```text
"basic"
```

or:

```text
"pro"
```

The `checked` expression determines which option is selected.

---

# 15. Select

You've already seen this pattern:

```jsx
const [country, setCountry] = useState("");

<select
  value={country}
  onChange={e => setCountry(e.target.value)}
>
  <option value="">
    Choose country
  </option>

  <option value="india">
    India
  </option>

  <option value="japan">
    Japan
  </option>
</select>
```

Again:

```text
state → value
change event → state update
```

Same underlying model.

---

# 16. Textarea

Textarea also works as a controlled input:

```jsx
const [message, setMessage] = useState("");

<textarea
  value={message}
  onChange={e => setMessage(e.target.value)}
/>
```

Don't think of it as some completely different React mechanism.

It's the same general pattern.

---

# 17. Form submission

You should generally handle the form itself:

```jsx
<form onSubmit={handleSubmit}>
```

rather than relying only on:

```jsx
<button onClick={handleSubmit}>
```

Why?

Because the action you're handling is **form submission**, not merely "button was clicked."

For example, a user can submit with Enter in many forms.

So:

```jsx
<form onSubmit={handleSubmit}>
```

is semantically better.

---

# 18. `preventDefault()`

Inside:

```jsx
function handleSubmit(e) {
  e.preventDefault();

  // your React logic
}
```

you're preventing the browser's default form submission behavior.

Then you can:

```text
validate
 ↓
send API request
 ↓
show error
 ↓
show success
```

without the browser navigating away as part of the traditional form submission.

---

# 19. Validation

Now let's introduce validation.

```jsx
const [email, setEmail] = useState("");
const [error, setError] = useState("");
```

Then:

```jsx
function handleSubmit(e) {
  e.preventDefault();

  if (!email.includes("@")) {
    setError("Enter a valid email.");
    return;
  }

  setError("");

  console.log("Valid!");
}
```

UI:

```jsx
<input
  type="email"
  value={email}
  onChange={e => setEmail(e.target.value)}
/>

{error && (
  <p>{error}</p>
)}
```

So:

```text
submit
 ↓
validate
 ↓
error state
 ↓
conditional rendering
```

Again, we're combining concepts you've already learned.

---

# 20. Field-level errors

Suppose your form has multiple fields.

One error string isn't enough.

You could have:

```jsx
const [errors, setErrors] = useState({});
```

Then:

```jsx
setErrors({
  email: "Invalid email",
  password: "Password is too short"
});
```

And:

```jsx
{errors.email && (
  <p>{errors.email}</p>
)}
```

This makes the state shape match the problem:

```text
errors
├── email
├── password
└── name
```

This is an example of designing state around the data you actually need.

---

# 21. Don't validate every conceivable way

Suppose your requirement is:

> Email must not be empty and must contain `@`.

Then:

```jsx
if (!email.trim()) {
  ...
}

if (!email.includes("@")) {
  ...
}
```

may be sufficient for a basic exercise.

Don't immediately build a 500-line validation framework.

Later you'll learn dedicated form libraries when the complexity actually justifies them.

---

# 22. Validating while typing

Because the value is already in state, you can derive validation information during render.

For example:

```jsx
const isEmailValid =
  email.includes("@");
```

Then:

```jsx
<button disabled={!isEmailValid}>
  Submit
</button>
```

Notice:

```text
email → derived isEmailValid
```

You usually don't need:

```jsx
const [isEmailValid, setIsEmailValid] = useState(false);
```

because validity can be calculated from the current email.

This is the **derived data** principle from the previous lesson.

---

# 23. Character count

Another simple example:

```jsx
const [message, setMessage] = useState("");
```

Then:

```jsx
<textarea
  value={message}
  onChange={e => setMessage(e.target.value)}
/>

<p>
  {message.length} / 500
</p>
```

`message.length` is derived.

No extra state required.

---

# 24. Conditional fields

Suppose the user chooses:

```text
Account type:
○ Personal
○ Business
```

and only business users need:

```text
Company name
[____________]
```

State:

```jsx
const [accountType, setAccountType] =
  useState("personal");
```

Then:

```jsx
{accountType === "business" && (
  <input
    name="company"
    value={form.company}
    onChange={handleChange}
  />
)}
```

This is another example of:

```text
state
 ↓
conditional rendering
```

---

# 25. A complete realistic form

Let's put all of this together.

```jsx
import { useState } from "react";

function SignupForm() {
  const [form, setForm] = useState({
    name: "",
    email: "",
    password: "",
    plan: "basic",
    subscribed: false
  });

  const [errors, setErrors] = useState({});

  function handleChange(e) {
    const { name, value, type, checked } = e.target;

    setForm(form => ({
      ...form,
      [name]: type === "checkbox"
        ? checked
        : value
    }));
  }

  function validate() {
    const nextErrors = {};

    if (!form.name.trim()) {
      nextErrors.name = "Name is required.";
    }

    if (!form.email.includes("@")) {
      nextErrors.email = "Enter a valid email.";
    }

    if (form.password.length < 8) {
      nextErrors.password =
        "Password must be at least 8 characters.";
    }

    return nextErrors;
  }

  function handleSubmit(e) {
    e.preventDefault();

    const nextErrors = validate();

    if (Object.keys(nextErrors).length > 0) {
      setErrors(nextErrors);
      return;
    }

    setErrors({});

    console.log(form);
  }

  return (
    <form onSubmit={handleSubmit}>
      <div>
        <label htmlFor="name">
          Name
        </label>

        <input
          id="name"
          name="name"
          value={form.name}
          onChange={handleChange}
        />

        {errors.name && (
          <p>{errors.name}</p>
        )}
      </div>

      <div>
        <label htmlFor="email">
          Email
        </label>

        <input
          id="email"
          name="email"
          type="email"
          value={form.email}
          onChange={handleChange}
        />

        {errors.email && (
          <p>{errors.email}</p>
        )}
      </div>

      <div>
        <label htmlFor="password">
          Password
        </label>

        <input
          id="password"
          name="password"
          type="password"
          value={form.password}
          onChange={handleChange}
        />

        {errors.password && (
          <p>{errors.password}</p>
        )}
      </div>

      <div>
        <label>
          Plan
        </label>

        <select
          name="plan"
          value={form.plan}
          onChange={handleChange}
        >
          <option value="basic">
            Basic
          </option>

          <option value="pro">
            Pro
          </option>

          <option value="enterprise">
            Enterprise
          </option>
        </select>
      </div>

      <label>
        <input
          type="checkbox"
          name="subscribed"
          checked={form.subscribed}
          onChange={handleChange}
        />

        Subscribe to newsletter
      </label>

      <button type="submit">
        Create account
      </button>
    </form>
  );
}
```

Don't memorize this.

Instead, understand the architecture.

---

# 26. Let's inspect the `handleChange` callback

This part may look intimidating:

```jsx
function handleChange(e) {
  const { name, value, type, checked } = e.target;

  setForm(form => ({
    ...form,
    [name]: type === "checkbox"
      ? checked
      : value
  }));
}
```

But it's actually just a sequence.

### Step 1

Get information from the element:

```js
const {
  name,
  value,
  type,
  checked
} = e.target;
```

### Step 2

Start with existing form state:

```js
...form
```

### Step 3

Replace the changed field:

```js
[name]: ...
```

### Step 4

If checkbox:

```js
checked
```

otherwise:

```js
value
```

### Step 5

Give the new object to React:

```js
setForm(...)
```

That's all.

When code looks dense, mentally expand it into steps like this.

---

# 27. Controlled vs uncontrolled

Now that you've seen controlled inputs, let's distinguish them.

## Controlled

```jsx
const [value, setValue] = useState("");

<input
  value={value}
  onChange={e => setValue(e.target.value)}
/>
```

React state owns the value.

## Uncontrolled

You can let the DOM maintain the value and access it when needed, often with a ref or form APIs.

Conceptually:

```text
Controlled:
React state ↔ input

Uncontrolled:
DOM/input owns its value
          ↓
       read when needed
```

For ordinary React forms where you need validation, conditional UI, live feedback, or state-dependent behavior, controlled inputs are often the easier mental model.

But uncontrolled inputs aren't "wrong."

They solve different problems.

---

# 28. One special case: file inputs

File inputs deserve a separate mention.

```jsx
<input type="file" />
```

The browser handles the selected file information specially, and you generally don't control its selected file by trying to drive it with a normal `value` state in the same way as a text input.

Usually you read the selected files from the event or via a ref.

For example:

```jsx
function handleFileChange(e) {
  const file = e.target.files?.[0];

  if (file) {
    console.log(file);
  }
}
```

So don't assume every input behaves identically.

---

# 29. Forms and component boundaries

Once forms get bigger, don't put everything into one massive component.

You might have:

```text
SignupForm
├── PersonalInfo
├── AccountPlan
├── PasswordFields
└── SubmitSection
```

The parent could own the form state:

```text
SignupForm
    │
    ├── PersonalInfo
    ├── AccountPlan
    └── PasswordFields
```

Then pass data and callbacks down.

This is exactly the state-ownership principle we learned earlier.

---

# 30. Here's the important design question again

Suppose:

```text
SignupForm
├── NameInput
├── EmailInput
├── PasswordInput
└── SubmitButton
```

Who should own the form data?

Usually the parent form component, because it needs the complete picture when submitting.

So:

```text
SignupForm
    ↓
form state
    ↓
children receive relevant values/callbacks
```

The child doesn't need to know how the whole form works.

It only needs something like:

```jsx
<EmailInput
  value={email}
  onChange={onEmailChange}
/>
```

---

# 31. A very important principle for forms

Don't think:

> "I need to use `useState` for every visible thing."

Think:

```text
What is the source of truth?
What can be derived?
What actually needs to change independently?
```

For our form:

```text
name        → state
email       → state
password    → state
plan        → state
subscribed  → state

emailIsValid → derived
passwordTooShort → derived
canSubmit → derived
```

That is a cleaner data model.

---

# 32. One more realistic example: checkout

Imagine:

```text
Cart
────────────────────
Laptop       ₹80,000
Mouse         ₹1,500

Coupon:
[ SAVE10       ] [Apply]

Subtotal: ₹81,500
Discount: ₹8,150
Total:    ₹73,350
```

Possible source state:

```text
cart
coupon
```

Derived values:

```text
subtotal
discount
total
```

Notice how this connects the previous lesson to forms:

```text
coupon input
     ↓
coupon state
     ↓
derived discount
     ↓
derived total
     ↓
UI
```

You don't need state for:

```text
subtotal
discount
total
```

if all three can be calculated from the current data.

---

# 33. Your biggest takeaway from forms

The basic controlled-input pattern is:

```jsx
<input
  value={state}
  onChange={e => setState(e.target.value)}
/>
```

The general data-flow pattern is:

```text
                  state
                    ↓
                  input
                    │
                user edits
                    ↓
                 event
                    ↓
               setState()
                    │
                    ↓
              new render
                    │
                    └────→ updated input
```

And for a complete form:

```text
input state
   ↓
validation / derived values
   ↓
submit event
   ↓
validation
   ↓
API call
```

We'll soon get to the API-call portion.

---

# Practice

Build a form with:

```text
Name:      [____________]

Email:     [____________]

Age:       [____________]

Country:   [ India ▼ ]

Newsletter [✓]

Password:  [____________]

[Submit]
```

Requirements:

```text
1. Every field should be controlled.
2. State should be the source of truth.
3. Show an error if name is empty.
4. Show an error if email is invalid.
5. Password must be at least 8 characters.
6. Don't submit if there are errors.
7. On successful submission, print the complete form object.
```

While building it, keep asking yourself:

> **What is state? What is derived? What event changes the state?**

Those three questions are more valuable than memorizing form syntax.

---

# Where we are now

You have covered:

```text
✓ Components
✓ JSX
✓ Props
✓ map()
✓ filter()
✓ find()
✓ reduce()
✓ Callbacks
✓ Closures
✓ State
✓ Events
✓ Conditional rendering
✓ Lists + keys
✓ Derived data
✓ State ownership
✓ Lifting state up
✓ Controlled forms
✓ Form validation
```

The next major concept is **`useEffect`**.

And I want to spend real time on it because it is where many otherwise-good React developers develop bad habits.

We'll answer:

```text
What exactly is an effect?
Why isn't fetching data just "another event"?
When does useEffect run?
What do dependencies actually mean?
Why do effects sometimes run more than expected?
What is cleanup?
Why do infinite effect loops happen?
When should you NOT use useEffect?
```

That lesson is where your understanding moves from basic React syntax toward actually understanding **React's rendering model**.