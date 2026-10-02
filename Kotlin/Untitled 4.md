
# Lesson 23 — Enums, Nested Classes, and Inner Classes

We're continuing the object-model part of Kotlin, but we're going to stay focused.

Today has two main ideas:

1. **Enums** — representing a fixed set of named possibilities.
    
2. **Nested vs inner classes** — putting one class inside another, and understanding whether the inner type has access to an outer object.
    

These concepts look small syntactically, but they solve very different modeling problems.

---

# Part 1 — Enums

Let's start with a very common problem.

Suppose a program has four directions:

```text
NORTH
SOUTH
EAST
WEST
```

We could represent them as strings:

```kotlin
val direction = "NORTH"
```

But that allows nonsense:

```kotlin
val direction = "BANANA"
```

The compiler has no reason to know that's invalid.

We could use integers:

```text
0 = NORTH
1 = SOUTH
2 = EAST
3 = WEST
```

but now the meaning is hidden behind arbitrary numbers.

Kotlin gives us a much better representation:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

Now:

```kotlin
val direction = Direction.NORTH
```

The variable isn't just "some value."

It's specifically a `Direction`.

---

# 1. What an enum actually represents

Think of an enum as:

> **a type whose allowed named values are explicitly defined.**

For:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

the type is:

```text
Direction
```

and its defined values are:

```text
Direction.NORTH
Direction.SOUTH
Direction.EAST
Direction.WEST
```

So:

```kotlin
val direction: Direction = Direction.NORTH
```

is valid.

But:

```kotlin
val direction: Direction = "NORTH"
```

isn't.

That's the advantage.

The type itself communicates the valid possibilities.

---

# 2. Why enums are better than strings

Imagine this:

```kotlin
fun move(direction: String) {
    // ...
}
```

The caller could pass:

```kotlin
move("NORTH")
move("north")
move("UP")
move("banana")
```

Now the function has to defend itself against invalid values.

Instead:

```kotlin
fun move(direction: Direction) {
    // ...
}
```

The compiler now prevents callers from passing arbitrary strings.

You can only pass a `Direction`.

That is one of the most important lessons of Kotlin's type system:

> **A good type can prevent invalid states before your code even runs.**

---

# 3. Enums work beautifully with `when`

Remember our earlier `when`:

```kotlin
val direction = Direction.NORTH

when (direction) {
    Direction.NORTH -> println("Up")
    Direction.SOUTH -> println("Down")
    Direction.EAST -> println("Right")
    Direction.WEST -> println("Left")
}
```

This is especially nice because the compiler understands the possible enum values.

You've already learned that `when` can be exhaustive.

Enums give it a very natural closed set of possibilities:

```text
Direction
├── NORTH
├── SOUTH
├── EAST
└── WEST
```

So the type system and `when` work together.

---

# 4. You don't necessarily need `else`

For a complete enum:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

you can write:

```kotlin
fun describe(direction: Direction): String {
    return when (direction) {
        Direction.NORTH -> "Up"
        Direction.SOUTH -> "Down"
        Direction.EAST -> "Right"
        Direction.WEST -> "Left"
    }
}
```

There is no `else`.

Why?

Because you've handled every defined `Direction`.

