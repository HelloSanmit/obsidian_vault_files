
# Lesson 22 — Visibility, `object`, and `companion object`

Today we're going to answer a different question from the last lesson.

Last time we asked:

> **How does a property control its own state?**

Now we're asking:

> **Who is allowed to see or use something, and does it belong to an individual object or to the class itself?**

These ideas are closely connected to encapsulation and object design.

---

# 1. Visibility: who can access this?

Consider:

```kotlin
class BankAccount {
    var balance = 0.0
}
```

Any code that can access the `BankAccount` object can normally access `balance`.

Sometimes that's exactly what we want.

But sometimes we want to restrict access.

For example:

```kotlin
class BankAccount {
    private var balance = 0.0
}
```

Now `balance` is hidden from code outside the class.

This is **visibility control**.

---

# 2. The four visibility modifiers

Kotlin has four main visibility modifiers:

```text
public
private
protected
internal
```

Let's understand what each one means rather than memorizing definitions.

---

## `public`

This is the normal default.

```kotlin
class Person {
    public val name = "Alex"
}
```

You could simply write:

```kotlin
class Person {
    val name = "Alex"
}
```

because `public` is the default visibility in Kotlin.

That means the declaration is accessible wherever the relevant containing declaration is accessible.

So this:

```kotlin
class Person(
    val name: String
)
```

has a public `name` property unless you explicitly say otherwise.

---

# 3. `private`

`private` means:

> This should only be accessible within its intended scope.

For a class member:

```kotlin
class BankAccount {
    private var balance = 0.0

    fun deposit(amount: Double) {
        balance += amount
    }
}
```

Outside:

```kotlin
val account = BankAccount()

account.deposit(100.0)
```

works.

But:

```kotlin
account.balance
```

doesn't.

The important design idea is:

> The class controls the state; users interact through the class's public interface.

That's encapsulation again.

---

# 4. `private` isn't only for properties

You can make methods private too:

```kotlin
class BankAccount {

    private fun logTransaction() {
        println("Transaction recorded")
    }

    fun deposit(amount: Double) {
        // ...
        logTransaction()
    }
}
```

Outside code cannot directly call:

```kotlin
account.logTransaction()
```

This lets a class expose a small public surface while keeping implementation details hidden.

---

# 5. `protected`

`protected` is associated with inheritance.

Suppose:

```kotlin
open class Animal {
    protected fun breathe() {
        println("Breathing")
    }
}
```

Then a subclass can use it:

```kotlin
class Dog : Animal() {
    fun live() {
        breathe()
    }
}
```

But ordinary outside code cannot simply do:

```kotlin
val dog = Dog()
dog.breathe()
```

So a useful mental model is:

```text
public
→ broadly accessible

private
→ restricted to its allowed scope

protected
→ available to the class and subclasses

internal
→ available within the same module
```

We'll come back to `internal` because "module" needs a little explanation.

---

# 6. What is `internal`?

Suppose you're building a project:

```text
MyApplication
```

and your code is split into many packages/files.

Sometimes you want something available throughout **your application/library module**, but not exposed as part of the module's public API.

That's where:

```kotlin
internal
```

comes in.

For example:

```kotlin
internal class DatabaseHelper
```

means code inside the same Kotlin module can use it, while code outside that module cannot access it as a public API.

This becomes much more meaningful once we study Gradle and multi-module projects.

For now, remember:

> **`internal` is module-level visibility.**

---

# 7. `private set` was a special case

We've already seen:

```kotlin
var score: Int = 0
    private set
```

This is a nice combination.

The property itself is accessible, but its setter isn't.

So:

```text
outside class
    ↓
read score       ✅
write score      ❌

inside class
    ↓
read score       ✅
write score      ✅
```

This is an example of controlling the visibility of an accessor rather than the whole property.

---

# 8. Why visibility matters

Suppose you write:

```kotlin
class Car {
    var fuel = 0
    var engineTemperature = 0
    var engineHealth = 100
    var doorCount = 4
    var speed = 0
}
```

