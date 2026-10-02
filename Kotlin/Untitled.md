
# Lesson 20 — Kotlin Properties and Initialization

We’ve learned how to create classes, constructors, methods, interfaces, inheritance, and data classes.

Now we need to understand one of Kotlin’s most important ideas:

> **A Kotlin property is more than just a variable stored inside an object.**

In Kotlin, you normally think in terms of **properties**, while Kotlin decides whether that property needs a field underneath, and can attach behavior to reading or writing it.

This single idea explains:

- getters
    
- setters
    
- custom accessors
    
- backing fields
    
- `init`
    
- initialization order
    
- `lateinit`
    
- `lazy`
    

---

# 1. A property is not the same thing as a field

Consider:

```kotlin
class Person {
    var name = "Alex"
}
```

You might initially think:

```text
Person
 └── field called name
```

But Kotlin's model is closer to:

```text
Person
 └── property called name
```

That distinction becomes important because a property can have behavior when you **get** or **set** it.

For a simple property:

```kotlin
var name = "Alex"
```

Kotlin effectively provides access like:

```text
read name  → getter
write name → setter
```

You don't normally write those yourself because Kotlin generates them automatically.

---

# 2. `val` and `var` properties

We've already seen `val` and `var` for ordinary variables.

The same distinction applies to properties.

```kotlin
class Person(
    val name: String,
    var age: Int
)
```

Now:

```kotlin
val person = Person("Alex", 25)

println(person.name)
person.age = 26
```

But:

```kotlin
person.name = "John"
```

doesn't work.

Why?

Because:

```kotlin
val name: String
```

has a readable property but no normal setter.

While:

```kotlin
var age: Int
```

has both a getter and a setter.

Conceptually:

```text
val
 └── get

var
 ├── get
 └── set
```

That distinction will become very useful soon.

---

# 3. Getters

Suppose we have:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
)
```

We could calculate the area using a function:

```kotlin
fun area(): Double {
    return width * height
}
```

But conceptually, `area` isn't really an action.

It's a **property of the rectangle**.

So Kotlin lets us write:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = width * height
}
```

Now:

```kotlin
val rectangle = Rectangle(10.0, 5.0)

println(rectangle.area)
```

Notice:

```kotlin
rectangle.area
```

not:

```kotlin
rectangle.area()
```

That's an important modeling distinction.

### Function

```kotlin
rectangle.calculateArea()
```

sounds like:

> perform an operation

### Property

```kotlin
rectangle.area
```

sounds like:

> this object has a value called area

The actual calculation can still happen every time you read it.

---

# 4. Computed properties

The `area` example is called a **computed property**.

It doesn't need to store an area separately.

```kotlin
val area: Double
    get() = width * height
```

The result is calculated when accessed.

For example:

```kotlin
class Circle(
    val radius: Double
) {
    val diameter: Double
        get() = radius * 2

    val circumference: Double
        get() = 2 * Math.PI * radius

    val area: Double
        get() = Math.PI * radius * radius
}
```

Notice something interesting:

There may be no need to store:

```text
diameter
circumference
area
```

because all three can be derived from `radius`.

This is one reason properties are powerful.

---

# 5. A property does not necessarily have a backing field

This is a very important concept.

Look at:

```kotlin
val area: Double
    get() = width * height
```

Where is `area` stored?

It isn't.

There is no stored value called `area`.

Instead:

```text
read area
   ↓
run getter
   ↓
calculate width × height
   ↓
return result
```

So:

> **Property ≠ necessarily stored field**

A property can be completely computed.

This is one of the places where beginners coming from a language with a strong "field = property" mental model get confused.

---

# 6. Custom setters

Now let's make a property control what happens when someone assigns a value.

Suppose:

```kotlin
class Person {
    var name: String = "Unknown"
}
```

Someone could write:

```kotlin
person.name = "   Alex   "
```

Maybe we want to automatically remove the surrounding whitespace.

We can customize the setter:

```kotlin
class Person {
    var name: String = "Unknown"
        set(value) {
            field = value.trim()
        }
}
```