That's useful because when the enum changes later:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST,
    NORTHEAST
}
```

your exhaustive `when` becomes incomplete, alerting you to update the relevant logic.

That's much safer than scattering string comparisons throughout a program.

---

# 5. Enum values are objects

This is a subtle but important point.

Consider:

```kotlin
Direction.NORTH
```

It's not just a text label.

It's an actual enum instance.

You can think of:

```text
Direction.NORTH
Direction.SOUTH
Direction.EAST
Direction.WEST
```

as predefined objects belonging to the `Direction` type.

That's why you can use properties and methods on enum entries.

---

# 6. Enums can have properties

An enum doesn't have to be nothing more than names.

For example:

```kotlin
enum class Direction(
    val dx: Int,
    val dy: Int
) {
    NORTH(0, 1),
    SOUTH(0, -1),
    EAST(1, 0),
    WEST(-1, 0)
}
```

Now:

```kotlin
println(Direction.NORTH.dx)
println(Direction.NORTH.dy)
```

gives:

```text
0
1
```

We've made the enum entries carry data.

So an enum can model:

```text
named possibility + associated data
```

---

# 7. Enums can have functions

For example:

```kotlin
enum class Direction(
    val dx: Int,
    val dy: Int
) {
    NORTH(0, 1),
    SOUTH(0, -1),
    EAST(1, 0),
    WEST(-1, 0);

    fun isVertical(): Boolean {
        return this == NORTH || this == SOUTH
    }
}
```

Now:

```kotlin
println(Direction.NORTH.isVertical())
```

works.

Notice the semicolon:

```kotlin
WEST(-1, 0);
```

Why?

Because after the enum entries, we're starting the enum class body with members such as functions.

It separates the entries from the rest of the class declaration.

---

# 8. `this` inside an enum

Inside:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST;

    fun isVertical(): Boolean {
        return this == NORTH || this == SOUTH
    }
}
```

`this` refers to the particular enum instance whose function is being executed.

So if you call:

```kotlin
Direction.NORTH.isVertical()
```

then:

```text
this
↓
Direction.NORTH
```

If you call:

```kotlin
Direction.EAST.isVertical()
```

then:

```text
this
↓
Direction.EAST
```

Same idea as `this` inside ordinary classes.

---

# 9. Each enum entry can have its own behavior

Here's where enums get more interesting.

```kotlin
enum class Operation {
    ADD {
        override fun apply(a: Int, b: Int): Int {
            return a + b
        }
    },

    SUBTRACT {
        override fun apply(a: Int, b: Int): Int {
            return a - b
        }
    };

    abstract fun apply(a: Int, b: Int): Int
}
```

Now:

```kotlin
println(Operation.ADD.apply(10, 3))
println(Operation.SUBTRACT.apply(10, 3))
```

produces:

```text
13
7
```

Each enum entry supplies its own implementation.

This is already getting into more advanced object-oriented modeling.

You don't need to use this every day, but you should understand that an enum can contain real behavior.

---

# 10. Enums vs `if`/`when`

Suppose you're modeling a traffic light.

You could do:

```kotlin
val color = "RED"
```

and then:

```kotlin
if (color == "RED") {
    ...
}
```

But you're encoding a domain concept as arbitrary text.

Instead:

```kotlin
enum class TrafficLight {
    RED,
    YELLOW,
    GREEN
}
```

Now:

```kotlin
fun action(light: TrafficLight) {
    when (light) {
        TrafficLight.RED -> println("Stop")
        TrafficLight.YELLOW -> println("Prepare")
        TrafficLight.GREEN -> println("Go")
    }
}
```

The type itself tells us:

> These are the possible traffic-light states.

---

# 11. When should you use an enum?

A useful question is:

> **Is the set of possibilities fixed and known?**

Examples:

```text
Days of week
Directions
Traffic light states
Order statuses
Card suits
Difficulty levels
```

These are often good candidates.

For example:

```kotlin
enum class Difficulty {
    EASY,
    MEDIUM,
    HARD
}
```

The set is conceptually closed.

---

# 12. When an enum isn't enough

Now consider a payment result.

There might be:

```text
successful payment
failed payment
payment requiring authentication
payment cancelled
```

But perhaps success carries:

```text
transaction ID
amount
```

Failure carries:

```text
error message
error code
```

An enum alone can represent the categories:

```kotlin
enum class PaymentResult {
    SUCCESS,
    FAILURE,
    REQUIRES_AUTHENTICATION,
    CANCELLED
}
```

But it can't naturally represent different data attached to each case.

That's where our future topic becomes important:

**sealed classes/interfaces.**

So remember:

