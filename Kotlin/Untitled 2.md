

# Lesson 21 — Deferred Initialization: `lateinit` and `lazy`

Now that you understand ordinary properties, getters, setters, backing fields, and `init`, we can look at a very common problem:

> **What if a property cannot be initialized yet, but you know it will have a value later?**

Kotlin gives us two very different tools for this:

```kotlin
lateinit
```

and

```kotlin
by lazy
```

They solve related problems, but they do **not** mean the same thing.

---

# 1. The problem `lateinit` solves

Normally, Kotlin wants non-null properties to be initialized.

This is invalid:

```kotlin
class User {
    var name: String
}
```

Kotlin asks:

> What is `name` when a `User` object is created?

You haven't given it an answer.

You could technically do:

```kotlin
class User {
    var name: String = ""
}
```

But now you've invented a fake value.

An empty string may mean:

> "The user has no name."

But perhaps the real meaning is:

> "We haven't assigned the name yet."

Those are different states.

---

# 2. Sometimes initialization genuinely happens later

Imagine:

```text
create object
    ↓
perform some setup
    ↓
provide dependency
    ↓
use object
```

The object exists before one of its dependencies is available.

For that situation, Kotlin allows:

```kotlin
class Service {
    lateinit var repository: Repository
}
```

`lateinit` literally tells Kotlin:

> This property won't be initialized during normal construction, but I promise to initialize it before anyone tries to use it.

---

# 3. A basic example

```kotlin
class User {
    lateinit var name: String
}
```

Then:

```kotlin
val user = User()

user.name = "Alex"

println(user.name)
```

This is fine.

The sequence is:

```text
User created
    ↓
name not initialized yet
    ↓
name = "Alex"
    ↓
name can now be read
```

---

# 4. What makes `lateinit` special?

Compare these two:

```kotlin
var name: String? = null
```

and:

```kotlin
lateinit var name: String
```

They communicate **different meanings**.

### Nullable property

```kotlin
String?
```

means:

> `null` is a legitimate state.

For example:

```text
A user may not have a middle name.
```

That's naturally nullable.

---

### `lateinit`

```kotlin
lateinit var name: String
```

means:

> This value is required, but it will be supplied later.

So:

```text
nullable
→ absence is allowed

lateinit
→ absence is temporary and should eventually be fixed
```

That distinction is much more important than the syntax.

---

# 5. `lateinit` is a promise

This is the mental model I want you to keep:

```text
lateinit
```

is essentially:

> **"Trust me; I'll initialize this later."**

That makes it convenient, but it also moves some safety responsibility to you.

Normally Kotlin protects you during compilation.

With `lateinit`, you're deliberately saying:

> "I know better in this particular situation."

So don't use it just because initialization feels inconvenient.

---

# 6. What if you access it too early?

Consider:

```kotlin
class User {
    lateinit var name: String
}
```

Now:

```kotlin
val user = User()

println(user.name)
```

You haven't initialized `name`.

So the program fails at runtime.

This is fundamentally different from an ordinary uninitialized non-null property, which Kotlin rejects earlier.

Think about the tradeoff:

```text
normal property
    ↓
compiler protects you

lateinit
    ↓
you promise to initialize it
    ↓
runtime can fail if you break that promise
```

This is why `lateinit` should be used intentionally.

---

# 7. `lateinit` is mutable

A `lateinit` property must be a `var`:

```kotlin
lateinit var name: String
```

not:

```kotlin
lateinit val name: String
```

Why?

Because the entire point is that you're going to assign it later.

For example:

```kotlin
user.name = "Alex"
```

and potentially later:

```kotlin
user.name = "Sam"
```

The property is mutable.

---

# 8. It's for non-null values

You don't normally write:

```kotlin
lateinit var name: String?
```

because that defeats the purpose.

A nullable property already has a representation for "not available":

```kotlin
null
```

`lateinit` is for the idea:

> "This should be a real non-null value; it just isn't initialized yet."

So these concepts should stay separate in your head:

```text
String?
→ may legitimately be null

lateinit String
→ should eventually contain a real String
```

---

# 9. A realistic example

Imagine a class that gets configured after construction:

```kotlin
class Application {
    lateinit var logger: Logger

    fun start() {
        logger.log("Application started")
    }
}
```

