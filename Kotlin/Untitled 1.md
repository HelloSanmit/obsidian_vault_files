
# Lesson 20 — Kotlin Properties: Getters, Setters, and Backing Fields

We're going to slow down here.

Rather than introducing `lateinit`, `lazy`, `object`, `companion object`, and several other features all at once, today's lesson will focus on one idea:

> **What exactly is a Kotlin property, and what happens when you read or write one?**

Once this becomes intuitive, several later Kotlin features become much easier.

---

## Start with something familiar

You already know this:

```kotlin
class Person(
    val name: String,
    var age: Int
)
```

And you can do:

```kotlin
val person = Person("Alex", 20)

println(person.name)

person.age = 21
```

At first glance, this looks extremely simple:

```text
person.name  → get the name
person.age   → get the age
person.age = 21 → change the age
```

But there's an important question hiding underneath:

### What actually happens when you write `person.age`?

In Kotlin, you're interacting with a **property**.

A property can control what happens when its value is read or changed.

That's today's subject.

---

# 1. Property vs variable

Outside a class:

```kotlin
val name = "Alex"
```

is a local variable.

Inside a class:

```kotlin
class Person {
    val name = "Alex"
}
```

`name` is a **property** of the object.

That distinction matters because properties can have behavior.

For example:

```kotlin
class Person {
    var name = "Alex"
}
```

You can write:

```kotlin
person.name
```

and:

```kotlin
person.name = "Sam"
```

Kotlin treats those as property access operations.

You don't normally have to think about getters and setters yet because Kotlin provides them automatically.

---

# 2. Every `var` property has a getter and setter

Take:

```kotlin
class Person {
    var age = 20
}
```

Conceptually, you can think of it as:

```text
age
 ├── getter → read the value
 └── setter → change the value
```

So:

```kotlin
println(person.age)
```

means:

```text
read age
→ getter
→ result
```

And:

```kotlin
person.age = 25
```

means:

```text
write age
→ setter
→ store new value
```

Kotlin writes the boring default behavior for you.

---

# 3. What about `val`?

For:

```kotlin
class Person {
    val name = "Alex"
}
```

you can read it:

```kotlin
println(person.name)
```

but you can't assign a new value:

```kotlin
person.name = "Sam" // error
```

Conceptually:

```text
val
 └── getter
```

while:

```text
var
 ├── getter
 └── setter
```

This is the same `val` vs `var` idea you already learned with local variables, now applied to object properties.

---

# 4. Let's make a getter ourselves

Suppose we have:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
)
```

A rectangle has an area.

We could write:

```kotlin
fun area(): Double {
    return width * height
}
```

and use:

```kotlin
rectangle.area()
```

But `area` isn't really an action.

It's a characteristic of the rectangle.

So Kotlin lets us express it as a property:

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
println(rectangle.area)
```

Notice the difference:

```kotlin
rectangle.area
```

rather than:

```kotlin
rectangle.area()
```

---

# 5. What is the getter doing?

This:

```kotlin
val area: Double
    get() = width * height
```

means approximately:

> Whenever somebody reads `area`, calculate `width * height` and return the result.

For example:

```kotlin
val rectangle = Rectangle(10.0, 5.0)

println(rectangle.area)
```

The mental process is:

```text
rectangle.area
      ↓
run getter
      ↓
width * height
      ↓
10.0 * 5.0
      ↓
50.0
```

The important part is that `area` doesn't have to be stored.

---

# 6. A computed property may have no stored value

Look at:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = width * height
}
```

Where is `area` stored?

Nowhere.

You have:

```text
width     → stored
height    → stored
area      → calculated
```

You can think of it like:

```text
property
   ↓
"How do I get this value?"
   ↓
run code
```

This is called a **computed property**.

That is a very useful concept.

---

# 7. Property or function?

This raises an important design question.

Suppose we need:

```kotlin
fun calculateArea(): Double
```

or:

```kotlin
val area: Double
    get() = width * height
```

They're both capable of calculating the same thing.

So how do we think about the choice?

A property usually reads naturally when we're describing a characteristic:

```kotlin
person.name
person.age
rectangle.area
account.balance
```

A function often reads naturally when we're asking an object to perform an operation:

```kotlin
account.withdraw(100.0)
person.sendMessage()
file.delete()
```

This is not a rigid rule, but it's a very useful modeling instinct.

---

# 8. Now let's look at setters

Suppose we write:

```kotlin
class Person {
    var name = "Unknown"
}
```

We can do:

```kotlin
person.name = "   Alex   "
```

But maybe we don't want spaces at the beginning and end.

We can define a custom setter:

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

prints:

```text
Alex
```

---

# 9. The mysterious `value`

Inside this:

```kotlin
set(value) {
    field = value.trim()
}
```

what is `value`?

It's the new value somebody is trying to assign.

When you write:

```kotlin
person.name = "   Alex   "
```

the setter effectively receives:

```text
value = "   Alex   "
```

Then:

```kotlin
value.trim()
```

produces:

```text
"Alex"
```

and that gets stored.

So:

```text
person.name = "   Alex   "
       ↓