```text
enum
→ fixed named possibilities, often sharing a common shape

sealed hierarchy
→ fixed possibilities that can have different structures/data
```

We'll study sealed types in Phase 7.

---

# Part 2 — Nested classes

Now let's switch topics.

Suppose we have:

```kotlin
class Computer
```

and inside it we want a helper type:

```text
Computer
└── Processor
```

Kotlin allows:

```kotlin
class Computer {

    class Processor
}
```

`Processor` is a **nested class**.

---

# 13. A nested class is not automatically tied to an outer object

This is very important.

Consider:

```kotlin
class Computer {
    class Processor
}
```

You can create:

```kotlin
val processor = Computer.Processor()
```

Notice that you do **not** need:

```kotlin
val computer = Computer()
```

first.

Why?

Because `Processor` is nested for organizational purposes, but it does not automatically belong to a specific `Computer` instance.

Think:

```text
Computer
└── Processor type
```

rather than:

```text
Computer object
└── Processor object
```

Those are different ideas.

---

# 14. Why nest a class?

One reason is organization.

Imagine:

```kotlin
class Network {

    class Request
    class Response
}
```

Now the names communicate that these types belong conceptually to the networking domain.

You can write:

```kotlin
val request = Network.Request()
val response = Network.Response()
```

instead of putting generic names like:

```text
Request
Response
```

into a huge global namespace.

It can make APIs easier to understand.

---

# 15. Nested class accessing outer members

Here's a common surprise:

```kotlin
class Computer(
    val brand: String
) {

    class Processor {
        fun printBrand() {
            // Can I access brand?
        }
    }
}
```

No.

Why?

Because `Processor` isn't attached to a specific `Computer` object.

Which computer's brand would it use?

There could be:

```text
Computer("Dell")
Computer("Apple")
Computer("Lenovo")
```

The nested `Processor` doesn't know which one you're talking about.

That's exactly what "nested but not inner" means.

---

# 16. `inner` changes the relationship

Now write:

```kotlin
class Computer(
    val brand: String
) {

    inner class Processor {
        fun printBrand() {
            println(brand)
        }
    }
}
```

Now `Processor` is an **inner class**.

An inner class is associated with an instance of the outer class.

So you must first create a `Computer`:

```kotlin
val computer = Computer("Lenovo")
```

Then:

```kotlin
val processor = computer.Processor()
```

Notice the difference.

For a nested class:

```kotlin
Computer.Processor()
```

For an inner class:

```kotlin
computer.Processor()
```

That little difference reflects a major conceptual difference.

---

# 17. Nested vs inner

Here's the most important comparison:

||Nested class|Inner class|
|---|---|---|
|Syntax|`class Processor`|`inner class Processor`|
|Requires outer object?|No|Yes|
|Can directly access outer instance members?|No|Yes|
|Relationship|Organizational|Tied to an outer instance|

So:

```text
nested
→ grouped inside another class

inner
→ grouped inside + connected to a particular outer object
```

That's the mental model.

---

# 18. Why does `inner` have access to the outer object?

Because conceptually it carries an association with its outer instance.

For example:

```kotlin
class Person(
    val name: String
) {

    inner class Greeting {
        fun sayHello() {
            println("Hello, $name")
        }
    }
}
```

Now:

```kotlin
val person = Person("Alex")
val greeting = person.Greeting()

greeting.sayHello()
```

prints:

```text
Hello, Alex
```

Which `name` is being used?

The name of the particular `Person` that created the `Greeting`.

That is what makes the class `inner`.

---

# 19. This also means the distinction is about state

Compare:

```kotlin
class Person(
    val name: String
) {

    class IDGenerator
}
```

`IDGenerator` doesn't need a particular person's state.

So nesting is enough.

But:

```kotlin
class Person(
    val name: String
) {

    inner class Greeting {
        fun text() = "Hello, $name"
    }
}
```

`Greeting` needs the state of a particular `Person`.