Some setup code might do:

```kotlin
val app = Application()

app.logger = Logger()

app.start()
```

That can make sense if the lifecycle guarantees that `logger` will be supplied before `start()`.

The danger is:

```kotlin
val app = Application()

app.start()
```

Now `start()` depends on a property that hasn't been initialized.

This suggests an important design question:

> **Can I structure the class so that the compiler guarantees this dependency exists?**

Often, the answer is yes.

For example, constructor injection:

```kotlin
class Application(
    val logger: Logger
)
```

Then:

```kotlin
val app = Application(Logger())
```

Now the object cannot exist without a logger.

That is generally safer when the dependency is truly required from the beginning.

So `lateinit` isn't automatically "better"; it exists for cases where initialization genuinely must be delayed.

---

# 10. `lateinit` versus constructor initialization

Compare:

### Constructor

```kotlin
class User(
    val name: String
)
```

Meaning:

> A `User` cannot exist without a name.

### `lateinit`

```kotlin
class User {
    lateinit var name: String
}
```

Meaning:

> A `User` object can exist temporarily without a name, but the name must be assigned before it is used.

That's a **different object lifecycle**.

This is the deeper reason behind `lateinit`.

---

# 11. Checking whether a `lateinit` property has been initialized

Kotlin provides a way to check certain `lateinit` properties before using them.

For example:

```kotlin
class User {
    lateinit var name: String

    fun printName() {
        if (::name.isInitialized) {
            println(name)
        } else {
            println("Name has not been initialized")
        }
    }
}
```

The syntax:

```kotlin
::name
```

is a property reference.

And:

```kotlin
::name.isInitialized
```

checks whether the `lateinit` property has received a value.

You don't need to memorize the mechanics yet.

What's important is that Kotlin provides a way to make the state explicit when that is useful.

---

# 12. Now let's meet `lazy`

`lateinit` and `lazy` are related enough that beginners often mix them up.

Here's the simplest distinction:

```text
lateinit
→ somebody assigns the value later

lazy
→ Kotlin creates the value automatically when it is first needed
```

Example:

```kotlin
val config by lazy {
    loadConfiguration()
}
```

You don't later write:

```kotlin
config = ...
```

The `lazy` mechanism handles the initialization.

---

# 13. Why would we want `lazy`?

Imagine:

```kotlin
fun loadConfiguration(): Configuration {
    println("Loading configuration...")
    return Configuration()
}
```

Suppose we do:

```kotlin
class App {
    val config = loadConfiguration()
}
```

The configuration is created immediately when the object is initialized.

But perhaps we don't actually need it.

Maybe an application can be created and sit around without ever using its configuration.

Then immediate initialization does unnecessary work.

Instead:

```kotlin
class App {
    val config by lazy {
        loadConfiguration()
    }
}
```

Now the idea becomes:

> Don't calculate this until someone actually asks for it.

---

# 14. Watch the timing

Consider:

```kotlin
class App {
    val config by lazy {
        println("Loading...")
        Configuration()
    }
}
```

Then:

```kotlin
val app = App()

println("App created")
```

The important thing is that merely creating `App` does not trigger the lazy block.

Later:

```kotlin
println(app.config)
```

causes the block to execute.

Conceptually:

```text
App()
 ↓
config not created yet

app.config
 ↓
first access
 ↓
run lazy block
 ↓
store result
 ↓
return result
```

---

# 15. It happens only once

Suppose:

```kotlin
class App {
    val config by lazy {
        println("Loading...")
        Configuration()
    }
}
```

and:

```kotlin
val app = App()

println(app.config)
println(app.config)
println(app.config)
```

The initialization block is evaluated on the first access, and later accesses reuse the initialized value.

So conceptually:

```text
first access
    ↓
calculate
    ↓
remember result

later access
    ↓
return remembered result
```

That's why the name `lazy` makes sense.

---

# 16. `lateinit` and `lazy` side by side

This comparison is important enough to remember.

||`lateinit`|`lazy`|
|---|---|---|
|Property kind|`var`|`val`|
|Who supplies value?|You|Lazy mechanism|
|Initialization|Explicit assignment|First access|
|Can be reassigned?|Yes|Not through the `val`|
|Main idea|Delayed assignment|Deferred computation|

