
# Phase 7 — Lesson 28: Delegation

This is the last major idea in Phase 7, and I want to spend real time on it because **delegation is one of those concepts that looks like syntax but is actually a design philosophy**.

You already know three ways objects can get behavior:

```text
write the behavior yourself
inherit behavior from a parent
use another object inside you
```

Delegation is about the third one.

And Kotlin makes that third approach extremely convenient.

---

# Let's start with a real problem

Suppose we have an interface:

```kotlin
interface Printer {
    fun print(text: String)
}
```

And we have a class that already knows how to print:

```kotlin
class ConsolePrinter : Printer {
    override fun print(text: String) {
        println(text)
    }
}
```

Now imagine we want another class:

```kotlin
class LoggingPrinter
```

which is also a `Printer`.

Maybe `LoggingPrinter` does some extra work:

```text
log something
then print
```

One approach is to implement everything ourselves:

```kotlin
class LoggingPrinter : Printer {
    override fun print(text: String) {
        println("Logging: $text")
        println(text)
    }
}
```

Fine.

But imagine the interface has 15 methods:

```kotlin
interface Printer {
    fun print(text: String)
    fun printHeader(text: String)
    fun printFooter(text: String)
    fun printError(text: String)
    fun printWarning(text: String)
    ...
}
```

And our `LoggingPrinter` doesn't actually want to implement all those methods.

We already have an object that knows how:

```kotlin
val printer = ConsolePrinter()
```

What we really want to say is:

> "I'm a Printer, but for most Printer behavior, use this other Printer."

That's delegation.

---

# The long way to delegate

Without Kotlin's delegation syntax, you could write:

```kotlin
class LoggingPrinter(
    private val printer: Printer
) : Printer {

    override fun print(text: String) {
        println("Logging: $text")
        printer.print(text)
    }

    override fun printHeader(text: String) {
        printer.printHeader(text)
    }

    override fun printFooter(text: String) {
        printer.printFooter(text)
    }

    override fun printError(text: String) {
        printer.printError(text)
    }

    // ...
}
```

Look at what we're doing.

Most of the class is just:

```text
receive method call
        ↓
pass it to printer
```

That's boilerplate.

Kotlin lets you express the same idea much more directly.

---

# Kotlin's delegation syntax

```kotlin
class LoggingPrinter(
    private val printer: Printer
) : Printer by printer
```

That's it.

This line:

```kotlin
: Printer by printer
```

means roughly:

> "Make `LoggingPrinter` implement `Printer`, and delegate the `Printer` interface implementation to `printer`."

Now this works:

```kotlin
val console = ConsolePrinter()
val logger = LoggingPrinter(console)

logger.print("Hello")
```

The `print()` call is forwarded to the `ConsolePrinter`.

---

# That's what `by` is doing here

You've already seen:

```kotlin
val config by lazy { ... }
```

and now:

```kotlin
class Something : Interface by object
```

The word `by` is involved in two related ideas:

```text
"Use this other thing to provide the implementation."
```

But the mechanism is different in each case.

For class delegation:

```kotlin
class LoggingPrinter(
    printer: Printer
) : Printer by printer
```

you're delegating **interface behavior**.

For a delegated property:

```kotlin
val config by lazy { ... }
```

you're delegating **property access**.

We'll do both today.

---

# First, understand class delegation completely

Suppose:

```kotlin
interface Clickable {
    fun click()
}
```

and:

```kotlin
class Button : Clickable {
    override fun click() {
        println("Button clicked")
    }
}
```

Now:

```kotlin
class LoggedButton(
    private val button: Clickable
) : Clickable by button
```

We can do:

```kotlin
val button = Button()
val logged = LoggedButton(button)

logged.click()
```

What happens?

Conceptually:

```text
logged.click()
      ↓
LoggedButton doesn't define click()
      ↓
delegate it to button
      ↓
button.click()
```

You don't have to manually write the forwarding code.

---

# But `LoggedButton` is still a `Clickable`

This is important.

```kotlin
val logged: Clickable = LoggedButton(button)
```