setter
       ↓
value = "   Alex   "
       ↓
trim()
       ↓
store "Alex"
```

---

# 10. Now the more interesting part: `field`

You might ask:

> Why are we writing `field` instead of `name`?

Because `field` refers to the property's underlying storage.

Consider:

```kotlin
class Person {
    var name: String = "Unknown"
        set(value) {
            field = value.trim()
        }
}
```

Think of it as:

```text
name = the property
field = the property's stored value
```

So:

```kotlin
field = value.trim()
```

means:

> Store the cleaned value in the property's underlying storage.

---

# 11. Why `name = ...` would be wrong

Suppose you tried:

```kotlin
class Person {
    var name: String = "Unknown"
        set(value) {
            name = value.trim()
        }
}
```

At first this seems reasonable.

But look at what happens:

```text
name = ...
   ↓
setter runs
   ↓
name = ...
   ↓
setter runs
   ↓
name = ...
   ↓
setter runs
   ↓
...
```

The setter keeps calling itself.

That's recursion.

So inside a custom accessor, when you want to work with the actual stored value, you use:

```kotlin
field
```

not the property itself.

---

# 12. Getters can have the same issue

This would be a problem:

```kotlin
val name: String
    get() = name
```

Why?

Reading `name` causes the getter to run.

The getter tries to read `name`.

That causes the getter to run again.

And so on.

When there's a backing field, the stored value is accessed with:

```kotlin
field
```

For example, a getter could conceptually be:

```kotlin
get() = field
```

although you normally don't need to write that because it's already the default behavior.

---

# 13. Kotlin writes the boring accessor for you

This:

```kotlin
var age = 20
```

already behaves like a normal property with storage and default access.

You don't need to write:

```kotlin
var age = 20
    get() = field
    set(value) {
        field = value
    }
```

The language gives you that behavior automatically.

We only write custom accessors when we actually want different behavior.

That distinction is worth remembering:

> **Don't write a custom getter/setter just because you can.**

Use one when the property needs special behavior.

---

# 14. Validation with a setter

Here's a more useful example.

Suppose age should never be negative.

```kotlin
class Person {
    var age: Int = 0
        set(value) {
            require(value >= 0)
            field = value
        }
}
```

Now:

```kotlin
val person = Person()

person.age = 25
```

works.

But:

```kotlin
person.age = -10
```

fails.

The setter becomes a gatekeeper.

```text
new value
   ↓
setter
   ↓
is it valid?
 ┌───────┴───────┐
yes              no
 ↓                ↓
store it        reject it
```

This is another way properties help with **encapsulation**.

---

# 15. Setters can enforce invariants

An **invariant** is a condition that should remain true for an object.

For example:

```text
age >= 0
```

might be an invariant.

So:

```kotlin
class Person {
    var age: Int = 0
        set(value) {
            require(value >= 0)
            field = value
        }
}
```

means:

> The class refuses to let `age` become invalid.

That's much better than letting every caller remember the rule.

---

# 16. `private set`

Now imagine:

```kotlin
class Player {
    var score: Int = 0
}
```

Anyone can do:

```kotlin
player.score = 1000000
```

Maybe that's not what we want.

Suppose the player should only gain points through methods.

We can write:

```kotlin
class Player {
    var score: Int = 0
        private set

    fun addPoints(points: Int) {
        require(points > 0)
        score += points
    }
}
```

Now outside the class:

```kotlin
println(player.score)
```

works.

But:

```kotlin
player.score = 500
```

doesn't.

The outside world can read the property, but only the class can modify it.

---

# 17. Why this is better than making the whole property private

We could instead write:

```kotlin
private var score = 0
```

But then outside code couldn't read it either.

With:

```kotlin
var score = 0
    private set
```

we get:

```text
outside:
    can read
    cannot write

inside:
    can read
    can write