So it needs to be `inner`.

A very useful question is:

> **Does the nested type need access to one specific outer object?**

If no → ordinary nested class may be enough.

If yes → `inner` may be appropriate.

---

# 20. Be careful with `inner`

An inner class creates a stronger relationship with the outer instance.

For example:

```kotlin
class LargeObject {
    inner class Helper
}
```

A `Helper` is associated with a `LargeObject`.

That relationship can be useful, but it can also make object lifecycles more complicated.

So don't add `inner` just because the class is visually located inside another class.

Use it because the inner object actually needs the outer instance.

---

# 21. A useful real-world example

Imagine a `Library` and a `Book`.

Could we write:

```kotlin
class Library {

    class Book
}
```

or:

```kotlin
class Library {

    inner class Book
}
```

Which one makes more sense depends on the domain.

If `Book` is simply a type conceptually grouped under `Library`, but doesn't need a particular library instance:

```kotlin
Library.Book
```

could make sense.

But if a `Book` needs to directly access the state of one specific `Library`:

```kotlin
library.someLibraryState
```

then `inner` creates that relationship.

Again, the keyword follows the design.

---

# 22. One subtle point: nested classes are not like inheritance

This is worth clearing up.

If we write:

```kotlin
class Computer {
    class Processor
}
```

`Processor` is **not** a subtype of `Computer`.

This does not mean:

```text
Processor IS-A Computer
```

It simply means:

```text
Processor is declared inside Computer
```

That's organization, not inheritance.

Just like:

```kotlin
class School {
    class Student
}
```

doesn't mean a `Student` is a kind of `School`.

---

# 23. Let's combine enums with classes

Now we can build a more realistic model.

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}

class Robot(
    var x: Int = 0,
    var y: Int = 0
) {

    fun move(direction: Direction) {
        when (direction) {
            Direction.NORTH -> y++
            Direction.SOUTH -> y--
            Direction.EAST -> x++
            Direction.WEST -> x--
        }
    }
}
```

Now:

```kotlin
val robot = Robot()

robot.move(Direction.NORTH)
robot.move(Direction.EAST)

println(robot.x)
println(robot.y)
```

This is a nice example of type-driven design:

```text
Direction
→ defines valid directions

Robot
→ owns position

move()
→ changes position according to the direction
```

We're not passing random strings like `"north"`.

The type system helps express the domain.

---

# 24. A slightly richer enum

We can make `Direction` carry its movement:

```kotlin
enum class Direction(
    val dx: Int,
    val dy: Int
) {
    NORTH(0, 1),
    SOUTH(0, -1),
    EAST(1, 0),
    WEST(-1, 0)
}
```

Then the robot becomes:

```kotlin
class Robot(
    var x: Int = 0,
    var y: Int = 0
) {
    fun move(direction: Direction) {
        x += direction.dx
        y += direction.dy
    }
}
```

Look at what's happened.

The `Robot` no longer needs to know:

```text
NORTH → y + 1
SOUTH → y - 1
...
```

The enum carries the data describing each direction.

That's a small but valuable design improvement.

---

# 25. Why this matters for bigger programs

You're beginning to see something important.

Programming isn't just:

> "What syntax do I write?"

It's increasingly:

> "Where should this information live?"

For example:

### Should `Robot` know the numeric movement offsets?

Maybe not.

### Should `Direction` know how it maps to movement?

Possibly.

That is why types, properties, functions, enums, interfaces, and classes all matter together.

We're learning **how to distribute responsibility across a program**.

---

# 26. A first look at enum utility functions

Enums provide some useful built-in functionality.

For example, conceptually you can inspect all enum entries.

You may encounter:

```kotlin
Direction.entries
```

and also older code using:

```kotlin
Direction.values()
```

For modern Kotlin code, `entries` is the newer API you'll increasingly see.

For now, don't worry about memorizing enum APIs.

The important idea is:

> You can work with the complete set of enum entries as data.

This will become useful for menus, validation, UI choices, parsing, and testing.

---

# 27. One more useful distinction: enum equality

Since enum entries represent fixed singleton-like values, comparing them is straightforward:

```kotlin
if (direction == Direction.NORTH) {
    println("Going north")
}
```

This is exactly the kind of use where `when` becomes natural:

```kotlin
when (direction) {
    Direction.NORTH -> ...
    Direction.SOUTH -> ...
    Direction.EAST -> ...
    Direction.WEST -> ...
}
```

Enums and exhaustive `when` are a particularly powerful combination.

---

# 28. Now think about the modeling hierarchy we've built

At the moment, you can represent:

### Arbitrary values

```kotlin
val age: Int
```

### Collections of values

```kotlin
List<Int>
```

### A fixed set of named possibilities

```kotlin
enum class Direction
```

### Objects with state and behavior

```kotlin
class Robot
```

### Contracts

```kotlin
interface Movable
```

### Different object implementations

```kotlin
open class Animal
```

And soon we'll add:

### A fixed set of different structured possibilities

```text
sealed class ...
```

### Strong domain-specific types

```text
value class ...
```

### Behavior reused through delegation

```text
by ...
```

That's the path from basic Kotlin into real type-oriented design.

---

# Practice

Take a moment to answer these without running code.

### Question 1

What does this represent?

```kotlin
enum class Status {
    PENDING,
    SUCCESS,
    FAILED
}
```

Why might this be preferable to:

```kotlin
val status: String
```

?

---

### Question 2

Why is this valid?

```kotlin
val direction = Direction.NORTH