Now:

```kotlin
val person = Person()

person.name = "   Alex   "

println(person.name)
```

Output:

```text
Alex
```

The flow is:

```text
person.name = "   Alex   "
             ↓
        setter receives value
             ↓
        value = "   Alex   "
             ↓
        value.trim()
             ↓
        stored property becomes "Alex"
```

---

# 7. What is `value`?

Inside a setter, Kotlin gives you a special variable called:

```kotlin
value
```

It represents the value being assigned.

For:

```kotlin
person.name = "Alex"
```

inside the setter:

```kotlin
value
```

is:

```text
"Alex"
```

So:

```kotlin
set(value) {
    field = value.trim()
}
```

means:

> Whenever someone tries to assign a name, take the incoming value, clean it, and store the cleaned result.

---

# 8. What is `field`?

This is one of the most important Kotlin-specific words you'll learn today.

Inside a property accessor, Kotlin provides:

```kotlin
field
```

when the property has a **backing field**.

For example:

```kotlin
var name: String = "Unknown"
    set(value) {
        field = value.trim()
    }
```

Here:

```text
name
 ↓
property

field
 ↓
actual storage used by the property
```

So `field` means approximately:

> the underlying stored value of this property.

---

# 9. Why can't we write `name = value.trim()`?

You might naturally try:

```kotlin
var name: String = "Unknown"
    set(value) {
        name = value.trim()
    }
```

This is wrong.

Why?

Because assigning to `name` invokes the setter again:

```text
name = ...
 ↓
setter
 ↓
name = ...
 ↓
setter
 ↓
name = ...
 ↓
setter
...
```

That creates recursive setter calls.

The same problem can happen with a getter.

This is wrong:

```kotlin
val name: String
    get() = name
```

The getter tries to read `name`, which calls the getter again, which reads `name`, and so on.

Instead, when a backing field exists, use:

```kotlin
field
```

So:

```kotlin
set(value) {
    field = value.trim()
}
```

is correct.

---

# 10. Default getters and setters

You don't always need to write accessors.

This:

```kotlin
var age: Int = 20
```

already has a normal getter and setter.

Conceptually:

```kotlin
var age: Int = 20
    get() = field
    set(value) {
        field = value
    }
```

You generally don't write the explicit version because Kotlin gives you the default behavior automatically.

That is why Kotlin code can remain concise.

---

# 11. `private set`

Here's a very useful pattern.

Suppose we have:

```kotlin
class Player {
    var score: Int = 0
}
```

Anyone can do:

```kotlin
player.score = 999999
```

But perhaps the **player itself** should control how the score changes.

We can write:

```kotlin
class Player {
    var score: Int = 0
        private set

    fun addPoints(points: Int) {
        score += points
    }
}
```

Now external code can do:

```kotlin
println(player.score)
```

but cannot do:

```kotlin
player.score = 999999
```

Instead it must use:

```kotlin
player.addPoints(100)
```

This is encapsulation.

The property is:

```text
public getter
private setter
```

So the outside world can **observe** the state but cannot arbitrarily **change** it.

This pattern is extremely common in real Kotlin code.

---

# 12. Encapsulation becomes much clearer now

Earlier we talked about:

> keeping an object's state controlled by the object itself.

Now we can see how Kotlin properties help us achieve that.

Consider:

```kotlin
class BankAccount {
    var balance: Double = 0.0
        private set

    fun deposit(amount: Double) {
        require(amount > 0)
        balance += amount
    }

    fun withdraw(amount: Double) {
        require(amount > 0)
        require(amount <= balance)

        balance -= amount
    }
}
```

The class controls the rules.

External code cannot simply do:

```kotlin
account.balance = -5000.0
```

The object protects its own invariant.

That idea will become increasingly important when we study architecture.

---

# 13. `init` blocks

Now let's move from properties to object initialization.

Consider:

```kotlin
class User(
    val username: String,
    val age: Int
) {
    init {
        println("User is being created")
    }
}
```