Everything is publicly exposed.

Now any other code can do:

```kotlin
car.speed = 9000
car.engineHealth = -500
car.engineTemperature = -1000
```

The class has no control over its own state.

Instead:

```kotlin
class Car {
    var speed = 0
        private set

    private var engineHealth = 100

    fun accelerate() {
        speed += 10
    }
}
```

Now the class controls how its state changes.

This is why visibility isn't just about hiding things.

It's about **protecting boundaries**.

---

# 9. Now let's change perspective: what if there is only one object?

So far we've done:

```kotlin
class User
```

and created instances:

```kotlin
val user1 = User()
val user2 = User()
val user3 = User()
```

Each is a separate object.

But sometimes you don't want three objects.

You want exactly one shared object.

For that, Kotlin has:

```kotlin
object
```

---

# 10. `object` declaration

Consider:

```kotlin
object Logger {
    fun log(message: String) {
        println(message)
    }
}
```

Notice that we didn't write:

```kotlin
class Logger
```

and we didn't manually create:

```kotlin
Logger()
```

Instead:

```kotlin
object Logger
```

declares a singleton-style object.

You can use it directly:

```kotlin
Logger.log("Application started")
```

There isn't a normal constructor call like:

```kotlin
Logger()
```

for an object declaration.

---

# 11. Why would we want one object?

Suppose your application has one shared configuration manager:

```kotlin
object Configuration {
    val environment = "development"
}
```

You can access it:

```kotlin
println(Configuration.environment)
```

There's one shared object representing that thing.

This can make sense for genuinely unique entities or stateless/shared utilities.

---

# 12. `object` versus `class`

Compare:

```kotlin
class Logger
```

with:

```kotlin
object Logger
```

A `class` is a blueprint from which you can create multiple instances:

```text
class Logger
   ↓
Logger #1
Logger #2
Logger #3
```

An `object` declaration directly gives you a single object:

```text
object Logger
      ↓
   Logger
```

So:

```text
class
→ describes a type of object

object
→ declares one object
```

That's a fundamental distinction.

---

# 13. Objects can have state

An `object` isn't restricted to functions.

For example:

```kotlin
object Counter {
    var value = 0

    fun increment() {
        value++
    }
}
```

Then:

```kotlin
Counter.increment()
Counter.increment()

println(Counter.value)
```

prints:

```text
2
```

That state belongs to the one `Counter` object.

There aren't separate counters unless you create a separate design for them.

---

# 14. But don't think "object = global variable"

You may see people describing `object` as "Kotlin's singleton."

That is useful, but incomplete.

A singleton-like object can be appropriate when the domain really has one shared instance.

But if you make everything an `object`, you can end up with:

- shared mutable state
    
- hidden dependencies
    
- difficult testing
    
- unclear ownership
    

So the lesson isn't:

> "Use `object` everywhere."

It's:

> **Use an object declaration when a single shared object is actually part of the design.**

---

# 15. `object` is also an expression

Kotlin has another use:

```kotlin
val logger = object {
    fun log() {
        println("Logging")
    }
}
```

This creates an **anonymous object**.

Anonymous objects are useful when you need a one-off object rather than a named class.

We won't go deeply into anonymous objects yet, but I want you to recognize that:

```kotlin
object
```

isn't limited to singleton declarations.

It can also create an object expression.

---

# 16. Now the confusing part: `companion object`

You may remember Java's `static`.

Kotlin doesn't have a `static` keyword for class members in the same way.

Instead, Kotlin provides:

```kotlin
companion object
```

For example:

```kotlin
class User {
    companion object {
        fun createGuest(): User {
            return User()
        }
    }
}
```

Now you can write:

```kotlin
val guest = User.createGuest()
```

This _looks_ like a static method.

But conceptually Kotlin is doing something slightly different.

---

# 17. A `companion object` is an object associated with a class

Think about:

```kotlin
class User {
    companion object {
        fun createGuest(): User {
            return User()
        }
    }
}
```

The class:

```text
User
```

has an associated object:

