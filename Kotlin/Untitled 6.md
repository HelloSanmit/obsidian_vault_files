
# Phase 7 — Lesson 25: Going Deeper with Sealed Types

Last lesson we discovered _why_ sealed types exist.

Today I want to push that idea a little further, because there is a point where you can know the syntax:

```kotlin
sealed interface Result
```

but still not really know **when a sealed type is the right model**.

So let's forget the syntax for a moment and think like someone designing a program.

---

Imagine you're writing a checkout system.

A payment can end in several ways:

```text
successful
failed
cancelled
needs authentication
```

You might initially write:

```kotlin
enum class PaymentStatus {
    SUCCESS,
    FAILED,
    CANCELLED,
    NEEDS_AUTHENTICATION
}
```

That's perfectly reasonable.

But now suppose a successful payment has:

```text
transactionId
```

a failure has:

```text
reason
```

and authentication requires:

```text
authenticationUrl
```

Now the simple enum isn't telling the whole story.

You could add a pile of nullable properties:

```kotlin
data class PaymentResult(
    val status: PaymentStatus,
    val transactionId: String?,
    val reason: String?,
    val authenticationUrl: String?
)
```

And immediately the model starts feeling wrong.

You can have:

```text
SUCCESS + reason
FAILED + transactionId
CANCELLED + authenticationUrl
```

Those combinations don't make sense, but the type system allows them.

So let's describe what we actually mean.

```kotlin
sealed interface PaymentResult
```

Then:

```kotlin
data class Success(
    val transactionId: String
) : PaymentResult

data class Failed(
    val reason: String
) : PaymentResult

data object Cancelled : PaymentResult

data class NeedsAuthentication(
    val authenticationUrl: String
) : PaymentResult
```

Now read the model again.

It says:

> A `PaymentResult` is one of these four things, and each thing carries the information appropriate to that thing.

That's the real power.

---

## A sealed hierarchy is a family

I want you to start seeing this:

```kotlin
sealed interface PaymentResult
```

not as "an interface with a special keyword."

See it as:

```text
                    PaymentResult
                          |
       ┌──────────┬───────┼──────────────┐
       |          |       |              |
    Success     Failed  Cancelled   NeedsAuthentication
       |          |                      |
 transactionId  reason             authenticationUrl
```

It's a **family of related possibilities**.

And the relationship is important enough that we want the compiler to know about it.

---

# The really nice part: you can reason from the parent type

Imagine a function:

```kotlin
fun handlePayment(result: PaymentResult) {
    when (result) {
        is Success -> {
            println("Transaction: ${result.transactionId}")
        }

        is Failed -> {
            println("Payment failed: ${result.reason}")
        }

        Cancelled -> {
            println("Payment cancelled")
        }

        is NeedsAuthentication -> {
            println("Authenticate here: ${result.authenticationUrl}")
        }
    }
}
```

When I read this function, I don't have to wonder:

> "Did the programmer forget some possible payment result?"

The sealed hierarchy answers that question.

And if we later add:

```kotlin
data object Refunded : PaymentResult
```

the compiler can force us to revisit relevant exhaustive `when` expressions.

That is a really nice relationship between:

```text
type design
and
control flow
```

---

# Now let's talk about `sealed class`

Everything we've done so far can also be expressed with a sealed class.

```kotlin
sealed class PaymentResult
```

and:

```kotlin
class Success(
    val transactionId: String
) : PaymentResult()
```

So what is the difference?

The biggest conceptual difference is that a class is a **class**, while an interface is an **interface**.

That sounds obvious, but it matters when designing the hierarchy.

---

## A sealed class can carry common implementation/state

For example:

```kotlin
sealed class Result(
    val timestamp: Long
)
```

Now every result has a timestamp.

Then:

```kotlin
data class Success(
    val value: String,
    val time: Long
) : Result(time)

data class Failure(
    val message: String,
    val time: Long
) : Result(time)
```

But perhaps we also want common behavior:

```kotlin
sealed class Result(
    val timestamp: Long
) {
    fun isRecent(currentTime: Long): Boolean {
        return currentTime - timestamp < 10_000
    }
}
```