For example:

```kotlin
lateinit var logger: Logger
```

means:

> I'll give you a logger later.

Whereas:

```kotlin
val logger by lazy {
    Logger()
}
```

means:

> Create the logger only when it's first needed.

---

# 17. Here's a subtle difference

Consider:

```kotlin
lateinit var service: Service
```

There is no automatic creation.

Something must explicitly do:

```kotlin
service = Service()
```

With:

```kotlin
val service by lazy {
    Service()
}
```

there's nothing else to assign.

Accessing:

```kotlin
service
```

causes initialization.

So:

```text
lateinit = delayed responsibility

lazy = delayed execution
```

That is the best mental shortcut.

---

# 18. Don't overuse either one

You might now think:

> Great! I'll just make everything `lateinit` or `lazy`.

Don't.

Prefer ordinary initialization when ordinary initialization works:

```kotlin
class User(
    val name: String
)
```

Prefer `lateinit` when:

> the property genuinely must be initialized later.

Prefer `lazy` when:

> initialization is expensive/unnecessary until first use, and deferred creation makes sense.

In other words:

```text
ordinary initialization
→ default choice

lateinit
→ special lifecycle requirement

lazy
→ special timing/performance requirement
```

---

# 19. A design example

Imagine:

```kotlin
class ReportGenerator {
    lateinit var template: Template

    val parser by lazy {
        Parser()
    }
}
```

These two properties have different stories.

### `template`

The object expects someone else to provide a template later.

```text
external setup
    ↓
template assigned
```

### `parser`

The class itself knows how to create the parser, but doesn't need it until it's actually used.

```text
first use
    ↓
create parser
```

That's exactly when the two mechanisms make sense.

---

# 20. One last important warning

Sometimes developers use `lateinit` to hide poor object design.

For example:

```kotlin
class Checkout {
    lateinit var paymentService: PaymentService
}
```

Then five different places in the program have to remember:

```kotlin
checkout.paymentService = ...
```

This makes the object's lifecycle harder to understand.

A constructor:

```kotlin
class Checkout(
    val paymentService: PaymentService
)
```

may make the dependency much clearer.

So whenever you see `lateinit`, ask:

> **Does this really need to be initialized later?**

That's a much better question than:

> "Can I use `lateinit` here?"

---

# Let's test your understanding

Don't run these immediately. Think first.

### Question 1

What is the conceptual difference between:

```kotlin
var name: String? = null
```

and:

```kotlin
lateinit var name: String
```

---

### Question 2

Which one represents:

> "The value isn't known yet, but when it exists it must be non-null."

```text
A. String?
B. lateinit String
```

---

### Question 3

Which one means:

> "Create this value when it is first accessed."

```text
A. lateinit
B. lazy
```

---

### Question 4

What happens here?

```kotlin
class Example {
    lateinit var text: String
}

val e = Example()
println(e.text)
```

Does it:

- print `null`
    
- print an empty string
    
- fail during compilation
    
- fail when `text` is accessed
    

Think about **why**, not just which option.

---

# Small coding exercise

Write this class:

```kotlin
class Profile
```

It should have:

```text
name
```

but you deliberately want to assign the name after the object is created.

Then write code that:

1. creates the `Profile`
    
2. assigns the name
    
3. prints the name
    

After that, create another property using `by lazy` that generates a string such as:

```text
"Profile: Alex"
```

and observe when that string actually gets created.

Don't worry about the exact implementation yet. The important thing is that you can explain the difference between the two mechanisms.

---

## The big picture so far

We've now built a pretty meaningful model of Kotlin properties:

```text
property
│
├── val
│   └── readable
│
├── var
│   ├── readable
│   └── writable
│
├── getter
│   └── can compute a value
│
├── setter
│   └── can validate/transform writes
│
├── backing field
│   └── actual storage when needed
│
├── init
│   └── initialization logic
│
├── lateinit
│   └── initialize later by assignment
│
└── lazy
    └── initialize automatically on first access
```

Next, we'll move on from initialization into another part of Kotlin's object model: **visibility, object declarations, and `companion object`**. Those concepts introduce a different question:

> **Who is allowed to access this thing, and does this thing belong to an object instance or to the class-level design?**