When you write:

```kotlin
val user = User("alex", 25)
```

the `init` block runs as part of object creation.

It's a place where you can perform initialization logic.

For example:

```kotlin
class User(
    val username: String,
    val age: Int
) {
    init {
        require(username.isNotBlank())
        require(age >= 0)
    }
}
```

Now object creation itself enforces valid state.

```kotlin
val user = User("", 25)
```

fails.

That's useful because the object cannot be created in an invalid state.

---

# 14. Property initialization happens before you get an object

Consider:

```kotlin
class Person(
    val name: String
) {
    val greeting = "Hello, $name"

    init {
        println(greeting)
    }
}
```

When:

```kotlin
val person = Person("Alex")
```

the object has to go through initialization before you can meaningfully use it.

A simplified mental model is:

```text
constructor receives arguments
        ↓
properties / init blocks execute
        ↓
object construction completes
        ↓
object is returned
```

But there is an additional subtlety:

## Initialization follows the order written in the class body.

For example:

```kotlin
class Example {
    val first = println("First")

    init {
        println("Init 1")
    }

    val second = println("Second")

    init {
        println("Init 2")
    }
}
```

When an object is created, the initialization happens in textual order.

Conceptually:

```text
First
Init 1
Second
Init 2
```

That's important.

The class body isn't just a random collection of declarations. Initialization logic has an order.

---

# 15. Initialization order can cause problems

Consider:

```kotlin
class Person {
    val greeting = name.uppercase()

    val name = "Alex"
}
```

The idea is dangerous because you're trying to use another property before its initialization has happened.

The general lesson is:

> **Know what has been initialized before you use it.**

When classes get large, initialization dependencies can become difficult to reason about.

This is one reason good class design matters.

---

# 16. `lateinit`

Now we reach an important special case.

Normally, a non-null property must have a value before the object is considered properly initialized.

For example:

```kotlin
class User {
    var name: String
}
```

This isn't acceptable because Kotlin doesn't know what `name` should contain when the object is created.

You could do:

```kotlin
var name: String = ""
```

but sometimes an empty value isn't meaningful.

What if we know:

> "This property will definitely be initialized later, but not during construction."

That's where `lateinit` comes in.

```kotlin
class User {
    lateinit var name: String
}
```

Now:

```kotlin
val user = User()

user.name = "Alex"
```

is valid.

---

# 17. Why would `lateinit` exist?

Imagine an object whose setup happens in stages:

```text
create object
    ↓
perform some setup
    ↓
assign dependency
    ↓
use object
```

For example:

```kotlin
class Service {
    lateinit var repository: Repository

    fun loadData() {
        repository.load()
    }
}
```

The expectation is:

```text
repository
must be assigned
before loadData()
is called
```

So `lateinit` says:

> Kotlin, don't require the initial value right now; I promise to assign it before use.

That is a **developer responsibility**.

---

# 18. What happens if you break that promise?

This:

```kotlin
val service = Service()

service.loadData()
```

before assigning:

```kotlin
service.repository
```

will result in an initialization-related runtime exception.

The important lesson isn't the exact exception name.

It's this:

> `lateinit` moves a safety check from compile time to runtime.

Compare:

```kotlin
val name: String
```

Kotlin says:

> Prove to me that this has a value before the object can be constructed.

With:

```kotlin
lateinit var name: String
```

you're saying:

> Trust me. I'll initialize this later.

That's convenient, but less safe.

---

# 19. `lateinit` vs nullable properties

You may wonder:

> Why not just use `String?`?

For example:

```kotlin
var name: String? = null
```

That means:

```text
"there may legitimately be no name"
```

Whereas:

```kotlin
lateinit var name: String
```

means:

```text
"there should always eventually be a name;
it simply isn't available yet"
```

These are conceptually different.

That's a very important distinction.

### Nullable

```kotlin
String?
```

means:

> absence is part of the state.

### `lateinit`

```kotlin
lateinit var name: String
```

means:

> the value is required, but initialization is delayed.