```text
User
└── companion object
```

And that companion object contains:

```text
createGuest()
```

So:

```kotlin
User.createGuest()
```

is really class-associated access to functionality stored on the companion object.

This is why it isn't exactly Java-style `static`.

---

# 18. Why is `companion object` useful?

One common use is factory methods.

Suppose constructing something involves a rule:

```kotlin
class User private constructor(
    val name: String
) {
    companion object {
        fun guest(): User {
            return User("Guest")
        }

        fun fromName(name: String): User {
            return User(name.trim())
        }
    }
}
```

Now:

```kotlin
val guest = User.guest()
val user = User.fromName("   Alex   ")
```

The companion object provides class-associated ways to create users.

---

# 19. Companion objects can have properties too

For example:

```kotlin
class MathHelper {
    companion object {
        const val PI = 3.14159
    }
}
```

Then:

```kotlin
println(MathHelper.PI)
```

There is no need to create a `MathHelper` instance just to get `PI`.

The companion object is associated with the class itself.

---

# 20. `companion object` is an actual object

This is the deeper idea.

Suppose:

```kotlin
class User {
    companion object {
        fun create() = User()
    }
}
```

That companion is itself an object.

It can implement interfaces too.

For example:

```kotlin
interface Factory<T> {
    fun create(): T
}

class User {
    companion object : Factory<User> {
        override fun create(): User {
            return User()
        }
    }
}
```

Now the companion object isn't just a place to dump "static-looking" functions.

It's an actual object participating in Kotlin's object model.

That's a very Kotlin way of thinking.

---

# 21. Named companion objects

You don't have to leave the companion unnamed.

You can write:

```kotlin
class User {
    companion object Factory {
        fun create(): User {
            return User()
        }
    }
}
```

Now the companion object's internal name is:

```text
Factory
```

This is useful in situations where you want to refer to the companion itself as an object.

For normal use:

```kotlin
User.create()
```

still works.

---

# 22. `object` vs `companion object`

This is one of the places beginners frequently confuse things.

### `object`

```kotlin
object Database
```

means:

> Declare a standalone singleton-like object named `Database`.

### `companion object`

```kotlin
class User {
    companion object {
        ...
    }
}
```

means:

> Declare an object associated with the `User` class.

So:

```text
object
→ standalone object

companion object
→ object associated with a class
```

---

# 23. Instance members vs companion members

Consider:

```kotlin
class User(
    val name: String
) {
    fun sayHello() {
        println("Hello, $name")
    }

    companion object {
        fun guest(): User {
            return User("Guest")
        }
    }
}
```

Now:

```kotlin
val user = User("Alex")

user.sayHello()
```

`sa­yHello()` belongs to the **User instance**.

It needs a particular user.

But:

```kotlin
User.guest()
```

doesn't need an existing user.

It belongs to the **companion object associated with the class**.

This gives us an important distinction:

```text
instance member
→ needs a particular object

companion member
→ associated with the class
```

---

# 24. Why can't a companion object directly access instance properties?

Consider:

```kotlin
class User(
    val name: String
) {
    companion object {
        fun printName() {
            println(name)
        }
    }
}
```

What's wrong?

Which `name`?

There could be:

```text
User("Alex")
User("Sam")
User("Priya")
```

Each instance has its own `name`.

The companion object doesn't represent one particular user instance.

Therefore it cannot simply say:

```text
name
```

because there is no instance attached to that access.

This is the same fundamental distinction:

```text
instance-specific state
vs
class-associated functionality
```

---

# 25. A useful analogy

Think of a `class` as a design for houses.

```text
House
```

Each actual house has:

```text
address
number of rooms
owner
```

Those belong to individual instances.

But perhaps the building company has a general function:

```text
estimateConstructionCost()
```

which is associated with the concept of a house rather than one particular house.

In Kotlin, you can model class-associated functionality using a companion object.

It's not a perfect analogy, but it helps with the distinction.

---

# 26. How today's topics connect

We've now added several layers to our object model.

