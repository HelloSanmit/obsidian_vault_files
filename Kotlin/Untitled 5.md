
You're right. I slipped back into a **course-template / textbook mode**: definition → table → numbered sections → exercises → recap. That isn't the way I want to teach you either.

You asked me to **teach the concept the way I naturally would if I were sitting beside you**, and that's what I'll do from here.

So let's actually redo our entry into Phase 7 that way.

---

# Phase 7 — Let's talk about sealed types

Forget the keyword `sealed` for a moment.

Suppose you're writing a program and somewhere you have this:

```kotlin
var status = "loading"
```

At first, that's perfectly understandable.

Then somewhere else:

```kotlin
status = "success"
```

And now you have another variable:

```kotlin
var data: String? = null
```

And perhaps another:

```kotlin
var error: String? = null
```

Eventually your code starts developing assumptions like:

```text
if status == "success"
    data must exist

if status == "error"
    error must exist

if status == "loading"
    data should probably be null
```

Do you see the problem?

The **relationship between these pieces of data exists only in your head**.

Kotlin sees:

```text
String
String?
String?
```

It doesn't know your rules.

You could accidentally end up with:

```text
status = "success"
data = null
error = "database failed"
```

and the types won't stop you.

That's the point at which I would start asking:

> "Why am I representing one concept using several unrelated variables?"

And that question leads us to sealed types.

---

## Let's model the thing itself

Instead of saying:

```kotlin
status = "success"
data = ...
error = ...
```

we can say:

```kotlin
sealed interface Result
```

and then describe what a `Result` can actually be:

```kotlin
data object Loading : Result

data class Success(val data: String) : Result

data class Failure(val error: String) : Result
```

Now our model looks like this:

```text
             Result
            /   |   \
           /    |    \
      Loading Success Failure
                |       |
              data     error
```

That is a much more truthful description of the problem.

A result isn't:

> a status string + some optional variables.

A result is:

> **one of a few specific things.**

And that's exactly what the sealed hierarchy expresses.

---

## This is where Kotlin gets interesting

Now imagine I write:

```kotlin
fun printResult(result: Result) {
    when (result) {
        Loading -> println("Loading...")
        is Success -> println("Got ${result.data}")
        is Failure -> println("Failed: ${result.error}")
    }
}
```

I like this code because when I read it, I can understand the whole world of `Result` by looking at it.

There isn't some invisible fourth possibility hiding somewhere.

There isn't a random string `"banana"`.

There isn't a `Success` with a missing value.

The structure of the type is telling me what reality is supposed to look like.

That's the real reason sealed classes/interfaces are useful.

Not:

> "Sealed classes are used for exhaustive `when`."

That's true, but that's almost the **side effect**.

The deeper thing is:

> **You are deliberately making the set of possibilities finite and explicit.**

The compiler can then take advantage of that.

---

# Now compare that with an enum

We already learned enums.

Suppose I write:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

This is perfect.

There are four possible directions.

And each one is basically the same kind of thing.

```text
Direction
├── NORTH
├── SOUTH
├── EAST
└── WEST
```

But imagine trying to use an enum for our `Result`:

```kotlin
enum class Result {
    LOADING,
    SUCCESS,
    FAILURE
}
```

We immediately hit a problem.

What does the successful result contain?

You would need some separate variable:

```kotlin
var result: Result
var data: String?
var error: String?
```

And we're back where we started.

With a sealed type:

```kotlin
data class Success(val data: String) : Result
data class Failure(val error: String) : Result
```

the data travels **with the case**.

That's the key difference.

An enum says:

> "Here are the names of the possible values."

A sealed hierarchy can say:

> "Here are the possible forms the value can take, and each form can have completely different data."

That's much richer.

---

# Let me show you another example, because this is where it really clicks

Suppose you're modeling a shape.

You could do:

```kotlin
enum class ShapeType {
    CIRCLE,
    RECTANGLE,
    TRIANGLE
}
```

But then you'd probably need:

```kotlin
var type: ShapeType
var radius: Double?
var width: Double?
var height: Double?
var base: Double?
```

Now you have a mess of values that don't all make sense together.

A triangle doesn't have a radius.

A circle doesn't have a width and height.

So instead:

```kotlin
sealed interface Shape
```

and:

```kotlin
data class Circle(
    val radius: Double
) : Shape

data class Rectangle(
    val width: Double,
    val height: Double
) : Shape

data class Triangle(
    val base: Double,
    val height: Double
) : Shape
```

Now look at what happened.

We've taken information that used to exist as **rules in our head** and put it into the **types themselves**.

A `Circle` has a radius.

A `Rectangle` has width and height.

A `Triangle` has base and height.

You can't accidentally construct:

```text
Circle(width = 10, errorMessage = "oops")
```

because the type doesn't describe such a thing.

This is what people mean when they talk about using types to make illegal states harder or impossible to represent.