Every subtype gets that behavior.

So a sealed class can be a place for **shared state and shared implementation**.

---

# A sealed interface is lighter

Suppose our types don't need common stored state or shared implementation.

We just want:

> "These types form one closed family."

Then:

```kotlin
sealed interface Result
```

is often the cleaner model.

For example:

```kotlin
sealed interface Result

data class Success(val value: String) : Result

data class Failure(val message: String) : Result

data object Loading : Result
```

The interface isn't trying to own state.

It's defining the family.

That's a useful way to think about the distinction.

---

# Here's where inheritance makes the difference important

Remember that a class can only have one superclass.

So:

```kotlin
class User : Person()
```

means `User` has `Person` as its superclass.

But a class can implement multiple interfaces:

```kotlin
class User : Person(), Serializable, Auditable
```

So imagine some type already needs to inherit from another class:

```kotlin
open class BaseEntity
```

and you want it to be part of a sealed family:

```kotlin
sealed interface Command
```

You can do:

```kotlin
class Move(
    val x: Int,
    val y: Int
) : BaseEntity(), Command
```

That flexibility is one reason sealed interfaces are useful.

The object can participate in an existing class hierarchy while also belonging to the sealed family.

---

# There's another thing I want you to understand: where the subclasses live

A sealed hierarchy is supposed to be **known and controlled**.

For example:

```kotlin
sealed class Result
```

and its direct subclasses are declared in permitted locations according to Kotlin's sealed-class rules.

That's why Kotlin can reason about the hierarchy.

This is fundamentally different from:

```kotlin
open class Result
```

where anybody can extend it elsewhere.

So think:

```text
open
→ "other code may extend this"

sealed
→ "the family is controlled"
```

That closed-world assumption is what gives the compiler useful knowledge.

---

# Sealed doesn't mean "private"

This is another thing beginners sometimes assume.

Suppose:

```kotlin
sealed interface Result
```

It doesn't mean that nobody outside the file can use `Result`.

You can absolutely expose:

```kotlin
fun loadUser(): Result
```

and callers can receive a `Result`.

What is controlled is the **set of direct subclasses**, not ordinary visibility of the type.

So:

```text
sealed
≠
private
```

Don't mix those ideas.

---

# Sealed doesn't mean "only one instance"

Also important.

Suppose:

```kotlin
sealed interface Result

data class Success(val value: String) : Result
```

You can create:

```kotlin
Success("A")
Success("B")
Success("C")
```

So there can be many `Success` objects.

The sealed restriction applies to the **kinds of result**, not how many instances of each kind exist.

---

# And this is why `data object` makes sense

Look at:

```kotlin
data object Loading : Result
```

Why `object`?

Because there's no data.

We don't need:

```kotlin
Loading()
Loading()
Loading()
```

with different values.

There is simply one meaningful loading case.

So:

```text
Success(value)
→ many possible values

Failure(message)
→ many possible values

Loading
→ one data-less case
```

This gives us a very natural representation.

---

# A sealed hierarchy can describe a state machine

This is something I really want you to notice.

Take:

```kotlin
sealed interface DownloadState

data object NotStarted : DownloadState

data object Downloading : DownloadState

data class Completed(
    val fileName: String
) : DownloadState

data class Failed(
    val reason: String
) : DownloadState
```

Now imagine the program:

```text
NotStarted
    |
    v
Downloading
   /   \
  /     \
 v       v
Completed Failed
```

The types themselves are describing the possible states.

And your program can use:

```kotlin
when (state) {
    NotStarted -> ...
    Downloading -> ...
    is Completed -> ...
    is Failed -> ...
}
```

You're starting to build something that looks much closer to a formal state machine.

And this is where sealed types become much more than "a fancy enum."

---

# But don't turn everything into a sealed hierarchy

This is just as important.

You could write:

```kotlin
sealed interface UserAction
```

and then create:

```text
Create
Delete
Update
Refresh
Search
Login
Logout
...
```

and eventually end up with 47 subclasses.

At that point, the model may be fighting you instead of helping you.

The question isn't:

> "Can I use sealed here?"

The question is:

> **"Is there a genuinely meaningful closed family of alternatives here?"**

