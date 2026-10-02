
# Lesson 14 — Kotlin's Type System: `Any`, `Unit`, `Nothing`, Casting, and Smart Casts

We learned earlier that:

```kotlin
val age = 25
```

means Kotlin can infer:

```text
age : Int
```

And:

```kotlin
val name = "Alice"
```

means:

```text
name : String
```

So far, we've mostly treated types as simple labels:

```text
Int
String
Boolean
Double
```

Today we're going to look underneath that.

We'll start answering questions like:

> Is everything in Kotlin an object?

> Is `Int` related to `String` somehow?

> What type can hold an arbitrary value?

> What is `Unit`?

> Why does Kotlin have something called `Nothing`?

> What happens when you don't know the actual type of a value?

> How can Kotlin discover a value's more specific type at runtime?

These ideas are extremely important later when we reach **classes, inheritance, generics, sealed classes, exceptions, and APIs**.

---

# 1. Types form a hierarchy

Imagine we have:

```kotlin
val age = 25
```

The type is:

```text
Int
```

Now imagine:

```kotlin
val name = "Alice"
```

The type is:

```text
String
```

These aren't completely unrelated islands.

Kotlin has a type hierarchy.

At the broadest conceptual level, you can think of:

```text
             Any
           /     \
       String    Int
         ...
```

`Any` is a very general Kotlin type.

The important intuition is:

> **`Any` can represent a value of essentially any non-null Kotlin type.**

For example:

```kotlin
val a: Any = 10
val b: Any = "Hello"
val c: Any = true
```

All three are valid.

So:

```text
10
 ↓
Int
 ↓
can also be viewed as Any
```

and:

```text
"Alice"
 ↓
String
 ↓
can also be viewed as Any
```

This is our first look at **subtyping**.

---

# 2. `Any` does not mean "I forgot the type"

Consider:

```kotlin
val value: Any = 10
```

It doesn't mean:

> "Kotlin doesn't know what this is."

Kotlin absolutely knows more about the value at runtime.

What you've said is:

> "Treat this value through the general `Any` type."

So conceptually:

```text
actual value
    ↓
Int

viewed through:
    ↓
Any
```

The distinction between an object's **actual runtime type** and the **static type through which we're currently viewing it** will become very important.

---

# 3. Why would you ever use `Any`?

Sometimes a function genuinely needs to accept different kinds of values.

For example:

```kotlin
fun printAnything(value: Any) {
    println(value)
}
```

Now:

```kotlin
printAnything(10)
printAnything("Hello")
printAnything(true)
```

all work.

But notice the tradeoff.

Inside:

```kotlin
fun printAnything(value: Any) {
```

you don't automatically know that `value` is an `Int`.

It could be:

```text
Int
String
Boolean
User
List<...>
...
```

So `Any` gives flexibility at the cost of specificity.

That's a pattern you'll see throughout programming:

> **The broader the type, the less specific the compiler can be about the value.**

---

# 4. `Any` vs `Any?`

This connects directly to our null-safety lesson.

You can write:

```kotlin
val value: Any = "Hello"
```

but `Any` itself is non-nullable.

If you want null to be allowed:

```kotlin
val value: Any? = null
```

So:

```text
Any
↓
any non-null value

Any?
↓
any value, including null
```

This is the same pattern we've already seen:

```text
String
String?
```

The `?` means the absence of a value is allowed.

---

# 5. Now let's talk about `Unit`

You've already seen functions like:

```kotlin
fun greet(name: String) {
    println("Hello, $name")
}
```

We said this function doesn't return a useful value.

Its return type is effectively:

```text
Unit
```

You could explicitly write:

```kotlin
fun greet(name: String): Unit {
    println("Hello, $name")
}
```

but normally you don't.

So what is `Unit`?

At a practical level:

> `Unit` represents the result of a function whose meaningful purpose is its side effect rather than producing useful data.

For example:

```kotlin
fun printMessage(message: String): Unit {
    println(message)
}
```

You call it:

```kotlin
printMessage("Hello")
```

and you care about the printing, not about receiving some meaningful result.

---

# 6. `Unit` is a real type

This may seem unnecessary.