---

# And now `when` becomes much more than a switch

We learned earlier that `when` can check types:

```kotlin
when (shape) {
    is Circle -> ...
    is Rectangle -> ...
    is Triangle -> ...
}
```

Before sealed types, we learned that kind of syntax almost as an isolated feature.

Now you can see why it exists.

The whole thing works together:

```text
sealed hierarchy
        ↓
known set of alternatives
        ↓
when can inspect the alternative
        ↓
smart cast gives access to its specific data
```

So here:

```kotlin
fun area(shape: Shape): Double {
    return when (shape) {
        is Circle -> Math.PI * shape.radius * shape.radius
        is Rectangle -> shape.width * shape.height
        is Triangle -> 0.5 * shape.base * shape.height
    }
}
```

when we're inside:

```kotlin
is Circle
```

Kotlin knows:

> "Ah, you're talking about the `Circle` case now."

So:

```kotlin
shape.radius
```

becomes available.

This is all the stuff you've already learned—`when`, `is`, smart casts—finally converging into something useful.

---

# There is a subtle distinction I want you to notice

We have now seen three different ideas:

```kotlin
enum class
sealed interface
object
```

They are **not three variations of the same feature**.

They answer different questions.

`enum` asks:

> What are the fixed named values?

`sealed` asks:

> What are the fixed kinds of values this type can be?

`object` asks:

> Is there one actual object representing this thing?

That last one becomes useful inside sealed hierarchies.

Look at:

```kotlin
sealed interface Result

data object Loading : Result
data class Success(val data: String) : Result
data class Failure(val error: String) : Result
```

Why is `Loading` an object?

Because there isn't any information attached to loading.

Every loading state would essentially be identical.

We don't need:

```text
Loading #1
Loading #2
Loading #3
```

So one object is enough.

But `Success` is different:

```kotlin
Success("hello")
Success("goodbye")
Success("42")
```

Those are different values, so we need a class.

That's a really nice example of several Kotlin concepts fitting together naturally.

---

# Sealed class vs sealed interface

Don't overcomplicate this yet.

The simplest way I'd explain it is:

A **sealed class** says:

> These things belong to this class hierarchy.

A **sealed interface** says:

> These things implement this closed contract.

For example:

```kotlin
sealed class Result
```

lets the root itself behave like a class and can hold common state/implementation.

Whereas:

```kotlin
sealed interface Result
```

is often more lightweight when what you really want is a closed family of types.

There's also an inheritance consequence.

A class can have only one superclass:

```kotlin
class Something : SomeBaseClass()
```

but it can implement multiple interfaces:

```kotlin
class Something : SomeBaseClass(), SomeInterface, Result
```

So sealed interfaces are useful when the alternatives already have other class relationships.

We'll get more precise about this later. Right now, I want you to understand the **reason for the feature**, not memorize inheritance trivia.

---

# Here's the question I want you to start asking

Whenever you see code like:

```kotlin
var type: String
var x: Int?
var y: Int?
var error: String?
```

don't immediately start writing `if` statements.

Stop and ask:

> **Are these really several unrelated variables, or are they different forms of one concept?**

If they're different forms of one concept, a sealed hierarchy may be a much better model.

That's a powerful programming habit.

---

# One more example

Imagine a network request can have these states:

```text
NotStarted
Loading
Success(data)
Failure(message)
```

I could represent that with four separate variables and a bunch of conventions.

Or I could just write what I mean:

```kotlin
sealed interface RequestState

data object NotStarted : RequestState

data object Loading : RequestState

data class Success(
    val data: String
) : RequestState

data class Failure(
    val message: String
) : RequestState
```

Then somewhere else:

```kotlin
fun show(state: RequestState) {
    when (state) {
        NotStarted -> println("Not started")
        Loading -> println("Loading...")
        is Success -> println("Data: ${state.data}")
        is Failure -> println("Error: ${state.message}")
    }
}
```

Read that code from top to bottom.

You can almost understand the entire state machine without needing a paragraph of documentation.

**That is what good modeling feels like.**

The code isn't merely executing the rules.

The code is **expressing the rules**.

---

# And this is where I want Phase 7 to go

We're not going to blast through:

```text
sealed classes
value classes
delegation
```

as three vocabulary words.

We're going to understand why each one exists.

Today the core idea is:

> **Sealed types let you represent a closed set of structured alternatives.**

Once that feels natural, we'll explore the edges:

- when sealed classes are better than enums,
    
- when sealed interfaces make more sense,
    
- nested sealed types,
    
- `data object`,
    
- common mistakes,
    
- and how to keep sealed hierarchies small and meaningful.
    

Then we'll move to **value classes**, where the question changes from:

> "What kinds of things can this value be?"

to:

> **"Can the type system distinguish two values that happen to use the same underlying representation?"**

That's where Kotlin's type system gets even more interesting.