Don't use nullable types merely because you haven't decided how to initialize something.

And don't use `lateinit` merely because dealing with `null` feels inconvenient.

---

# 20. `lazy`

Now we have another interesting initialization strategy.

Suppose calculating a value is expensive:

```kotlin
fun loadConfiguration(): Configuration {
    println("Loading...")
    return Configuration()
}
```

We could do:

```kotlin
val config = loadConfiguration()
```

But then it loads immediately when the object is initialized.

Maybe we don't need the configuration immediately.

Instead:

```kotlin
val config by lazy {
    loadConfiguration()
}
```

Now the computation happens only when `config` is first accessed.

For example:

```kotlin
class Application {
    val config by lazy {
        loadConfiguration()
    }
}
```

Then:

```kotlin
val app = Application()
```

doesn't necessarily perform the configuration loading yet.

When you do:

```kotlin
println(app.config)
```

the lazy block runs.

---

# 21. How `lazy` behaves

Think of it like:

```text
property accessed
       ↓
has it been initialized?
   ↙           ↘
 no            yes
 ↓              ↓
calculate       return
store result    stored result
 ↓
return result
```

So the first access triggers initialization.

Later accesses reuse the already-created value.

For example:

```kotlin
val number by lazy {
    println("Calculating...")
    42
}
```

Then:

```kotlin
println(number)
println(number)
println(number)
```

The calculation block runs once, not three times.

---

# 22. `lateinit` and `lazy` are almost opposites conceptually

This is worth remembering.

### `lateinit`

You say:

> "I will provide this value later."

```kotlin
lateinit var service: Service
```

The programmer explicitly assigns it.

---

### `lazy`

You say:

> "Create this value automatically when it is first needed."

```kotlin
val service by lazy {
    createService()
}
```

The property itself controls initialization.

So:

```text
lateinit
→ deferred assignment

lazy
→ deferred computation
```

That's a very useful mental model.

---

# 23. Stored vs computed vs deferred properties

We now have three important kinds of properties.

### Stored

```kotlin
var name = "Alex"
```

The value is stored.

### Computed

```kotlin
val fullName: String
    get() = "$firstName $lastName"
```

The value is derived when accessed.

### Lazily initialized

```kotlin
val config by lazy {
    loadConfiguration()
}
```

The value is computed only when first needed and then retained.

Those are fundamentally different behaviors even though they all look like properties at the usage site:

```kotlin
object.name
object.fullName
object.config
```

This is one of the elegant parts of Kotlin.

---

# 24. A complete example

Let's put several ideas together.

```kotlin
class BankAccount(
    val accountNumber: String,
    initialBalance: Double
) {

    var balance: Double = initialBalance
        private set

    val isEmpty: Boolean
        get() = balance == 0.0

    lateinit var ownerName: String

    val accountLabel by lazy {
        "Account-$accountNumber"
    }

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
}
```

There is a lot going on here.

### `accountNumber`

```kotlin
val accountNumber: String
```

Stored, readable, not reassigned.

### `initialBalance`

```kotlin
initialBalance: Double
```

It's a constructor parameter, but not a property.

This is an important distinction we learned earlier.

It exists during construction but isn't automatically accessible later as:

```kotlin
account.initialBalance
```

### `balance`

```kotlin
var balance: Double = initialBalance
    private set
```

Stored property.

Anyone can read it.

Only the class can change it directly.

### `isEmpty`

```kotlin
val isEmpty: Boolean
    get() = balance == 0.0
```

Computed property.

Nothing needs to separately store `true` or `false`.

### `ownerName`

```kotlin
lateinit var ownerName: String
```

Required non-null value, but initialized later.

### `accountLabel`

```kotlin
val accountLabel by lazy {
    "Account-$accountNumber"
}
```

Created only when first accessed.

### `init`

```kotlin
init {
    require(initialBalance >= 0)
}
```

Ensures the object begins in a valid state.

That's a fairly good miniature example of Kotlin-style object modeling.

---

# 25. A subtle design question: property or function?

This is an important skill, not just syntax.