That's the design skill.

---

# Here's a case where an enum is still better

Suppose:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

Why complicate that into:

```kotlin
sealed interface Direction

data object North : Direction
data object South : Direction
data object East : Direction
data object West : Direction
```

You _can_.

But what did you gain?

Probably not much.

The four cases are simple named values.

An enum expresses that extremely well.

So don't think:

```text
sealed > enum
```

That's not the lesson.

It's:

```text
simple fixed values
→ enum

structured fixed alternatives
→ sealed hierarchy
```

---

# Here's a really useful test

When you're deciding between an enum and a sealed type, ask:

### "Does each case need its own kind of data?"

If the answer is no:

```kotlin
enum class Priority {
    LOW,
    MEDIUM,
    HIGH
}
```

may be perfect.

If the answer is yes:

```kotlin
sealed interface SearchResult

data class Found(
    val results: List<String>
) : SearchResult

data object NoResults : SearchResult

data class Failed(
    val reason: String
) : SearchResult
```

starts making much more sense.

That's not a hard law, but it's a very good first instinct.

---

# Here's another beautiful property of sealed types

Suppose I give you:

```kotlin
fun render(result: SearchResult): String
```

and you know only the declared type:

```text
SearchResult
```

you don't need to know which concrete subtype it is beforehand.

Polymorphism allows the actual value to be different.

But unlike an open class hierarchy, the set of possible alternatives is still controlled.

So you're combining two ideas:

```text
polymorphism
+
closed-world knowledge
```

That's unusually useful.

---

# One thing from earlier lessons is coming back

Remember this?

```kotlin
is
```

We learned that:

```kotlin
if (x is String) {
    ...
}
```

lets Kotlin refine its understanding of the type.

Now combine that with sealed types.

```kotlin
fun describe(result: Result): String {
    return when (result) {
        is Success -> result.value
        is Failure -> result.message
        Loading -> "Loading"
    }
}
```

Inside:

```kotlin
is Success
```

Kotlin understands:

```text
result : Success
```

rather than merely:

```text
result : Result
```

So it can expose:

```kotlin
result.value
```

This is why our earlier lessons weren't isolated after all.

You're seeing the pieces connect:

```text
interfaces
      +
inheritance
      +
polymorphism
      +
is
      +
smart casts
      +
when
      +
sealed types
```

That's the kind of understanding we're trying to build.

---

# There's a deeper programming idea hiding here

Consider these two approaches.

### Approach A

```text
state = "FAILED"
message = ...
data = ...
```

The program has a bunch of values plus a bunch of assumptions.

### Approach B

```text
Failed(message)
```

The state and its required data are one thing.

Approach B is powerful because the **representation reflects the domain**.

You're no longer relying on the programmer to remember:

> "When this flag has this value, that other variable needs to be non-null."

The structure itself tells you.

That's one of the central ideas of type-driven programming.

---

# Your turn

I don't want to give you ten exercises here.

I want you to wrestle with one.

Imagine a file download system.

A download can be:

```text
waiting
downloading
paused
completed
failed
```

But:

- `downloading` should contain the percentage complete
    
- `paused` should contain how much was downloaded
    
- `completed` should contain the filename
    
- `failed` should contain the error message
    
- `waiting` contains no additional data
    

Design a:

```kotlin
sealed interface DownloadState
```

for this.

Then write:

```kotlin
fun describe(state: DownloadState): String
```

using `when`.

Before writing code, sketch the hierarchy on paper:

```text
DownloadState
├── ?
├── ?
├── ?
├── ?
└── ?
```

and decide which cases should be classes and which should be objects.

That design step is actually more important than the syntax.

---

## One last thought before we move on

I want you to notice the progression we've made in the type system:

```text
String
    ↓
"this is some text"

enum
    ↓
"this is one of these known named values"

sealed type
    ↓
"this is one of these known structured alternatives"
```

And then the next concept will take us in a different direction.

Suppose you have:

```kotlin
val userId: String
val productId: String
val orderId: String
```

They're all `String`.

But are they really the same _kind of thing_?

That's the problem **value classes** are going to address.

And that will be our next lesson.