works.

So the outside world sees:

```text
LoggedButton
    ↓
is a Clickable
```

But internally:

```text
LoggedButton
    ↓
uses button
    ↓
to fulfill the Clickable contract
```

This is a very useful distinction.

---

# Delegation is not inheritance

This is where I want you to be very careful.

Suppose:

```kotlin
class Dog : Animal()
```

That's inheritance.

`Dog` is saying:

> "I am an `Animal` because I belong to that class hierarchy."

Now:

```kotlin
class DogTrainer(
    private val dog: Animal
) : Animal by dog
```

would not even be the same idea, and ordinary class delegation isn't a replacement for arbitrary class inheritance.

Delegation is about **providing an interface implementation through another object**.

So:

```text
inheritance
→ "I am a specialized version of this class"

delegation
→ "I can provide this interface's behavior using this other object"
```

That's a much healthier mental distinction.

---

# Delegation is really composition

Earlier we talked about:

> composition over inheritance.

Remember:

```kotlin
class Car(
    private val engine: Engine
)
```

A `Car` **has an** `Engine`.

That's composition.

Delegation takes composition and says:

> "Not only do I contain this object; let this object provide some of my behavior automatically."

So:

```text
composition
→ I contain another object

delegation
→ I contain another object and forward responsibility to it
```

This is why delegation is often described as a form of composition.

---

# Let's see when delegation becomes useful

Imagine:

```kotlin
interface Storage {
    fun save(data: String)
    fun load(): String
}
```

We already have:

```kotlin
class FileStorage : Storage {
    override fun save(data: String) {
        println("Saving to file")
    }

    override fun load(): String {
        return "file data"
    }
}
```

Now perhaps we want:

```kotlin
class CachedStorage(
    private val storage: Storage
) : Storage by storage
```

At this point, `CachedStorage` behaves like the underlying storage.

We can then override only the behavior we want to change.

That's the really nice part.

---

# You can override delegated members

Suppose:

```kotlin
interface Storage {
    fun save(data: String)
    fun load(): String
}
```

and:

```kotlin
class FileStorage : Storage {
    override fun save(data: String) {
        println("Saving: $data")
    }

    override fun load(): String {
        return "file data"
    }
}
```

Now:

```kotlin
class CachedStorage(
    private val storage: Storage
) : Storage by storage {

    override fun load(): String {
        println("Checking cache")
        return storage.load()
    }
}
```

Now:

```text
save()
→ delegated automatically

load()
→ our own implementation
```

So delegation isn't:

> "I give up all control."

It's:

> "Use this object as the default implementation, and I'll take over whatever I need."

This is extremely useful.

---

# Think of it as a default implementation source

For:

```kotlin
class CachedStorage(
    private val storage: Storage
) : Storage by storage
```

you can mentally read it as:

> "For methods I haven't implemented myself, `storage` handles them."

That's a very useful approximation.

---

# What if both the delegate and our class have state?

Suppose:

```kotlin
interface Counter {
    fun increment()
    fun value(): Int
}
```

Delegate:

```kotlin
class SimpleCounter : Counter {
    private var count = 0

    override fun increment() {
        count++
    }

    override fun value(): Int {
        return count
    }
}
```

Then:

```kotlin
class LoggedCounter(
    private val counter: Counter
) : Counter by counter {

    override fun increment() {
        println("Incrementing")
        counter.increment()
    }
}
```

Now `LoggedCounter` has its own behavior while the actual count still lives in the delegate.

That is perfectly valid.

---

# What if we call another delegated method from our own method?

This is subtle.

Suppose:

```kotlin
interface Worker {
    fun work()
    fun start()
}
```

and:

```kotlin
class RealWorker : Worker {
    override fun work() {
        println("Real work")
    }

    override fun start() {
        work()
    }
}
```

Now:

```kotlin
class DelegatingWorker(
    private val worker: Worker
) : Worker by worker {

    override fun work() {
        println("DelegatingWorker work")
    }
}
```

You might wonder:

> When `start()` is called through `DelegatingWorker`, and `RealWorker.start()` internally calls `work()`, will it call `DelegatingWorker.work()`?

No.

The delegate's own implementation is executing.

So:

```text
DelegatingWorker.start()
        ↓
delegated to RealWorker.start()
        ↓
RealWorker's `this`
        ↓
RealWorker.work()
```

not:

```text
DelegatingWorker.work()
```

This is a subtle consequence of the fact that the delegate is a separate object.

The delegated method doesn't somehow rewrite the delegate's internal `this`.

That's worth understanding because it prevents surprising behavior.

---

# Delegation does not magically merge two objects

Suppose:

```kotlin
class A : Interface by b
```

It doesn't mean:

```text
A + B become one object
```

There are still two objects.

```text
A object
 └── reference to B object
```

When a method isn't implemented by `A`, Kotlin routes the interface call through that reference.

That's all.

The two objects maintain their own state.

---

# Why is this better than inheritance sometimes?

Suppose we have:

```text
Logger
```

and:

```text
Database
```

and:

```text
NetworkClient
```

You could try to create enormous inheritance trees.

But these things aren't naturally parent/child relationships.

A database isn't a specialized logger.

A network client isn't a specialized database.

Composition is more natural.

Delegation lets one object say:

> "I can behave like this interface because another object already knows how."

That gives you flexibility without creating artificial class hierarchies.

---

# Now let's switch to delegated properties

This is the other half of delegation.

We already wrote:

```kotlin
val config by lazy {
    loadConfiguration()
}
```

Now you can see what that really means.

Instead of the property implementing all of its own behavior, another object handles the property access logic.

Think:

```text
config
  ↓
delegate
  ↓
"tell me how to get this value"
```

---

# A normal property

Normally:

```kotlin
var name = "Alex"
```

means roughly:

```text
read name
→ get stored value

write name
→ store new value
```

With a delegated property:

```kotlin
var name by something
```

you're saying:

> "Let `something` handle how this property is read and/or written."

---

# The delegate has a job

For a read-only property:

```kotlin
val x by delegate
```

the delegate needs to know how to provide the value.

For a mutable property:

```kotlin
var x by delegate
```

the delegate may need to know both:

```text
how to read
how to write
```

Kotlin uses specially named operations under the hood:

```kotlin
getValue(...)
setValue(...)
```

You don't need to implement these yourself to use the standard delegates.

But understanding that they exist makes delegated properties much less magical.

---

# `lazy` is itself a delegate

This:

```kotlin
val config by lazy {
    loadConfiguration()
}
```

means, conceptually:

```text
config
  ↓
delegates its access to the lazy mechanism
  ↓
lazy mechanism decides:
  "Have I initialized yet?"
  ↓
if not → create value
if yes → return saved value
```

So `lazy` isn't some special property syntax unrelated to delegation.

It's one of the most famous examples of a delegated property.

---

# Let's look at another standard delegate: `observable`

Kotlin provides property delegation tools that let you react when a property changes.

For example:

```kotlin
import kotlin.properties.Delegates

var name: String by Delegates.observable("Unknown") { _, old, new ->
    println("Changed from $old to $new")
}
```

Then:

```kotlin
name = "Alex"
name = "Sam"
```

causes the delegate to observe the changes.

Conceptually:

```text
assignment
   ↓
delegate
   ↓
notice old/new values
   ↓
run callback
   ↓
store/update value
```

This is another illustration of the bigger idea:

> **The property doesn't have to own all of its behavior itself.**

---

# `vetoable`

There's another standard delegate:

```kotlin
var age: Int by Delegates.vetoable(0) { _, _, new ->
    new >= 0
}
```

Conceptually:

> Before accepting the new value, ask the delegate whether it is allowed.

So:

```kotlin
age = 20
```

can be accepted.

But:

```kotlin
age = -5
```

can be rejected.

This is similar in spirit to the custom setter you learned earlier.

But now the behavior can be packaged into a reusable delegation mechanism.

That's the key difference.