```text
class
│
├── instance properties
│      ├── val
│      └── var
│
├── instance methods
│
├── visibility
│      ├── public
│      ├── private
│      ├── protected
│      └── internal
│
└── companion object
        ├── class-associated functions
        └── class-associated properties
```

And separately:

```text
object
→ one standalone object
```

This is starting to give you a much clearer picture of Kotlin's design.

---

# 27. A complete example

Let's put these ideas together.

```kotlin
class BankAccount(
    val accountNumber: String,
    initialBalance: Double
) {

    var balance: Double = initialBalance
        private set

    init {
        require(initialBalance >= 0)
    }

    fun deposit(amount: Double) {
        require(amount > 0)
        balance += amount
    }

    fun withdraw(amount: Double) {
        require(amount > 0)
        require(amount <= balance)

        balance -= amount
    }

    companion object {
        fun empty(accountNumber: String): BankAccount {
            return BankAccount(accountNumber, 0.0)
        }
    }
}
```

Now:

```kotlin
val account = BankAccount.empty("12345")
```

The factory-like function belongs to the companion object.

Then:

```kotlin
account.deposit(100.0)
```

is an instance operation.

And:

```kotlin
println(account.balance)
```

reads an instance property.

But:

```kotlin
account.balance = -50.0
```

is forbidden because the setter is private.

You can now see several concepts cooperating:

```text
companion object
      ↓
create the object

instance
      ↓
owns the state

private setter
      ↓
protect the state

methods
      ↓
control mutations
```

That's actual object design rather than isolated syntax.

---

# 28. One important distinction: `object` doesn't replace classes

You might be tempted to think:

> "If `object` gives me an object, why ever use a class?"

Because classes are useful when you need **multiple independent instances**.

For example:

```kotlin
class User(
    val name: String
)
```

lets you create:

```kotlin
val alex = User("Alex")
val sam = User("Sam")
val priya = User("Priya")
```

Each has separate state.

An `object` gives you one shared instance:

```kotlin
object AppConfig
```

So:

```text
class
→ potentially many independent objects

object
→ one shared object
```

---

# 29. Don't memorize visibility mechanically

Here's a better way to think.

When designing something, ask:

### Who should know this exists?

Maybe:

```text
everyone → public
class internals → private
subclasses → protected
my module → internal
```

And then ask:

### Who should be allowed to change it?

That's where patterns such as:

```kotlin
var score: Int = 0
    private set
```

become useful.

Good Kotlin code often has a **small public surface** and a more controlled internal implementation.

---

# Practice

Before moving on, try these mentally.

### 1

What is the difference between:

```kotlin
class Logger
```

and:

```kotlin
object Logger
```

?

---

### 2

Why does this work?

```kotlin
class User {
    companion object {
        fun guest() = User()
    }
}

val user = User.guest()
```

but the companion object can't directly access an instance property such as:

```kotlin
class User(
    val name: String
)
```

?

---

### 3

What can outside code do here?

```kotlin
class Player {
    var score = 0
        private set
}
```

Can it:

```text
read score?
change score?
```

---

### 4

What visibility would you choose conceptually for something that should be usable only by subclasses?

```text
public
private
protected
internal
```

---

# Small coding exercise

Create:

```kotlin
class Counter
```

with:

```text
value
increment()
reset()
```

Requirements:

- `value` starts at `0`
    
- outside code can read `value`
    
- outside code cannot directly change `value`
    
- `increment()` increases it
    
- `reset()` sets it back to zero
    

Then create:

```kotlin
object AppInfo
```

containing:

```text
applicationName
version
```

and print them.

Finally, add a `companion object` to `Counter` with:

```kotlin
fun create(): Counter
```

that returns a new `Counter`.

The important part isn't how quickly you finish it. Try to explain to yourself **which pieces belong to an instance, which belong to the class-associated companion, and which are hidden by visibility**.

Next we'll cover **enums and nested/inner classes**, and then we'll be ready to enter the more powerful modeling features of Phase 7: **sealed classes/interfaces, value classes, and delegation**.