Why not simply say:

> "The function doesn't return anything"?

Because Kotlin treats functions consistently as things that have return types.

So:

```text
(Int) -> Int
```

means:

> takes an `Int`, returns an `Int`.

And:

```text
(String) -> Unit
```

means:

> takes a `String`, performs its operation, and returns `Unit`.

This becomes particularly useful when we pass functions around.

For example:

```kotlin
fun execute(action: () -> Unit) {
    action()
}
```

Now the type tells us:

> `action` is a function that takes no arguments and returns `Unit`.

---

# 7. `Unit` is not the same thing as `null`

This distinction matters.

`Unit` represents a successful completion with one specific `Unit` value.

`null` represents absence of a value.

So:

```text
Unit
↓
there is a meaningful "completed" result type

null
↓
there is no value
```

Consider:

```kotlin
fun greet() {
    println("Hello")
}
```

This function returns `Unit`.

It isn't returning `null`.

Similarly:

```kotlin
val name: String? = null
```

is a nullable String.

Different concepts.

---

# 8. `Nothing` is much stranger

Now we get to one of Kotlin's most interesting types.

```text
Nothing
```

At first this looks like an odd name.

What does it mean?

A useful practical definition is:

> **`Nothing` represents a computation that never successfully produces a value.**

For example:

```kotlin
fun fail(): Nothing {
    throw IllegalStateException("Something went wrong")
}
```

This function never reaches a normal return.

It throws an exception.

Therefore there is no successful value coming back.

Its return type can be:

```text
Nothing
```

---

# 9. Another example of `Nothing`

Consider:

```kotlin
fun infiniteLoop(): Nothing {
    while (true) {
    }
}
```

This function never returns normally.

It just continues forever.

So its return type can be `Nothing`.

The core idea is:

```text
Unit
↓
function completes normally, but meaningful result isn't needed

Nothing
↓
function cannot produce a normal result at all
```

That's a very different concept.

---

# 10. Why is `Nothing` useful?

Because the type system can reason about code paths that don't continue normally.

For example:

```kotlin
val name: String = null ?: error("No name")
```

The `error(...)` function returns `Nothing`.

So if `name` is null:

```text
error(...)
   ↓
program does not continue normally
```

There is no value that needs to be produced by the failing branch.

Therefore Kotlin can still treat the overall expression as compatible with `String`.

This will become much clearer when we study the type system in more depth.

---

# 11. `Nothing` is the bottom of the type hierarchy

Conceptually, you can think of Kotlin's hierarchy something like:

```text
        Any
       /   \
     ...   ...
       \   /
      Nothing
```

This is a simplified conceptual picture, but it captures an important mathematical relationship:

> `Nothing` is a subtype of every type.

That's why a `Nothing` expression can appear where some other type is expected.

For example:

```kotlin
val name: String = error("Missing name")
```

The expression:

```kotlin
error("Missing name")
```

has type:

```text
Nothing
```

and `Nothing` can fit into the required type position because that execution path never actually gives you a value.

This is an advanced idea, so don't worry if it takes time to feel intuitive.

---

# 12. Now let's return to `Any`

Suppose:

```kotlin
val value: Any = "Hello"
```

Can we do:

```kotlin
println(value.length)
```

No.

Why?

Because the **static type** is:

```text
Any
```

And an arbitrary `Any` value doesn't necessarily have a `length`.

It could be:

```text
10
true
User(...)
"Hello"
```

So Kotlin won't let you assume it's a String.

This is exactly what strong typing is supposed to do.

---

# 13. Type checking with `is`

We can ask what kind of value something is.

Kotlin uses:

```kotlin
is
```

For example:

```kotlin
val value: Any = "Hello"

if (value is String) {
    println(value.length)
}
```

Now Kotlin knows that inside the `if` branch:

```text
value : String
```

even though outside the branch its declared type is:

```text
Any
```

This is called a **smart cast**.

---

# 14. What exactly happened?

At the beginning:

```text
value : Any
```

We then check:

```kotlin
value is String
```

Suppose the result is `true`.

Kotlin now knows:

```text
value is actually a String
```

So inside:

```kotlin
if (value is String) {
    println(value.length)
}
```