Suppose we have:

```kotlin
fun calculateAge(): Int
```

versus:

```kotlin
val age: Int
    get() = ...
```

When should something be a property?

A useful intuition is:

### Property

Use when it conceptually behaves like a characteristic/value of the object.

```kotlin
user.name
user.age
rectangle.area
account.balance
```

### Function

Use when it represents an action, operation, or something that reads naturally as something being done.

```kotlin
account.withdraw(100.0)
user.sendMessage()
file.delete()
```

This isn't an absolute law.

But it's a useful modeling instinct.

---

# 26. Another subtle point: `val` does not mean the value can never change internally

Consider:

```kotlin
class Person {
    val name: String
        get() = generateName()
}
```

This is a `val`, but that doesn't necessarily mean repeated reads must produce the same result.

The key idea is:

> `val` means the property cannot be assigned to through that property reference.

It does **not** universally mean:

> "calling the getter must always produce the exact same value."

Similarly, a `val` can refer to a mutable object:

```kotlin
class Team {
    val players = mutableListOf<String>()
}
```

You can't do:

```kotlin
team.players = mutableListOf()
```

but you can do:

```kotlin
team.players.add("Alex")
```

So again:

```text
val
≠
deep immutability
```

We've seen this idea before with collections; now you can see how it applies to properties too.

---

# 27. The mental model I want you to keep

At this point, don't memorize a pile of syntax.

Think of a Kotlin property as something with a **public interface** that can have different implementation strategies.

For example:

```text
              PROPERTY
                  │
       ┌──────────┼──────────┐
       │          │          │
    stored     computed    deferred
       │          │          │
     field      getter      lazy
```

And for mutable properties:

```text
var
├── getter
└── setter
       │
       └── may use backing field
```

And initialization strategies:

```text
normal initialization
lateinit
lazy
```

These are tools for expressing **how an object's state is represented and controlled**.

That is much more important than memorizing individual keywords.

---

# 28. Practice

Don't just read these. Predict what happens before running them.

### Exercise 1 — Custom getter

Complete this:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = ???
}
```

Make `area` return the rectangle's area.

---

### Exercise 2 — Custom setter

Create:

```kotlin
class User {
    var username: String = ""
        set(value) {
            // store a trimmed version
        }
}
```

So:

```kotlin
user.username = "   alex   "
println(user.username)
```

prints:

```text
alex
```

---

### Exercise 3 — Controlled mutation

Create a class:

```kotlin
class Counter
```

with:

```kotlin
var value: Int
```

but external code should only be able to **read** `value`.

The class should provide:

```kotlin
increment()
decrement()
```

to modify it.

Hint:

```kotlin
private set
```

---

### Exercise 4 — Initialization

What does this print?

```kotlin
class Example {

    val first = println("First")

    init {
        println("Init")
    }

    val second = println("Second")
}
```

Don't run it immediately. Predict first.

---

### Exercise 5 — `lateinit` vs `lazy`

For each situation, decide which idea fits better:

**A**

> "This service must be assigned by the framework later."

**B**

> "This expensive object should be created only when someone first asks for it."

The answers should make sense conceptually, not just because you memorized the keywords.

---

# One challenge

Build this class:

```kotlin
class Student(
    val name: String,
    initialAge: Int
)
```

Give it:

```text
age
isAdult
grade
```

with these rules:

- `age` can be changed, but must never become negative.
    
- `isAdult` should be calculated from `age`.
    
- `grade` can be read from outside but changed only inside the class.
    
- The constructor should reject a negative starting age.
    

Try to use:

```text
custom setter
computed property
private set
init
```

This single exercise will tie today's concepts together.

---

## What comes next

After properties and initialization, we'll finish the remaining core pieces of Kotlin classes:

**visibility → object declarations → companion objects → nested/inner classes → enums**

Then we'll move into **sealed classes/interfaces, value classes, and delegation**, which is where Phase 7 really begins.

And importantly, you now have enough understanding to see _why_ those features exist rather than just learning more keywords.