A custom setter is behavior attached specifically to that property.

A delegate can encapsulate **a reusable policy**.

---

# That's a very important distinction

Suppose you have this pattern everywhere:

```text
trim incoming string
```

You could write custom setters over and over.

Or you could package the behavior into reusable delegation logic.

Likewise:

```text
logging
validation
observation
lazy initialization
storage elsewhere
```

can all potentially become reusable property behaviors.

Delegation is therefore partly about **reusing behavior cleanly**.

---

# You can create your own property delegate

This is where delegation becomes really interesting.

Suppose we want a property that always stores a trimmed string.

We can create:

```kotlin
class TrimmedStringDelegate(
    private var storedValue: String = ""
) {
    operator fun getValue(
        thisRef: Any?,
        property: kotlin.reflect.KProperty<*>
    ): String {
        return storedValue
    }

    operator fun setValue(
        thisRef: Any?,
        property: kotlin.reflect.KProperty<*>,
        value: String
    ) {
        storedValue = value.trim()
    }
}
```

Then:

```kotlin
class User {
    var name: String by TrimmedStringDelegate()
}
```

Now:

```kotlin
val user = User()

user.name = "   Alex   "

println(user.name)
```

prints:

```text
Alex
```

The property itself isn't implementing the trimming.

The delegate is.

---

# Don't worry if that example feels difficult

It should.

This is the first time we're looking at the machinery behind delegated properties.

You don't need to memorize:

```kotlin
getValue
setValue
KProperty
thisRef
```

right now.

The important thing is understanding the architecture:

```text
User.name
   ↓
delegate
   ↓
delegate decides how to read/write
```

The strange-looking generic machinery is just Kotlin's protocol for making that possible.

We'll revisit it later when we reach advanced Kotlin.

---

# Delegates can store the value somewhere else

Here's where this gets even more interesting.

A delegated property doesn't have to store its state in the object in the ordinary way.

For example, a delegate could store values in:

```text
a Map
database
configuration object
another object
preferences
cache
```

Imagine:

```kotlin
class UserSettings {
    private val values = mutableMapOf<String, Any?>()

    // imagine delegated access here
}
```

Kotlin even supports property delegation to maps.

Conceptually:

```text
property
   ↓
map
   ↓
lookup by property name
```

So delegation isn't necessarily about "custom getter/setter syntax."

It's about **giving another component responsibility for property storage/access behavior**.

---

# Why this is such a powerful idea

Consider a normal property:

```kotlin
var username = ""
```

You're saying:

> "This property directly owns its value."

A delegated property:

```kotlin
var username by something
```

says:

> "The actual rules for this property live somewhere else."

That lets you separate:

```text
what the property is called
```

from:

```text
how the property behaves
```

That separation can be extremely useful.

---

# Let's compare custom setters and delegates

### Custom setter

```kotlin
var name = ""
    set(value) {
        field = value.trim()
    }
```

Everything is local.

```text
property
└── custom behavior
```

Very simple.

---

### Delegate

```kotlin
var name by TrimmedStringDelegate()
```

Now:

```text
property
└── delegate
     └── behavior
```

That behavior can potentially be reused by many properties/classes.

For example:

```kotlin
class User {
    var name by TrimmedStringDelegate()
}

class Product {
    var title by TrimmedStringDelegate()
}
```

The same policy is reused.

That's when delegation starts earning its complexity.

---

# There is an important rule here

Don't reach for delegation just because the language provides it.

If this is all you need:

```kotlin
var name = ""
    set(value) {
        field = value.trim()
    }
```

that's perfectly good.

Creating a custom delegate for one property might make the code worse.

Delegation becomes interesting when you have:

```text
the same property behavior
repeated many times
```

or:

```text
complex property behavior that deserves its own component
```

or:

```text
another object already owns the required behavior
```

Those are strong reasons.

---

# Class delegation and property delegation are the same broad philosophy

Now look at both forms.

### Class delegation

```kotlin
class LoggingPrinter(
    printer: Printer
) : Printer by printer
```