we can use:

```kotlin
value.length
```

without explicitly casting.

Conceptually:

```text
Any
 ↓
is String?
 ↓
yes
 ↓
treat as String
```

This is one of Kotlin's nicest type-system features.

---

# 15. Smart casts aren't limited to `String`

For example:

```kotlin
val value: Any = 42

if (value is Int) {
    println(value + 10)
}
```

Inside the branch, Kotlin knows:

```text
value : Int
```

so:

```kotlin
value + 10
```

is valid.

Another:

```kotlin
val value: Any = true

if (value is Boolean) {
    println(!value)
}
```

The compiler understands the narrowed type.

---

# 16. You can test for the opposite with `!is`

For example:

```kotlin
if (value !is String) {
    println("Not a string")
}
```

Read:

> If `value` is not a String...

This is useful when handling multiple possibilities.

For example:

```kotlin
fun describe(value: Any) {
    if (value !is String) {
        println("This isn't text")
        return
    }

    println("String length: ${value.length}")
}
```

Notice the early `return`.

After this:

```kotlin
if (value !is String) {
    return
}
```

the remaining code is reached only when `value` **is** a String.

Kotlin can often reason about that control flow.

---

# 17. Smart casts are based on what Kotlin can prove

Here's a subtle but important point.

Consider:

```kotlin
var value: Any = "Hello"

if (value is String) {
    println(value.length)
}
```

This may be smart-cast appropriately in a simple case.

But mutable variables can be trickier.

Suppose another piece of code could change `value` between the check and the use.

Then Kotlin may refuse the smart cast because it can't prove the value remains the same type.

This leads to an important principle:

> **The compiler only gives you a smart cast when it can prove the relevant condition remains valid.**

That's why immutable local values are often easier for the compiler to reason about.

This will connect back to our `val` vs `var` discussion.

---

# 18. Explicit casting with `as`

Sometimes you know the actual type and want to tell Kotlin explicitly.

For example:

```kotlin
val value: Any = "Hello"

val text = value as String
```

Now:

```text
text : String
```

and you can do:

```kotlin
println(text.length)
```

The operator:

```text
as
```

is an explicit cast.

You're telling Kotlin:

> Treat this value as this type.

But there is a risk.

---

# 19. Unsafe casts can fail

Consider:

```kotlin
val value: Any = 42

val text = value as String
```

This is wrong.

The actual value is an `Int`, not a `String`.

At runtime, the cast fails.

So:

```text
Any
 ↓
as String
 ↓
❌ runtime failure
```

This is why `as` should be used when you genuinely know the type relationship you're asserting.

---

# 20. Safe casts with `as?`

We've seen this briefly before.

Instead of:

```kotlin
value as String
```

you can write:

```kotlin
value as? String
```

If the cast succeeds:

```text
"Hello"
```

If it fails:

```text
null
```

So:

```kotlin
val value: Any = 42

val text = value as? String
```

gives:

```text
text : String?
text = null
```

This is a beautiful example of how two Kotlin ideas connect:

```text
safe cast
   ↓
nullable result
```

The type itself communicates:

> The conversion might not succeed.

---

# 21. `is` vs `as?`

You have two different ways to approach an uncertain type.

### Check first

```kotlin
if (value is String) {
    println(value.length)
}
```

You're asking:

> Is this a String?

Kotlin then smart-casts it.

### Try to convert

```kotlin
val text = value as? String
```

You're asking:

> Can I safely treat this as a String?

If not, you get `null`.

Neither is universally better.

They express different intentions.

---

# 22. `when` becomes very powerful with type checking

We haven't properly studied `when` yet, but you've already seen it in the curriculum.

Consider:

```kotlin
fun describe(value: Any) {
    when (value) {
        is String -> println("String: $value")
        is Int -> println("Integer: $value")
        is Boolean -> println("Boolean: $value")
        else -> println("Something else")
    }
}
```

If:

```kotlin
describe("Hello")
```

we go into:

```text
is String
```

If:

```kotlin
describe(42)
```

we go into:

```text
is Int
```

The compiler understands the type inside each branch.

So:

```text
Any
 ↓
when
 ├── String
 ├── Int
 ├── Boolean
 └── something else
```

This becomes much more powerful later with **sealed classes**, where `when` can model a fixed set of states.

---

# 23. Type checking is not just about "what object is this?"

It's also about behavior available from the type.

Suppose:

```kotlin
fun printLength(value: Any) {
    if (value is String) {
        println(value.length)
    }
}
```

Before the check, this isn't valid:

```kotlin
value.length
```

because `Any` has no `length`.

After the check, it becomes valid because `String` has that property.

So type narrowing effectively says:

> "Now that I know what kind of value this is, I know which operations are available."

That's a core principle of typed programming.

---

# 24. Nullable type + smart cast

Now let's combine two earlier ideas.

Suppose:

```kotlin
val name: String? = "Alice"
```

We know:

```text
name : String?
```

If we write:

```kotlin
if (name != null) {
    println(name.length)
}
```

Kotlin narrows:

```text
String?
 ↓
after non-null check
 ↓
String
```

This is another form of smart casting.

So smart casts aren't only about `Any` and explicit type checks.

They also work with nullability.

---

# 25. Here's a useful chain of reasoning

Suppose:

```kotlin
val value: Any? = "Hello"
```

What possibilities are represented?

```text
Any?
│
├── String
├── Int
├── Boolean
├── ...
└── null
```

Now:

```kotlin
if (value is String) {
    println(value.length)
}
```

Inside the branch, we've narrowed an enormous set of possibilities down to:

```text
String
```

That is essentially **type refinement**.

We'll eventually discuss type refinement and algebraic-style modeling much more deeply.

---

# 26. Why Kotlin's type system is so useful

Imagine a function:

```kotlin
fun process(value: Any) {
    ...
}
```

Because it accepts anything, the caller has a lot of flexibility.

But the function itself has less information.

Compare:

```kotlin
fun process(name: String) {
    ...
}
```

Here the compiler can immediately give you:

- String-specific operations
    
- null guarantees
    
- better autocomplete
    
- type checking
    
- clearer API expectations
    

This is why **specific types are generally preferable when you know what you actually require**.

Use `Any` because the abstraction genuinely needs arbitrary values, not merely because you don't want to decide on a type.

---

# 27. A practical example

Let's build a function that accepts arbitrary values:

```kotlin
fun describe(value: Any): String {
    return when (value) {
        is String -> "Text with ${value.length} characters"
        is Int -> "Integer: $value"
        is Boolean -> "Boolean: $value"
        else -> "Unknown value"
    }
}
```

Now:

```kotlin
println(describe("Kotlin"))
println(describe(42))
println(describe(true))
```

Possible output:

```text
Text with 6 characters
Integer: 42
Boolean: true
```

Notice what's happening.

The function begins with:

```text
Any
```

then uses runtime type information to narrow the possibilities.

This is a very common pattern when dealing with heterogeneous data.

---

# 28. But heterogeneous data is often a design smell

Don't take that example as:

> "Use `Any` everywhere and inspect types."

Usually, if your program frequently needs:

```kotlin
when (value) {
    is String -> ...
    is Int -> ...
    is Boolean -> ...
}
```

it may indicate that you should create a more meaningful domain type.

Later we might model that with:

```kotlin
sealed interface Value
```

and specific implementations such as:

```kotlin
data class TextValue(...)
data class NumberValue(...)
data class BooleanValue(...)
```

Now the possible states are explicitly modeled.

This is where Kotlin's type system becomes a design tool rather than just a compiler feature.

We'll study that later.

---

# 29. A note about primitive types

You may remember us saying:

```text
Int
Double
Boolean
```

and also that Kotlin has a type hierarchy.

On the JVM, there's an interesting complication.

Kotlin's `Int` is represented efficiently using JVM primitive types in many situations, rather than always being a heap-allocated Java-style object.

But Kotlin's language model lets you treat `Int` as a normal Kotlin type.

This is one reason our later JVM lesson will be useful.

You don't need to worry about the implementation details now.

Just remember:

> Kotlin's clean type model doesn't map one-to-one onto the raw JVM representation in every situation.

We'll dig into boxing and unboxing later.