when (direction) {
    Direction.NORTH -> println("N")
    Direction.SOUTH -> println("S")
    Direction.EAST -> println("E")
    Direction.WEST -> println("W")
}
```

What does the compiler know about `Direction` that it wouldn't know about arbitrary strings?

---

### Question 3

What's the difference?

```kotlin
class Outer {
    class Nested
}
```

versus:

```kotlin
class Outer {
    inner class Inner
}
```

Focus on the **relationship to an outer object**, not just the keyword.

---

### Question 4

Why does this work?

```kotlin
class Person(
    val name: String
) {
    inner class Greeting {
        fun text() = "Hello, $name"
    }
}
```

But if `Greeting` were just:

```kotlin
class Greeting
```

it wouldn't automatically have access to `name`.

---

# Coding exercise

Build an enum:

```kotlin
enum class Day
```

with all seven days.

Then write:

```kotlin
fun isWeekend(day: Day): Boolean
```

using `when`.

Your function should return `true` for Saturday and Sunday and `false` for the others.

Then build:

```kotlin
class Calendar
```

with a nested class:

```kotlin
class Event
```

Don't make `Event` `inner` yet.

Then ask yourself:

> Does an event actually need access to a particular `Calendar` instance?

That's the design question I care about more than the syntax.

---

# Challenge

Create a small model for a media player.

You need:

```text
PlaybackState:
    STOPPED
    PLAYING
    PAUSED
```

Then:

```kotlin
class MusicPlayer
```

with a property representing its current state.

Give it functions:

```text
play()
pause()
stop()
```

and use `when` somewhere to describe the current state.

Then think:

> Is `PlaybackState` better represented as a string, an enum, or something more powerful?

Don't worry if you aren't certain. That question is deliberately preparing you for the next phase.

---

## The important ideas from today

Keep these distinctions clear:

```text
enum
→ fixed set of named values

nested class
→ class declared inside another class for organization

inner class
→ nested class that is tied to a specific outer instance
```

And one particularly important connection:

```text
enum
    ↓
fixed possibilities
    ↓
when
    ↓
exhaustive handling
```

That combination is one of Kotlin's strengths.

Next, we'll make the jump into **Phase 7 proper: sealed classes and sealed interfaces**. That's where you'll see why an enum isn't always enough when different possibilities need to carry different kinds of data.