Means:

> Let another object provide this interface behavior.

### Property delegation

```kotlin
val config by lazy {
    loadConfiguration()
}
```

Means:

> Let another object provide the behavior for accessing this property.

The common idea is:

```text
"I don't need to implement this responsibility directly.
I'll give that responsibility to another object."
```

That's delegation.

---

# Delegation vs inheritance

Let's put the three major ideas next to each other.

### Inheritance

```kotlin
class Dog : Animal()
```

You create a subtype relationship.

```text
Dog IS-A Animal
```

---

### Composition

```kotlin
class Car(
    private val engine: Engine
)
```

You create a "has-a" relationship.

```text
Car HAS-A Engine
```

---

### Delegation

```kotlin
class LoggingPrinter(
    printer: Printer
) : Printer by printer
```

You say:

```text
LoggingPrinter IS-A Printer
but
Printer behavior is provided by another object
```

So delegation often gives you:

```text
interface compatibility
+
composition
```

without needing inheritance from an implementation class.

That's its sweet spot.

---

# This is why delegation can reduce inheritance problems

Imagine an inheritance tree:

```text
Device
├── Printer
├── Scanner
└── ...
```

Then you need some object that behaves like multiple things.

Inheritance can become awkward because classes can only have one superclass.

Composition/delegation gives you more freedom:

```text
Object
 ├── Printer implementation
 ├── Scanner implementation
 └── other behavior
```

and the object can delegate different responsibilities through different interfaces.

This connects directly back to something you learned in Phase 6:

> **Composition can often be more flexible than inheritance.**

Delegation is one of Kotlin's ways of making that composition practical.

---

# One important limitation

Class delegation is based around interfaces.

You cannot simply say:

```kotlin
class MyClass : SomeConcreteClass() by otherObject
```

and expect Kotlin to delegate an ordinary class inheritance hierarchy for you.

The feature is designed around delegation of interface implementations.

That's an important boundary to remember.

---

# Multiple delegation

This becomes particularly interesting.

Suppose:

```kotlin
interface Printer {
    fun print()
}

interface Scanner {
    fun scan()
}
```

and:

```kotlin
class PrinterImpl : Printer {
    override fun print() {
        println("Printing")
    }
}

class ScannerImpl : Scanner {
    override fun scan() {
        println("Scanning")
    }
}
```

Now you could have:

```kotlin
class MultiFunctionDevice(
    printer: Printer,
    scanner: Scanner
) : Printer by printer, Scanner by scanner
```

So:

```text
MultiFunctionDevice
     |
     ├── Printer behavior → printer
     └── Scanner behavior → scanner
```

That's a beautiful example of composition.

The object doesn't have to manually forward everything.

---

# And now you can override specific pieces

```kotlin
class MultiFunctionDevice(
    printer: Printer,
    scanner: Scanner
) : Printer by printer, Scanner by scanner {

    override fun print() {
        println("Preparing device...")
        printer.print()
    }
}
```

Now:

```text
Printer.print
→ custom implementation

Scanner.scan
→ delegated implementation
```

This gives you very fine-grained control.

---

# Delegation and responsibility

This is the real design lesson.

Suppose:

```text
Object A
Object B
Object C
```

and A shouldn't know how to perform B's responsibility.

Instead of:

```text
A implements everything
```

we can say:

```text
A coordinates
B performs responsibility #1
C performs responsibility #2
```

Delegation gives us a clean mechanism for this.

That's why it's valuable in larger designs.

---

# Let's tie the whole Phase 7 together

Look at what we've learned.

### Enum

```text
"What fixed named values can this be?"
```

### Sealed type

```text
"What fixed structured alternatives can this be?"
```

### Value class

```text
"What meaningful concept does this single value represent?"
```

### Delegation

```text
"Who should actually perform this responsibility?"
```

That's a surprisingly coherent phase.

We've moved from syntax into **modeling decisions**.

---

# I want you to stop and think about this example

Suppose:

```kotlin
interface Repository {
    fun save(data: String)
    fun load(): String
}
```