---

# 30. Let's look at `Nothing` one more time

Consider:

```kotlin
fun requireUser(user: String?): String {
    return user ?: error("User is missing")
}
```

What happens?

If `user` is non-null:

```text
user
 ↓
String
 ↓
return
```

If `user` is null:

```text
user
 ↓
null
 ↓
error(...)
 ↓
execution doesn't continue normally
```

Because:

```text
error(...) : Nothing
```

the compiler can understand that the failing branch doesn't need to provide an ordinary String.

This is one reason `Nothing` is useful even though you rarely write it directly.

---

# 31. `Nothing` and exceptions

Another example:

```kotlin
fun fail(): Nothing {
    throw RuntimeException("Failure")
}
```

If you write:

```kotlin
val result: Int = fail()
```

the type system can allow this because `fail()` never actually gives you a value.

The execution either:

```text
throws
```

or never returns.

There is no successful value to contradict the `Int` requirement.

This is a subtle but elegant part of the type system.

---

# 32. Let's connect all the special types

You should now have this rough picture:

```text
Any
│
├── String
├── Int
├── Boolean
├── Double
├── Your future classes...
│
└── ...
```

And:

```text
Any?
│
├── Any
├── null
└── nullable types
```

While:

```text
Unit
```

means:

> A normal completion with no meaningful result.

And:

```text
Nothing
```

means:

> No normal result can ever be produced.

These are very different roles.

---

# 33. Your exercises

### Exercise 1 — `Any`

Create:

```kotlin
val value: Any = ...
```

Try assigning:

```text
an Int
a String
a Boolean
a Double
```

to it.

Observe that all are valid.

Then try using an operation that only exists on one of the specific types.

See what the compiler says.

---

### Exercise 2 — `is`

Create:

```kotlin
val value: Any = "Kotlin"
```

Use `is String` to safely access:

```kotlin
value.length
```

Then change the value to an integer and observe the branch behavior.

---

### Exercise 3 — `as`

Try:

```kotlin
val value: Any = "Hello"
val text = value as String
```

Then intentionally make the underlying value an `Int` and see what happens.

---

### Exercise 4 — `as?`

Repeat the previous exercise using:

```kotlin
as? String
```

Observe the difference.

---

### Exercise 5 — smart cast

Write a function:

```text
describe(value: Any)
```

that prints different messages for:

```text
String
Int
Boolean
```

using `when` and `is`.

---

### Exercise 6 — `Nothing`

Write:

```kotlin
fun fail(message: String): Nothing
```

that throws an exception.

Then understand why this is valid:

```kotlin
val result: String = fail("Something failed")
```

even though `fail()` obviously doesn't return a String.

Don't worry if that last point still feels strange.

It should.

It's your first encounter with an idea from the deeper type theory behind Kotlin.

---

# Challenge

Create:

```kotlin
fun describeValue(value: Any?): String
```

It should distinguish:

```text
null
String
Int
Boolean
Double
```

For example, conceptually:

```text
describeValue("hello")
→ "String: hello"

describeValue(42)
→ "Int: 42"

describeValue(null)
→ "No value"
```

Use:

```text
when
is
```

and smart casting.

Don't use `as` unless there's a specific reason.

The goal is to practice **narrowing a broad type into something more specific**.

---

# The deeper idea

We started this course thinking about types as:

```text
Int
String
Boolean
```

Now you're seeing that types actually form part of the **logic of the program**.

A type tells us:

```text
what a value can be
what operations are available
what assumptions are safe
what values are possible
what values are impossible
```

And Kotlin can use control flow to refine what it knows:

```text
Any
 ↓
is String?
 ↓
yes
 ↓
String
```

or:

```text
String?
 ↓
!= null?
 ↓
String
```

That is why Kotlin's type system isn't merely a collection of annotations.

It's actively participating in reasoning about your program.

---

The next lesson is going to bring together two ideas we've touched several times:

**`when` + conditions + ranges + type checking + expressions.**

We'll learn why Kotlin's `when` is much more powerful than a traditional `switch`, how exhaustive `when` works, how `when` can return values, and how it becomes especially powerful when modeling different kinds of states.