```

That's often exactly what we want.

This pattern is worth recognizing immediately when you see it in real Kotlin code.

---

# 18. Let's connect this back to OOP

Earlier we said:

> Encapsulation means an object controls its own state.

Now we have a concrete mechanism for doing that.

Imagine a bank account:

```kotlin
class BankAccount(
    val accountNumber: String
) {
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

The user of the class says:

```kotlin
account.deposit(100.0)
account.withdraw(20.0)
```

rather than directly manipulating:

```kotlin
account.balance = -999999.0
```

The class owns the rules.

This is not merely Kotlin syntax.

It's object-oriented design.

---

# 19. A subtle but important distinction

Look carefully at this:

```kotlin
class User(
    val name: String,
    initialAge: Int
)
```

There are two different things here:

```text
name
initialAge
```

`name` is a property because it has `val` in the constructor.

`initialAge` is only a constructor parameter.

So:

```kotlin
val user = User("Alex", 20)
```

Later:

```kotlin
println(user.name)
```

works.

But:

```kotlin
println(user.initialAge)
```

does not.

Why?

Because `initialAge` was only needed while constructing the object.

This distinction is going to matter a lot when we discuss initialization.

---

# 20. Let's introduce initialization carefully

When you create an object:

```kotlin
val person = Person("Alex", 20)
```

the object doesn't magically appear fully formed.

Kotlin has to initialize its properties.

For example:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

The constructor provides the initial data needed by the object.

You can also have initialization logic inside the class:

```kotlin
class Person(
    val name: String,
    val age: Int
) {
    init {
        println("Creating Person")
    }
}
```

When you do:

```kotlin
val person = Person("Alex", 20)
```

the `init` block executes as part of construction.

---

# 21. `init` is not a function you call yourself

You don't do:

```kotlin
person.init()
```

That's not how it works.

Instead:

```text
construct object
      ↓
initialization happens
      ↓
object becomes available
```

So:

```kotlin
class Person(
    val name: String
) {
    init {
        println("Hello $name")
    }
}
```

and:

```kotlin
val person = Person("Alex")
```

will run the `init` logic automatically.

---

# 22. `init` is especially useful for validation

Consider:

```kotlin
class Person(
    val name: String,
    val age: Int
) {
    init {
        require(name.isNotBlank())
        require(age >= 0)
    }
}
```

This says:

> A `Person` cannot be successfully created unless these conditions are satisfied.

That is powerful.

We are effectively saying:

```text
valid input
   ↓
object can exist

invalid input
   ↓
object creation fails
```

The object protects its own validity from the beginning.

---

# 23. Initialization order

Here's something worth paying attention to.

```kotlin
class Example {

    val first = println("First")

    init {
        println("Init")
    }

    val second = println("Second")
}
```

What do you predict happens?

It prints:

```text
First
Init
Second
```

Why?

Because initialization proceeds in the order in which the property initializers and `init` blocks appear in the class body.

The class body has an execution order during construction.

This becomes increasingly important once classes get more complicated.

---

# 24. A dangerous ordering example

Look at this:

```kotlin
class Person {

    val greeting = name.uppercase()

    val name = "Alex"
}
```

This creates a problem because `greeting` is trying to use `name` before `name` has been initialized.

The lesson is:

> **Initialization order matters.**

Don't think of all class properties as if they somehow become available simultaneously.

---

# 25. Let's pause here

Notice how much we've covered without even reaching:

```kotlin
lateinit
lazy
object
companion object
enum
sealed class
```

That's intentional.

I don't want you memorizing ten keywords.

I want the core model to become natural:

```text
A class has properties.
A property may store data.
A property may compute data.
A property may control writes.
Initialization establishes the object's initial state.
Accessors can enforce rules around that state.
```

Once that is solid, the next features will have a reason to exist.

---

# Try these before looking at the answers

## Exercise 1

What happens here?

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = width * height
}
```

Answer these questions yourself:

1. Is `area` stored?
    
2. When is `width * height` calculated?
    
3. Why do we write `rectangle.area` rather than `rectangle.area()`?
    

---

## Exercise 2

Complete this:

```kotlin
class User {
    var username: String = ""
        set(value) {
            // store a trimmed version
        }
}
```

You should be able to figure out what belongs inside the setter.

---

## Exercise 3

Write a class:

```kotlin
class Counter
```

with a property:

```kotlin
value
```

Rules:

- outside code can read `value`
    
- outside code cannot directly change `value`
    
- the class has `increment()`
    
- the class has `decrement()`
    

The important keyword you'll need is:

```kotlin
private set
```

---

## Exercise 4 — prediction

Without running it, what does this print?

```kotlin
class Example {

    val a = println("A")

    init {
        println("B")
    }

    val c = println("C")
}
```

---

## Exercise 5 — design thinking

Which would you model as a property and which as a function?

```text
A person's name
A rectangle's area
A bank account's balance
Sending an email
Withdrawing money
A person's age
```

There isn't necessarily one mechanically correct answer for every case. I want you to explain your reasoning.

---

## One challenge

Build this:

```kotlin
class Student(
    val name: String,
    initialAge: Int
)
```

Requirements:

- `age` can change
    
- `age` cannot become negative
    
- `isAdult` should automatically reflect the current age
    
- the constructor should reject a negative `initialAge`
    
- outside code can see the student's grade, but only the class can change it
    

Try to use:

```text
custom setter
computed property
private set
init
```

Don't worry about making it perfect. **This is where you should start writing code rather than just reading.**

When you send me your solution, I'll go through it line by line and show you exactly what Kotlin is doing.