We have:

```kotlin
class DatabaseRepository : Repository {
    override fun save(data: String) {
        println("Save to database")
    }

    override fun load(): String {
        return "database data"
    }
}
```

Now we want a repository that logs everything.

You have at least two choices.

### Manually forward everything

```kotlin
class LoggingRepository(
    private val repository: Repository
) : Repository {

    override fun save(data: String) {
        println("save() called")
        repository.save(data)
    }

    override fun load(): String {
        println("load() called")
        return repository.load()
    }
}
```

### Delegation

```kotlin
class LoggingRepository(
    private val repository: Repository
) : Repository by repository {

    override fun save(data: String) {
        println("save() called")
        repository.save(data)
    }

    override fun load(): String {
        println("load() called")
        return repository.load()
    }
}
```

In this particular example, because we're overriding both methods, the `by` isn't buying us much.

But imagine `Repository` eventually has 20 methods and we only want to customize 2.

Then delegation becomes very attractive.

---

# Here's the question you should ask when you see `by`

Whenever you see:

```kotlin
Something by somethingElse
```

don't immediately think:

> "`by` syntax."

Ask:

> **What responsibility is being handed to the other object?**

For:

```kotlin
class A : B by b
```

it's interface behavior.

For:

```kotlin
val x by lazy { ... }
```

it's property access behavior.

That mental distinction will save you a lot of confusion.

---

# Your exercises

I want you to do these in increasing difficulty.

### Exercise 1 — understand delegation

Create:

```kotlin
interface Printer {
    fun print(text: String)
}
```

Then:

```kotlin
class ConsolePrinter : Printer
```

Then:

```kotlin
class MyPrinter(
    printer: Printer
) : Printer by printer
```

Create a `ConsolePrinter`, pass it into `MyPrinter`, and call `print()` through `MyPrinter`.

Then explain to yourself:

> Who actually performed the printing?

---

### Exercise 2 — override one method

Give the interface two methods:

```kotlin
interface Printer {
    fun print(text: String)
    fun printError(text: String)
}
```

Make your delegate implement both.

Then make `MyPrinter` delegate the interface but override only:

```kotlin
printError()
```

You should be able to predict which object's implementation runs for each method.

---

### Exercise 3 — multiple delegation

Create:

```kotlin
interface Printer {
    fun print()
}

interface Scanner {
    fun scan()
}
```

Create one class implementing `Printer` and another implementing `Scanner`.

Then make:

```kotlin
class Machine(
    ...
) : Printer by ..., Scanner by ...
```

Now your `Machine` can perform both operations without manually forwarding either one.

---

### Exercise 4 — delegated property

Use:

```kotlin
val answer by lazy {
    println("Calculating...")
    42
}
```

Then access it three times.

Your job is to explain:

> Why does `"Calculating..."` happen only on the first access?

Don't memorize the answer. Connect it to the idea of a property delegating its access behavior.

---

# One final exercise is the most important

Think about these situations:

```text
A class needs behavior another object already provides.

A property should be initialized only when accessed.

Several properties need the same reusable validation behavior.

A class needs to implement two interfaces using two different existing objects.
```

For each one, ask:

> **Could delegation be the natural solution?**

Not:

> "Can Kotlin technically do this?"

That's a much weaker question.

---

# And with that, Phase 7 is essentially complete

We've now reached the end of the major topics we planned for this phase:

```text
Phase 7
│
├── sealed classes/interfaces
│
├── value classes
│
└── delegation
```

But I don't want to immediately jump to Phase 8 just because the checklist is technically complete.

You've now encountered several powerful type/modeling ideas:

```text
enum
sealed
data class
value class
interface
inheritance
composition
delegation
```

The next step is to start seeing **how these choices differ when you are actually designing code**.

Then we'll move into **Phase 8 — Generics**.

And generics are a big step.

You're going from:

```text
"I know what type this value is."
```

to:

> **"I want to write code that works with many types while still preserving type safety."**

That's where Kotlin's type system gets considerably deeper.